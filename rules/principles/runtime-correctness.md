# Runtime Correctness

Princípios universais sobre bugs estruturais que **só aparecem em tempo de execução** — em geral no caminho menos exercitado pelos testes. Não são problemas de estilo; são problemas que vão crashar ou produzir resultado errado quando o caminho específico for percorrido.

Linter pega esses padrões em Python (Ruff `B023`, `F821`) e TypeScript em parte (compilador pega muitos `F821` equivalentes), mas o reviewer ainda assim deve sinalizar quando aparecer — porque alguns repos têm linters relaxados, e revisor humano lê com contexto que o linter não tem.

---

## CLAUDE_MD:loop-variable-captured-in-closure

**Princípio:** ao criar uma função (lambda, callback, closure, async handler) dentro de um loop que referencia a variável de loop, a função vai capturar a **variável**, não o **valor naquele iteração**. Quando ela for chamada depois, vai ler o valor final do loop. Bug clássico de late-binding.

**Severidade:** `major` (`BUG:loop-variable-closure`).

**Aplica em:** Python (qualquer versão), JavaScript pré-`let` (apenas com `var`), Go antes da 1.22, qualquer linguagem com closures de escopo léxico. Em Python a captura por nome é a regra, então o bug é especialmente fácil de cometer.

### ❌ Python — todas as funções recebem o último `x`

```py
handlers = []
for x in items:
    handlers.append(lambda: process(x))

# Quando handlers[0]() rodar, vai usar o último x, não o primeiro
```

Variantes do mesmo bug:

```py
# Lambdas em comprehension
callbacks = [lambda: print(i) for i in range(5)]
# Todos imprimem 4

# Closure async
tasks = []
for url in urls:
    tasks.append(asyncio.create_task(fetch_and_log(url)))  # OK se fetch usa url imediatamente
    tasks.append(asyncio.create_task(asyncio.sleep(0).then(lambda: log(url))))  # bug
```

### ✅ Python — bind explícito via argumento default

```py
handlers = []
for x in items:
    handlers.append(lambda x=x: process(x))
```

Ou (mais legível) extraia para função:

```py
def make_handler(x):
    return lambda: process(x)

handlers = [make_handler(x) for x in items]
```

### ❌ JavaScript — `var` em closure

```js
var handlers = [];
for (var i = 0; i < items.length; i++) {
  handlers.push(() => process(items[i]));
}
// items[i] vai ser items.length (undefined) quando handler rodar
```

### ✅ JavaScript — `let` cria binding por iteração

```js
const handlers = [];
for (let i = 0; i < items.length; i++) {
  handlers.push(() => process(items[i]));
}
```

Em TypeScript moderno o problema raramente aparece (lint pega `var` no loop), mas vale o reviewer apontar quando aparece — geralmente é código portado ou auto-gerado.

**Quando NÃO se aplica:**

- A função criada **é chamada imediatamente dentro do mesmo iteração** (ex.: `[transform(x) for x in items]` onde `transform` é executada na hora — não há captura tardia).
- A variável de loop **não é referenciada** no corpo da closure (a função não usa `x` mesmo).

---

## CLAUDE_MD:reference-to-undefined-name

**Princípio:** referenciar um nome (variável, função, classe, atributo) que não está definido no escopo é um `NameError` em runtime — bug que só aparece quando o caminho de execução chega ali. Tipos comuns: nome digitado errado, import esquecido, símbolo removido por refactor parcial, nome definido só num branch do `if`.

**Severidade:** `blocker` (`BUG:undefined-name`).

**Aplica em:** linguagens dinâmicas (Python, JS sem TypeScript, Ruby). Em TS/Go o compilador já bloqueia — mas o reviewer ainda flag pode acontecer em strings de query, template literals, dynamic imports, etc.

### ❌ Python — variável definida só em um branch

```py
def process(item: Item) -> Result:
    if item.is_valid:
        result = compute(item)
    return result  # NameError se item.is_valid for False
```

### ✅ Python — definir em todos os caminhos OU retornar cedo

```py
def process(item: Item) -> Result:
    if not item.is_valid:
        return Result.invalid()
    return compute(item)
```

### ❌ Python — typo / símbolo removido

```py
from src.adapters.email import send_email

def notify_user(user: User) -> None:
    sned_email(user.email, "Hello")  # typo, vai crashar quando chamado
```

### ❌ Python — `from X import Y` ausente após refactor

```py
def process_order(order: Order) -> None:
    payment = PaymentProcessor()  # PaymentProcessor não está mais importado
    payment.charge(order)
```

### ✅ Python — Ruff `F821` pega tudo isso

Configure `ruff check --select F` no pre-commit. Quando aparecer no reviewer, é porque o linter local foi pulado.

### ❌ TypeScript — em string template para query (linter cego)

```ts
const userId = req.params.id;
const result = await prisma.$queryRaw`SELECT * FROM users WHERE id = ${usrId}`;
// usrId é typo; query é raw então TS não verifica
```

**Quando NÃO se aplica:**

- Nomes resolvidos dinamicamente em runtime (`globals()[name]`, `eval`, plugin system) — fora de escopo desta regra. Esses casos justificam comentário explicando *como* o nome é resolvido.
- Atributos opcionais de objetos onde `getattr(obj, "name", default)` é o contrato (não é "indefinido", é "ausente com fallback").
