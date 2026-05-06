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

O fluxo (PR → staging → produção):

1. **PR aberta** → `pr-checks.yml` roda `docker build` (sem push) pra validar o Dockerfile antes do merge. Não deploya nada.
2. **Merge em `main`** → `build_and_deploy.yaml` builda imagem, pusha pra ECR e bumpa `image.tag` em commit `[skip ci]`. Pra `target_env ∈ {workspace, payments}` o bump é em `chart/values-k8s-staging.yaml` (CD contínuo de staging); pra `target_env ∈ {staging, shared-services}` o bump é direto no values primário.
3. **ArgoCD staging** sincroniza e deploya no cluster `k8s-staging`.
4. **Quando staging está estável**, dispara o workflow `Release to production` (em **Actions → Release to production → Run workflow**). Ele lê a `image.tag` atual de staging, bumpa o values primário (`chart/values-k8s-${target_env}.yaml`) e force-pusha branch `release`.
5. **ArgoCD prod** segue o branch `release` (via `targetRevision` condicional na ApplicationSet) e deploya em workspace/payments.

> ℹ️ Apps com `target_env ∈ {staging, shared-services}` **não têm release flow** — o bootstrap deleta o `release.yml` automaticamente, e o `targetRevision` da ApplicationSet fica `main` puro.

> ⚠️ Pra rollback de prod: dispare `Release to production` passando o `ref` do commit anterior (input do workflow_dispatch). Fast-forward backwards no branch `release`.

### Setup automatizado pelo bootstrap

Tudo que o `bootstrap.yml` configura automaticamente no repo da app:

- **`.github/CODEOWNERS`** com regra `chart/values-k8s-*.yaml @pagamericantech/infrastructure @pagamericantech/engineering` — protege todos os values do chart, que são mantidos pelos workflows.
- Pra apps com `target_env ∈ {workspace, payments}`:
  - Cria branch `release` apontando pro main HEAD.
  - Tenta criar ruleset "Protect release branch" (bloqueia `deletion` + `non_fast_forward`, bypass pro app `pagamerican-gitops`). Se o token não tiver `Administration:Write`, emite warning — nesse caso configure manualmente em **Settings → Rules → Rulesets**.

`release.yml` autentica via `pagamerican-gitops` (secrets `ORG_GITOPS_BOT_*`), não via `GITHUB_TOKEN` default — é o único actor com bypass do ruleset.

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
