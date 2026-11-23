**Tarefa.** Aplicando o framework **B-A-B**, escrever o prompt de IA que, recebendo esse manifest, produza a versão modernizada.
**Entregue.** Prompt, modelo, output e justificativa mostrando como Before, After e Bridge aparecem no prompt.


### 1. Prompt

```

# Before
O manifest do Chronos que o George escreveu três anos atrás precisa ser modernizado pois muita coisa que hoje é obrigatória no padrão da empresa ainda não está presente nele. Segue o manifest abaixo:

apiVersion: apps/v1 kind: Deployment metadata: name: chronos-api namespace: production spec: replicas: 1 selector: matchLabels: app: chronos-api template: metadata: labels: app: chronos-api spec: containers: - name: api image: chronos-api:latest ports: - containerPort: 8080 env: - name: DB_PASSWORD value: "P@ssw0rd2023!" - name: JWT_SECRET value: "hvt-jwt-prod-secret"


# After
A versão moderna precisa ter alta disponibilidade, imagem versionada (nada de `latest`), secrets fora do manifest, resource requests e limits, liveness e readiness probes, `securityContext` não-root e as demais práticas de produção que hoje são padrão na empresa.

# Bridge
Crie um plano de modernização em fases, considerando segurança e estabilidade do ambiente. Para cada fase inclua: o que será alterado, pré-requisitos, estimativa de esforço, riscos principais e plano de rollback.

```

 
```


```

### 2. Modelo

```
Haiku 5.5
```


### 3. Output
---

#### Alerta de segurança (fazer antes de qualquer mudança)

O manifest original contém a senha do banco e o segredo do JWT em texto puro. Como ele está versionado, esses valores devem ser considerados **comprometidos**, mesmo que o repositório seja privado, porque ficam no histórico do Git e em qualquer cópia do manifest. Recomendo:

1. Rotacionar a senha do banco e o `JWT_SECRET`.
2. Remover os valores do histórico do Git (`git filter-repo` ou BFG) e invalidar clones antigos.
3. Verificar os logs de acesso do banco no período em que os valores ficaram expostos.
4. Avaliar o impacto da troca do JWT: todos os tokens emitidos com o segredo antigo deixarão de valer, então os usuários precisarão fazer login de novo. Se a aplicação suportar múltiplas chaves (por `kid`), dá para fazer a troca sem derrubar sessões.

#### Manifest modernizado

Os pontos marcados com `<...>` ou com comentário `# CONFIRMAR` dependem de informações que não estão no manifest original.

yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app.kubernetes.io/name: chronos-api
    app.kubernetes.io/part-of: chronos
    app.kubernetes.io/version: "<VERSAO>"          # CONFIRMAR
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
        app.kubernetes.io/name: chronos-api
        app.kubernetes.io/version: "<VERSAO>"
    spec:
      serviceAccountName: chronos-api              # ServiceAccount dedicada
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chronos-api
      containers:
        - name: api
          image: <REGISTRY>/chronos-api:<VERSAO>@sha256:<DIGEST>
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - secretRef:
                name: chronos-api-secrets          # criado pelo External Secrets / Vault
          env:
            - name: DB_HOST                        # CONFIRMAR nomes reais das variáveis
              valueFrom:
                configMapKeyRef:
                  name: chronos-api-config
                  key: DB_HOST
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              memory: 512Mi                        # sem limit de CPU: evita throttling
          startupProbe:
            httpGet:
              path: /health/live                   # CONFIRMAR endpoints
              port: http
            periodSeconds: 2
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 20
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chronos-api
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chronos-api
  minReplicas: 3
  maxReplicas: 6                                   # dimensionar pelo pool de conexões do banco
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Os segredos não aparecem no manifest. O Secret `chronos-api-secrets` deve ser gerado a partir do cofre (Vault, AWS Secrets Manager ou equivalente) por meio do External Secrets Operator, ou então selado com Sealed Secrets. Os valores antigos não devem ser reaproveitados.

### Plano de modernização em fases

Os esforços são estimativas para um engenheiro, sem contar revisão e homologação, e devem ser validados com o time.

#### Fase 0: Contenção de segurança

- **O que muda:** rotação de credenciais, limpeza do histórico do Git, auditoria de acesso.
- **Pré-requisitos:** acesso ao banco e ao cofre; comunicação prévia aos times que dependem do JWT.
- **Esforço:** 1 a 2 dias.
- **Riscos principais:** logout em massa dos usuários; aplicação que não suporta trocar o segredo sem reiniciar; quebra de integrações que usam a senha antiga.
- **Rollback:** durante a janela de troca, manter o valor antigo válido no banco ou aceitar as duas chaves do JWT. Se a rotação causar falha, restaurar a credencial anterior até corrigir a causa.

#### Fase 1: Secrets externalizados e imagem versionada

- **O que muda:** Secret vindo do cofre via External Secrets; imagem com tag fixa e digest; pipeline com scan de vulnerabilidades (Trivy ou similar); `latest` removido.
- **Pré-requisitos:** External Secrets Operator instalado; registry corporativo; confirmação da versão atualmente em execução (o `latest` pode apontar para builds diferentes em cada nó).
- **Esforço:** 3 a 5 dias.
- **Riscos principais:** nomes de variáveis de ambiente divergentes entre o código e o cofre; a imagem versionada se comportar de forma diferente da que está em `latest`.
- **Rollback:** `kubectl rollout undo deployment/chronos-api -n production`; manter o Secret antigo até a validação completa.

#### Fase 2: Probes e recursos

- **O que muda:** startup, readiness e liveness probes; requests e limits de CPU e memória baseados em métricas reais.
- **Pré-requisitos:** endpoints de saúde implementados ou identificados; pelo menos duas semanas de métricas de consumo em produção.
- **Esforço:** 2 a 4 dias.
- **Riscos principais:** liveness agressiva causando loop de reinício; limite de memória baixo gerando OOMKill. Começar só com readiness é a forma mais segura.
- **Rollback:** reverter a revisão no Deployment ou remover as probes em um commit separado.

#### Fase 3: Hardening do container

- **O que muda:** `runAsNonRoot`, `readOnlyRootFilesystem`, `drop ALL` nas capabilities, `seccompProfile`, `emptyDir` para `/tmp`.
- **Pré-requisitos:** imagem que roda como usuário não-root (ajuste no Dockerfile); mapeamento de todos os caminhos onde a aplicação escreve.
- **Esforço:** 2 a 5 dias, dependendo de quantos caminhos de escrita existirem.
- **Riscos principais:** a aplicação tentar gravar em um caminho read-only e falhar só em alguns fluxos, como upload ou logs em arquivo.
- **Rollback:** reverter a revisão. Testar em staging com carga e com os fluxos que escrevem em disco.

#### Fase 4: Alta disponibilidade

- **O que muda:** de 1 para 3 réplicas; PodDisruptionBudget; `topologySpreadConstraints` por zona; HPA; estratégia de rolling update sem indisponibilidade.
- **Pré-requisitos:** confirmar que a aplicação é stateless (sessões, cache local e arquivos temporários); cluster com nós em mais de uma zona; **dimensionamento do pool de conexões do banco** (3 a 6 réplicas multiplicam as conexões).
- **Esforço:** 3 a 5 dias, incluindo testes de falha de nó e de zona.
- **Riscos principais:** esgotamento de conexões no banco; aumento de custo; estado em memória que só aparece com mais de uma réplica.
- **Rollback:** escalar de volta para 1 réplica, remover o HPA e ajustar o PDB. Nenhuma mudança de dados é envolvida.

#### Fase 5: Padronização e governança

- **O que muda:** labels padrão, ServiceAccount dedicada, NetworkPolicy com menor privilégio, políticas de admissão (Kyverno ou OPA Gatekeeper), validação de manifests no CI (kubeconform, kube-linter), deploy via GitOps (Argo CD ou Flux).
- **Pré-requisitos:** padrões da empresa documentados; ferramenta de GitOps disponível; mapa das dependências de rede do Chronos.
- **Esforço:** 5 a 8 dias.
- **Riscos principais:** políticas bloqueando deploys de outros serviços; NetworkPolicy derrubando comunicação não documentada.
- **Rollback:** colocar as políticas em modo `audit` antes de `enforce`; remover NetworkPolicy por commit no GitOps.

### Riscos transversais e pré-requisitos gerais

- **Documentação e dono do serviço:** o George escreveu o manifest há três anos, então vale registrar quem é o responsável atual e documentar as dependências (banco, filas, integrações).
- **Staging equivalente a produção:** cada fase deve ser validada em um ambiente com a mesma configuração e carga aproximada.
- **Backup do banco antes da Fase 0 e da Fase 4.**
- **Métricas de linha de base** (latência, taxa de erro, consumo de recursos) para comparar antes e depois de cada fase.
- **Janela de mudança** e plano de comunicação para cada fase que afeta a produção.
- **Rollback geral:** se todas as mudanças forem feitas via GitOps, cada fase vira um commit reversível. Com o Deployment, `kubectl rollout undo` resolve a maioria dos casos em segundos.

**Esforço total estimado:** 3 a 5 semanas de um engenheiro, considerando sequenciamento e validações. A Fase 0 não deve esperar pelas demais.

### Informações necessárias para fechar o plano

1. Versão atual da imagem em execução e o registry usado.
2. Endpoints de health check da aplicação.
3. Lista completa de variáveis de ambiente (o manifest original não declara o host do banco nem outras configurações).
4. Se a aplicação mantém estado em memória ou em disco.
5. Quantas zonas o cluster tem e qual é o limite de conexões do banco.
6. Se o time já usa External Secrets, Vault, Argo CD ou Kyverno.


### 4. Justificativa

Na saída do prompt podemos ver o plano montado em fases, considerando o estado atual do manifest e onde queremos chegar depois da execução do plano. No meu prompt, solicitei que considerasse segurança e estabilidade do ambiente. Pelo plano gerado a fase 0 e 1 tratou exatamente disso ao abordar rotação de credencias. Nas fases 2 a 4 ja começa a ser tratado sobre a estabilidade, alta disponibilidade e etc. Na fase 5 já foi uma viajada porque ficou bem genérico no prompt como eu disse "demais práticas de produção que hoje são padrão na empresa" mas o pré-requisito tratou bem isso ao dizer sobre o padrões da empresa documentados; ferramenta de GitOps disponível; mapa das dependências de rede do Chronos.