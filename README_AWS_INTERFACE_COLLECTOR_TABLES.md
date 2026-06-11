# CTI Hunting AWS — arquitetura revisada com Dynamo por collector

Esta versão remove a Lambda roteadora central `cti_hunting_api`. O roteamento HTTP fica no API Gateway e cada recurso da API aponta para uma Lambda menor.

## Lambdas

### API

- `cti_hunting_health`
- `cti_hunting_assets`
- `cti_hunting_scans`
- `cti_hunting_schedules`
- `cti_hunting_safelist`
- `cti_hunting_osint_jobs`
- `cti_hunting_reports`
- `cti_hunting_config`
- `cti_hunting_auth`
- `cti_hunting_recalc`

### OSINT

- `cti_hunting_osint_orchestrator`
- `cti_hunting_collect_serp`
- `cti_hunting_collect_virustotal`
- `cti_hunting_collect_apura`
- `cti_hunting_collect_intelx`
- `cti_hunting_collect_domaintools`
- `cti_hunting_collect_dnsprobe`

### Relatórios

- `cti_hunting_report`

## Dynamos

### Tabelas principais

| Tabela | Partition key | Sort key | Observação |
|---|---|---|---|
| `cti_hunting_assets` | `id` | - | Assets monitorados |
| `cti_hunting_scans` | `id` | - | Histórico consolidado |
| `cti_hunting_safelist` | `asset_key` | - | Itens corrigidos/safelisted |
| `cti_hunting_config` | `key` | - | Configurações não sensíveis |
| `cti_hunting_jobs` | `id` | - | Jobs manuais, agendados e relatórios |

### Tabelas dos collectors

Cada collector grava na própria tabela. Todas usam o mesmo schema:

| Tabela | Partition key | Sort key |
|---|---|---|
| `cti_hunting_collect_serp` | `scan_id` | `item_key` |
| `cti_hunting_collect_virustotal` | `scan_id` | `item_key` |
| `cti_hunting_collect_apura` | `scan_id` | `item_key` |
| `cti_hunting_collect_intelx` | `scan_id` | `item_key` |
| `cti_hunting_collect_domaintools` | `scan_id` | `item_key` |
| `cti_hunting_collect_dnsprobe` | `scan_id` | `item_key` |

Itens esperados:

```text
COLLECTOR#<provider>
LOG#<timestamp>#<seq>
FINDING#<module>#<fingerprint|uuid>
```

## Fluxo manual

```text
Front-end
  -> API Gateway
  -> cti_hunting_osint_jobs
  -> cti_hunting_jobs: cria job
  -> cti_hunting_osint_orchestrator
  -> collectors async
  -> cada collector grava em sua própria Dynamo
  -> collector notifica orquestradora
  -> orquestradora lê as tabelas dos collectors
  -> consolida resultado no job
```

## Fluxo agendado

```text
EventBridge Scheduler
  -> cti_hunting_osint_orchestrator
  -> carrega asset em cti_hunting_assets
  -> cria job scheduled_osint
  -> chama collectors async
  -> collectors gravam nas próprias tabelas
  -> orquestradora consolida
  -> salva scan final em cti_hunting_scans
  -> atualiza próxima data do asset
  -> recria/atualiza o schedule do asset
```

## Variáveis de ambiente principais

Use em todas as Lambdas que precisarem:

```text
CTI_HUNTING_SECRETS_NAME=cti_hunting_secrets
CTI_HUNTING_ASSETS_TABLE=cti_hunting_assets
CTI_HUNTING_SCANS_TABLE=cti_hunting_scans
CTI_HUNTING_SAFELIST_TABLE=cti_hunting_safelist
CTI_HUNTING_CONFIG_TABLE=cti_hunting_config
CTI_HUNTING_JOBS_TABLE=cti_hunting_jobs
```

Na orquestradora:

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

Nos collectors:

```text
CTI_HUNTING_ORCHESTRATOR_LAMBDA=cti_hunting_osint_orchestrator
CTI_HUNTING_COLLECTOR_TTL_DAYS=90
```

Na Lambda de relatório:

```text
CTI_HUNTING_REPORT_FROM_EMAIL=<email verificado no SES>
```

Para schedules:

```text
CTI_HUNTING_ORCHESTRATOR_LAMBDA_ARN=<ARN da cti_hunting_osint_orchestrator>
CTI_HUNTING_SCHEDULER_ROLE_ARN=<ARN da role usada pelo EventBridge Scheduler>
CTI_HUNTING_SCHEDULER_GROUP=default
CTI_HUNTING_SCHEDULER_TIMEZONE=America/Sao_Paulo
CTI_HUNTING_SCHEDULE_TIME=00:00:00
```

## API Gateway HTTP API — rotas

Configure CORS no API Gateway, não nas Lambdas.

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

## Permissões IAM resumidas

### Lambdas de API

- DynamoDB apenas nas tabelas que usam.
- `cti_hunting_osint_jobs`: `lambda:InvokeFunction` para `cti_hunting_osint_orchestrator`.
- `cti_hunting_reports`: `lambda:InvokeFunction` para `cti_hunting_report`.
- `cti_hunting_assets`: `scheduler:CreateSchedule`, `scheduler:UpdateSchedule`, `scheduler:DeleteSchedule` e `iam:PassRole` para a role do Scheduler.

### Orquestradora

- Leitura/escrita em `cti_hunting_jobs`, `cti_hunting_scans`, `cti_hunting_assets`, `cti_hunting_safelist`, `cti_hunting_config`.
- Leitura/escrita nas tabelas `cti_hunting_collect_*`.
- `lambda:InvokeFunction` nos collectors.
- `scheduler:CreateSchedule`, `scheduler:UpdateSchedule` e `iam:PassRole` para reagendar assets.

### Collectors

- `secretsmanager:GetSecretValue` na secret configurada.
- Leitura/escrita somente na própria tabela Dynamo.
- `lambda:InvokeFunction` para `cti_hunting_osint_orchestrator`.

### Report

- Leitura em `cti_hunting_scans`.
- Leitura/escrita em `cti_hunting_jobs`.
- `ses:SendRawEmail`.

## Upload pela interface AWS

1. Crie ou abra cada Lambda.
2. Runtime: Python 3.12.
3. Handler: `lambda_function.handler`.
4. Faça upload do ZIP correspondente.
5. Anexe a Layer `cti_hunting_python_layer_collector_tables.zip` nas Lambdas que usam bibliotecas externas.
6. Configure as variáveis de ambiente.
7. Configure as rotas no HTTP API.
8. Configure CORS no HTTP API.
9. Publique o front-end no bucket S3 usado como origem do CloudFront.
10. Crie o comportamento `/api/*` no CloudFront apontando para o API Gateway.
