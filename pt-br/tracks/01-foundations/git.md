# Git

> Git não é apenas um sistema de backup. É um grafo acíclico dirigido de snapshots, um protocolo de colaboração e uma ferramenta de debug. Entender seu modelo de objetos e estratégias de branching transforma você de alguém que memoriza comandos em alguém que raciocina sobre controle de versão.

---

## 1. O que é e por que importa

Git é o sistema de controle de versão mais usado do mundo, criado por Linus Torvalds em 2005 para gerenciar o kernel Linux. Diferente de sistemas mais antigos (SVN, CVS) que rastreavam diferenças entre versões de arquivos, o Git armazena **snapshots completos** de todo o repositório a cada commit.

Entender o Git profundamente permite:
- Recuperar de erros com confiança (reflog, reset, cherry-pick)
- Debugar regressões eficientemente (bisect)
- Colaborar sem pisar nas mudanças uns dos outros
- Manter um histórico limpo e navegável (rebase interativo, conventional commits)
- Impor gates de qualidade automaticamente (hooks)
- Escolher a estratégia de branching certa para sua equipe

---

## 2. Conceitos Fundamentais

### O Modelo de Objetos do Git (DAG)

Git armazena tudo como **objetos** em `.git/objects/`. Todo objeto é identificado por um hash SHA-1 (ou SHA-256 em versões mais novas) do seu conteúdo. Há quatro tipos de objeto:

```
blob     — o conteúdo bruto de um único arquivo
tree     — uma listagem de diretório (mapeia nomes de arquivo para SHAs de blob/tree)
commit   — um snapshot apontando para uma tree + commit(s) pai(s) + metadados
tag      — um ponteiro nomeado para um commit específico (tags anotadas)
```

Um commit se parece com isso internamente:
```
commit abc1234
  tree  def5678        ← aponta para o snapshot da tree raiz
  parent bbb9876       ← commit anterior (nenhum para o primeiro commit)
  author Alice <alice@example.com> 1704067200 +0000
  committer Alice <alice@example.com> 1704067200 +0000

  Add user authentication
```

Cada objeto tree:
```
tree def5678
  blob aaa1111  README.md
  blob bbb2222  package.json
  tree ccc3333  src/         ← tree aninhada para o diretório src/
```

**Insight chave:** Commits são **snapshots imutáveis**, não diffs. Quando você muda um arquivo e faz commit, o Git cria uma nova tree com um novo blob para o arquivo alterado e reutiliza blobs para arquivos inalterados (via deduplicação baseada em SHA). É isso que torna o Git rápido: fazer checkout de um branch é apenas mudar qual SHA de tree está ativo.

**Branches e HEAD são apenas ponteiros:**
```
main       → abc1234 (SHA do commit)
feature/x  → def5678 (SHA de commit diferente)
HEAD       → main    (HEAD aponta para o branch atual)
           → abc1234 (quando detached)
```

Branches são armazenados como arquivos em `.git/refs/heads/`. Um branch é literalmente um arquivo de 41 bytes contendo um SHA de commit. Criar um branch é O(1) — quase gratuito.

---

### As Três Áreas

```
Working Tree           Staging Area (Index)        Repositório (.git/)
─────────────────      ──────────────────────      ──────────────────
Seus arquivos          O que vai entrar             Snapshots commitados
reais enquanto         no próximo commit            (histórico permanente)
você edita

git add ──────────────────────────>
                 git commit ──────────────────────>
git checkout ───────────────────────────────────────────────────<
```

Entender esse modelo explica por que:
- `git diff` mostra working tree vs staging area
- `git diff --staged` mostra staging area vs último commit
- `git reset HEAD~1` move o HEAD sem tocar na working tree (por padrão)
- `git stash` salva working tree + staging area e restaura ambos para o último commit

---

## 3. Como Funciona

### Estratégias de Branching

**Trunk-Based Development (TBD):**
```
main: A──B──C──D──E──F──G──H (todos commitam aqui, sem branches de longa duração)
               ↑         ↑
        feature flag  feature flag
            on             off

Características:
- Todos integram para main ao menos uma vez por dia
- Features incompletas ocultas atrás de feature flags
- Deploy contínuo — todo commit em main é deployável
- Requer excelente cobertura de testes e CI
- Melhor para: equipes experientes, microsserviços, alta frequência de deploy
```

**GitHub Flow:**
```
main:      A─────────────────G──── (sempre deployável)
                   ↑         ↑
feature/x: B──C──D──E  (curta duração, PR para main, depois deletado)
                   ↑
              Code review, verificações de CI, depois merge

Características:
- Feature branches vivem no máximo 1-3 dias
- PR obrigatório para fazer merge em main
- Simples: dois estados "permanentes" — working e deployed
- Melhor para: web apps, equipes pequenas/médias, deploys frequentes
```

**Git Flow:**
```
main:     A─────────────────────────E── (apenas releases de produção)
           \                       /
develop:   B──C──D──F──G──H──I──J──

           ↑ feature branches ↑    ↑ release branch ↑
feature/x: C──D (branch do develop, merge de volta para develop)
release/1.2:   H──I (branch do develop, merge para main + develop)
hotfix/bug:         K──L (branch do main, merge para main + develop)

Características:
- Pesado — muitos tipos de branch com regras estritas
- Bom para: software versionado com ciclos de release (apps mobile, bibliotecas)
- Excessivo para a maioria das aplicações web
```

> **Escolha baseado na frequência de deploy.** Se você faz deploy múltiplas vezes por dia, TBD ou GitHub Flow. Se tem releases trimestrais com números de versão, Git Flow.

---

### Rebase vs Merge

**Merge** — preserva histórico completo, cria um merge commit:
```
Antes:
main:    A──B──C
              \
feature:       D──E

Depois de git merge feature (em main):
main:    A──B──C──────F  ← merge commit, dois pais (C e E)
              \      /
feature:       D──E
```

**Rebase** — replaya commits em cima do alvo, cria histórico linear:
```
Antes:
main:    A──B──C
              \
feature:       D──E (D,E baseados em B — antes de C ser adicionado)

Depois de git rebase main (em feature):
main:    A──B──C
                 \
feature:          D'──E' (D' e E' são commits NOVOS com novos SHAs)
```

Depois o merge fast-forward dá histórico linear limpo:
```
Depois de git merge --ff-only feature:
main:    A──B──C──D'──E'
```

**Quando fazer rebase:**
- Antes de fazer merge de um feature branch para main — limpe os commits, coloque-os em cima do main mais recente
- Para atualizar um branch de longa duração com mudanças do main: `git rebase main`
- Rebase interativo para squash de commits WIP antes do PR

> **Nunca faça rebase de commits que foram enviados para um branch compartilhado.** Rebase cria novos commits (novos SHAs). Se seus colegas têm os commits antigos, o histórico deles diverge do seu e a reconciliação é dolorosa. A regra de ouro: só faça rebase de commits que existem apenas na sua máquina local (ou em branches que só você usa).

---

### Rebase Interativo

`git rebase -i HEAD~N` abre um editor para manipular os últimos N commits:

```
pick abc1234 Add user model
pick def5678 Fix typo in user model
pick ghi9012 Add user controller
pick jkl3456 WIP: feature pela metade
pick mno7890 Termina a feature
```

Comandos disponíveis:
```
pick    — usa o commit como está
reword  — usa o commit mas edita a mensagem
squash  — mescla com o commit anterior (combina mensagens)
fixup   — mescla com o commit anterior (descarta a mensagem deste commit)
edit    — pausa o rebase aqui, permite fazer amend no commit
drop    — remove este commit
reorder — arrasta linhas para reordenar commits
```

Exemplo — limpar antes do PR:
```
pick abc1234 Add user model
fixup def5678 Fix typo in user model     ← dobrado em abc1234 silenciosamente
pick ghi9012 Add user controller
squash jkl3456 WIP: feature pela metade  ← combinado com mno7890
squash mno7890 Termina a feature        ← editor pede mensagem combinada
```

Resultado: um histórico limpo de dois commits pronto para revisão no PR.

---

### Git Bisect — Busca Binária por Bugs

`git bisect` faz uma busca binária pelo histórico de commits para encontrar qual commit introduziu um bug. Esta é uma das funcionalidades mais subutilizadas e poderosas do Git.

```bash
# Inicia o bisect
git bisect start

# Marca o commit atual como ruim (tem o bug)
git bisect bad

# Marca um commit que você sabe que estava bom (sem bug)
git bisect good v2.0.0   # ou um SHA específico: git bisect good abc1234

# Git faz checkout do commit do ponto médio
# Você testa — roda seu teste, verifica o bug

# Se este commit tem o bug:
git bisect bad

# Se este commit é bom:
git bisect good

# Git continua estreitando até encontrar o primeiro commit ruim
# Imprime: "abc1234 is the first bad commit"

# Limpa
git bisect reset

# Automatiza com um script
git bisect run npm test   # roda npm test; exit 0 = bom, não-zero = ruim
```

Com 1000 commits, git bisect encontra o commit ruim em no máximo 10 passos (log2(1000) ≈ 10).

---

### Stash

```bash
git stash                          # salva working tree + index no stash
git stash push -m "WIP: feature pela metade"   # com mensagem descritiva
git stash push -p                  # interativo — escolhe quais hunks guardar
git stash push -- path/to/file.ts  # guarda apenas arquivo específico

git stash list                     # mostra todos os stashes
git stash show stash@{0}           # mostra diff do stash mais recente
git stash show -p stash@{1}        # mostra patch completo

git stash pop                      # aplica o stash mais recente e o remove da lista
git stash apply stash@{1}          # aplica stash específico (mantém na lista)
git stash drop stash@{0}           # remove stash específico sem aplicar
git stash clear                    # remove TODOS os stashes
git stash branch new-branch        # cria branch do stash (útil se stash tem conflitos)
```

---

### Reflog — Recuperando Commits Perdidos

O reflog é um diário local de onde HEAD e as pontas de branch estiveram. Registra toda vez que HEAD se move (checkout, reset, rebase, merge, commit, amend).

```bash
git reflog                         # mostra reflog do HEAD
git reflog show main               # mostra reflog do branch main

# Saída:
# abc1234 HEAD@{0}: commit: Add feature X
# def5678 HEAD@{1}: checkout: moving from feature/x to main
# ghi9012 HEAD@{2}: rebase (finish): ...
# jkl3456 HEAD@{3}: commit: WIP commit que foi perdido

# Recupera um commit "perdido" após um reset ruim
git checkout jkl3456               # vai para o commit (HEAD detached)
git checkout -b recovery           # cria um branch para preservá-lo
# ou faz cherry-pick:
git cherry-pick jkl3456
```

> Git quase nunca perde dados de verdade. Se você fez `git reset --hard` e perdeu commits, `git reflog` quase certamente vai te mostrar o SHA para recuperar. Entradas do reflog expiram após 90 dias por padrão.

---

### Git Hooks

Hooks são scripts em `.git/hooks/` que rodam automaticamente em pontos específicos do workflow do Git.

```
pre-commit       — antes do git commit criar o commit (lint, testes, formatação)
prepare-commit-msg — após o arquivo de mensagem de commit ser criado, antes do editor
commit-msg       — valida a mensagem de commit (ex.: impor Conventional Commits)
post-commit      — após o commit ser criado (notificações, automação local)
pre-push         — antes de fazer push (rodar suite completa de testes, prevenir push de secrets)
post-merge       — após um merge bem-sucedido (rodar npm install se package.json mudou)
pre-rebase       — antes do rebase (verificações de segurança)
```

```bash
# Exemplo: hook pre-commit que roda ESLint
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
set -e

# Obtém arquivos .ts/.tsx staged
STAGED=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(ts|tsx)$' || true)
[ -z "$STAGED" ] && exit 0

echo "Rodando ESLint nos arquivos staged..."
echo "$STAGED" | xargs npx eslint --max-warnings=0
EOF
chmod +x .git/hooks/pre-commit
```

> Para hooks compartilhados com a equipe, use **husky** (projetos Node.js) ou coloque hooks em um diretório `.githooks/` e configure com `git config core.hooksPath .githooks`. Hooks em `.git/hooks/` não são commitados no repositório.

```bash
# Configura diretório de hooks compartilhados
git config core.hooksPath .githooks
# Agora .githooks/pre-commit é rastreado e compartilhado via repositório
```

---

### Resolução de Conflitos

```bash
# Quando um merge/rebase tem conflitos:
git status                    # mostra arquivos em conflito com 'both modified'

# Arquivo em conflito fica assim (git insere esses marcadores automaticamente):
# <<< HEAD
# const x = 1;               ← sua mudança (branch atual)
# ===
# const x = 2;               ← mudança que está sendo mesclada
# >>> feature/x

# Opções de resolução:
# 1. Edite o arquivo manualmente — remova os marcadores, mantenha o que quiser
# 2. git checkout --ours file.ts   — pega sua versão
# 3. git checkout --theirs file.ts — pega a versão que está sendo mesclada
# 4. git mergetool                 — abre ferramenta visual de merge configurada

# Após resolver todos os conflitos:
git add resolved-file.ts
git commit                    # para merge
git rebase --continue         # para rebase (ou --skip / --abort)
```

---

## 4. Exemplos de Código

### Aliases úteis
```bash
# Adicione ao ~/.gitconfig
[alias]
  lg = log --oneline --graph --all --decorate
  s  = status -s
  d  = diff
  ds = diff --staged
  undo = reset HEAD~1
  unstage = reset HEAD --
  last = log -1 HEAD --stat
  aliases = config --get-regexp '^alias'
  contributors = shortlog -sn --no-merges
  branches = branch -vv --sort=-committerdate
```

```bash
# Log rápido com grafo
git log --oneline --graph --all
# * abc1234 (HEAD -> main) Add authentication
# * def5678 Refactor user service
# | * ghi9012 (feature/payments) Add payment processing
# |/
# * jkl3456 Initial commit

# Veja o que mudou no último commit
git show HEAD --stat

# Veja quem mudou cada linha de um arquivo por último
git blame -L 10,30 src/auth.ts   # apenas linhas 10-30
git blame --follow src/auth.ts   # segue renomeações

# Busca em mensagens de commit
git log --grep="fix" --oneline

# Busca em todos os commits quando uma string foi introduzida/removida
git log -S "secretToken" --all   # todos os branches

# Veja commits por autor
git shortlog -sn --no-merges

# Compara dois branches
git diff main...feature/x        # o que está em feature/x mas não em main
git log main..feature/x          # commits em feature/x que não estão em main
```

### Padrões de .gitignore
```gitignore
# Dependências
node_modules/
vendor/
.venv/

# Outputs de build
dist/
build/
*.js.map

# Arquivos de ambiente (NUNCA faça commit desses)
.env
.env.*
!.env.example       # exceção — mantém o template

# Logs
*.log
logs/

# Arquivos do OS
.DS_Store
Thumbs.db

# Arquivos de editor
.idea/
.vscode/
*.swp
*.swo

# Relatórios de cobertura
coverage/
.nyc_output/

# Casa qualquer profundidade
**/*.env
**/node_modules/

# TypeScript (se não quiser commitar output compilado)
*.js       # cuidado — apenas se você nunca commitar arquivos JS puros
!jest.config.js
```

> Se você acidentalmente commitou `node_modules` antes de adicionar `.gitignore`:
> ```bash
> git rm -r --cached node_modules
> echo "node_modules/" >> .gitignore
> git add .gitignore
> git commit -m "chore: remove node_modules from tracking"
> ```

---

## 5. Erros Comuns e Armadilhas

> **Commitar secrets**: Chaves de API, senhas e tokens commitados no git são comprometidos para sempre — persistem no histórico mesmo após você deletá-los. Use `git log -S "secret"` para encontrar onde estavam. Rotacione as chaves imediatamente após descobrir um vazamento. Ferramentas: `git-secrets`, `truffleHog`, `gitleaks` para escanear secrets. Use `.gitignore` para arquivos `.env` e configuração baseada em ambiente.

> **Force-push em branches compartilhados**: `git push --force` reescreve o histórico remoto. Qualquer pessoa que fez fetch daquele branch agora tem um histórico divergente. Use `git push --force-with-lease` que falha se o remoto tem commits que você não viu (mais seguro). Nunca force-push em `main` ou `develop`.

> **Commits enormes**: Commits atômicos são mais fáceis de revisar, mais fáceis de reverter e mais fáceis de bisectar. Um commit que muda 15 arquivos em 3 preocupações diferentes é difícil de revisar e impossível de cherry-pick. Mantenha commits focados em uma única mudança lógica.

> **Mensagens de commit ruins**: Commits são documentação. `"fix bug"` não diz nada. Uma boa mensagem: `"fix(auth): handle token expiry during concurrent requests"`. Siga Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`).

> **Ignorar `.gitignore` até depois de commitar `node_modules`**: O repositório cresce centenas de megabytes, `git status` fica inutilizavelmente lento e operações git demoram uma eternidade. Adicione `.gitignore` antes do primeiro commit. Use `git rm -r --cached` para remover arquivos já rastreados.

> **Fazer rebase em branches compartilhados**: Nunca faça rebase de commits que foram enviados e que outros já puxaram. Isso cria históricos divergentes e força colegas a fazerem reconciliação dolorosa com `git rebase --onto` ou resetando seus branches locais.

---

## 6. Quando Usar / Não Usar

**Use `merge` quando:**
- Integrando um feature branch em `main` (preserva o histórico completo do feature branch)
- Em branches públicos/compartilhados (nunca faça rebase em branches compartilhados)

**Use `rebase` quando:**
- Atualizando seu feature branch local com mudanças do main (antes de abrir o PR)
- Rebase interativo para limpar commits WIP antes da revisão do PR
- Criando um histórico linear para uma feature antes de fazer merge

**Use `stash` quando:**
- Precisar trocar rapidamente de contexto (pull, checkout, hotfix urgente) com mudanças não commitadas
- Quiser testar algo sem as mudanças não commitadas presentes

**Use `cherry-pick` quando:**
- Precisar de um commit específico de outro branch (backport de um fix para um release branch)
- Acidentalmente commitou no branch errado

**Use `bisect` quando:**
- Sabe que um bug foi introduzido em um commit entre a versão X (boa) e HEAD (ruim)
- O bug é testável programaticamente (use `git bisect run`)

**Use `reflog` quando:**
- Acidentalmente fez reset, deletou um branch ou perdeu commits

---

## 7. Cenário Real

### Usando git bisect para encontrar uma regressão de performance

**Situação:** A aplicação estava rápida na v1.5.0 (lançada 3 meses atrás, 200 commits atrás). Agora está lenta. Você precisa encontrar qual commit introduziu a regressão.

```bash
# Escreva um teste automatizado para a regressão
cat > test-performance.sh << 'EOF'
#!/bin/bash
# Retorna 0 (bom) se rápido, não-zero (ruim) se lento
npm install --silent
DURATION=$(node -e "
  const start = Date.now();
  const result = require('./src/utils/processor').process(testData);
  console.log(Date.now() - start);
")
[ "$DURATION" -lt 100 ]  # bom se abaixo de 100ms
EOF
chmod +x test-performance.sh

# Inicia o bisect
git bisect start

# Marca o HEAD atual como ruim
git bisect bad

# Marca v1.5.0 como bom
git bisect good v1.5.0

# Bisect automatizado — git roda o script em cada ponto médio
git bisect run ./test-performance.sh

# Saída do git:
# Bisecting: 100 revisions left to test after this (roughly 7 steps)
# [sha1] ...
# Bisecting: 50 revisions left to test after this (roughly 6 steps)
# ...
# abc1234 is the first bad commit
# Author: Dave <dave@example.com>
# Date: Thu Jan 18 14:32:11 2024 +0000
#     refactor(processor): simplify loop logic

# Inspeciona o commit ofensor
git show abc1234

# Limpa
git bisect reset

# Agora você sabe exatamente qual commit e qual autor introduziu a regressão
# Faça git blame nas linhas afetadas, entenda a mudança e corrija
```

---

### Configurando um hook pre-commit com configuração compartilhada

```bash
# Cria diretório .githooks (commitado no repo, compartilhado com a equipe)
mkdir .githooks

cat > .githooks/pre-commit << 'EOF'
#!/bin/sh
set -e

STAGED_TS=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(ts|tsx)$' || true)
STAGED_ALL=$(git diff --cached --name-only --diff-filter=ACMR)

# Roda ESLint nos arquivos TypeScript staged
if [ -n "$STAGED_TS" ]; then
  echo "ESLint..."
  echo "$STAGED_TS" | xargs npx eslint --max-warnings=0
fi

# Roda verificação Prettier em todos os arquivos staged
if [ -n "$STAGED_ALL" ]; then
  echo "Prettier..."
  echo "$STAGED_ALL" | xargs npx prettier --check --ignore-unknown
fi

echo "Verificações pre-commit passaram."
EOF
chmod +x .githooks/pre-commit

cat > .githooks/commit-msg << 'EOF'
#!/bin/sh
# Impõe formato Conventional Commits
MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,100}"
if ! echo "$MSG" | grep -Eq "$PATTERN"; then
  echo "ERRO: Mensagem de commit não segue o formato Conventional Commits."
  echo "Esperado: tipo(escopo): descrição"
  echo "Exemplo:  feat(auth): add OAuth2 login"
  exit 1
fi
EOF
chmod +x .githooks/commit-msg

# Configura git para usar .githooks/ para todos os membros da equipe
# (adicione isso ao README ou script de onboarding)
git config core.hooksPath .githooks
```

---

## 8. Perguntas de Entrevista

**Q1: O que é git rebase e quando você deve usá-lo?**

R: Rebase replaya commits de um branch em cima de outro, criando novos commits com novos SHAs. O resultado é um histórico linear como se o branch tivesse sido criado a partir da ponta atual do alvo, não de onde foi originalmente criado. Use rebase: (1) Antes de fazer merge de um feature branch para main — `git rebase main` no seu feature branch coloca seus commits após os últimos commits do main; (2) Rebase interativo para limpar commits WIP (squash, reword, drop) antes de um PR; (3) Atualizando um feature branch de longa duração com mudanças do main. Nunca faça rebase de commits já enviados para um branch compartilhado — reescreve o histórico e força colegas a reconciliar históricos divergentes.

---

**Q2: Como você recupera um branch deletado?**

R: Use `git reflog` para encontrar o SHA do último commit no branch deletado. O reflog registra todo movimento do HEAD, incluindo quando você fez checkout do branch e quando foi deletado. `git reflog | grep "nome-do-branch"` mostra entradas relevantes. Então `git checkout -b branch-recuperado <SHA>` recria o branch a partir daquele commit. Isso funciona porque deletar um branch apenas remove o arquivo de ponteiro em `.git/refs/heads/` — os objetos de commit permanecem até a coleta de lixo (que não roda em entradas de reflog alcançáveis, que expiram após 90 dias por padrão).

---

**Q3: O que é git bisect?**

R: Git bisect é uma busca binária pelo histórico de commits para encontrar o commit que introduziu um bug. Você diz ao git `bisect start`, marca o estado atual como ruim (`bisect bad`) e um estado sabidamente bom como bom (`bisect good v1.5.0`). Git faz checkout do commit do ponto médio. Você testa e diz ao git `bisect good` ou `bisect bad`. Git divide o espaço de busca pela metade a cada vez. Com N commits, encontra o culpado em log₂(N) passos. O poder: `git bisect run <script>` automatiza o teste — git roda seu script em cada ponto médio, usando o exit code para determinar bom/ruim, sem interação humana.

---

**Q4: Explique as trocas entre merge e rebase.**

R: **Merge** preserva o histórico verdadeiro — quando os branches divergiram, como foram integrados, quem trabalhou em quê. A desvantagem são os merge commits que podem poluir o log em um repositório movimentado. **Rebase** produz um histórico limpo e linear que é mais fácil de ler e raciocinar. A desvantagem é que reescreve o histórico (novos SHAs), o que é perigoso para branches compartilhados, e perde a informação sobre quando o trabalho foi feito concorrentemente. Merge é mais seguro e correto; rebase é mais limpo. A maioria das equipes faz rebase de feature branches locais antes de fazer merge em main, mas usa merge ao integrar para manter rastreabilidade.

---

**Q5: O que é um estado de HEAD detached?**

R: Normalmente, HEAD aponta para um nome de branch (ex.: `ref: refs/heads/main`), que por sua vez aponta para o último commit naquele branch. Quando você faz `git checkout <SHA>` ou `git checkout v1.5.0`, HEAD aponta diretamente para um SHA de commit em vez de para um branch. Isso é "HEAD detached". Você pode fazer commits nesse estado, mas eles não têm nenhum branch os rastreando. Quando você faz checkout de outro branch, esses commits parecem ter sido perdidos (são alcançáveis via reflog mas nenhum branch aponta para eles). Correção: `git checkout -b novo-branch` enquanto está em HEAD detached para criar um branch que captura seu trabalho.

---

**Q6: Como você desfaz o último commit?**

R: Depende do que você quer desfazer e se o commit foi enviado. Para um commit local, não enviado: (1) `git reset HEAD~1` — move HEAD um commit para trás, mantém mudanças na working tree (staged ou não dependendo de `--mixed`/`--soft`/`--hard`). (2) `git reset --soft HEAD~1` — mantém mudanças staged. (3) `git reset --hard HEAD~1` — descarta todas as mudanças (destrutivo). Para um commit enviado: (4) `git revert HEAD` — cria um novo commit que desfaz o anterior (seguro para branches compartilhados, preserva o histórico). Nunca use `reset --hard` em commits enviados a menos que queira fazer force-push, o que é perigoso em branches compartilhados.

---

**Q7: O que é um hook do git?**

R: Um hook do git é um script que roda automaticamente em um ponto específico do workflow do Git. Hooks ficam em `.git/hooks/` (ou em um diretório `core.hooksPath` configurado). Hooks chave: `pre-commit` (roda linting/formatação antes do commit ser criado), `commit-msg` (valida o formato da mensagem de commit), `pre-push` (roda testes antes de fazer push). Hooks não são commitados com o repositório por padrão (`.git/` não é rastreado). Para compartilhar hooks com a equipe, armazene-os em um diretório commitado (ex.: `.githooks/`) e configure `git config core.hooksPath .githooks`. Ou use husky em projetos Node.js.

---

**Q8: Explique o modelo de objetos do git.**

R: Git armazena quatro tipos de objeto: blobs (conteúdo de arquivo), trees (diretórios mapeando nomes para SHAs de blob/tree), commits (snapshots apontando para uma tree + commit pai + metadados) e tags anotadas (ponteiros nomeados para commits). Todos os objetos são identificados por um hash SHA do seu conteúdo — conteúdo idêntico produz o mesmo SHA, fornecendo deduplicação automática. Branches e HEAD são apenas ponteiros (arquivos de texto contendo um SHA). Criar um branch é O(1). Commits são imutáveis — você não pode mudar um commit, apenas criar novos. `git rebase` cria novos commits (novos SHAs) com novos pais; os commits antigos permanecem no object store até serem coletados pelo garbage collector.

---

## 9. Exercícios

**Exercício 1 — Git bisect para encontrar um bug:**

1. Crie um repositório com 20+ commits, um dos quais introduz um bug (ex.: uma função retorna output errado)
2. Pratique `git bisect start`, marque o atual como ruim, marque o commit 1 como bom
3. Identifique manualmente o commit ruim passo a passo
4. Redefina e faça novamente com `git bisect run` usando um script de teste

Dica: Crie um script de teste que faz `exit 0` em sucesso e `exit 1` em falha. `git bisect run sh test.sh`.

---

**Exercício 2 — Hook pre-commit para ESLint:**

Configure um hook pre-commit que:
- Identifica arquivos `.ts` e `.tsx` staged
- Roda `eslint --max-warnings=0` neles
- Falha o commit se ESLint reportar quaisquer erros ou warnings
- Pula a verificação se nenhum arquivo TypeScript está staged

Coloque em `.githooks/pre-commit`, torne executável e configure `git config core.hooksPath .githooks`.

---

**Exercício 3 — Alias git log:**

Crie um alias git `lg` que mostra:
- Uma linha por commit
- Grafo ASCII de branches e merges
- Todos os branches (locais e remotos)
- Saída colorida com nomes de branch/tag
- Datas relativas

```bash
git config --global alias.lg "log --oneline --graph --all --decorate --color"
# Melhore com:
git config --global alias.lg "log --graph --format='%C(yellow)%h%C(reset) %C(green)(%ar)%C(reset) %C(bold blue)%an%C(reset) %C(white)%s%C(reset)%C(red)%d%C(reset)' --all"
```

Teste criando alguns branches e commits para ver o grafo em ação.

---

**Exercício 4 — Resolva um conflito de merge deliberadamente:**

1. Crie um branch a partir do main
2. Em main: mude a linha 5 de um arquivo e faça commit
3. No branch: mude a mesma linha 5 de forma diferente e faça commit
4. Tente fazer merge do branch em main — haverá conflito
5. Abra o arquivo em conflito e resolva manualmente (mantenha uma versão)
6. Stage o arquivo resolvido e complete o merge
7. Use `git log --graph --oneline` para verificar que o merge commit aparece

Bônus: repita com `git rebase main` em vez de merge, resolva o conflito de rebase e compare os históricos resultantes.

---

## 10. Leitura Adicional

- **Pro Git** (gratuito online): https://git-scm.com/book/en/v2 — o livro Git abrangente oficial de Scott Chacon
- **"Git Internals"** (do Pro Git, Capítulo 10): https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain — como o git funciona internamente
- **"A Successful Git Branching Model"** de Vincent Driessen (Git Flow): https://nvie.com/posts/a-successful-git-branching-model/
- **"Please stop recommending Git Flow"** (caso para Trunk-Based Development): https://georgestocker.com/2020/03/04/please-stop-recommending-git-flow/
- **Trunk-Based Development**: https://trunkbaseddevelopment.com — guia abrangente
- **Especificação Conventional Commits**: https://www.conventionalcommits.org/ — padrão de mensagem de commit
- **Referência git-scm.com**: https://git-scm.com/docs — páginas man oficiais
- **Oh Shit, Git!**: https://ohshitgit.com — guia em linguagem simples para desfazer erros comuns
- **Learn Git Branching** (interativo): https://learngitbranching.js.org — tutorial visual e interativo
- **Husky** (Git hooks para Node.js): https://typicode.github.io/husky/
