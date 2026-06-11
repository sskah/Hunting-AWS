# CTI Hunting — Migração para REST API + variáveis de ambiente por Lambda

## 0. Resumo do que mudou (e do que NÃO precisa mudar)

- **As Lambdas não precisam ser alteradas.** Cada Lambda lê o evento de forma
  agnóstica ao tipo de API: ela aceita tanto o payload **REST (v1)** quanto o
  **HTTP API (v2)**. No código (`request_info`):
  - método: `requestContext.http.method` (v2) **ou** `httpMethod` (REST v1);
  - caminho: `rawPath` (v2) **ou** `path` (REST v1);
  - além disso ela corta tudo que vem antes de `/api/`, então o prefixo do
    *stage* (ex.: `/prod/api/health`) é tratado automaticamente.
  - O roteamento por sub-rota (`/assets/{id}`, `/scans/{id}/correction`, etc.)
    é feito **dentro** da Lambda, lendo o próprio path — ela **não** depende de
    *path parameters* do API Gateway. Por isso o `{proxy+}` funciona.
- **Condição única:** usar **Lambda Proxy Integration (AWS_PROXY)** e garantir
  que o caminho recebido contenha `/api/...`.
- **Front-end:** todas as chamadas já são `"/api/<rota>"`. Só muda **onde** o
  front encontra a API (ver seção 4). Recomendado manter mesma origem via
  CloudFront para **não precisar de CORS** — e é justamente isso que permite
  não mexer nas Lambdas (elas não enviam cabeçalhos `Access-Control-*`).

---

## 1. Mínimo OBRIGATÓRIO (variáveis sem default utilizável)

Quase todas as variáveis têm *default* igual ao nome real do recurso
(`CTI_HUNTING_ASSETS_TABLE` → `cti_hunting_assets`, etc.). Ou seja, se você
nomeou tudo conforme o README, a maioria é **opcional**. As que **precisam**
ser definidas porque o default é vazio / específico do ambiente:

| Variável | Em quais Lambdas | Por quê |
|---|---|---|
| `CTI_HUNTING_ORCHESTRATOR_LAMBDA_ARN` | `cti_hunting_assets`, `cti_hunting_schedules`, `cti_hunting_osint_orchestrator`* | ARN alvo do EventBridge Scheduler (default vazio) |
| `CTI_HUNTING_SCHEDULER_ROLE_ARN` | `cti_hunting_assets`, `cti_hunting_schedules`, `cti_hunting_osint_orchestrator`* | Role que o Scheduler assume (default vazio) |
| `CTI_HUNTING_FRONTEND_URL` | `cti_hunting_auth` | Monta o `redirect_uri` do OAuth (ver seção 5). Alternativa: `OAUTH_REDIRECT_URI` na secret/config |
| `CTI_HUNTING_REPORT_FROM_EMAIL` | `cti_hunting_report` | Remetente SES (aceita `CTI_HUNTING_SES_FROM_EMAIL` como alternativa) |

\* No orquestrador, os dois ARNs só são necessários se ele recriar/atualizar
schedules.

> Tudo o que estiver nas tabelas abaixo e **não** estiver aqui é “defina se quiser
> ser explícito ou se o nome do recurso difere do default”.

---

## 2. Variáveis por Lambda — API Gateway

Legenda: **R** = obrigatória · **D** = opcional (já tem default = nome real).

### `cti_hunting_health`
Faz *ping* nos collectors e no report.
```
CTI_HUNTING_SECRETS_NAME              = cti_hunting_secrets            (D)
CTI_HUNTING_ASSETS_TABLE              = cti_hunting_assets            (D)
CTI_HUNTING_SCANS_TABLE               = cti_hunting_scans             (D)
CTI_HUNTING_SAFELIST_TABLE            = cti_hunting_safelist          (D)
CTI_HUNTING_CONFIG_TABLE              = cti_hunting_config            (D)
CTI_HUNTING_JOBS_TABLE                = cti_hunting_jobs              (D)
CTI_HUNTING_COLLECT_SERP_LAMBDA       = cti_hunting_collect_serp      (D)
CTI_HUNTING_COLLECT_VIRUSTOTAL_LAMBDA = cti_hunting_collect_virustotal(D)
CTI_HUNTING_COLLECT_APURA_LAMBDA      = cti_hunting_collect_apura     (D)
CTI_HUNTING_COLLECT_INTELX_LAMBDA     = cti_hunting_collect_intelx    (D)
CTI_HUNTING_COLLECT_DOMAINTOOLS_LAMBDA= cti_hunting_collect_domaintools(D)
CTI_HUNTING_COLLECT_DNSPROBE_LAMBDA   = cti_hunting_collect_dnsprobe  (D)
CTI_HUNTING_COLLECT_SERP_TABLE        = cti_hunting_collect_serp      (D)
CTI_HUNTING_COLLECT_VIRUSTOTAL_TABLE  = cti_hunting_collect_virustotal(D)
CTI_HUNTING_COLLECT_APURA_TABLE       = cti_hunting_collect_apura     (D)
CTI_HUNTING_COLLECT_INTELX_TABLE      = cti_hunting_collect_intelx    (D)
CTI_HUNTING_COLLECT_DOMAINTOOLS_TABLE = cti_hunting_collect_domaintools(D)
CTI_HUNTING_COLLECT_DNSPROBE_TABLE    = cti_hunting_collect_dnsprobe  (D)
CTI_HUNTING_REPORT_LAMBDA             = cti_hunting_report            (D)
```
IAM: invoke nos collectors + report; leitura nas tabelas que checar.

### `cti_hunting_assets`
Cria/atualiza assets **e** sincroniza schedules no EventBridge.
```
CTI_HUNTING_ASSETS_TABLE          = cti_hunting_assets   (D)
CTI_HUNTING_SCANS_TABLE           = cti_hunting_scans    (D)
CTI_HUNTING_SAFELIST_TABLE        = cti_hunting_safelist (D)
CTI_HUNTING_CONFIG_TABLE          = cti_hunting_config   (D)
CTI_HUNTING_JOBS_TABLE            = cti_hunting_jobs     (D)
CTI_HUNTING_ASSETS_SCHEDULE_INDEX = schedule-index       (D)
CTI_HUNTING_ORCHESTRATOR_LAMBDA_ARN = <ARN do orquestrador>   (R)
CTI_HUNTING_SCHEDULER_ROLE_ARN      = <ARN da role do Scheduler> (R)
CTI_HUNTING_SCHEDULER_GROUP       = default              (D)
CTI_HUNTING_SCHEDULER_TIMEZONE    = America/Sao_Paulo    (D)
CTI_HUNTING_SCHEDULE_TIME         = 00:00:00             (D)
```
IAM: R/W em `cti_hunting_assets`; `scheduler:CreateSchedule/UpdateSchedule/DeleteSchedule`; `iam:PassRole` da role do Scheduler.

### `cti_hunting_scans`
```
CTI_HUNTING_SCANS_TABLE    = cti_hunting_scans    (D)
CTI_HUNTING_SAFELIST_TABLE = cti_hunting_safelist (D)
CTI_HUNTING_CONFIG_TABLE   = cti_hunting_config   (D)
```
IAM: R/W em `cti_hunting_scans` (e leitura em safelist/config se usar correção).

### `cti_hunting_safelist`
```
CTI_HUNTING_SAFELIST_TABLE = cti_hunting_safelist (D)
```
IAM: R/W em `cti_hunting_safelist`.

### `cti_hunting_schedules`
Lê/grava config de agenda e escreve schedules.
```
CTI_HUNTING_CONFIG_TABLE            = cti_hunting_config        (D)
CTI_HUNTING_ASSETS_TABLE            = cti_hunting_assets        (D)
CTI_HUNTING_ORCHESTRATOR_LAMBDA_ARN = <ARN do orquestrador>     (R)
CTI_HUNTING_SCHEDULER_ROLE_ARN      = <ARN da role do Scheduler>(R)
CTI_HUNTING_SCHEDULER_GROUP         = default                  (D)
CTI_HUNTING_SCHEDULER_TIMEZONE      = America/Sao_Paulo         (D)
CTI_HUNTING_SCHEDULE_TIME           = 00:00:00                 (D)
```
IAM: igual ao assets (Scheduler + PassRole) + R/W em config.

### `cti_hunting_osint_jobs`
Cria job e **invoca** o orquestrador.
```
CTI_HUNTING_JOBS_TABLE          = cti_hunting_jobs               (D)
CTI_HUNTING_ASSETS_TABLE        = cti_hunting_assets             (D)
CTI_HUNTING_CONFIG_TABLE        = cti_hunting_config             (D)
CTI_HUNTING_ORCHESTRATOR_LAMBDA = cti_hunting_osint_orchestrator (D)  ← nome, não ARN
```
IAM: R/W em `cti_hunting_jobs`; `lambda:InvokeFunction` no orquestrador.

### `cti_hunting_reports`
Cria job de relatório e **invoca** o report.
```
CTI_HUNTING_SCANS_TABLE = cti_hunting_scans  (D)
CTI_HUNTING_JOBS_TABLE  = cti_hunting_jobs   (D)
CTI_HUNTING_REPORT_LAMBDA = cti_hunting_report (D)  ← nome, não ARN
```
IAM: R/W em `cti_hunting_jobs`, leitura em `cti_hunting_scans`; `lambda:InvokeFunction` no report.

### `cti_hunting_config`
```
CTI_HUNTING_CONFIG_TABLE = cti_hunting_config   (D)
CTI_HUNTING_SECRETS_NAME = cti_hunting_secrets  (D)
```
IAM: R/W em `cti_hunting_config`; `secretsmanager:GetSecretValue` (para `/config/full`).

### `cti_hunting_auth`  ⚠️ precisa da **Layer** (usa `requests`)
```
CTI_HUNTING_CONFIG_TABLE = cti_hunting_config   (D)
CTI_HUNTING_SECRETS_NAME = cti_hunting_secrets  (D)
CTI_HUNTING_FRONTEND_URL = https://SEU_DOMINIO_CLOUDFRONT   (R)*
```
\* Em vez de `CTI_HUNTING_FRONTEND_URL`, você pode gravar `OAUTH_REDIRECT_URI`
na secret/config (tem prioridade). O `client_id`/`client_secret` do OAuth ficam
na **Secret** (`OAUTH_MICROSOFT_CLIENT_ID`, `OAUTH_MICROSOFT_CLIENT_SECRET`),
não em variável de ambiente. IAM: leitura em config + `GetSecretValue`.

### `cti_hunting_recalc`
```
CTI_HUNTING_SCANS_TABLE  = cti_hunting_scans   (D)
CTI_HUNTING_CONFIG_TABLE = cti_hunting_config  (D)
```
IAM: R/W em `cti_hunting_scans`; leitura em config.

---

## 3. Variáveis por Lambda — OSINT / Report

### `cti_hunting_osint_orchestrator`
```
CTI_HUNTING_ASSETS_TABLE              = cti_hunting_assets             (D)
CTI_HUNTING_SCANS_TABLE               = cti_hunting_scans             (D)
CTI_HUNTING_SAFELIST_TABLE            = cti_hunting_safelist          (D)
CTI_HUNTING_CONFIG_TABLE              = cti_hunting_config            (D)
CTI_HUNTING_JOBS_TABLE                = cti_hunting_jobs              (D)
CTI_HUNTING_ASSETS_SCHEDULE_INDEX     = schedule-index                (D)
CTI_HUNTING_COLLECTOR_TTL_DAYS        = 90                            (D)
CTI_HUNTING_SCHEDULER_TIMEZONE        = America/Sao_Paulo             (D)
CTI_HUNTING_COLLECT_SERP_LAMBDA       = cti_hunting_collect_serp      (D)
CTI_HUNTING_COLLECT_VIRUSTOTAL_LAMBDA = cti_hunting_collect_virustotal(D)
CTI_HUNTING_COLLECT_APURA_LAMBDA      = cti_hunting_collect_apura     (D)
CTI_HUNTING_COLLECT_INTELX_LAMBDA     = cti_hunting_collect_intelx    (D)
CTI_HUNTING_COLLECT_DOMAINTOOLS_LAMBDA= cti_hunting_collect_domaintools(D)
CTI_HUNTING_COLLECT_DNSPROBE_LAMBDA   = cti_hunting_collect_dnsprobe  (D)
CTI_HUNTING_COLLECT_SERP_TABLE        = cti_hunting_collect_serp      (D)
CTI_HUNTING_COLLECT_VIRUSTOTAL_TABLE  = cti_hunting_collect_virustotal(D)
CTI_HUNTING_COLLECT_APURA_TABLE       = cti_hunting_collect_apura     (D)
CTI_HUNTING_COLLECT_INTELX_TABLE      = cti_hunting_collect_intelx    (D)
CTI_HUNTING_COLLECT_DOMAINTOOLS_TABLE = cti_hunting_collect_domaintools(D)
CTI_HUNTING_COLLECT_DNSPROBE_TABLE    = cti_hunting_collect_dnsprobe  (D)
# Só se o orquestrador recriar schedules:
CTI_HUNTING_ORCHESTRATOR_LAMBDA_ARN   = <ARN do orquestrador>   (R se aplicável)
CTI_HUNTING_SCHEDULER_ROLE_ARN        = <ARN da role>           (R se aplicável)
CTI_HUNTING_SCHEDULER_GROUP           = default                 (D)
CTI_HUNTING_SCHEDULE_TIME             = 00:00:00                (D)
```
IAM: R/W em jobs/scans/assets/safelist/config + todas `cti_hunting_collect_*`;
`lambda:InvokeFunction` nos collectors; Scheduler+PassRole se recriar schedules.

### Collectors — `cti_hunting_collect_{serp,virustotal,apura,intelx,domaintools,dnsprobe}`
Cada um usa **apenas a sua própria tabela**:
```
CTI_HUNTING_SECRETS_NAME        = cti_hunting_secrets             (D)
CTI_HUNTING_ORCHESTRATOR_LAMBDA = cti_hunting_osint_orchestrator  (D)
CTI_HUNTING_COLLECTOR_TTL_DAYS  = 90                              (D)
CTI_HUNTING_SECRET_CACHE_TTL    = 60                              (D)
CTI_HUNTING_COLLECT_<NOME>_TABLE = cti_hunting_collect_<nome>     (D)  ← a SUA tabela
DOMAINTOOLS_MONITOR_ID          = <id>                            (D, só DomainTools)
```
Mapeamento da tabela por collector:
`serp→CTI_HUNTING_COLLECT_SERP_TABLE`, `virustotal→..._VIRUSTOTAL_TABLE`,
`apura→..._APURA_TABLE`, `intelx→..._INTELX_TABLE`,
`domaintools→..._DOMAINTOOLS_TABLE`, `dnsprobe→..._DNSPROBE_TABLE`.
As chaves de API (`SERPAPI_KEY`, `VIRUSTOTAL_KEY`, `APURA_API_KEY`, `INTELX_KEY`,
`DOMAINTOOLS_API_USERNAME`/`DOMAINTOOLS_API_KEY`) ficam na **Secret**, não em env.
IAM: `GetSecretValue`; R/W só na própria tabela; `InvokeFunction` no orquestrador.
Precisam da **Layer** (usam libs externas).

### `cti_hunting_report`  ⚠️ precisa da **Layer**
```
CTI_HUNTING_SCANS_TABLE      = cti_hunting_scans   (D)
CTI_HUNTING_JOBS_TABLE       = cti_hunting_jobs    (D)
CTI_HUNTING_SECRETS_NAME     = cti_hunting_secrets (D)
CTI_HUNTING_REPORT_FROM_EMAIL= remetente@seu-dominio   (R)  (ou CTI_HUNTING_SES_FROM_EMAIL)
```
IAM: leitura em `cti_hunting_scans`, R/W em `cti_hunting_jobs`, `ses:SendRawEmail`.

---

## 4. REST API Gateway — recursos, métodos e integração

Use **HTTP method `ANY`** + **Lambda Proxy (AWS_PROXY)** em cada recurso.
Como cada Lambda resolve a sub-rota sozinha, crie um recurso por domínio e um
filho coringa `{proxy+}` apontando para a **mesma** Lambda:

```
/api
  /health              ANY → cti_hunting_health
  /assets              ANY → cti_hunting_assets
    /{proxy+}          ANY → cti_hunting_assets
  /scans               ANY → cti_hunting_scans
    /{proxy+}          ANY → cti_hunting_scans
  /safelist
    /{proxy+}          ANY → cti_hunting_safelist
  /schedule
    /{proxy+}          ANY → cti_hunting_schedules
  /osint               ANY → cti_hunting_osint_jobs
    /{proxy+}          ANY → cti_hunting_osint_jobs
  /report              ANY → cti_hunting_reports
    /{proxy+}          ANY → cti_hunting_reports
  /reports
    /{proxy+}          ANY → cti_hunting_reports
  /config              ANY → cti_hunting_config
    /{proxy+}          ANY → cti_hunting_config
  /auth
    /{proxy+}          ANY → cti_hunting_auth
  /recalc              ANY → cti_hunting_recalc
```

Passos no console:
1. **API Gateway → Create API → REST API (Build)**, não “HTTP API”.
2. Crie os recursos acima. Em cada um marque **“Configure as proxy resource”**
   ao criar o `{proxy+}` (cria método `ANY` automaticamente).
3. Integração: **Lambda Function**, marque **“Use Lambda Proxy integration”**,
   selecione a Lambda do domínio. Aceite a permissão de invoke quando solicitado.
4. **Deploy API** para um *stage* (ex.: `prod`). A URL fica
   `https://{id}.execute-api.{region}.amazonaws.com/prod`.

> O prefixo do stage (`/prod`) no caminho não atrapalha: a Lambda corta tudo
> antes de `/api/`. Por isso `GET /prod/api/health` chega como `/api/health`.

Mapa de rotas → Lambda (todas cobertas pelos recursos acima):

| Método(s) | Rota | Lambda |
|---|---|---|
| GET | `/api/health` | health |
| GET POST | `/api/assets` · `/api/assets/{id}` (GET/PUT/DELETE) · `/api/assets/{id}/toggle-auto` (POST) | assets |
| GET POST DELETE | `/api/scans` · `/api/scans/{id}` · `/api/scans/{id}/correction` (PUT) | scans |
| GET POST DELETE | `/api/safelist/{asset}` (+ `/{module}/{fp}`) | safelist |
| GET PUT | `/api/schedule/config` · `/api/schedule/next` (GET) | schedules |
| POST GET | `/api/osint` · `/api/osint/jobs` · `/api/osint/jobs/{id}` · `/api/osint/jobs/{id}/cancel` | osint_jobs |
| POST GET | `/api/report` · `/api/report/pdf` · `/api/reports/email` · `/api/reports/jobs/{id}` | reports |
| GET PUT POST | `/api/config` · `/api/config/full` · `/api/config/reset` | config |
| GET POST | `/api/auth/config` · `/api/auth/exchange` | auth |
| POST | `/api/recalc` | recalc |

### CORS (importante para “não mexer nas Lambdas”)
- **Recomendado:** servir front e API na **mesma origem** (CloudFront com
  comportamento `/api/*` → origem = este REST API, *origin path* = `/prod`).
  Mesma origem ⇒ **sem CORS** ⇒ as Lambdas não precisam emitir cabeçalhos.
- Se chamar a URL `execute-api` direto (origem diferente), o “Enable CORS” do
  API Gateway só responde o **preflight OPTIONS**. Com integração **proxy**, as
  respostas de GET/POST também precisariam de `Access-Control-Allow-Origin`, que
  estas Lambdas **não** enviam. Como você não quer alterá-las, **use a mesma
  origem (CloudFront)**.

---

## 5. Correção do login OAuth (voltava para a tela de login)

**Causa raiz.** No fluxo original, **só** `login.html` executava a troca do
código (`completeOAuth`). O `index.html` tem uma trava que manda todo acesso
não autenticado de volta para `login.html`. Se o provedor redirecionar para
qualquer página que **não** seja `login.html` (por ex. a raiz do site, que
serve `index.html`), o código nunca é trocado, nenhuma sessão é gravada e o
`index.html` “rebota” para o login — exatamente o sintoma observado.

**O que mudou no front (já aplicado):**
- O tratamento do callback foi movido para `auth.js`, num **guard único** que
  roda em **qualquer** página: se a URL tiver `?code=&state=`, ele finaliza a
  troca ali e segue para `index.html` com `location.replace()` (sem deixar o
  `?code=` no histórico). Assim o login funciona mesmo que o `redirect_uri`
  aponte para a raiz/`index.html`.
- Removido o login local (admin/usuário) — agora é **somente OAuth**.
- `expires_at` é normalizado (aceita segundos **ou** milissegundos), evitando
  sessão “nascer expirada”.

**Configuração que você precisa garantir:**
1. `cti_hunting_auth` com `CTI_HUNTING_FRONTEND_URL = https://SEU_DOMINIO`
   (ou `OAUTH_REDIRECT_URI` na secret). O `redirect_uri` resultante será
   `https://SEU_DOMINIO/login.html`.
2. No **Azure AD (App Registration) → Authentication → Redirect URIs**,
   registre **exatamente** essa mesma URL (`https://SEU_DOMINIO/login.html`),
   tipo **SPA** (PKCE, sem client secret) ou **Web** se usar client secret.
3. Secret com `OAUTH_MICROSOFT_CLIENT_ID` (e `OAUTH_MICROSOFT_CLIENT_SECRET`
   se o app for “Web”). Com PKCE/SPA, o secret é opcional.
4. Servir front e API na mesma origem (CloudFront) — ver seção 4.

---

## 6. Checklist rápido
- [ ] Layer Python anexada em: `auth`, `report` e nos 6 collectors.
- [ ] `CTI_HUNTING_ORCHESTRATOR_LAMBDA_ARN` e `CTI_HUNTING_SCHEDULER_ROLE_ARN`
      em `assets` e `schedules`.
- [ ] `CTI_HUNTING_FRONTEND_URL` em `auth` + Redirect URI igual no Azure.
- [ ] `CTI_HUNTING_REPORT_FROM_EMAIL` em `report` + remetente verificado no SES.
- [ ] REST API com recursos `{proxy+}` por domínio, **Lambda Proxy**, deploy no stage.
- [ ] CloudFront `/api/*` → REST API (origin path = `/prod`); `js/env.js` com base vazia.
- [ ] Invalidar cache do CloudFront após subir o front.
