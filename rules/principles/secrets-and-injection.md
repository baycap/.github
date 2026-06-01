# Secrets e Injection

Princípios universais de segurança. Aplicam-se a qualquer linguagem que toque dados de usuário, banco de dados, ou armazene credenciais. Segurança não tem linguagem favorita — os exemplos abaixo cobrem Python e TypeScript/JavaScript, mas o princípio vale igualmente para Go, Java, Rust, etc.

Somos uma gestora de crédito: violações destas três regras não são "estilo ruim" — são risco regulatório e financeiro concreto.

---

## CLAUDE_MD:sql-injection-via-string-concatenation

**Princípio:** queries SQL nunca devem ser construídas concatenando ou interpolando input externo no texto da query. Use sempre parâmetros (`?`, `:name`, `$1`) ou um query builder/ORM que parametriza por padrão.

**Severidade:** `blocker` (`SEC:sql-injection`).

**Aplica em:** qualquer código que monta SQL. Vale para raw SQL, ORMs que aceitam fragmentos crus (`text(...)`, `Prisma.sql\`...\``), e templates.

### ❌ Python — concatenação ou f-string com input externo

```py
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
cursor.execute("SELECT * FROM users WHERE name = '" + name + "'")
session.execute(text(f"DELETE FROM orders WHERE status = '{status}'"))
```

### ✅ Python — parametrizado

```py
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
session.execute(
    text("DELETE FROM orders WHERE status = :status"),
    {"status": status},
)
# Ou via ORM — parametrização implícita:
session.query(User).filter(User.id == user_id).first()
```

### ❌ TypeScript — template literal em query

```ts
const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`);
const rows = await db.$queryRawUnsafe(`SELECT * FROM accounts WHERE id = ${id}`);
```

### ✅ TypeScript — parâmetros explícitos

```ts
const result = await db.query("SELECT * FROM users WHERE email = $1", [email]);
// Ou via Prisma — parametrização implícita:
await prisma.user.findUnique({ where: { email } });
// Quando precisar de raw com Prisma, use a tag template:
await prisma.$queryRaw`SELECT * FROM accounts WHERE id = ${id}`;
```

**Quando NÃO se aplica:** SQL totalmente estático (sem nenhum input externo no texto). Mesmo assim, prefira a API do query builder do ORM em vez de raw — fica imune a regressões futuras quando alguém "só vai adicionar um filtro dinâmico aqui".

---

## CLAUDE_MD:hardcoded-credentials

**Princípio:** senhas, tokens, chaves de API, certificados, secrets em geral nunca vão no código-fonte. Vivem em secret manager (AWS Secrets Manager, Vault, Doppler) e chegam ao processo via configuração carregada em runtime.

**Severidade:** `blocker` (`SEC:hardcoded-credential`).

**Aplica em:** strings literais atribuídas a variáveis com nome de credencial (`password`, `token`, `api_key`, `secret`, `private_key`, `client_secret`), ou strings com aparência de token/chave em contexto de autenticação.

### ❌ Python

```py
DATABASE_PASSWORD = "P@ssw0rd!2024"
OPENAI_API_KEY = "sk-proj-abc123..."

def get_admin_password() -> str:
    return "admin123"

requests.post(url, headers={"Authorization": "Bearer eyJhbGciOi..."})
```

### ✅ Python

```py
from src.core.settings import settings

DATABASE_PASSWORD = settings.database_password  # carrega via Pydantic Settings + Secrets Manager
requests.post(url, headers={"Authorization": f"Bearer {settings.api_token}"})
```

### ❌ TypeScript

```ts
const STRIPE_KEY = "sk_live_4242abcdef...";
const config = { jwtSecret: "my-super-secret-key-123" };
```

### ✅ TypeScript

```ts
const STRIPE_KEY = process.env.STRIPE_KEY;
if (!STRIPE_KEY) throw new Error("STRIPE_KEY not configured");

const config = { jwtSecret: process.env.JWT_SECRET ?? throwMissing("JWT_SECRET") };
```

**Quando NÃO se aplica:**

- Exemplos em docstring/comentário que claramente não são credenciais reais (`"Bearer <your-token-here>"`).
- Valores em testes que são óbvia e exclusivamente de teste (`"test-token"`, `"fake-key-for-unit-test"`). Mesmo assim, prefira gerar via fixture/factory para evitar que alguém copie e cole em produção.
- Strings com formato de token mas que são identificadores públicos (ex.: client IDs OAuth, que não são secretos).

---

## CLAUDE_MD:silent-exception-swallow

**Princípio:** capturar uma exceção e ignorar (`pass`, `continue`, return silencioso, `catch {}` vazio) esconde bugs reais e falhas operacionais. Toda captura precisa pelo menos um destes três comportamentos:

1. **Tratamento legítimo no domínio** — com comentário curto explicando o porquê (ex.: fallback documentado para entrada ausente).
2. **`logger.exception(...)` antes de re-raise ou continue** — preserva o stack trace nos logs e permite seguir.
3. **Captura nominal de exceção esperada com path alternativo explícito** — `except FileNotFoundError: return default_config()`.

Não atende esses critérios → não capture. Deixe propagar.

**Severidade:** `major` (`SEC:silent-swallow`).

**Aplica em:** try/catch em qualquer linguagem. Inclui `except:` sem tipo (bare except) e captura genérica de `Exception`/`Error`.

### ❌ Python — silencioso

```py
try:
    process(item)
except Exception:
    pass

try:
    risky()
except:
    continue
```

### ✅ Python — batch resiliente, com log

```py
for item in items:
    try:
        process(item)
    except ProcessingError:
        logger.exception("Failed to process item id=%s", item.id)
        failed.append(item.id)
        continue
```

### ✅ Python — captura nominal com fallback documentado

```py
def load_config() -> Config:
    try:
        return Config.parse_file(CONFIG_PATH)
    except FileNotFoundError:
        # primeira execução, ainda não há arquivo de config
        return Config.default()
```

### ❌ TypeScript

```ts
try {
  await process(item);
} catch {
  /* ignored */
}

try {
  await api.call();
} catch (e) {
  // swallow
}
```

### ✅ TypeScript

```ts
try {
  await process(item);
} catch (err) {
  logger.error("Failed to process item", { itemId: item.id, err });
  throw err;
}
```

**Quando NÃO se aplica:** captura nominal de uma exceção específica que faz parte do contrato do código (ex.: `cache.get()` que retorna `null` em miss). O sinal de alerta é exceção *genérica* + ausência de log + ausência de path alternativo claro.
