# Gerenciamento de Secrets

## Visão Geral

Um secret é qualquer valor que concede acesso a um sistema: API keys, senhas de banco de dados, chaves de assinatura JWT, client secrets OAuth, chaves de criptografia. O mau gerenciamento de secrets é uma das causas mais comuns de vazamentos de dados — e a maioria dessas brechas é evitável com um pequeno conjunto de práticas consistentes. Este capítulo cobre como armazenar, acessar, rotacionar e auditar secrets em uma aplicação Node.js/TypeScript.

---

## Pré-requisitos

- Node.js e TypeScript básicos
- Compreensão de variáveis de ambiente
- Familiaridade com pipelines de CI/CD (GitHub Actions ou similar)

---

## Conceitos Fundamentais

### O ciclo de vida de um secret

```
Gerar → Armazenar → Acessar → Rotacionar → Revogar
```

Cada etapa tem modos de falha:

| Etapa | Falha comum |
|-------|------------|
| Gerar | Aleatoriedade fraca, valores previsíveis |
| Armazenar | Commitado no git, armazenado em texto plano |
| Acessar | Logado, passado em query params de URL |
| Rotacionar | Nunca rotacionado, mesmo secret por anos |
| Revogar | Sem mecanismo para revogar secrets comprometidos |

### Hierarquia de secrets

1. **Variáveis de ambiente** — mais simples, funciona em todo lugar, frágil em escala
2. **Secret managers** (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault) — centralizados, auditáveis, suportam rotação
3. **Sealed secrets / vaults criptografados** — para Kubernetes, secrets criptografados em repouso no controle de versão

---

## Exemplos Práticos

### Padrão básico com variáveis de ambiente

Nunca hardcode secrets no código-fonte. Carregue-os do ambiente e valide na inicialização.

```typescript
// src/config.ts
import { z } from 'zod';

const ConfigSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_ACCESS_SECRET: z.string().min(32),
  JWT_REFRESH_SECRET: z.string().min(32),
  SMTP_PASSWORD: z.string().min(1),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
});

function loadConfig() {
  const result = ConfigSchema.safeParse(process.env);

  if (!result.success) {
    console.error('Configuração inválida:');
    for (const issue of result.error.issues) {
      console.error(`  ${issue.path.join('.')}: ${issue.message}`);
    }
    process.exit(1);
  }

  return result.data;
}

export const config = loadConfig();
```

Este padrão faz a aplicação encerrar na inicialização se algum secret obrigatório estiver ausente ou malformado, em vez de falhar silenciosamente em tempo de execução.

---

### Gerando secrets fortes

```typescript
import crypto from 'crypto';

// Para secrets de assinatura JWT, API keys, CSRF tokens
export function generateSecret(bytes = 32): string {
  return crypto.randomBytes(bytes).toString('base64url');
}

// Gere um novo secret JWT
// node -e "import('crypto').then(c => console.log(c.randomBytes(64).toString('base64url')))"
```

Para tokens de uso único (verificação de e-mail, redefinição de senha):

```typescript
export function generateOTP(length = 6): string {
  // Inteiro criptograficamente seguro no intervalo [0, 10^length)
  const max = Math.pow(10, length);
  const random = crypto.randomInt(max);
  return random.toString().padStart(length, '0');
}

export function generateResetToken(): string {
  return crypto.randomBytes(32).toString('hex'); // string hex de 64 caracteres
}
```

---

### Gerenciamento de arquivos .env

```bash
# .env.example — commitado no git, sem valores reais
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
JWT_ACCESS_SECRET=change-me-minimum-32-characters-long
JWT_REFRESH_SECRET=change-me-different-from-access-secret
STRIPE_SECRET_KEY=sk_test_...

# .env — NÃO commitado no git (no .gitignore)
DATABASE_URL=postgresql://prod_user:actual_password@db.prod.example.com:5432/myapp
JWT_ACCESS_SECRET=generated-64-char-secret-here
```

```bash
# .gitignore — sempre inclua esses
.env
.env.local
.env.production
*.key
*.pem
```

Use `dotenv` em desenvolvimento:

```typescript
// src/index.ts — carregue dotenv apenas em desenvolvimento
if (process.env.NODE_ENV !== 'production') {
  const { config } = await import('dotenv');
  config();
}

// Em produção, injete variáveis de ambiente via sua plataforma (Docker, K8s, etc.)
```

---

### Padrão de rotação de secrets

Rotacionar secrets sem downtime exige suportar tanto o valor antigo quanto o novo durante uma janela de transição.

```typescript
// src/lib/token.ts — suporte a múltiplos secrets de assinatura durante a rotação
const CURRENT_SECRET = process.env.JWT_ACCESS_SECRET!;
const PREVIOUS_SECRET = process.env.JWT_ACCESS_SECRET_PREVIOUS; // opcional

export function verifyAccessToken(token: string) {
  // Tenta o secret atual primeiro
  try {
    return jwt.verify(token, CURRENT_SECRET);
  } catch (err) {
    // Se o atual falhar e tivermos um secret anterior, tente-o
    if (PREVIOUS_SECRET) {
      try {
        return jwt.verify(token, PREVIOUS_SECRET);
      } catch {
        // ambos falharam
      }
    }
    throw err;
  }
}
```

**Procedimento de rotação:**
1. Gere um novo secret
2. Defina `JWT_ACCESS_SECRET_PREVIOUS = valor_antigo`
3. Defina `JWT_ACCESS_SECRET = novo_valor`
4. Deploy — tokens assinados com o secret antigo ainda são aceitos
5. Aguarde todos os tokens antigos expirarem (15 minutos para access tokens)
6. Remova `JWT_ACCESS_SECRET_PREVIOUS`
7. Deploy novamente

---

### Integração com HashiCorp Vault (Node.js)

```typescript
// src/lib/vault.ts
import vault from 'node-vault';

const client = vault({
  apiVersion: 'v1',
  endpoint: process.env.VAULT_ADDR!,
  token: process.env.VAULT_TOKEN!,
});

export async function getSecret(path: string): Promise<Record<string, string>> {
  const result = await client.read(`secret/data/${path}`);
  return result.data.data;
}

// Uso na inicialização
const dbSecrets = await getSecret('myapp/database');
const connectionString = `postgresql://${dbSecrets.username}:${dbSecrets.password}@${dbSecrets.host}/${dbSecrets.name}`;
```

Em produção, use AppRole ou método de autenticação Kubernetes em vez de um `VAULT_TOKEN` estático.

---

### AWS Secrets Manager

```typescript
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: 'us-east-1' });

export async function getAwsSecret(secretName: string): Promise<Record<string, string>> {
  const command = new GetSecretValueCommand({ SecretId: secretName });
  const response = await client.send(command);

  if (!response.SecretString) {
    throw new Error(`Secret ${secretName} não tem valor string`);
  }

  return JSON.parse(response.SecretString);
}

// Na inicialização — armazene os secrets em cache na memória pelo tempo de vida do processo
let cachedSecrets: Record<string, string> | null = null;

export async function loadSecrets(): Promise<void> {
  cachedSecrets = await getAwsSecret('myapp/production');
}
```

---

### GitHub Actions — secrets em CI/CD

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          JWT_ACCESS_SECRET: ${{ secrets.JWT_ACCESS_SECRET }}
        run: |
          # Secrets são injetados como variáveis de ambiente — nunca exibidos nos logs
          npm run deploy
```

Secrets configurados na interface do GitHub em Settings > Secrets são mascarados em toda saída de log. Nunca use `echo $SECRET` — mesmo secrets mascarados não devem ser impressos.

---

### Auditoria: encontrando secrets na sua codebase

```bash
# Use trufflehog para escanear secrets no histórico git
npx trufflehog git file://. --only-verified

# Use gitleaks para um scan mais rápido
gitleaks detect --source . -v
```

Execute essas ferramentas em pre-commit hooks e no CI para capturar commits acidentais.

---

## Padrões Comuns e Boas Práticas

- **Valide todos os secrets na inicialização** — falhe rapidamente se a configuração obrigatória estiver ausente
- **Um secret por serviço/finalidade** — nunca reutilize API keys entre ambientes ou serviços
- **Rotacione secrets regularmente** — no mínimo, rotacione após a saída de qualquer membro da equipe
- **Use tokens de curta duração** — prefira tokens que expiram a chaves estáticas de longa duração
- **Audite o acesso a secrets** — secret managers na nuvem registram cada leitura; revise os logs
- **Nunca logue secrets** — use a opção `redact` do `pino` para remover campos sensíveis
- **Use `.env.example`** — documente os secrets necessários sem expor os valores

---

## Anti-Padrões a Evitar

- Commitar arquivos `.env` no controle de versão (mesmo em repos "privados" — repos privados também sofrem brechas)
- Armazenar secrets em scripts do `package.json` ou inline no `docker-compose.yml`
- Passar secrets como query parameters de URL (`?api_key=secret`)
- Enviar secrets por e-mail ou Slack (use um gerenciador de senhas)
- Usar o mesmo secret em desenvolvimento, staging e produção
- Rotacionar secrets alterando apenas um caractere no final
- Depender de segurança por obscuridade (esconder a chave em um comentário ou arquivo de configuração)

---

## Debugging e Resolução de Problemas

**"A aplicação encerra com 'Configuração inválida' na inicialização"**
Execute a validação de configuração manualmente: `node -e "require('./src/config.js')"` e leia as mensagens de erro do Zod. Elas informarão exatamente quais variáveis de ambiente estão ausentes ou malformadas.

**"Meu secret está aparecendo nos logs"**
Configure o `pino` com um array `redact`. Campos comuns para redact: `['password', 'token', 'secret', 'authorization', 'cookie', '*.password', '*.token']`.

**"Rotacionar o secret JWT deslogou todos os usuários"**
Você não suportou uma janela de transição. Implemente o padrão de secret duplo (atual + anterior) e aguarde os tokens existentes expirarem antes de remover o secret antigo.

---

## Cenários do Mundo Real

**Cenário: Lidando com uma API key vazada**

1. Revogue imediatamente a chave no dashboard do provedor
2. Gere e faça deploy de uma nova chave
3. Verifique nos logs se houve uso não autorizado com a chave vazada
4. Escaneie o histórico git com trufflehog para confirmar que não há outros vazamentos
5. Rotacione todos os secrets relacionados (o secret comprometido pode ter dado acesso a outros)
6. Adicione um pre-commit hook para prevenir vazamentos futuros

**Cenário: Gerenciamento de secrets em múltiplos ambientes**

```
development: .env (no gitignore) com valores dummy/locais
staging:     GitHub Actions secrets (injetados no momento do deploy)
production:  AWS Secrets Manager (buscados em tempo de execução)
```

Cada ambiente tem secrets independentes. Uma chave de dev comprometida não pode ser usada em produção.

---

## Leitura Complementar

- [12-Factor App: Config](https://12factor.net/config)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Documentação do HashiCorp Vault](https://developer.hashicorp.com/vault/docs)
- [AWS Secrets Manager best practices](https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html)

---

## Resumo

A regra cardinal do gerenciamento de secrets é simples: secrets nunca devem entrar no controle de versão. A partir daí, a disciplina se constrói em camadas — gere secrets aleatórios fortes, valide-os na inicialização, injete-os via variáveis de ambiente ou um secret manager, rotacione-os regularmente e audite seu uso. Para a maioria das aplicações, variáveis de ambiente mais GitHub Actions secrets são suficientes. Ao escalar, adote um secret manager (Vault ou AWS Secrets Manager) que oferece auditoria centralizada e rotação automática. O investimento em higiene de secrets é pequeno comparado ao custo de uma brecha.
