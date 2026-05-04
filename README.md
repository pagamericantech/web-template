# app-template

Template repository para novos apps da plataforma. Espelha o padrão usado em
[`landscape`](https://github.com/pagamericantech/landscape): single-image
buildada via workflow reutilizável de [`platform-workflows`](https://github.com/pagamericantech/platform-workflows)
e deploy via chart compartilhado de
[`platform-charts`](https://github.com/pagamericantech/platform-charts).

## Estrutura

```
.
├── .github/workflows/build_and_deploy.yaml   # chama _reusable-docker-build-push.yml
├── chart/
│   └── values-k8s-shared-services.yaml       # override do chart compartilhado
├── Dockerfile                                # build do app (placeholder)
├── .dockerignore
└── .gitignore
```

O fluxo é:

1. Push em `main` → workflow builda imagem, pusha pra ECR (cria o repo se não existir).
2. Workflow atualiza `chart/values-k8s-shared-services.yaml` com a nova `image.tag` em commit `[skip ci]`.
3. ArgoCD (configurado em `platform-addons`) detecta a mudança no values e faz o deploy via chart de `platform-charts`.

## Como criar um novo app a partir desse template

```bash
gh repo create pagamericantech/<app-name> \
  --template pagamericantech/app-template \
  --private \
  --clone

cd <app-name>
```

Em seguida, substitua os placeholders no repo recém-clonado:

| Placeholder        | Onde aparece                                                | Exemplo                          |
| ------------------ | ----------------------------------------------------------- | -------------------------------- |
| `__APP_NAME__`     | `chart/values-k8s-shared-services.yaml`                     | `risk-dashboard`                 |
| `__IMAGE_NAME__`   | `.github/workflows/build_and_deploy.yaml` + `chart/values…` | `pga-risk-dashboard`             |
| `__CHART_NAME__`   | comentário no topo de `chart/values…`                       | `platform-k8s-web-chart`         |
| `__OWNER__`        | labels em `chart/values…`                                   | `risk-engineering`               |
| `__LANGUAGE__`     | `application.language` em `chart/values…`                   | `go`, `node`, `python`, `static` |

Substituição em massa (macOS/BSD sed):

```bash
APP=meu-app
IMAGE=pga-meu-app
CHART=platform-k8s-web-chart
OWNER=meu-time
LANG=go

find . -type f \( -name '*.yaml' -o -name '*.yml' \) -not -path './.git/*' -print0 | \
  xargs -0 sed -i '' \
    -e "s/__APP_NAME__/${APP}/g" \
    -e "s/__IMAGE_NAME__/${IMAGE}/g" \
    -e "s/__CHART_NAME__/${CHART}/g" \
    -e "s/__OWNER__/${OWNER}/g" \
    -e "s/__LANGUAGE__/${LANG}/g"
```

## Charts disponíveis em `platform-charts`

Escolha o que casa com o tipo do app e ajuste `__CHART_NAME__` + `chart/values-k8s-shared-services.yaml`:

- [`platform-k8s-web-chart`](https://github.com/pagamericantech/platform-charts/tree/main/platform-k8s-web-chart) — apps HTTP (Deployment + HTTPRoute + Rollouts canary)
- [`platform-k8s-worker-chart`](https://github.com/pagamericantech/platform-charts/tree/main/platform-k8s-worker-chart) — workers/consumers sem entrada HTTP
- [`platform-k8s-cronjob-chart`](https://github.com/pagamericantech/platform-charts/tree/main/platform-k8s-cronjob-chart) — jobs agendados

Os values aqui são um esqueleto inspirado em `landscape` (web). Para worker/cronjob,
ajuste removendo `httpRoute`/`healthcheck`/`containerPort` e adicionando os campos
específicos do chart escolhido — consulte o `values.yaml` do chart correspondente.

## Marcar como template no GitHub

Após criar o repo `app-template` no GitHub, ative o flag de template:

```bash
gh repo edit pagamericantech/app-template --template
```

Isso habilita o botão "Use this template" e permite o `gh repo create --template`.
