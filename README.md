# Atlas · Observability 📊

> Parte do **Projeto Atlas** — plataforma acadêmica desenvolvida para o **IFRN Campus Pau dos Ferros** como Projeto Integrador de Sistemas Distribuídos. O Atlas conecta alunos a trilhas de conhecimento e bolsas, com avaliação automática de código por IA.

Stack de **observabilidade** do Atlas: coleta de métricas com **Prometheus** e visualização com **Grafana** (dashboards e datasource provisionados automaticamente). Roda como uma stack separada que enxerga os serviços pela rede do projeto.

## O que este repositório contém

- **`prometheus/prometheus.yml`** — configuração de scrape das métricas de cada serviço.
- **`grafana/provisioning/`** — datasource (Prometheus) e dashboards provisionados (ex.: `atlas-overview`).
- **`docker-compose.yml`** — sobe Prometheus (`v2.55.1`) e Grafana (`11.3.0`).

## Como funciona

Cada serviço do Atlas expõe um endpoint `/metrics`:
- Serviços **Django** via `django-prometheus`.
- **ai-service** (FastAPI) via `prometheus-fastapi-instrumentator`.

O Prometheus faz *scrape* a cada 15s dos alvos:

| Job | Alvo |
|---|---|
| auth | `auth-service:8000` |
| feed | `feed-service:8000` |
| tracks | `tracks-service:8000` |
| scholarship | `scholarship-service:8000` |
| notification | `notification-service:8000` |
| ai | `ai-service:8003` |

O Grafana consome o Prometheus como datasource e exibe os dashboards provisionados. Combinado com `pg_stat_statements` / `pg_stat_activity` no PostgreSQL, cobre o monitoramento exigido pelo guia do projeto.

## Ecossistema

| Repositório | Responsabilidade |
|---|---|
| atlas-auth-service | Identidade: SUAP OAuth2, JWT, perfis de usuário |
| atlas-track-service | Trilhas, módulos, conteúdos, progresso e submissão de desafios |
| atlas-scholarship-service | Bolsas, candidaturas, banco de talentos e notas |
| atlas-feed-service | Feed institucional: posts, comentários, curtidas e banners |
| atlas-notification-service | Notificações (consumidor central via RabbitMQ) |
| atlas-ai-service | Avaliação de repositórios GitHub por LLM local (Ollama) |
| atlas-frontend | SPA React + TypeScript (aluno e professor) |
| atlas-infra | Docker Compose, Nginx (gateway), Postgres/Redis/RabbitMQ, deploy e backup |
| **atlas-observability** | **Prometheus + Grafana (métricas dos serviços)** |

## Executando

```bash
cp .env.example .env      # ex.: credenciais/admin do Grafana
docker compose up -d
# Prometheus e Grafana ficam disponíveis nas portas configuradas no compose
```

> Requer que os serviços do Atlas estejam na mesma rede Docker (`atlas-network`), definida pelo [atlas-infra](https://github.com/Atlas-IFRN/atlas-infra).

## Variáveis de ambiente

Baseie seu `.env` no `.env.example` (ex.: usuário/senha admin do Grafana).

## CI/CD

Workflow de deploy em `.github/workflows/deploy.yml`.
