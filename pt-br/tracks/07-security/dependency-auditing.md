# Auditoria de Dependências

## Visão Geral

Aplicações Node.js modernas dependem de centenas de pacotes. Cada um é um vetor de ataque em potencial — uma dependência comprometida ou vulnerável pode expor toda a aplicação. Auditoria de dependências é a prática de verificar sistematicamente sua árvore de dependências em busca de vulnerabilidades conhecidas, manter os pacotes atualizados e impedir que pacotes maliciosos entrem na sua cadeia de suprimentos. Este capítulo cobre ferramentas, fluxos de trabalho e as práticas operacionais que tornam a segurança de dependências sustentável.

---

## Pré-requisitos

- Gerenciamento de pacotes Node.js (npm, pnpm ou Bun)
- Compreensão básica de pipelines de CI/CD (GitHub Actions)
- Familiaridade com `package.json` e lockfiles

---

## Conceitos Fundamentais

### A superfície de risco das dependências

```
Seu código → dependências diretas → dependências transitivas → ... → pacotes do SO
```

Uma aplicação Next.js típica tem ~1.000 pacotes em `node_modules`. Você controla diretamente ~50; o restante é puxado transitivamente. Uma vulnerabilidade em qualquer um deles pode afetá-lo.

### Níveis de severidade de vulnerabilidade

| Nível | Pontuação CVSS | Exemplo | Ação |
|-------|---------------|---------|------|
| Crítico | 9.0–10.0 | Execução remota de código | Corrija imediatamente |
| Alto | 7.0–8.9 | Bypass de autenticação, exposição de dados | Corrija em até 24 horas |
| Médio | 4.0–6.9 | Divulgação de informações | Corrija dentro de um sprint |
| Baixo | 0.1–3.9 | Problemas menores | Monitore, corrija quando possível |

---

## Exemplos Práticos

### npm audit

```bash
# Execute uma auditoria básica
npm audit

# Reporte apenas alto e crítico
npm audit --audit-level=high

# Corrija problemas corrigíveis automaticamente
npm audit fix

# Corrija incluindo mudanças incompatíveis (bumps de versão major)
npm audit fix --force

# Saída JSON para parsing no CI
npm audit --json
```

Exemplo de saída:

```
found 3 vulnerabilities (1 moderate, 2 high)
  run `npm audit fix` to fix 1 of them.
  2 vulnerabilities require manual review.
  See the full report for details.
```

### Integração com CI usando GitHub Actions

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 8 * * 1' # toda segunda-feira às 8h UTC

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install dependencies
        run: npm ci

      - name: Run security audit
        run: npm audit --audit-level=high
        # Quebra o build em vulnerabilidades altas ou críticas
```

### Snyk — varredura avançada de vulnerabilidades

O Snyk vai além do `npm audit` — verifica vulnerabilidades que ainda não estão no banco de dados advisory do npm e fornece orientação de remediação.

```bash
# Instale o Snyk CLI
npm install -g snyk

# Autentique
snyk auth

# Verifique vulnerabilidades
snyk test

# Teste e falhe em severidade alta
snyk test --severity-threshold=high

# Monitore seu projeto (reporta ao dashboard do Snyk)
snyk monitor
```

```yaml
# GitHub Actions com Snyk
- name: Run Snyk vulnerability scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

### Dependabot — atualizações automáticas de dependências

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "08:00"
      timezone: "America/Sao_Paulo"
    groups:
      # Agrupa atualizações de patch e minor para reduzir o volume de PRs
      patch-and-minor:
        update-types:
          - "minor"
          - "patch"
    ignore:
      # Bumps de versão major exigem revisão manual
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
    reviewers:
      - "your-github-username"
    labels:
      - "dependencies"
```

O Dependabot abre PRs automaticamente quando atualizações estão disponíveis. Agrupe atualizações por patch/minor para manter o volume de PRs gerenciável.

### Verificando a integridade do lockfile

```bash
# npm ci usa o package-lock.json exatamente — falha se estiver inconsistente
npm ci

# Nunca use npm install no CI — pode atualizar o lockfile
```

Sempre faça commit do lockfile (`package-lock.json`, `pnpm-lock.yaml`, `bun.lockb`) no controle de versão. Sem um lockfile, `npm install` pode instalar versões diferentes em máquinas diferentes.

### Auditoria de licenças

Nem todas as dependências são seguras para uso em produtos comerciais. Verifique as licenças:

```bash
# license-checker lista todas as licenças das dependências
npx license-checker --onlyAllow 'MIT;ISC;BSD-2-Clause;BSD-3-Clause;Apache-2.0;CC0-1.0;0BSD'
```

```typescript
// Script no package.json
{
  "scripts": {
    "check:licenses": "license-checker --onlyAllow 'MIT;ISC;BSD-2-Clause;BSD-3-Clause;Apache-2.0'"
  }
}
```

Dependências com licença GPL em uma aplicação comercial de código fechado podem criar obrigações legais.

### Prevenção de ataques à cadeia de suprimentos

Ataques à cadeia de suprimentos (como o incidente `event-stream` e a sabotagem do `node-ipc`) introduzem código malicioso em pacotes dos quais você depende.

Mitigações:

```bash
# Fixe versões exatas no package.json (sem intervalos ^ ou ~)
{
  "dependencies": {
    "fastify": "4.28.1",  // exato — não "^4.28.1"
    "zod": "3.23.8"
  }
}
```

```bash
# Verifique a integridade do pacote após instalar
npm ci --ignore-scripts  # pula scripts postinstall
```

```bash
# Use npm pack para inspecionar o conteúdo de um pacote antes de instalar
npm pack fastify --dry-run
```

```bash
# Verifique typosquatting antes de instalar um novo pacote
# Confirme nome do pacote, autor e downloads semanais em npmjs.com
```

### Encontrando pacotes desatualizados

```bash
# Liste pacotes desatualizados
npm outdated

# Atualize para as versões compatíveis mais recentes
npx npm-check-updates -u --target minor
npm install
```

---

## Padrões Comuns e Boas Práticas

- **Execute `npm audit` no CI** — falhe em severidade alta, para que vulnerabilidades não cheguem à produção
- **Habilite o Dependabot** — automatize a descoberta de atualizações de dependências
- **Use `npm ci` no CI** — instalações determinísticas a partir do lockfile; falha se o lockfile estiver inconsistente
- **Revise os PRs do Dependabot com seu processo normal de revisão** — não faça merge automático sem verificar
- **Mantenha uma tarefa mensal de "saúde de dependências"** — atualize o que o Dependabot não pega (versões major)
- **Verifique licenças** em projetos comerciais — GPL em produto de código fechado é um risco legal
- **Audite novas dependências antes de adicioná-las** — verifique downloads no npm, stars no GitHub, status de manutenção

---

## Anti-Padrões a Evitar

- Não commitar o lockfile — permite que versões diferentes sejam instaladas em máquinas diferentes
- Usar `--legacy-peer-deps` para silenciar avisos de peer dependencies — esconde problemas reais de compatibilidade
- `npm audit fix --force` sem ler o que muda — force pode introduzir mudanças incompatíveis
- Ignorar PRs do Dependabot indefinidamente — acumulam e se tornam difíceis de gerenciar
- Instalar pacotes com poucos downloads ou criados recentemente sem verificá-los
- Executar scripts `postinstall` de pacotes de terceiros sem revisão

---

## Debugging e Resolução de Problemas

**"npm audit reporta vulnerabilidades mas fix não as resolve"**
Algumas vulnerabilidades estão em dependências transitivas que não podem ser atualizadas sem quebrar a dependência direta. Opções:
1. Verifique se a dependência direta lançou uma correção — atualize-a
2. Use `overrides` no `package.json` para fixar uma versão segura da dependência transitiva:

```json
{
  "overrides": {
    "lodash": "^4.17.21"
  }
}
```

3. Abra uma issue com o mantenedor da dependência direta

**"Dependabot abre 50 PRs de uma vez"**
Configure agrupamento no `dependabot.yml` para agrupar atualizações de patch/minor e defina uma frequência `weekly` em vez de `daily`.

**"License checker falha em um pacote que precisamos"**
Verifique os termos reais da licença do pacote — alguns são permissivos apesar do rótulo. Se necessário, contate o autor do pacote. Como último recurso, use um fork com mudança de licença.

---

## Cenários do Mundo Real

**Cenário: Respondendo a um CVE crítico**

1. `npm audit --json` — identifique os pacotes afetados e o CVE
2. Verifique se uma correção está disponível: `npm audit fix`
3. Se não: encontre o advisory, verifique se há um workaround
4. Verifique seu código: você usa o caminho de código vulnerável?
5. Se vulnerável e sem correção: isole o pacote atrás de uma fronteira de serviço, notifique as partes interessadas
6. Monitore o GitHub do pacote para uma correção, assine notificações de release

**Cenário: Política de zero dependências para código crítico de segurança**

Para operações criptográficas, handlers de autenticação e fluxos de pagamento, minimize dependências:

```typescript
// Use built-ins do Node.js para crypto — sem dependências de terceiros
import { createHash, randomBytes, timingSafeEqual } from 'crypto';

// Use apenas pacotes bem estabelecidos e amplamente auditados para hashing de senhas
import bcrypt from 'bcrypt'; // 10M+ downloads semanais, ativamente mantido, auditorias de segurança
```

---

## Leitura Complementar

- [Documentação do npm Audit](https://docs.npmjs.com/cli/v10/commands/npm-audit)
- [Guia de segurança Node.js do Snyk](https://snyk.io/learn/nodejs-security/)
- [Documentação do GitHub Dependabot](https://docs.github.com/en/code-security/dependabot)
- [OWASP Software Composition Analysis](https://owasp.org/www-community/Component_Analysis)
- [Segurança da cadeia de suprimentos — framework SLSA](https://slsa.dev/)

---

## Resumo

Auditoria de dependências é um processo contínuo, não uma tarefa única. A linha base é `npm audit` no CI com `--audit-level=high` — qualquer vulnerabilidade alta ou crítica quebra o build. O Dependabot automatiza a descoberta de atualizações e abre PRs em um cronograma, evitando que suas dependências fiquem perigosamente desatualizadas. Para projetos comerciais, a auditoria de licenças é igualmente importante. A prática mais profunda — fixar versões exatas, auditar novos pacotes antes de adicioná-los e revisar os PRs do Dependabot com cuidado — transforma o gerenciamento de dependências de uma corrida reativa para um fluxo de trabalho controlado e de baixo risco.
