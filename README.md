# Atlas Observability

Stack de observabilidade do projeto **Atlas** — Prometheus + Grafana em Docker.

Coleta métricas dos microsserviços Atlas (auth, feed, tracks, scholarship,
notification via [`django-prometheus`](https://github.com/korfuri/django-prometheus)
e ai via [`prometheus-fastapi-instrumentator`](https://github.com/trallnag/prometheus-fastapi-instrumentator))
e as apresenta num dashboard genérico de visão geral.

## Componentes

| Serviço      | Imagem                     | Porta (host) | Função                                  |
|--------------|----------------------------|--------------|-----------------------------------------|
| `prometheus` | `prom/prometheus:v2.55.1`  | `9090`       | Coleta e armazena as métricas (TSDB)    |
| `grafana`    | `grafana/grafana:11.3.0`   | `3000`       | Dashboards. Login: **admin / atlas**    |

## Pré-requisitos

Este stack **anexa-se à rede `atlas-network`**, criada pelo stack principal
(`atlas-infra`). Suba o Atlas primeiro (`docker compose up -d` no `atlas-infra`)
para que a rede exista e os containers dos serviços estejam no ar.

O Prometheus raspa cada serviço pelo nome do container na rede interna
(`auth-service:8000`, `feed-service:8000`, ... e `ai-service:8003`) — nada é
exposto pelo gateway nginx.

## Uso

```bash
# a partir da raiz deste repositório
docker compose up -d
```

- **Grafana:** http://localhost:3000 — usuário `admin`, senha `atlas`
- **Prometheus:** http://localhost:9090 (aba *Status → Targets* para ver o scrape)

O datasource Prometheus e o dashboard **"Atlas — Visão Geral dos Serviços"**
são provisionados automaticamente no boot (não precisa importar nada).

### Credenciais

Definidas no `docker-compose.yml` (`admin` / `atlas`). Para trocar, copie
`.env.example` para `.env` e ajuste `GF_SECURITY_ADMIN_USER` /
`GF_SECURITY_ADMIN_PASSWORD` antes do **primeiro** boot (depois disso a senha
já foi gravada no volume `grafana_data`; use a UI ou `grafana-cli` para mudar).

## Dashboard

O painel **Atlas — Visão Geral** traz, com um seletor de `$service`:

- **Visão geral:** serviços no ar, requisições/s, erros 5xx/s, latência p95 global
- **HTTP:** req/s por serviço, latência p50/p95, respostas por status, erros 4xx/5xx
- **Recursos:** memória residente e CPU por processo

## Estrutura

```
.
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml                    # jobs de scrape dos serviços
└── grafana/
    └── provisioning/
        ├── datasources/prometheus.yml    # datasource Prometheus (uid fixo)
        └── dashboards/
            ├── dashboards.yml            # provider de dashboards
            └── atlas-overview.json       # dashboard genérico
```

## Integração nos serviços

Cada serviço Django expõe `/metrics` via `django-prometheus`:

```python
# settings/base.py
INSTALLED_APPS = [..., "django_prometheus"]
MIDDLEWARE = [
    "django_prometheus.middleware.PrometheusBeforeMiddleware",  # primeiro
    ...
    "django_prometheus.middleware.PrometheusAfterMiddleware",   # último
]

# config/urls.py
urlpatterns += [path("", include("django_prometheus.urls"))]  # /metrics
```

O `ai-service` (FastAPI) usa `prometheus-fastapi-instrumentator`, que expõe
`/metrics` no mesmo padrão.
