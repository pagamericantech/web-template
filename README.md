# web-template

Template repository para apps **HTTP/web** da plataforma. Espelha o padrão usado em
[`landscape`](https://github.com/pagamericantech/landscape): single-image buildada
via workflow reutilizável de [`platform-workflows`](https://github.com/pagamericantech/platform-workflows)
e deploy via [`platform-k8s-web-chart`](https://github.com/pagamericantech/platform-charts/tree/main/platform-k8s-web-chart).

Para outros tipos de workload, use o template correspondente:

- **`web-template`** (este) — apps HTTP (Deployment + HTTPRoute + Argo Rollouts canary).
- [`worker-template`](https://github.com/pagamericantech/worker-template) — consumers/workers sem entrada HTTP (kafka/sqs/rabbitmq).
- [`cronjob-template`](https://github.com/pagamericantech/cronjob-template) — jobs agendados.

## Estrutura

```
.
├── .github/workflows/build_and_deploy.yaml   # chama _reusable-docker-build-push.yml
├── chart/
│   └── values-k8s-shared-services.yaml       # override do platform-k8s-web-chart
├── Dockerfile                                # build do app (placeholder)
├── .dockerignore
└── .gitignore
```

O fluxo é:

1. Push em `main` → workflow builda imagem, pusha pra ECR (cria o repo se não existir).
2. Workflow atualiza `chart/values-k8s-shared-services.yaml` com a nova `image.tag` em commit `[skip ci]`.
3. ArgoCD detecta a mudança no values (multi-source ApplicationSet em `platform-applications`) e faz o deploy via `platform-k8s-web-chart`.

## Como criar um novo app a partir desse template

```bash
gh repo create pagamericantech/<app-name> \
  --template pagamericantech/web-template \
  --private \
  --clone

cd <app-name>
```

Em seguida, substitua os placeholders no repo recém-clonado:

| Placeholder        | Onde aparece                                                | Exemplo                          |
| ------------------ | ----------------------------------------------------------- | -------------------------------- |
| `__APP_NAME__`     | `chart/values-k8s-shared-services.yaml`                     | `risk-dashboard`                 |
| `__IMAGE_NAME__`   | `.github/workflows/build_and_deploy.yaml` + `chart/values…` | `pga-risk-dashboard`             |
| `__OWNER__`        | labels em `chart/values…`                                   | `risk-engineering`               |
| `__LANGUAGE__`     | `application.language` em `chart/values…`                   | `go`, `node`, `python`, `static` |

Substituição em massa (macOS/BSD sed):

```bash
APP=meu-app
IMAGE=pga-meu-app
OWNER=meu-time
LANG=go

find . -type f \( -name '*.yaml' -o -name '*.yml' \) -not -path './.git/*' -print0 | \
  xargs -0 sed -i '' \
    -e "s/__APP_NAME__/${APP}/g" \
    -e "s/__IMAGE_NAME__/${IMAGE}/g" \
    -e "s/__OWNER__/${OWNER}/g" \
    -e "s/__LANGUAGE__/${LANG}/g"
```

Depois, ajuste `chart/values-k8s-shared-services.yaml` conforme necessário:

- `httpRoute.hostnames` — hostname público/interno do app (default usa `<app>.k8s-shared-services.pagamerican.tech`).
- `httpRoute.parentRefs[].name` — `internal` (default) ou `external` se for hostname público em `*.pagamerican.app`.
- `containerPort` / `healthcheck.*.path` — porta/paths do seu app.
- `autoscaling` — KEDA com cpu/memory por default; ver `values.yaml` do chart pra triggers extras (kafka/sqs/rabbitmq/datadog).

## Marcar como template no GitHub

Após criar o repo `web-template` no GitHub (público), ative o flag de template:

```bash
gh repo edit pagamericantech/web-template --template
```

Isso habilita o botão "Use this template" e permite o `gh repo create --template`.
