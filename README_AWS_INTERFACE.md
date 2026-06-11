# CTI Hunting AWS — implementação final via AWS Console

Este pacote contém somente a arquitetura atual do projeto. Foram removidos os artefatos das versões anteriores, incluindo a Lambda roteadora `cti_hunting_api`, os pacotes `modular`, `reviewed` e a versão single-Lambda.

## Estrutura do pacote

```text
cti_hunting_aws_project_final/
├── README_AWS_INTERFACE.md
├── deploy/
│   ├── frontend/
│   │   └── cti_hunting_frontend_static.zip
│   ├── layer/
│   │   └── cti_hunting_python_layer.zip
│   └── lambdas/
│       ├── api/
│       ├── collectors/
│       ├── osint/
│       └── report/
└── source/
    ├── backend/
    └── frontend/
```

Use os arquivos em `deploy/` para upload na AWS Console. Use `source/` apenas para revisão e manutenção do código.

## Arquitetura vigente

```text
CloudFront + S3
  └── Front-end estático

API Gateway HTTP API
  ├── cti_hunting_health
  ├── cti_hunting_assets
  ├── cti_hunting_scans
  ├── cti_hunting_schedules
  ├── cti_hunting_safelist
  ├── cti_hunting_osint_jobs
  ├── cti_hunting_reports
  ├── cti_hunting_config
  ├── cti_hunting_auth
  └── cti_hunting_recalc

EventBridge Scheduler
  └── cti_hunting_osint_orchestrator
        ├── cti_hunting_collect_serp        → Dynamo cti_hunting_collect_serp
        ├── cti_hunting_collect_virustotal  → Dynamo cti_hunting_collect_virustotal
        ├── cti_hunting_collect_apura       → Dynamo cti_hunting_collect_apura
        ├── cti_hunting_collect_intelx      → Dynamo cti_hunting_collect_intelx
        ├── cti_hunting_collect_domaintools → Dynamo cti_hunting_collect_domaintools
        └── cti_hunting_collect_dnsprobe    → Dynamo cti_hunting_collect_dnsprobe

Relatórios
  └── cti_hunting_report
```

## Lambdas e pacotes de upload

### API Gateway

| Lambda | ZIP |
|---|---|
| `cti_hunting_health` | `deploy/lambdas/api/cti_hunting_health.zip` |
| `cti_hunting_assets` | `deploy/lambdas/api/cti_hunting_assets.zip` |
| `cti_hunting_scans` | `deploy/lambdas/api/cti_hunting_scans.zip` |
| `cti_hunting_schedules` | `deploy/lambdas/api/cti_hunting_schedules.zip` |
| `cti_hunting_safelist` | `deploy/lambdas/api/cti_hunting_safelist.zip` |
| `cti_hunting_osint_jobs` | `deploy/lambdas/api/cti_hunting_osint_jobs.zip` |
| `cti_hunting_reports` | `deploy/lambdas/api/cti_hunting_reports.zip` |
| `cti_hunting_config` | `deploy/lambdas/api/cti_hunting_config.zip` |
| `cti_hunting_auth` | `deploy/lambdas/api/cti_hunting_auth.zip` |
| `cti_hunting_recalc` | `deploy/lambdas/api/cti_hunting_recalc.zip` |

### OSINT

| Lambda | ZIP |
|---|---|
| `cti_hunting_osint_orchestrator` | `deploy/lambdas/osint/cti_hunting_osint_orchestrator.zip` |
| `cti_hunting_collect_serp` | `deploy/lambdas/collectors/cti_hunting_collect_serp.zip` |
| `cti_hunting_collect_virustotal` | `deploy/lambdas/collectors/cti_hunting_collect_virustotal.zip` |
| `cti_hunting_collect_apura` | `deploy/lambdas/collectors/cti_hunting_collect_apura.zip` |
| `cti_hunting_collect_intelx` | `deploy/lambdas/collectors/cti_hunting_collect_intelx.zip` |
| `cti_hunting_collect_domaintools` | `deploy/lambdas/collectors/cti_hunting_collect_domaintools.zip` |
| `cti_hunting_collect_dnsprobe` | `deploy/lambdas/collectors/cti_hunting_collect_dnsprobe.zip` |

### Relatórios

| Lambda | ZIP |
|---|---|
| `cti_hunting_report` | `deploy/lambdas/report/cti_hunting_report.zip` |

### Front-end e Layer

| Item | ZIP |
|---|---|
| Front-end estático | `deploy/frontend/cti_hunting_frontend_static.zip` |
| Layer Python | `deploy/layer/cti_hunting_python_layer.zip` |

## Dynamos esperadas

### Tabelas principais

| Tabela | Partition key | Sort key | Uso |
|---|---|---|---|
| `cti_hunting_assets` | `id` | - | Assets monitorados |
| `cti_hunting_scans` | `id` | - | Histórico consolidado |
| `cti_hunting_safelist` | `asset_key` | - | Correções e safelist |
| `cti_hunting_config` | `key` | - | Configurações não sensíveis |
| `cti_hunting_jobs` | `id` | - | Jobs manuais, agendados e relatórios |

### Tabelas de collectors

Cada collector grava na própria tabela. Todas usam o mesmo schema:

| Tabela | Partition key | Sort key |
|---|---|---|
| `cti_hunting_collect_serp` | `scan_id` | `item_key` |
| `cti_hunting_collect_virustotal` | `scan_id` | `item_key` |
| `cti_hunting_collect_apura` | `scan_id` | `item_key` |
| `cti_hunting_collect_intelx` | `scan_id` | `item_key` |
| `cti_hunting_collect_domaintools` | `scan_id` | `item_key` |
| `cti_hunting_collect_dnsprobe` | `scan_id` | `item_key` |

Itens esperados nas tabelas dos collectors:

```text
COLLECTOR#<provider>
LOG#<timestamp>#<seq>
FINDING#<module>#<fingerprint|uuid>
```

## Secrets Manager

A secret principal deve ser referenciada por:

```text
CTI_HUNTING_SECRETS_NAME=cti_hunting_secrets
```

Chaves esperadas na secret, conforme os módulos habilitados:

```text
SERPAPI_KEY
VIRUSTOTAL_KEY
APURA_API_KEY
INTELX_KEY
DOMAINTOOLS_API_USERNAME
DOMAINTOOLS_API_KEY
```

Se alguma fonte não for usada, a respectiva chave pode ficar ausente. O collector deve retornar erro controlado/health warning, não quebrar os demais módulos.

## Variáveis de ambiente

### Variáveis comuns

Configure nas Lambdas que acessam as respectivas tabelas:

```text
CTI_HUNTING_SECRETS_NAME=cti_hunting_secrets
CTI_HUNTING_ASSETS_TABLE=cti_hunting_assets
CTI_HUNTING_SCANS_TABLE=cti_hunting_scans
CTI_HUNTING_SAFELIST_TABLE=cti_hunting_safelist
CTI_HUNTING_CONFIG_TABLE=cti_hunting_config
CTI_HUNTING_JOBS_TABLE=cti_hunting_jobs
```

### Orquestradora

```text
CTI_HUNTING_COLLECT_SERP_LAMBDA=cti_hunting_collect_serp
CTI_HUNTING_COLLECT_VIRUSTOTAL_LAMBDA=cti_hunting_collect_virustotal
CTI_HUNTING_COLLECT_APURA_LAMBDA=cti_hunting_collect_apura
CTI_HUNTING_COLLECT_INTELX_LAMBDA=cti_hunting_collect_intelx
CTI_HUNTING_COLLECT_DOMAINTOOLS_LAMBDA=cti_hunting_collect_domaintools
CTI_HUNTING_COLLECT_DNSPROBE_LAMBDA=cti_hunting_collect_dnsprobe

CTI_HUNTING_COLLECT_SERP_TABLE=cti_hunting_collect_serp
CTI_HUNTING_COLLECT_VIRUSTOTAL_TABLE=cti_hunting_collect_virustotal
CTI_HUNTING_COLLECT_APURA_TABLE=cti_hunting_collect_apura
CTI_HUNTING_COLLECT_INTELX_TABLE=cti_hunting_collect_intelx
CTI_HUNTING_COLLECT_DOMAINTOOLS_TABLE=cti_hunting_collect_domaintools
CTI_HUNTING_COLLECT_DNSPROBE_TABLE=cti_hunting_collect_dnsprobe
```

### Collectors

Em cada Lambda de coleta:

```text
CTI_HUNTING_SECRETS_NAME=cti_hunting_secrets
CTI_HUNTING_ORCHESTRATOR_LAMBDA=cti_hunting_osint_orchestrator
CTI_HUNTING_COLLECTOR_TTL_DAYS=90
```

Adicione também a tabela específica do collector:

```text
CTI_HUNTING_COLLECT_SERP_TABLE=cti_hunting_collect_serp
CTI_HUNTING_COLLECT_VIRUSTOTAL_TABLE=cti_hunting_collect_virustotal
CTI_HUNTING_COLLECT_APURA_TABLE=cti_hunting_collect_apura
CTI_HUNTING_COLLECT_INTELX_TABLE=cti_hunting_collect_intelx
CTI_HUNTING_COLLECT_DOMAINTOOLS_TABLE=cti_hunting_collect_domaintools
CTI_HUNTING_COLLECT_DNSPROBE_TABLE=cti_hunting_collect_dnsprobe
```

Use apenas a variável da tabela correspondente em cada collector.

### Relatórios

```text
CTI_HUNTING_REPORT_FROM_EMAIL=<email verificado no SES>
CTI_HUNTING_SCANS_TABLE=cti_hunting_scans
CTI_HUNTING_JOBS_TABLE=cti_hunting_jobs
```

### Schedules

Use nas Lambdas que criam/atualizam schedules e na orquestradora:

```text
CTI_HUNTING_ORCHESTRATOR_LAMBDA_ARN=<ARN da cti_hunting_osint_orchestrator>
CTI_HUNTING_SCHEDULER_ROLE_ARN=<ARN da role usada pelo EventBridge Scheduler>
CTI_HUNTING_SCHEDULER_GROUP=default
CTI_HUNTING_SCHEDULER_TIMEZONE=America/Sao_Paulo
CTI_HUNTING_SCHEDULE_TIME=00:00:00
```

## API Gateway HTTP API — rotas

Configure o CORS no API Gateway, não nas Lambdas.

| Método | Rota | Lambda |
|---|---|---|
| GET | `/api/health` | `cti_hunting_health` |
| GET | `/api/assets` | `cti_hunting_assets` |
| POST | `/api/assets` | `cti_hunting_assets` |
| GET | `/api/assets/{asset_id}` | `cti_hunting_assets` |
| PUT | `/api/assets/{asset_id}` | `cti_hunting_assets` |
| DELETE | `/api/assets/{asset_id}` | `cti_hunting_assets` |
| POST | `/api/assets/{asset_id}/toggle-auto` | `cti_hunting_assets` |
| GET | `/api/scans` | `cti_hunting_scans` |
| POST | `/api/scans` | `cti_hunting_scans` |
| DELETE | `/api/scans` | `cti_hunting_scans` |
| GET | `/api/scans/{scan_id}` | `cti_hunting_scans` |
| DELETE | `/api/scans/{scan_id}` | `cti_hunting_scans` |
| PUT | `/api/scans/{scan_id}/correction` | `cti_hunting_scans` |
| GET | `/api/safelist/{asset_name}` | `cti_hunting_safelist` |
| POST | `/api/safelist/{asset_name}` | `cti_hunting_safelist` |
| DELETE | `/api/safelist/{asset_name}` | `cti_hunting_safelist` |
| DELETE | `/api/safelist/{asset_name}/{module}/{fp}` | `cti_hunting_safelist` |
| GET | `/api/schedule/config` | `cti_hunting_schedules` |
| PUT | `/api/schedule/config` | `cti_hunting_schedules` |
| GET | `/api/schedule/next` | `cti_hunting_schedules` |
| POST | `/api/osint` | `cti_hunting_osint_jobs` |
| POST | `/api/osint/jobs` | `cti_hunting_osint_jobs` |
| GET | `/api/osint/jobs/{job_id}` | `cti_hunting_osint_jobs` |
| POST | `/api/osint/jobs/{job_id}/cancel` | `cti_hunting_osint_jobs` |
| POST | `/api/report` | `cti_hunting_reports` |
| POST | `/api/report/pdf` | `cti_hunting_reports` |
| POST | `/api/reports/email` | `cti_hunting_reports` |
| GET | `/api/reports/jobs/{job_id}` | `cti_hunting_reports` |
| GET | `/api/config` | `cti_hunting_config` |
| GET | `/api/config/full` | `cti_hunting_config` |
| PUT | `/api/config` | `cti_hunting_config` |
| POST | `/api/config/reset` | `cti_hunting_config` |
| GET | `/api/auth/config` | `cti_hunting_auth` |
| POST | `/api/auth/exchange` | `cti_hunting_auth` |
| POST | `/api/recalc` | `cti_hunting_recalc` |

## Passo a passo de implementação pela AWS Console

### 1. Criar ou revisar as Dynamos

Você informou que as Dynamos já foram criadas. Confirme apenas os nomes e as chaves listadas acima.

Para as tabelas dos collectors, habilite TTL se quiser expirar dados intermediários:

```text
ttl_epoch
```

### 2. Criar ou revisar as secrets

Você informou que as secrets já foram criadas. Confirme o nome usado em `CTI_HUNTING_SECRETS_NAME` e as chaves usadas pelos collectors habilitados.

### 3. Criar a Layer Python

1. Acesse **AWS Lambda**.
2. Vá em **Layers**.
3. Clique em **Create layer**.
4. Nome: `cti_hunting_python_layer`.
5. Faça upload de `deploy/layer/cti_hunting_python_layer.zip`.
6. Runtime compatível: Python 3.12.
7. Crie a layer.

### 4. Criar as Lambdas

Para cada Lambda listada neste README:

1. Acesse **AWS Lambda**.
2. Clique em **Create function**.
3. Escolha **Author from scratch**.
4. Nome: exatamente o nome da Lambda, por exemplo `cti_hunting_assets`.
5. Runtime: Python 3.12.
6. Architecture: x86_64, salvo se sua conta usar arm64 por padrão.
7. Handler: `lambda_function.handler`.
8. Crie a função.
9. Em **Code**, faça upload do ZIP correspondente em `deploy/lambdas/...`.
10. Em **Layers**, anexe `cti_hunting_python_layer` apenas nas Lambdas que usam bibliotecas externas.
11. Em **Configuration > Environment variables**, configure as variáveis necessárias.

### 5. Configurar permissões IAM

Use roles separadas ou uma role por grupo. O mínimo recomendado:

#### Lambdas de API

- DynamoDB apenas nas tabelas usadas por cada Lambda.
- `cti_hunting_osint_jobs`: permissão `lambda:InvokeFunction` para `cti_hunting_osint_orchestrator`.
- `cti_hunting_reports`: permissão `lambda:InvokeFunction` para `cti_hunting_report`.
- `cti_hunting_assets` e `cti_hunting_schedules`: permissões do EventBridge Scheduler quando criarem/alterarem schedules.

#### Orquestradora

- DynamoDB read/write em:
  - `cti_hunting_jobs`
  - `cti_hunting_scans`
  - `cti_hunting_assets`
  - `cti_hunting_safelist`
  - `cti_hunting_config`
  - `cti_hunting_collect_*`
- `lambda:InvokeFunction` nos collectors.
- Permissões do EventBridge Scheduler se ela recriar schedules.

#### Collectors

- `secretsmanager:GetSecretValue` na secret configurada.
- DynamoDB read/write apenas na própria tabela.
- `lambda:InvokeFunction` para `cti_hunting_osint_orchestrator`.

#### Relatório

- DynamoDB read em `cti_hunting_scans`.
- DynamoDB read/write em `cti_hunting_jobs`.
- `ses:SendRawEmail`.

### 6. Criar o API Gateway HTTP API

1. Acesse **API Gateway**.
2. Clique em **Create API**.
3. Escolha **HTTP API**.
4. Configure as integrações Lambda para as Lambdas de API.
5. Crie as rotas conforme a tabela de rotas deste README.
6. Configure CORS no API Gateway:
   - Allowed origins: domínio do CloudFront ou `*` durante testes.
   - Allowed methods: `GET,POST,PUT,DELETE,OPTIONS`.
   - Allowed headers: `Content-Type,Authorization`.
7. Faça deploy para o stage desejado.

### 7. Configurar EventBridge Scheduler

Para OSINT agendado:

1. Acesse **Amazon EventBridge**.
2. Vá em **Scheduler**.
3. Crie ou use o grupo definido em `CTI_HUNTING_SCHEDULER_GROUP`.
4. A target Lambda deve ser `cti_hunting_osint_orchestrator`.
5. A role do Scheduler deve ter permissão para invocar a orquestradora.
6. O payload deve conter pelo menos `asset_id` e uma indicação de execução agendada, quando criado manualmente.

O fluxo normal do projeto também permite que a própria aplicação crie/atualize schedules a partir dos dados do asset.

### 8. Configurar SES para envio de relatórios

1. Acesse **Amazon SES**.
2. Verifique o remetente configurado em `CTI_HUNTING_REPORT_FROM_EMAIL`.
3. Se sua conta estiver em sandbox, verifique também os destinatários usados nos testes.
4. Garanta permissão `ses:SendRawEmail` na role da Lambda `cti_hunting_report`.

### 9. Publicar o front-end no S3

1. Extraia `deploy/frontend/cti_hunting_frontend_static.zip`.
2. Suba os arquivos extraídos no bucket S3 usado como origem do CloudFront.
3. Ajuste `js/env.js` com a URL base do API Gateway ou do comportamento `/api/*` no CloudFront, conforme sua configuração.

### 10. Configurar CloudFront

1. Origem principal: bucket S3 do front-end.
2. Origem adicional: API Gateway HTTP API.
3. Crie comportamento `/api/*` apontando para o API Gateway.
4. Garanta que métodos HTTP necessários sejam permitidos.
5. Invalide o cache após atualizar o front-end.

### 11. Testes mínimos

1. Acesse o front-end pelo CloudFront.
2. Teste login/configuração, se aplicável.
3. Abra a área de logs/health check.
4. Valide `GET /api/health`.
5. Cadastre um asset.
6. Execute OSINT manual.
7. Verifique se o job foi criado em `cti_hunting_jobs`.
8. Verifique se cada collector gravou em sua própria Dynamo.
9. Verifique se a orquestradora consolidou o scan em `cti_hunting_scans`.
10. Gere um relatório e teste envio por email.
