**Tarefa.** Aplicando o framework **R-T-F**, escrever o prompt de IA que produza esse script bash.
**Entregue.** Prompt, modelo, output e justificativa mostrando como Role, Task e Format aparecem no prompt.

### 1. Prompt

```

[ROLE]
Você é um especialista em AWS com conhecimentos avançados de Linux e PostgreSQL.

[TASK]
Criar uma rotina de backup do banco de dados PostgreSQL do Ledger. O script precisa fazer o dump com `pg_dump`, compactar com `gzip`, subir o arquivo para o bucket S3 `hvt-ledger-backups` via `aws s3 cp`, manter 30 dias de retenção no S3 (removendo os mais antigos), registrar cada execução em `/var/log/ledger-backup.log` com timestamp, e sair com exit code adequado em caso de falha.
Contexto:
- Host: `ledger-db.internal.hvt.io`
- Porta: `5432`
- Banco: `ledger_prod`
- Usuário de backup: `backup_user`
- Senha: variável de ambiente `PGPASSWORD`, populada pelo AWS Secrets Manager via IAM role da instância
- Região AWS: `us-east-1`
- SO da instância: Ubuntu 22.04 LTS
- Diretório de trabalho com 80 GB livres: `/var/backups/ledger`
- Tamanho médio atual do dump compactado: ~12 GB
Requisitos:
- Seguir boas práticas para criação de rotinas de backup em Linux na AWS
- O script deve ser executado diariamente de forma automática
- Senhas não podem ficar expostas

[FORMAT]
Responda com:
1. Script completo, com comentários curtos em cada instrução
2. Uma tabela "Boa prática | Onde foi aplicada | Por quê"
Sem introdução ou texto fora desses dois itens.


```

### 2. Modelo

```
Opus 5.5
```


### 3. Output

##### 3.1 Script


```bash

#!/usr/bin/env bash
# /usr/local/sbin/ledger-backup.sh: backup diário do ledger_prod para o S3
# Executar como usuário dedicado (ex.: "ledger-backup"), nunca como postgres/root interativo.

set -Eeuo pipefail                       # aborta em erro, variável indefinida ou falha em pipe
umask 077                                # todo arquivo criado fica legível só pelo dono

# ---------------- Configuração ----------------
readonly DB_HOST="ledger-db.internal.hvt.io"   # host do PostgreSQL
readonly DB_PORT="5432"                        # porta do PostgreSQL
readonly DB_NAME="ledger_prod"                 # banco alvo
readonly DB_USER="backup_user"                 # usuário com privilégio mínimo de leitura
readonly SECRET_ID="${LEDGER_BACKUP_SECRET_ID:-ledger/backup_user}"  # fallback se PGPASSWORD não vier no ambiente
readonly S3_BUCKET="hvt-ledger-backups"        # bucket de destino
readonly S3_PREFIX="postgres/${DB_NAME}"       # prefixo isolado (retenção só atua aqui)
readonly AWS_REGION="us-east-1"                # região do bucket
readonly WORK_DIR="/var/backups/ledger"        # área temporária local
readonly LOG_FILE="/var/log/ledger-backup.log" # log de execuções
readonly LOCK_FILE="/run/lock/ledger-backup.lock"  # trava contra execução concorrente
readonly RETENTION_DAYS=30                     # retenção no S3
readonly MIN_KEEP=7                            # nunca deixa menos que N backups no S3
readonly MIN_FREE_GB=30                        # ~2,5x o dump atual (12 GB) + folga
readonly DUMP_TIMEOUT="6h"                     # evita dump travado para sempre
readonly UPLOAD_ATTEMPTS=3                     # tentativas de upload

# Códigos de saída (facilitam alarme no CloudWatch/monitoramento)
readonly E_DEPS=2 E_LOCK=3 E_SECRET=4 E_DISK=5 E_CONN=6 E_DUMP=7 E_VALIDATE=8 E_UPLOAD=9 E_RETENTION=10

export AWS_DEFAULT_REGION="$AWS_REGION"        # região fixa para o CLI
export AWS_RETRY_MODE="standard"               # retry com backoff nativo do CLI
export AWS_MAX_ATTEMPTS=5                      # tentativas internas por chamada
export PGSSLMODE="${PGSSLMODE:-require}"       # tráfego do dump criptografado (ideal: verify-full + CA)
export PGCONNECT_TIMEOUT=30                    # não fica pendurado se o banco não responder

readonly TS="$(date -u +%Y%m%dT%H%M%SZ)"                 # timestamp UTC (ordenável)
readonly FILE_NAME="${DB_NAME}_${TS}.dump.gz"            # nome do artefato
readonly LOCAL_FILE="${WORK_DIR}/${FILE_NAME}"           # caminho local final
readonly PARTIAL_FILE="${LOCAL_FILE}.partial"            # caminho durante a escrita
readonly SHA_FILE="${LOCAL_FILE}.sha256"                 # checksum do artefato
readonly S3_KEY="${S3_PREFIX}/${FILE_NAME}"              # chave do objeto
PGPASS_TMP=""                                            # arquivo de senha temporário

# ---------------- Funções utilitárias ----------------
log() {                                                  # grava linha com timestamp ISO-8601 UTC
  printf '%s [%s] [pid %s] %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$1" "$$" "$2" | tee -a "$LOG_FILE" >&2
}

die() {                                                  # loga erro e sai com código específico
  log ERROR "$2"
  exit "$1"
}

on_err() {                                               # captura falhas não tratadas explicitamente
  log ERROR "falha inesperada na linha $1 (comando: $2)"
}

cleanup() {                                              # sempre executa ao sair
  local rc=$?
  [[ -n "$PGPASS_TMP" ]] && shred -u "$PGPASS_TMP" 2>/dev/null || rm -f "$PGPASS_TMP"  # destrói a senha em disco
  rm -f "$PARTIAL_FILE" "$LOCAL_FILE" "$SHA_FILE"        # remove artefatos locais (o S3 é a cópia oficial)
  log INFO "fim da execução (exit=${rc}, duração=${SECONDS}s)"
  exit "$rc"
}

trap 'on_err $LINENO "$BASH_COMMAND"' ERR                # log de erro com linha
trap cleanup EXIT                                        # limpeza garantida
trap 'die 130 "interrompido por sinal"' INT TERM         # trata kill/Ctrl+C

# ---------------- Pré-checagens ----------------
touch "$LOG_FILE" 2>/dev/null || { echo "sem permissão em $LOG_FILE" >&2; exit "$E_DEPS"; }  # log precisa existir
log INFO "início do backup de ${DB_NAME}@${DB_HOST} -> s3://${S3_BUCKET}/${S3_KEY}"

for cmd in pg_dump pg_restore pg_isready gzip aws jq flock sha256sum timeout ionice shred; do
  command -v "$cmd" >/dev/null || die "$E_DEPS" "dependência ausente: $cmd"   # falha cedo se faltar binário
done

exec 9>"$LOCK_FILE" || die "$E_LOCK" "não foi possível abrir $LOCK_FILE"     # descritor da trava
flock -n 9 || die "$E_LOCK" "outra execução já está em andamento"            # impede sobreposição

install -d -m 700 "$WORK_DIR"                                                # garante diretório privado
free_gb="$(df --output=avail -BG "$WORK_DIR" | tail -1 | tr -dc '0-9')"      # espaço livre em GB
(( free_gb >= MIN_FREE_GB )) || die "$E_DISK" "espaço insuficiente: ${free_gb}G livres (mínimo ${MIN_FREE_GB}G)"
log INFO "espaço livre em ${WORK_DIR}: ${free_gb}G"

# ---------------- Credencial (sem expor senha) ----------------
load_password() {
  local pw="${PGPASSWORD:-}"                                                 # usa o valor já injetado, se existir
  if [[ -z "$pw" ]]; then                                                    # senão, busca no Secrets Manager via IAM role
    pw="$(aws secretsmanager get-secret-value --secret-id "$SECRET_ID" \
          --query SecretString --output text)" || return 1
    [[ "$pw" == \{* ]] && pw="$(jq -r '.password // empty' <<< "$pw")"       # suporta segredo em JSON
  fi
  [[ -n "$pw" ]] || return 1                                                 # senha vazia = falha
  pw="${pw//\\/\\\\}"; pw="${pw//:/\\:}"                                     # escapa caracteres especiais do .pgpass
  PGPASS_TMP="$(mktemp "${WORK_DIR}/.pgpass.XXXXXX")"                        # arquivo 0600 (umask 077)
  printf '%s:%s:%s:%s:%s\n' "$DB_HOST" "$DB_PORT" "$DB_NAME" "$DB_USER" "$pw" > "$PGPASS_TMP"
  export PGPASSFILE="$PGPASS_TMP"                                            # libpq lê a senha do arquivo
  unset PGPASSWORD pw                                                        # tira a senha do ambiente dos filhos
}
load_password || die "$E_SECRET" "não foi possível obter a credencial do banco"

pg_isready -h "$DB_HOST" -p "$DB_PORT" -d "$DB_NAME" -U "$DB_USER" -q \
  || die "$E_CONN" "banco indisponível em ${DB_HOST}:${DB_PORT}"             # testa conectividade antes do dump

# ---------------- Dump + compactação ----------------
log INFO "iniciando pg_dump (formato custom, compactação via gzip)"
set +e                                                                       # captura status de cada etapa do pipe
nice -n 10 ionice -c2 -n7 timeout "$DUMP_TIMEOUT" \
  pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" \
          --format=custom -Z 0 --no-password --lock-wait-timeout=120s \
          2>>"$LOG_FILE" \
  | gzip -6 > "$PARTIAL_FILE"                                                # compacta em streaming (sem arquivo bruto)
dump_rc=("${PIPESTATUS[@]}")                                                 # [0]=pg_dump [1]=gzip
set -e
(( dump_rc[0] == 0 )) || die "$E_DUMP" "pg_dump falhou (rc=${dump_rc[0]}; 124 = timeout)"
(( dump_rc[1] == 0 )) || die "$E_DUMP" "gzip falhou (rc=${dump_rc[1]})"
mv "$PARTIAL_FILE" "$LOCAL_FILE"                                             # renomeia só quando completo
log INFO "dump concluído: $(du -h "$LOCAL_FILE" | cut -f1)"

# ---------------- Validação ----------------
gzip -t "$LOCAL_FILE" || die "$E_VALIDATE" "arquivo gzip corrompido"         # integridade do gzip
set +e
gunzip -c "$LOCAL_FILE" | pg_restore --list > /dev/null 2>>"$LOG_FILE"       # confirma que o dump é restaurável
list_rc=("${PIPESTATUS[@]}")                                                 # gunzip pode receber SIGPIPE; vale o rc do pg_restore
set -e
(( list_rc[1] == 0 )) || die "$E_VALIDATE" "pg_restore não reconheceu o arquivo"
sha256sum "$LOCAL_FILE" | awk '{print $1}' > "$SHA_FILE"                     # checksum para auditoria/restore
sha="$(cat "$SHA_FILE")"
log INFO "validação OK (sha256=${sha})"

# ---------------- Upload para o S3 ----------------
upload_ok=0
for (( i = 1; i <= UPLOAD_ATTEMPTS; i++ )); do                               # retry com backoff exponencial
  if aws s3 cp "$LOCAL_FILE" "s3://${S3_BUCKET}/${S3_KEY}" \
       --sse aws:kms --storage-class STANDARD_IA --checksum-algorithm SHA256 \
       --metadata "sha256=${sha},db=${DB_NAME},source=$(hostname -s)" \
       --only-show-errors 2>>"$LOG_FILE" \
     && aws s3 cp "$SHA_FILE" "s3://${S3_BUCKET}/${S3_KEY}.sha256" \
       --sse aws:kms --only-show-errors 2>>"$LOG_FILE"; then
    upload_ok=1; break
  fi
  log WARN "upload falhou (tentativa ${i}/${UPLOAD_ATTEMPTS})"
  sleep $(( 30 * 2 ** (i - 1) ))                                             # 30s, 60s, 120s
done
(( upload_ok == 1 )) || die "$E_UPLOAD" "upload falhou após ${UPLOAD_ATTEMPTS} tentativas"

remote_size="$(aws s3api head-object --bucket "$S3_BUCKET" --key "$S3_KEY" \
               --query ContentLength --output text)" || die "$E_UPLOAD" "objeto não encontrado após upload"
local_size="$(stat -c%s "$LOCAL_FILE")"
[[ "$remote_size" == "$local_size" ]] \
  || die "$E_UPLOAD" "tamanho divergente (local=${local_size}, s3=${remote_size})"   # confere o que subiu
log INFO "upload verificado: s3://${S3_BUCKET}/${S3_KEY} (${local_size} bytes)"

# ---------------- Retenção (só após backup novo confirmado) ----------------
apply_retention() {
  local cutoff listing total deleted=0 ts key
  cutoff="$(date -u -d "${RETENTION_DAYS} days ago" +%Y-%m-%dT%H:%M:%S)"     # limite de corte em UTC
  listing="$(aws s3api list-objects-v2 --bucket "$S3_BUCKET" --prefix "${S3_PREFIX}/" \
             --query 'Contents[?ends_with(Key, `.dump.gz`)].[LastModified, Key]' \
             --output text)" || return 1                                     # lista só os dumps do prefixo
  [[ -z "$listing" || "$listing" == "None" ]] && { log WARN "nenhum backup listado no S3"; return 0; }
  total="$(wc -l <<< "$listing")"                                            # quantidade atual de backups
  while read -r ts key; do                                                   # do mais antigo para o mais novo
    [[ "${ts:0:19}" < "$cutoff" ]] || break                                  # parou de achar antigos
    (( total - deleted > MIN_KEEP )) || { log WARN "mínimo de ${MIN_KEEP} backups atingido; retenção interrompida"; break; }
    aws s3 rm "s3://${S3_BUCKET}/${key}" --only-show-errors 2>>"$LOG_FILE" || return 1        # remove o dump
    aws s3 rm "s3://${S3_BUCKET}/${key}.sha256" --only-show-errors 2>>"$LOG_FILE" \
      || log WARN "checksum não removido: ${key}.sha256"                                       # remove o checksum
    log INFO "removido por retenção: ${key} (${ts})"
    (( ++deleted ))
  done < <(sort <<< "$listing")
  log INFO "retenção aplicada: ${deleted} removido(s), $(( total - deleted )) mantido(s)"
}
apply_retention || die "$E_RETENTION" "backup OK, mas a limpeza de retenção falhou"

log INFO "backup concluído com sucesso"
exit 0

# ---------------- Implantação (referência) ----------------
# Permissões:  install -o ledger-backup -g ledger-backup -m 750 ledger-backup.sh /usr/local/sbin/
# Log:         /etc/logrotate.d/ledger-backup -> weekly, rotate 12, compress, create 0640 ledger-backup adm
# Agendamento: systemd timer (OnCalendar=*-*-* 03:00:00 UTC, Persistent=true) ou
#              cron: 0 3 * * * ledger-backup /usr/local/sbin/ledger-backup.sh
# IAM (role da instância), mínimo:
#   s3:PutObject, s3:GetObject (HeadObject) e s3:DeleteObject em arn:aws:s3:::hvt-ledger-backups/postgres/ledger_prod/*
#   s3:ListBucket em arn:aws:s3:::hvt-ledger-backups (condição s3:prefix = postgres/ledger_prod/*)
#   secretsmanager:GetSecretValue no ARN do segredo; kms:GenerateDataKey/Decrypt na chave usada
# Cliente: pg_dump/pg_restore da mesma versão major do servidor (ou superior), via repositório PGDG.

```


#### 3.2 Boas práticas aplicadas

|Boa prática|Onde foi aplicada|Por quê|
|---|---|---|
|Modo estrito do Bash|`set -Eeuo pipefail`|Impede que uma falha silenciosa (inclusive no meio de um pipe) gere um backup “de sucesso” vazio ou truncado|
|Senha fora de argumentos e do ambiente|`load_password` com `.pgpass` temporário 0600, `PGPASSFILE`, `unset PGPASSWORD`, `shred` no `cleanup`|A senha não aparece em `ps`, histórico, log nem em `/proc/<pid>/environ` do `pg_dump`|
|Credencial via IAM Role + Secrets Manager|`aws secretsmanager get-secret-value` (fallback)|Elimina chaves fixas na instância e permite rotacionar a senha sem mexer no script|
|Princípio do menor privilégio|Usuário dedicado, `backup_user`, política IAM restrita ao prefixo|Limita o impacto se a instância ou o usuário forem comprometidos|
|Arquivos privados|`umask 077`, `install -d -m 700`|O dump contém dados financeiros e não pode ficar legível para outros usuários do SO|
|Trava contra concorrência|`flock -n` em `/run/lock/ledger-backup.lock`|Evita dois dumps simultâneos disputando disco, banco e S3|
|Verificações prévias (fail fast)|Checagem de binários, espaço em disco e `pg_isready`|Falha em segundos com causa clara, em vez de quebrar horas depois|
|Dump em streaming|`pg_dump --format=custom -Z 0 \| gzip`|Não grava um arquivo bruto intermediário (economiza disco) e mantém o formato restaurável por `pg_restore`|
|Tratamento do status do pipe|`PIPESTATUS` após o dump e a validação|Identifica se quem falhou foi o `pg_dump` ou o `gzip` e registra o código exato|
|Escrita atômica|`.partial` seguido de `mv`|Um arquivo com nome final nunca está incompleto|
|Baixo impacto no host|`nice`, `ionice`, `timeout`, `--lock-wait-timeout`|Reduz a concorrência por CPU e I/O e evita processos pendurados ou locks indefinidos|
|Criptografia em trânsito|`PGSSLMODE=require` (ideal: `verify-full`) e HTTPS do AWS CLI|Protege os dados entre o banco, a instância e o S3|
|Criptografia em repouso|`--sse aws:kms`|Protege o backup no S3 com chave auditável no KMS|
|Validação do backup|`gzip -t` + `pg_restore --list` + `sha256sum`|Um backup só tem valor se puder ser restaurado. O checksum permite conferir a integridade no restore|
|Upload resiliente e verificado|Retry com backoff, `AWS_RETRY_MODE`, `--checksum-algorithm`, `head-object` comparando tamanho|Contorna falhas transitórias de rede e confirma que o objeto completo chegou ao S3|
|Custo de armazenamento|`--storage-class STANDARD_IA`|Backups são raramente lidos, e essa classe custa menos que a STANDARD|
|Retenção segura|`apply_retention` roda só após upload confirmado, filtra o prefixo e respeita `MIN_KEEP`|Nunca apaga backups antigos se o novo falhou e nunca zera o histórico|
|Retenção complementar no S3|Recomendação: regra de Lifecycle (ex.: 35 dias) e versionamento/Object Lock|Funciona como rede de segurança se o script parar de rodar e protege contra exclusão maliciosa|
|Log estruturado e rotacionado|`log()` com ISO-8601 UTC, PID e nível, mais `logrotate`|Facilita a auditoria e a correlação com CloudWatch, sem encher o disco|
|Códigos de saída específicos|`E_DEPS` … `E_RETENTION`|O monitoramento (systemd, CloudWatch Agent, alarmes) distingue o tipo de falha|
|Limpeza garantida|`trap cleanup EXIT` e `trap … INT TERM`|Remove a senha temporária e os arquivos locais mesmo em caso de erro ou interrupção|
|Horário e nomes em UTC|`date -u` no nome do arquivo e no log|Ordenação lexicográfica correta e sem ambiguidade de fuso ou horário de verão|
|Compatibilidade de versão|Nota sobre `pg_dump` do PGDG|Um cliente com versão major inferior à do servidor faz o `pg_dump` falhar ou gerar dump incompatível|

### 4. Justificativa

As instruções foram colocadas de forma explícita no prompt, separando em Role, Task e Format, como diz o framework. O resultado pode ser visto na resposta do modelo.

- **Role** define o papel do modelo, sua área de atuação. Talvez uma das mais importantes, pois trás o conhecimento necessário para executar a tarefa. Na resposta do modelo, podemos ver que foram tomadas decisões com base no conhecimento de um especialista Linux e em PostgreSQL ao configurar o modo estrito do Bash que impede que uma falha silenciosa gere um backup “de sucesso” vazio ou truncado. A parte de retenção também foi bem aplicada, removendo o arquivo só depois de confirmado um backup com sucesso e, principalmente, a validação de que o dump é restaurável é uma boa prática de quem já sofreu com um backup corrompido.

- **Task** é a parte que demanda maior detalhamento. Nela foi definido o que se espera que seja feito e onde contextualizamos com as informações que já temos, como a infraestrutura, informações do banco de dados, dos servidores e etc. O requisito de senhas não expostas, por exemplo, foi atendido com um `.pgpass` temporário (permissão 0600), a remoção de `PGPASSWORD` do ambiente e a destruição do arquivo ao final da execução.

- **Format** é onde defini como queria o resultado. Para ser objetivo, pedi o script completo com comentários curtos em cada instrução e uma tabela de boas práticas, sem texto introdutório. A resposta seguiu essas instruções.