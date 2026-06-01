# Error Handling

Princípios universais sobre como tratar erros — independentes de linguagem. Aplicam-se a Python, TypeScript, Go, Java, qualquer linguagem com tratamento de exceções ou retorno de erro explícito.

A regra mestre: **a informação sobre por que algo falhou é cara — não jogue fora**. Stack trace original, exceção encadeada, contexto operacional. Engenheiro que debuga em produção precisa dessas pistas.

> Existe overlap natural com `python.md` (seção "Exception Handling"), que mantém regras Python-specific (semântica de status HTTP, `str(e)` em `HTTPException`, etc.). Este arquivo cobre o **princípio universal** — o `python.md` cobre as **manifestações específicas** da linguagem.

---

## CLAUDE_MD:preserve-error-chain

**Princípio:** ao capturar uma exceção e relançar (ou criar uma nova exceção de domínio em cima), preserve a causa original. Stack trace e exceção encadeada permitem reconstruir o caminho do erro em produção. Reencapsular sem preservar a cadeia é equivalente a deletar metade da pista.

**Severidade:** `major` (`BUG:lost-error-context`).

**Aplica em:** qualquer ponto de código que captura uma exceção e relança como um novo tipo (tradução de erro de infraestrutura para erro de domínio, mapeamento HTTP, retry com transformação, etc.).

### ❌ Python — perde a cadeia (re-raise nu com nova exceção)

```py
try:
    s3_client.put_object(Bucket=bucket, Key=key, Body=data)
except ClientError:
    raise UploadFailedError(key)  # cadeia original perdida

# Pior: mascara o tipo da exceção original em texto solto
try:
    api.call()
except HTTPStatusError as e:
    raise IntegrationError(f"API failed: {e}")
```

### ✅ Python — `raise ... from <original>`

```py
try:
    s3_client.put_object(Bucket=bucket, Key=key, Body=data)
except ClientError as e:
    raise UploadFailedError(key) from e

try:
    api.call()
except HTTPStatusError as e:
    raise IntegrationError("Upstream API failed") from e
```

A sintaxe `raise X from Y` é a forma idiomática em Python — o traceback mostra "during handling of the above exception, another exception occurred", e ferramentas de observabilidade (Sentry, Datadog) capturam ambas.

### ❌ TypeScript — relança sem `cause`

```ts
try {
  await s3.putObject({ Bucket, Key, Body });
} catch (err) {
  throw new UploadFailedError(key);  // cadeia perdida
}

// Pior: serializa a exceção original em texto
try {
  await api.call();
} catch (err) {
  throw new Error(`API failed: ${err}`);
}
```

### ✅ TypeScript — `Error.cause` (ES2022+)

```ts
try {
  await s3.putObject({ Bucket, Key, Body });
} catch (err) {
  throw new UploadFailedError(key, { cause: err });
}

try {
  await api.call();
} catch (err) {
  throw new IntegrationError("Upstream API failed", { cause: err });
}
```

**Quando NÃO se aplica:**

- O erro original **não deve** ser exposto ao código que consumirá a exceção de domínio por razão deliberada (ex.: vazamento de informação interna). Mesmo nesses casos, prefira **logar** a exceção original com `logger.exception` ANTES de levantar a exceção sanitizada — preserva a pista no log mesmo que o cliente não veja.
- Captura nominal com path alternativo claro onde a exceção original não é "uma falha" e sim parte do contrato (ex.: `FileNotFoundError` em primeira execução — você está retornando default, não relançando).

---

## CLAUDE_MD:broad-exception-catch

**Princípio:** capturar `Exception` (Python), `Error` (TS/JS), `error` genérico (Go quando ignorado) **sem motivo declarado** mascara bugs. Capture o tipo específico que você sabe lidar. Reserve a captura genérica para fronteiras explícitas do sistema (handlers de fila/worker que devem isolar uma mensagem ruim do resto do batch, top-level handlers de aplicação que precisam transformar qualquer falha em telemetria) — e mesmo nesses casos, com `logger.exception(...)` obrigatório.

**Severidade:** `major` (`BUG:broad-catch`).

**Aplica em:** qualquer linguagem com try/catch genérico. Em Go (sem exceções), o equivalente é `if err != nil { return nil }` sem repropagar — mesmo princípio.

### ❌ Python — captura genérica sem motivo

```py
try:
    payload = parse_request(body)
    user = repository.get_user(payload.user_id)
except Exception as e:
    logger.error("Something failed")
    return None
```

Esconde: `JSONDecodeError`, `ValidationError`, `DatabaseError`, `AttributeError` (bug de programação). O tratamento mistura todos sem distinção.

### ✅ Python — captura específica

```py
try:
    payload = parse_request(body)
except ValidationError as e:
    raise InvalidPayloadError() from e

try:
    user = repository.get_user(payload.user_id)
except UserNotFoundError:
    return None  # contrato do endpoint: usuário ausente retorna 404
```

### ✅ Python — fronteira legítima de batch (with logger.exception)

```py
for message in messages:
    try:
        handler.process(message)
    except Exception:
        logger.exception("Failed to process message id=%s", message.id)
        dlq.send(message)
```

Captura genérica é aceita aqui porque (a) é **fronteira explícita** (handler de fila) e (b) há **log estruturado** + **path alternativo claro** (DLQ).

### ❌ TypeScript — catch genérico silencioso

```ts
try {
  const payload = parseRequest(body);
  const user = await repository.getUser(payload.userId);
} catch (err) {
  console.error("Something failed");
  return null;
}
```

### ✅ TypeScript — checagem de tipo no catch

```ts
try {
  const payload = parseRequest(body);
} catch (err) {
  if (err instanceof ZodError) {
    throw new InvalidPayloadError({ cause: err });
  }
  throw err;
}
```

**Quando NÃO se aplica:**

- **Batch resiliente por item** (handlers SQS/EventBridge/Kafka, jobs em loop sobre arquivo grande): captura genérica + `logger.exception` + caminho de DLQ/skip.
- **Top-level handler de framework** (FastAPI `app.exception_handler(Exception, ...)`, middleware de Next.js): última linha de defesa transformando qualquer erro em telemetria + resposta HTTP genérica. Mesmo aqui, exceções específicas devem ter handlers próprios *antes*.

Fora desses dois cenários, captura genérica é sinal de "não pensei nos modos de falha". Refator.
