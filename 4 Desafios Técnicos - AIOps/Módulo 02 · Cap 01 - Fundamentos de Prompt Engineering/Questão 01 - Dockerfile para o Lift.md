**Tarefa.** Aplicando o framework **R-T-F**, escrever o prompt de IA que produza esse Dockerfile.
**Entregue.** Prompt, modelo, output e justificativa mostrando como Role, Task e Format aparecem no prompt.

### 1. Prompt

```

#ROLE
Você é um engenheiro DevOps sênior, especialista em containerização de
aplicações Python e deploy em Kubernetes.

#TASK
Crie o Dockerfile para a aplicação "Lift", que está migrando de VMs
para um cluster Kubernetes.
Contexto:
- API Python/Flask, porta 8080
- Estrutura: app.py, requirements.txt, lib/ (auth.py, storage.py), tests/
- Dependências: Flask==3.0.0, gunicorn==21.2.0, requests==2.31.0, python-dotenv==1.0.0, psycopg2-binary==2.9.9
- Comando de produção: gunicorn --bind 0.0.0.0:8080 --workers 4 app:app
- Variáveis obrigatórias no runtime: DATABASE_URL e API_KEY (sensíveis)
Requisitos:
- Seguir boas práticas de criação de imagens
- Segredos NÃO podem ficar na imagem; serão injetados pelo Kubernetes
- A pasta tests/ não deve ir para a imagem de produção

#FORMAT
Responda com:
1. O Dockerfile completo, com comentários curtos em cada instrução
2. O conteúdo do .dockerignore
3. Uma tabela "Boa prática | Onde foi aplicada | Por quê"
Sem introdução ou texto fora desses três itens.


```

### 2. Modelo

```
Opus 5.5
```


### 3. Output

##### 3.1 Dockerfile


```dockerfile
# syntax=docker/dockerfile:1

# ============================================================
# Estágio 1: builder (instala as dependências num venv isolado)
# ============================================================
# Imagem oficial slim com versão fixa (em produção, fixe também o digest @sha256:...)
FROM python:3.12-slim-bookworm AS builder

# Pip sem cache e sem checagem de versão: build mais rápido e camada menor
ENV PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Diretório de trabalho do build
WORKDIR /build

# Cria um virtualenv para copiar só as dependências para o estágio final
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copia só o requirements primeiro para aproveitar o cache de camadas
COPY requirements.txt .

# Instala as dependências fixadas (psycopg2-binary já traz a libpq, sem gcc)
RUN pip install --no-cache-dir -r requirements.txt

# ============================================================
# Estágio 2: runtime (imagem final, enxuta e sem ferramentas de build)
# ============================================================
FROM python:3.12-slim-bookworm AS runtime

# Metadados OCI para rastreabilidade no registry
LABEL org.opencontainers.image.title="lift-api" \
      org.opencontainers.image.description="API Lift (Flask + Gunicorn)"

# Sem .pyc em disco, logs sem buffer (stdout/stderr vão direto para o kubectl logs)
# Nenhum segredo aqui: DATABASE_URL e API_KEY chegam via Secret do Kubernetes
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

# Usuário e grupo sem privilégios, com UID/GID numéricos (compatível com runAsNonRoot)
RUN groupadd --system --gid 10001 lift \
 && useradd --system --uid 10001 --gid lift --no-create-home --shell /usr/sbin/nologin lift

# Diretório da aplicação
WORKDIR /app

# Traz apenas o venv pronto do builder (sem pip cache, sem arquivos de build)
COPY --from=builder /opt/venv /opt/venv

# Copia explicitamente só o código de produção (tests/ nunca entra)
# Arquivos ficam de posse do root: o usuário da app só lê, não altera
COPY app.py .
COPY lib/ ./lib/

# Executa como usuário não-root
USER 10001:10001

# Documenta a porta da API
EXPOSE 8080

# Forma exec: Gunicorn vira PID 1 e recebe o SIGTERM do Kubernetes corretamente
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

#### 3.2 .dockerignore


```gitignore
# Testes e artefatos de teste (não vão para produção)
tests/
.pytest_cache/
.coverage
htmlcov/
.tox/

# Segredos e configurações locais (python-dotenv lê .env; nunca pode ir na imagem)
.env
.env.*
*.env
*.pem
*.key

# Ambientes virtuais e bytecode locais
.venv/
venv/
env/
__pycache__/
*.py[cod]

# Controle de versão e CI
.git/
.gitignore
.github/
.gitlab-ci.yml

# IDEs e SO
.vscode/
.idea/
*.swp
.DS_Store

# Manifestos de deploy e arquivos do próprio build
k8s/
helm/
Dockerfile
.dockerignore
docker-compose*.yml

# Documentação
*.md
docs/
```

#### 3.3 Boas práticas aplicadas

|Boa prática|Onde foi aplicada|Por quê|
|---|---|---|
|Imagem base oficial, slim e com tag fixa|`FROM python:3.12-slim-bookworm`|Menor superfície de ataque e tamanho; tag fixa garante builds reproduzíveis (o digest `@sha256` reforça isso)|
|Multi-stage build|Estágios `builder` e `runtime`|A imagem final recebe só o venv pronto, sem cache do pip nem resíduos de instalação|
|Ordem de camadas para cache|`COPY requirements.txt` antes do código|Mudanças no código não invalidam a camada de dependências, deixando o CI mais rápido|
|Dependências fixadas e sem cache|`requirements.txt` com `==` + `--no-cache-dir`|Builds determinísticos e camada menor|
|Segredos fora da imagem|Nenhum `ENV`/`ARG` com `DATABASE_URL`/`API_KEY`; `.env` no `.dockerignore`|Qualquer valor na imagem fica legível via `docker history` ou no registry; o Kubernetes injeta em runtime|
|Exclusão de testes|`tests/` no `.dockerignore` + `COPY` explícito de `app.py` e `lib/`|Dupla proteção: menos código e dependências desnecessárias em produção|
|Usuário não-root com UID numérico|`useradd --uid 10001` + `USER 10001:10001`|Limita o impacto de uma invasão; UID numérico permite ao Kubernetes validar `runAsNonRoot`|
|Código imutável em runtime|Arquivos copiados de posse do root|O processo da aplicação não consegue alterar o próprio código|
|Logs em stdout sem buffer|`PYTHONUNBUFFERED=1`|Logs aparecem em tempo real no `kubectl logs` e no coletor do cluster|
|Sem bytecode em disco|`PYTHONDONTWRITEBYTECODE=1`|Evita escrita no filesystem, compatível com `readOnlyRootFilesystem`|
|CMD em forma exec|`CMD ["gunicorn", ...]`|Gunicorn recebe o SIGTERM diretamente e faz graceful shutdown no rolling update|
|Porta documentada|`EXPOSE 8080`|Deixa explícito o contrato com o `containerPort` do Deployment|
|Metadados OCI|`LABEL org.opencontainers.image.*`|Facilita rastreabilidade e inventário no registry|

### 4. Justificativa


As instruções foram colocadas de forma explícita no prompt, separando em Role, Task e Format, como diz o framework. O resultado pode ser visto na resposta do modelo.

- Role define o papel do modelo, sua área de atuação. Talvez uma das mais importantes, pois trás o conhecimento necessário para executar a tarefa. Na resposta do modelo, podemos ver que foram tomadas decisões com base no conhecimento de um especialista DevOps Python como utilizar imagem base oficial, slim e com tag fixa, secrets fora da imagem e usuário não-root com UID numérico. Nenhuma destas configurações foram pedidas, mas foram feitas de acordo com a role definida.

- Task é a parte que demanda maior detalhamento. Nela foi definido o que se espera que seja feito e onde contextualizamos com as informações que já temos, como a stack, estrutura e dependências do projeto. Defini alguns requisitos e deixei a instrução para seguir as boas práticas para avaliar o conhecimento do modelo a partir da definição da Role. Compensei este risco, pedindo a tabela de boas práticas aplicadas.

- Format é onde defino como quero o resultado, sendo mais objetivo pedi o conteúdo dos arquivos dockerfile e dockerignore, com comentários curtos em cada instrução. Vemos pelo arquivo gerado que foram seguidas as intruções.
  