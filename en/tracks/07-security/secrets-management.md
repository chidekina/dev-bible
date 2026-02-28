# Secrets Management

## Overview

A secret is any value that grants access to a system: API keys, database passwords, JWT signing keys, OAuth client secrets, encryption keys. Mishandling secrets is one of the most common causes of data breaches — and most of those breaches are preventable with a small set of consistent practices. This chapter covers how to store, access, rotate, and audit secrets in a Node.js/TypeScript application.

---

## Prerequisites

- Basic Node.js and TypeScript
- Understanding of environment variables
- Familiarity with CI/CD pipelines (GitHub Actions or similar)

---

## Core Concepts

### The secret lifecycle

```
Generate → Store → Access → Rotate → Revoke
```

Each stage has failure modes:

| Stage | Common failure |
|-------|---------------|
| Generate | Weak randomness, predictable values |
| Store | Committed to git, stored in plaintext |
| Access | Logged, passed in URL query params |
| Rotate | Never rotated, same secret for years |
| Revoke | No mechanism to revoke compromised secrets |

### Secrets hierarchy

1. **Environment variables** — simplest, works everywhere, fragile at scale
2. **Secret managers** (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault) — centralized, auditable, supports rotation
3. **Sealed secrets / encrypted vaults** — for Kubernetes, secrets encrypted at rest in source control

---

## Hands-On Examples

### Basic environment variable pattern

Never hardcode secrets in source code. Load them from the environment and validate at startup.

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
    console.error('Invalid configuration:');
    for (const issue of result.error.issues) {
      console.error(`  ${issue.path.join('.')}: ${issue.message}`);
    }
    process.exit(1);
  }

  return result.data;
}

export const config = loadConfig();
```

This pattern crashes the application at startup if any required secret is missing or malformed, rather than failing silently at runtime.

---

### Generating strong secrets

```typescript
import crypto from 'crypto';

// For JWT signing secrets, API keys, CSRF tokens
export function generateSecret(bytes = 32): string {
  return crypto.randomBytes(bytes).toString('base64url');
}

// Generate a new JWT secret
// node -e "import('crypto').then(c => console.log(c.randomBytes(64).toString('base64url')))"
```

For one-time use tokens (email verification, password reset):

```typescript
export function generateOTP(length = 6): string {
  // Cryptographically secure integer in range [0, 10^length)
  const max = Math.pow(10, length);
  const random = crypto.randomInt(max);
  return random.toString().padStart(length, '0');
}

export function generateResetToken(): string {
  return crypto.randomBytes(32).toString('hex'); // 64-char hex string
}
```

---

### .env file management

```bash
# .env.example — committed to git, no real values
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
JWT_ACCESS_SECRET=change-me-minimum-32-characters-long
JWT_REFRESH_SECRET=change-me-different-from-access-secret
STRIPE_SECRET_KEY=sk_test_...

# .env — NOT committed to git (in .gitignore)
DATABASE_URL=postgresql://prod_user:actual_password@db.prod.example.com:5432/myapp
JWT_ACCESS_SECRET=generated-64-char-secret-here
```

```bash
# .gitignore — always include these
.env
.env.local
.env.production
*.key
*.pem
```

Use `dotenv` in development:

```typescript
// src/index.ts — only load dotenv in development
if (process.env.NODE_ENV !== 'production') {
  const { config } = await import('dotenv');
  config();
}

// In production, inject env vars via your platform (Docker, K8s, etc.)
```

---

### Secret rotation pattern

Rotating secrets without downtime requires supporting both the old and new value for a transition window.

```typescript
// src/lib/token.ts — support multiple signing secrets during rotation
const CURRENT_SECRET = process.env.JWT_ACCESS_SECRET!;
const PREVIOUS_SECRET = process.env.JWT_ACCESS_SECRET_PREVIOUS; // optional

export function verifyAccessToken(token: string) {
  // Try current secret first
  try {
    return jwt.verify(token, CURRENT_SECRET);
  } catch (err) {
    // If current fails and we have a previous secret, try it
    if (PREVIOUS_SECRET) {
      try {
        return jwt.verify(token, PREVIOUS_SECRET);
      } catch {
        // both failed
      }
    }
    throw err;
  }
}
```

**Rotation procedure:**
1. Generate new secret
2. Set `JWT_ACCESS_SECRET_PREVIOUS = old_value`
3. Set `JWT_ACCESS_SECRET = new_value`
4. Deploy — tokens signed with old secret are still accepted
5. Wait for all old tokens to expire (15 minutes for access tokens)
6. Remove `JWT_ACCESS_SECRET_PREVIOUS`
7. Deploy again

---

### HashiCorp Vault integration (Node.js)

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

// Usage at startup
const dbSecrets = await getSecret('myapp/database');
const connectionString = `postgresql://${dbSecrets.username}:${dbSecrets.password}@${dbSecrets.host}/${dbSecrets.name}`;
```

For production, use AppRole or Kubernetes auth method instead of a static `VAULT_TOKEN`.

---

### AWS Secrets Manager

```typescript
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: 'us-east-1' });

export async function getAwsSecret(secretName: string): Promise<Record<string, string>> {
  const command = new GetSecretValueCommand({ SecretId: secretName });
  const response = await client.send(command);

  if (!response.SecretString) {
    throw new Error(`Secret ${secretName} has no string value`);
  }

  return JSON.parse(response.SecretString);
}

// At startup — cache secrets in memory for the lifetime of the process
let cachedSecrets: Record<string, string> | null = null;

export async function loadSecrets(): Promise<void> {
  cachedSecrets = await getAwsSecret('myapp/production');
}
```

---

### GitHub Actions — secrets in CI/CD

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
          # Secrets are injected as env vars — never echoed in logs
          npm run deploy
```

Secrets set in GitHub's UI under Settings > Secrets are masked in all log output. Never `echo $SECRET` — even masked secrets should not be printed.

---

### Audit: finding secrets in your codebase

```bash
# Use trufflehog to scan for secrets in git history
npx trufflehog git file://. --only-verified

# Use gitleaks for a faster scan
gitleaks detect --source . -v
```

Run these in pre-commit hooks and in CI to catch accidental commits.

---

## Common Patterns & Best Practices

- **Validate all secrets at startup** — fail fast if required config is missing
- **One secret per service/purpose** — never reuse API keys across environments or services
- **Rotate secrets regularly** — at minimum, rotate after any team member departure
- **Use short-lived tokens** — prefer tokens that expire over long-lived static keys
- **Audit secret access** — cloud secret managers log every read; review the logs
- **Never log secrets** — use `pino`'s `redact` option to strip sensitive fields
- **Use `.env.example`** — document required secrets without exposing values

---

## Anti-Patterns to Avoid

- Committing `.env` files to version control (even in "private" repos — private repos get breached too)
- Storing secrets in `package.json` scripts or `docker-compose.yml` inline
- Passing secrets as URL query parameters (`?api_key=secret`)
- Sending secrets in email or Slack (use a password manager instead)
- Using the same secret in development, staging, and production
- Rotating secrets by changing one character at the end
- Relying on security through obscurity (hiding the key in a comment or config file)

---

## Debugging & Troubleshooting

**"The app crashes with 'Invalid configuration' at startup"**
Run the config validation manually: `node -e "require('./src/config.js')"` and read the Zod error messages. They will tell you exactly which env vars are missing or malformed.

**"My secret is showing up in logs"**
Configure `pino` with a `redact` array. Common fields to redact: `['password', 'token', 'secret', 'authorization', 'cookie', '*.password', '*.token']`.

**"Rotating the JWT secret logged all users out"**
You did not support a transition window. Implement the dual-secret pattern (current + previous) and wait for existing tokens to expire before removing the old secret.

---

## Real-World Scenarios

**Scenario: Handling a leaked API key**

1. Immediately revoke the key in the provider's dashboard
2. Generate and deploy a new key
3. Check logs for unauthorized usage with the leaked key
4. Scan git history with trufflehog to confirm there are no other leaks
5. Rotate all related secrets (the compromised secret may have given access to others)
6. Add a pre-commit hook to prevent future leaks

**Scenario: Multi-environment secret management**

```
development: .env (gitignored) with dummy/local values
staging:     GitHub Actions secrets (injected at deploy time)
production:  AWS Secrets Manager (fetched at runtime)
```

Each environment has independent secrets. A compromised dev key cannot be used in production.

---

## Further Reading

- [12-Factor App: Config](https://12factor.net/config)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [AWS Secrets Manager best practices](https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html)

---

## Summary

The cardinal rule of secrets management is simple: secrets never touch source control. From there, the discipline builds in layers — generate strong random secrets, validate them at startup, inject them via environment variables or a secrets manager, rotate them regularly, and audit their usage. For most applications, environment variables plus GitHub Actions secrets is sufficient. As you scale, adopt a secrets manager (Vault or AWS Secrets Manager) that provides centralized auditing and automatic rotation. The investment in secrets hygiene is small compared to the cost of a breach.
