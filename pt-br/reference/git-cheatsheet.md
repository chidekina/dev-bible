# Git Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## Configuração Inicial

```bash
git config --global user.name "Seu Nome"
git config --global user.email "voce@exemplo.com"
git config --global core.editor "vim"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global rebase.autoStash true
git config --list --show-origin       # exibir toda a config com origem
git config --global alias.lg "log --oneline --graph --decorate --all"
```

---

## Fluxo Diário

```bash
# Status e informações
git status
git status -s                         # formato resumido
git diff                              # alterações não staged
git diff --staged                     # alterações staged
git diff HEAD                         # todas as alterações desde o último commit
git diff main..feature                # entre branches

# Staging
git add file.ts                       # stage de arquivo específico
git add src/                          # stage de diretório
git add -p                            # staging interativo por hunk
git add -u                            # stage de todas as alterações rastreadas
git add .                             # stage de tudo (use com cuidado)
git restore --staged file.ts          # remover do stage
git restore file.ts                   # descartar alterações no diretório de trabalho

# Committing
git commit -m "feat: add user auth"
git commit --amend                    # modificar o último commit (não enviado)
git commit --amend --no-edit          # amend sem alterar a mensagem
git commit -a -m "fix: quick patch"   # stage de rastreados + commit

# Push
git push origin feature/my-branch
git push -u origin feature/my-branch  # definir upstream
git push --force-with-lease           # force push seguro
git push --tags                       # enviar todas as tags
```

---

## Branches

```bash
git branch                            # listar branches locais
git branch -a                         # listar todos (incluindo remotos)
git branch feature/new-thing          # criar branch
git switch feature/new-thing          # mudar para a branch
git switch -c feature/new-thing       # criar e mudar
git switch -c feature/x origin/feature/x  # rastrear branch remota

git branch -d feature/done            # deletar (seguro — apenas se merged)
git branch -D feature/wip             # forçar deleção
git branch -m old-name new-name       # renomear

git push origin --delete feature/done # deletar branch remota

# Comparar
git log main..feature                 # commits em feature que não estão em main
git diff main...feature               # diff a partir do ancestral comum
```

---

## Merge

```bash
git merge feature/my-branch           # merge na branch atual
git merge --no-ff feature/my-branch   # sempre criar merge commit
git merge --squash feature/my-branch  # squash em um único commit
git merge --abort                     # abortar merge em andamento

# Após conflito:
# 1. Editar arquivos para resolver conflitos
# 2. git add <arquivos-resolvidos>
# 3. git commit
```

---

## Rebase

```bash
git rebase main                       # rebase da branch atual em cima de main
git rebase -i HEAD~3                  # rebase interativo dos últimos 3 commits
git rebase -i origin/main             # interativo a partir do ponto de divergência
git rebase --abort                    # abortar rebase
git rebase --continue                 # continuar após resolver conflito
git rebase --skip                     # pular commit conflitante atual

# Comandos do rebase interativo (no editor):
# pick   = usar commit como está
# reword = alterar mensagem do commit
# edit   = pausar para emendar o commit
# squash = fundir no commit anterior (manter todas as mensagens)
# fixup  = fundir no anterior (descartar esta mensagem)
# drop   = remover commit completamente
```

---

## Operações Remotas

```bash
git remote -v                         # listar remotos
git remote add origin git@github.com:user/repo.git
git remote remove origin
git remote rename origin upstream

git fetch                             # buscar todos os remotos
git fetch origin                      # buscar remoto específico
git fetch --prune                     # também remover branches remotas deletadas

git pull                              # fetch + merge
git pull --rebase                     # fetch + rebase (histórico mais limpo)
git pull origin main                  # pull de branch específica

# Rastreamento
git branch -u origin/main main        # definir upstream
git branch -vv                        # exibir informações de rastreamento
```

---

## Stash

```bash
git stash                             # guardar diretório de trabalho + index
git stash push -m "WIP: feature pela metade"
git stash push -p                     # interativo (escolher hunks)
git stash push --include-untracked    # também guardar arquivos não rastreados

git stash list                        # listar todos os stashes
git stash show stash@{0}              # mostrar diff do stash
git stash show -p stash@{1}           # patch completo

git stash pop                         # aplicar + remover o stash do topo
git stash apply stash@{1}             # aplicar sem remover
git stash drop stash@{0}              # remover um stash
git stash clear                       # remover todos os stashes
git stash branch feature/x stash@{0} # criar branch a partir do stash
```

---

## Histórico e Log

```bash
git log
git log --oneline                     # uma linha por commit
git log --oneline --graph --all       # grafo visual de branches
git log --stat                        # mostrar arquivos alterados
git log -p                            # mostrar patch completo
git log -n 10                         # últimos 10 commits
git log --author="Alice"
git log --since="2024-01-01" --until="2024-12-31"
git log --grep="fix:"                 # buscar nas mensagens de commit
git log -S "functionName"             # buscar string adicionada/removida
git log -- path/to/file               # commits que tocaram um arquivo
git log main..HEAD                    # commits que não estão em main

git show HEAD                         # mostrar último commit + diff
git show abc1234                      # mostrar commit específico
git show HEAD:src/app.ts              # mostrar arquivo em determinado commit

# Quem alterou esta linha?
git blame file.ts
git blame -L 10,25 file.ts            # faixa de linhas específica
git log -p -S "suspectFunction" file.ts  # quando foi introduzido?
```

---

## Desfazendo Alterações

```bash
# Desfazer alterações no diretório de trabalho (destrutivo)
git restore file.ts                   # descartar alterações não staged
git restore .                         # descartar todas as alterações não staged
git clean -fd                         # remover arquivos/diretórios não rastreados
git clean -fdn                        # simulação antes de executar

# Desfazer alterações staged
git restore --staged file.ts

# Desfazer commits (seguro — adiciona novo commit)
git revert HEAD                       # reverter último commit
git revert abc1234                    # reverter commit específico
git revert HEAD~3..HEAD               # reverter intervalo

# Reset (move o ponteiro da branch — perigoso se já enviado)
git reset --soft HEAD~1               # desfazer commit, manter staged
git reset --mixed HEAD~1              # desfazer commit + remover stage (padrão)
git reset --hard HEAD~1               # desfazer commit + descartar alterações

# Desfazer um merge problemático
git reset --hard ORIG_HEAD            # após merge (antes de outros commits)
```

---

## Reflog — Sua Rede de Segurança

```bash
git reflog                            # todos os movimentos do HEAD já registrados
git reflog show feature/my-branch     # reflog de uma branch específica

# Recuperar branch deletada
git reflog                            # encontrar o SHA antes da deleção
git switch -c recovered-branch abc1234

# Recuperar após reset problemático
git reset --hard HEAD@{3}             # voltar 3 movimentos atrás

# Reflog expira (padrão: 90 dias para alcançáveis, 30 para não alcançáveis)
git reflog expire --expire=now --all  # expirar manualmente (use com cuidado)
```

---

## Bisect — Encontrar o Commit que Quebrou Tudo

```bash
git bisect start
git bisect bad                        # commit atual está com problema
git bisect good v2.0.0                # último commit/tag que funcionava
# Git faz checkout do ponto médio — teste e então:
git bisect good                       # este funciona
git bisect bad                        # este está quebrado
# Repita até o git mostrar o commit problemático
git bisect reset                      # voltar ao HEAD original

# Bisect automatizado
git bisect run npm test               # executar suite de testes automaticamente
```

---

## Tags

```bash
git tag                               # listar tags
git tag v1.0.0                        # tag leve
git tag -a v1.0.0 -m "Release 1.0"   # tag anotada (recomendada)
git tag -a v1.0.0 abc1234 -m "Tag em commit antigo"
git push origin v1.0.0                # enviar tag individual
git push origin --tags                # enviar todas as tags
git tag -d v1.0.0                     # deletar tag local
git push origin --delete v1.0.0      # deletar tag remota
git show v1.0.0                       # exibir informações da tag
```

---

## Cherry-Pick

```bash
git cherry-pick abc1234               # aplicar commit único
git cherry-pick abc1234..def5678      # aplicar intervalo (início exclusivo)
git cherry-pick abc1234^..def5678     # aplicar intervalo (início inclusivo)
git cherry-pick -n abc1234            # aplicar sem commitar
git cherry-pick --abort
git cherry-pick --continue
```

---

## Submodules

```bash
git submodule add https://github.com/user/lib.git libs/lib
git submodule update --init --recursive   # inicializar + buscar submodules
git submodule update --remote             # atualizar para o remoto mais recente
git clone --recurse-submodules <url>      # clonar com submodules
```

---

## Aliases Úteis

```bash
git config --global alias.st "status -s"
git config --global alias.co "switch"
git config --global alias.br "branch"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.unstage "restore --staged"
git config --global alias.visual "!gitk"
git config --global alias.aliases "config --get-regexp alias"
```

---

## Referência Rápida do Git Flow

```
main        — código pronto para produção
develop     — branch de integração
feature/*   — novas funcionalidades (branch de develop)
release/*   — preparação de release (branch de develop, merge em main + develop)
hotfix/*    — correções urgentes (branch de main, merge em main + develop)
```

```bash
# Fluxo de feature
git switch -c feature/user-auth develop
# ... trabalho ...
git switch develop && git merge --no-ff feature/user-auth
git branch -d feature/user-auth

# Fluxo de hotfix
git switch -c hotfix/1.0.1 main
# ... correção ...
git switch main && git merge --no-ff hotfix/1.0.1
git tag -a v1.0.1 -m "Hotfix 1.0.1"
git switch develop && git merge --no-ff hotfix/1.0.1
git branch -d hotfix/1.0.1
```

---

## Padrões do .gitignore

```gitignore
# Diretórios
node_modules/
dist/
.cache/

# Arquivos
.env
.env.local
*.log
*.tmp

# Negação (des-ignorar)
!.env.example

# Wildcards
**/*.test.js     # qualquer profundidade
src/**/*.snap    # dentro de src em qualquer profundidade

# Sistema operacional
.DS_Store
Thumbs.db
```
