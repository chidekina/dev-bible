# GitHub Actions & CI/CD

## Visão Geral

CI/CD (Continuous Integration / Continuous Delivery) é a prática de automaticamente fazer build, testar e fazer deploy do código a cada mudança. Elimina a cerimônia do "dia de deploy", detecta regressões imediatamente e dá às equipes a confiança para entregar dezenas de vezes por dia.

**Continuous Integration (CI):** Cada push aciona testes automatizados. A branch main está sempre em estado deployável.

**Continuous Delivery (CD):** Cada build aprovado é automaticamente deployado para staging. O deploy para produção é um clique.

**Continuous Deployment:** Cada build aprovado é automaticamente deployado para produção — sem intervenção humana.

GitHub Actions é a plataforma de CI/CD integrada ao GitHub. Os workflows ficam como arquivos YAML em `.github/workflows/`, rodam na infraestrutura do GitHub e se integram nativamente com pull requests, branches e environments.

---

## Pré-requisitos

- Um repositório no GitHub
- Entendimento básico de Docker (para builds containerizados)
- Projeto Node.js/TypeScript com uma suite de testes

---

## Conceitos Fundamentais

### Anatomia de um Workflow

```yaml
name: CI                          # nome exibido na UI do GitHub

on:                               # gatilhos
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:                             # unidades de trabalho paralelas
  test:                           # ID do job
    runs-on: ubuntu-latest        # ambiente do runner
    steps:                        # steps sequenciais dentro de um job
      - uses: actions/checkout@v4 # fazer checkout do código
      - run: npm test             # comando shell
```

### Blocos de Construção Principais

| Conceito | Descrição |
|---------|-----------|
| **Workflow** | Arquivo YAML em `.github/workflows/` |
| **Gatilho** (`on`) | O que inicia o workflow: push, PR, schedule, manual |
| **Job** | Conjunto de steps que rodam no mesmo runner |
| **Step** | Uma tarefa única — seja uma action `uses` ou um comando `run` |
| **Action** | Unidade reutilizável de trabalho (`actions/checkout`, `docker/login-action`) |
| **Runner** | A VM ou container que executa os jobs |
| **Environment** | Alvo de deploy nomeado com regras de proteção e secrets |
| **Secret** | Valor encriptado armazenado nas configurações do repositório/environment |
| **Matrix** | Rodar o mesmo job com múltiplas combinações de parâmetros |

### Expressões e Contexto

```yaml
# Acessar objetos de contexto
${{ github.sha }}          # SHA do commit
${{ github.ref_name }}     # nome da branch ou tag
${{ github.actor }}        # usuário que acionou o workflow
${{ secrets.MY_SECRET }}   # secret do repositório
${{ vars.MY_VAR }}         # variável do repositório (não-secret)
${{ env.MY_ENV }}          # variável de ambiente definida no workflow

# Expressões condicionais
if: github.ref == 'refs/heads/main'
if: github.event_name == 'pull_request'
if: failure()              # rodar apenas se o step anterior falhou
if: always()               # rodar independente do resultado anterior
```

---

## Exemplos Práticos

### Exemplo 1: Pipeline de CI para Node.js

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Test (Node ${{ matrix.node-version }})
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [20, 22]

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: test
          POSTGRES_DB: app_test
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
        ports:
          - 5432:5432

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run type-check

      - name: Lint
        run: npm run lint

      - name: Unit tests
        run: npm run test:unit

      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgres://app:test@localhost:5432/app_test

      - name: Build
        run: npm run build
```

### Exemplo 2: Build e Push de Docker

```yaml
name: Build and Push

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          target: production
```

### Exemplo 3: CI/CD Completo com Deploy

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─── Job 1: Testes ──────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: test
          POSTGRES_DB: app_test
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - run: npm run test
        env:
          DATABASE_URL: postgres://app:test@localhost:5432/app_test

  # ─── Job 2: Build da Imagem Docker ──────────────────────────────
  build:
    name: Build Image
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ─── Job 3: Deploy para VPS ─────────────────────────────────────
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://myapp.com

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/myapp
            echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker compose pull
            docker compose up -d --remove-orphans
            docker system prune -f
```

### Exemplo 4: Jobs Agendados e Cron

```yaml
name: Maintenance

on:
  schedule:
    - cron: '0 2 * * *'   # diariamente às 2 AM UTC
  workflow_dispatch:        # botão de acionamento manual na UI do GitHub

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Delete old packages
        uses: actions/delete-package-versions@v5
        with:
          package-name: myapp
          package-type: container
          min-versions-to-keep: 10
          delete-only-untagged-versions: true

  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Backup database
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            pg_dump $DATABASE_URL | gzip > /backups/$(date +%Y%m%d).sql.gz
            find /backups -mtime +30 -delete
```

### Exemplo 5: Workflow de Release com Versionamento Semântico

```yaml
name: Release

on:
  push:
    tags: ['v*.*.*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # histórico completo para changelog

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:latest

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true  # gera automaticamente a partir de PRs/commits
          files: |
            dist/*.tar.gz
```

---

## Padrões e Boas Práticas

### Cache de Dependências

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 22
    cache: npm           # cacheia ~/.npm

# Ou manualmente:
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: npm-
```

### Workflows Reutilizáveis

Defina uma vez, chame de múltiplos workflows:

`.github/workflows/reusable-test.yml`:
```yaml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '22'
    secrets:
      DATABASE_URL:
        required: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - run: npm ci && npm test
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

Chamador:
```yaml
jobs:
  test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '22'
    secrets:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### Controle de Concorrência

Evitar múltiplos deploys rodando simultaneamente:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true   # cancelar execução mais antiga quando nova chega
```

### Filtros por Caminho — Rodar Apenas Quando Arquivos Relevantes Mudam

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package*.json'
      - 'Dockerfile'
      - '.github/workflows/**'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

### Regras de Proteção de Environment

Em GitHub Settings > Environments, configure:
- Revisores obrigatórios antes do deploy
- Branches de deploy (apenas `main` pode deployar para produção)
- Timer de espera (delay antes do auto-deploy)

```yaml
deploy:
  environment:
    name: production
    url: https://myapp.com
  # GitHub exigirá aprovação dos revisores configurados
```

### Estratégia de Matrix para Testes Cross-Platform

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [20, 22]
  fail-fast: false   # continuar outros jobs da matrix se um falhar

runs-on: ${{ matrix.os }}
```

---

## Anti-padrões a Evitar

### Hardcoding de Secrets

```yaml
# RUIM
run: docker login -u myuser -p mysecretpassword

# BOM
run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u ${{ secrets.REGISTRY_USER }} --password-stdin
```

### Não Fixar Versões de Actions

```yaml
# RUIM — pode quebrar se o autor da action fizer push de mudanças quebrando para @main
uses: actions/checkout@main

# RUIM — a tag v4 pode apontar para um commit diferente
uses: actions/checkout@v4

# MELHOR — fixar no SHA exato do commit por segurança
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

### Jobs Monolíticos Gigantes

```yaml
# RUIM — se o deploy falhar, não dá para re-rodar apenas o deploy
jobs:
  all:
    steps:
      - run: npm test
      - run: docker build
      - run: deploy

# BOM — jobs separados com needs
jobs:
  test: ...
  build:
    needs: test
  deploy:
    needs: build
```

### Upload de Dados Sensíveis para Artifacts

```yaml
# RUIM — arquivos .env podem conter secrets
- uses: actions/upload-artifact@v4
  with:
    path: .    # faz upload de tudo incluindo .env
```

### Rodar Jobs Longos sem Timeout

```yaml
jobs:
  test:
    timeout-minutes: 15   # falhar se os testes levarem mais de 15 minutos
```

---

## Debug & Troubleshooting

### Habilitar Log de Debug

Em Repository Settings > Secrets, adicione:
```
ACTIONS_STEP_DEBUG = true
ACTIONS_RUNNER_DEBUG = true
```

### Sessão de Debug SSH

Adicione um step para abrir um túnel SSH temporário dentro de um runner com falha:

```yaml
- name: Debug with tmate
  if: failure()
  uses: mxschmitt/action-tmate@v3
  timeout-minutes: 15
```

### Inspecionar Execução do Workflow

```bash
# GitHub CLI
gh run list                         # listar execuções recentes
gh run view 1234567890              # ver detalhes da execução
gh run view 1234567890 --log        # logs completos
gh run view 1234567890 --log-failed # apenas logs de steps com falha
gh run rerun 1234567890             # re-executar todos os jobs
gh run rerun 1234567890 --failed    # re-executar apenas jobs com falha
```

### Falhas Comuns

```
Error: Process completed with exit code 1.
```
Verifique a saída do step. O código de saída vem do comando que você executou.

```
Error: Resource not accessible by integration
```
Bloco `permissions` ausente. Adicione as permissões necessárias ao job.

```
Error: No space left on device
```
Disco do runner está cheio. Adicione `docker system prune -f` entre steps pesados.

```
Error: Context access might be invalid: secrets.MY_SECRET
```
Secret não definido nas configurações do repositório, ou environment errado.

---

## Cenários do Mundo Real

### Cenário 1: Deploy de Preview em PRs

```yaml
name: Preview Deploy

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy PR preview
        id: deploy
        run: |
          PREVIEW_URL=$(./scripts/deploy-preview.sh ${{ github.event.number }})
          echo "url=$PREVIEW_URL" >> $GITHUB_OUTPUT

      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `Preview deployado: ${{ steps.deploy.outputs.url }}`
            })
```

### Cenário 2: Atualização Automática de Dependências com Testes

Use `dependabot.yml` + este workflow para auto-merge de PRs do Dependabot que passem nos testes:

```yaml
name: Auto-merge Dependabot

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  auto-merge:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: write
    steps:
      - name: Auto-merge minor/patch updates
        uses: actions/github-script@v7
        with:
          script: |
            const { data: pr } = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });
            const isMinor = pr.title.match(/bump .* from .* to .*/i);
            if (isMinor) {
              await github.rest.pulls.merge({
                owner: context.repo.owner,
                repo: context.repo.repo,
                pull_number: context.issue.number,
                merge_method: 'squash'
              });
            }
```

### Cenário 3: Build e Publicação de Pacote NPM

```yaml
name: Publish

on:
  push:
    tags: ['v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          registry-url: https://registry.npmjs.org

      - run: npm ci
      - run: npm test
      - run: npm run build

      - name: Publish to NPM
        run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## Leitura Complementar

- [Documentação do GitHub Actions](https://docs.github.com/en/actions)
- [Referência de sintaxe de workflow](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Hardening de segurança para GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [act — rodar Actions localmente](https://github.com/nektos/act)

---

## Resumo

| Conceito | Ponto Principal |
|---------|-----------------|
| Gatilhos de workflow | `push`, `pull_request`, `schedule`, `workflow_dispatch` |
| Jobs | Rodam em paralelo por padrão; use `needs` para ordenar |
| Cache | Sempre cachear `node_modules` ou `~/.npm` — economiza 1-3 min por execução |
| Secrets | Armazene no GitHub Settings, referencie como `${{ secrets.NOME }}` |
| Environments | Alvos de deploy nomeados com regras de proteção |
| Matrix | Testar contra múltiplas versões do Node ou SOs em uma config |
| Workflows reutilizáveis | DRY — defina uma vez, chame de múltiplos workflows |
| Concorrência | Evite deploys simultâneos com grupo `concurrency` |
| Fixar actions | Fixe no SHA do commit por segurança, não apenas em tags de versão |
| Debug | `ACTIONS_STEP_DEBUG=true` + tmate para acesso SSH ao vivo |

CI/CD não é luxo — é a fundação da entrega confiável de software. Um badge de CI passando significa que o código funciona. Um pipeline de CD automatizado significa que o deploy deixou de ser um evento.
