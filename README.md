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

- **Grafana:** http://localhost:3000/grafana/ — usuário `admin`, senha `atlas`
- **Prometheus:** http://localhost:9090 (aba *Status → Targets* para ver o scrape)

Em produção o Grafana é exposto pelo gateway nginx em **`/grafana/`**
(ex.: `http://144.126.151.140/grafana/`) — ver seção *Deploy*. O `docker-compose`
já configura o Grafana em sub-path (`GF_SERVER_SERVE_FROM_SUB_PATH`), então o
mesmo caminho vale no acesso direto pela porta 3000.

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
├── .github/workflows/deploy.yml          # deploy SSH para produção
├── prometheus/
│   └── prometheus.yml                    # jobs de scrape dos serviços
└── grafana/
    └── provisioning/
        ├── datasources/prometheus.yml    # datasource Prometheus (uid fixo)
        └── dashboards/
            ├── dashboards.yml            # provider de dashboards
            └── atlas-overview.json       # dashboard genérico
```

## Deploy (produção)

Deploy automático via GitHub Actions (`.github/workflows/deploy.yml`) a cada
push na `main`, ou manualmente em *Actions → Deploy observability → Run*.

O workflow entra por SSH no servidor e:

1. Confere se a rede `atlas-network` existe (criada pelo stack principal — ele
   precisa estar no ar);
2. Clona o repo em **`/home/production/observability`** (ou atualiza para
   `origin/main` se já existir);
3. Sobe o stack com `docker compose up -d`.

O Grafana fica acessível pelo gateway nginx do `atlas-infra` em **`/grafana/`**
(a rota `location /grafana/` já está no `nginx/nginx.conf` do atlas-infra,
apontando para `atlas-grafana:3000`).

### Secrets necessários (no repositório)

Os mesmos dos outros repositórios Atlas — *Settings → Secrets and variables →
Actions*:

| Secret         | Valor                     |
|----------------|---------------------------|
| `SSH_HOST`     | IP do servidor            |
| `SSH_USER`     | usuário SSH               |
| `SSH_PASSWORD` | senha SSH                 |

> **Segurança:** o Grafana sobe com `admin`/`atlas`. Troque a senha em produção
> (via UI ou `GF_SECURITY_ADMIN_PASSWORD` antes do 1º boot) já que a rota
> `/grafana/` é pública pelo gateway.

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
