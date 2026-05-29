# Code Complexity

Princípios universais sobre **estrutura do código** (não Big-O de execução). Aplicam-se a qualquer linguagem: função muito longa, muitos caminhos independentes, profundidade de aninhamento alta.

A regra mestre: **carga cognitiva é finita**. Função que cabe na cabeça do leitor em uma passada é função fácil de testar, refatorar e debugar. Função que não cabe gera bugs quando alguém a edita seis meses depois sem entender o todo.

Os thresholds abaixo são pontos de partida pragmáticos — não verdade matemática. O reviewer aplica como sinal, não como veredito; refator é convite, não obrigação automática.

---

## CLAUDE_MD:cyclomatic-complexity

**Princípio:** cada caminho independente que o código pode percorrer (cada `if`, `elif`, `else`, `for`, `while`, `except`, `case`, operador booleano `and`/`or` em expressão) soma 1 à complexidade ciclomática (McCabe). Função com muitos caminhos é difícil de testar exaustivamente e difícil de seguir no debug.

**Severidade:**

- `major` quando complexidade ≥ **10** (linha original do McCabe).
- `blocker` quando complexidade ≥ **15** — refator é obrigatório, não opcional.

**Aplica em:** qualquer linguagem. Em Python o Ruff calcula com `C901` (configurável via `mccabe.max-complexity`). Em TypeScript o ESLint tem `complexity`. O reviewer cita o slug quando vê o padrão estrutural, mesmo em repos onde o lint não está ativo.

### ❌ Função com muitos branches

```py
def process_transaction(tx: Transaction, user: User, account: Account) -> Result:
    if tx.amount <= 0:
        return Result.invalid("amount")
    if tx.amount > account.limit:
        if user.is_premium and tx.amount < account.limit * 1.5:
            ...
        else:
            return Result.rejected("over_limit")
    if tx.currency != account.currency:
        if tx.currency in SUPPORTED_CURRENCIES:
            ...
        elif user.is_premium:
            ...
        else:
            return Result.rejected("currency")
    if user.is_blocked:
        return Result.rejected("blocked")
    if account.is_frozen:
        return Result.rejected("frozen")
    if not user.has_verified_email:
        return Result.pending_verification()
    # ... mais 8 branches
    return Result.approved()
```

### ✅ Decomposição em funções menores

```py
def process_transaction(tx: Transaction, user: User, account: Account) -> Result:
    if (rejection := _check_eligibility(user, account)) is not None:
        return rejection
    if (rejection := _check_amount(tx, account, user)) is not None:
        return rejection
    if (rejection := _check_currency(tx, account, user)) is not None:
        return rejection
    return Result.approved()

def _check_eligibility(user: User, account: Account) -> Result | None:
    if user.is_blocked:
        return Result.rejected("blocked")
    if account.is_frozen:
        return Result.rejected("frozen")
    if not user.has_verified_email:
        return Result.pending_verification()
    return None
```

Cada função fica com complexidade < 5, é testável isoladamente, e o nome explica o que cada checagem faz.

**Quando NÃO se aplica:**

- **State machine explícita / dispatcher** — funções com muitos casos onde cada caso é trivial e o "many branches" é a essência do problema (parser de protocolo, conversor de moeda, switch de tipos). Documente com comentário `# noqa` apontando o padrão. Idealmente extraia para um dict de handlers ou `match`.
- **Validador linear** — função que faz N checagens independentes do mesmo objeto. Mesmo nesse caso, considere extrair as checagens para funções nomeadas — fica fácil entender o "o que" sem ler o "como".
- **Função gerada por código** (fixture builder, schema serializer).

---

## CLAUDE_MD:function-length

**Princípio:** função longa esconde múltiplas responsabilidades. Mesmo sem complexidade ciclomática alta, função com 80+ linhas é difícil de revisar, testar isoladamente, e reusar partes.

**Severidade:**

- `major` quando ≥ **60 linhas** efetivas (excluindo docstring, blank lines, comments).
- `blocker` quando ≥ **100 linhas**.

**Aplica em:** qualquer linguagem.

### ❌ Função orquestradora gigante

```py
async def import_csv(file_path: str) -> ImportResult:
    # 120 linhas misturando:
    # - leitura do arquivo
    # - parsing de cada linha
    # - validação por linha
    # - enrich via API externa
    # - persistência
    # - logging
    # - construção do relatório
    ...
```

### ✅ Decomposição por responsabilidade

```py
async def import_csv(file_path: str) -> ImportResult:
    raw_rows = await _read_rows(file_path)
    parsed = [_parse(row) for row in raw_rows]
    valid, invalid = _validate(parsed)
    enriched = await _enrich(valid)
    persisted = await _persist(enriched)
    return _build_report(persisted=persisted, invalid=invalid)
```

Cada helper fica testável separadamente. A função principal lê como uma narrativa.

**Quando NÃO se aplica:**

- Funções compostas principalmente por **dados** (long literal, configuração inline, fixture grande) — extrair pode piorar a leitura.
- **Migrations/scripts** linearmente sequenciais sem reuso natural — preferência por clareza linear vs decomposição forçada.

---

## CLAUDE_MD:nesting-depth

**Princípio:** cada nível de aninhamento (`if` dentro de `for` dentro de `try` dentro de `with`) dobra a carga cognitiva de leitura. Para entender o que acontece na linha mais funda, o leitor precisa manter mentalmente todas as condições acima.

**Severidade:**

- `major` quando profundidade ≥ **4 níveis** dentro de uma função.
- `blocker` quando profundidade ≥ **5 níveis**.

**Aplica em:** qualquer linguagem.

### ❌ Profundidade 5

```py
def process_orders(orders):
    for order in orders:
        if order.is_valid:
            for item in order.items:
                if item.in_stock:
                    try:
                        result = process(item)
                        if result.ok:
                            persist(result)  # 5 níveis dentro
                    except ProcessingError:
                        log.exception(...)
```

### ✅ Early return / guard clauses

```py
def process_orders(orders: list[Order]) -> None:
    for order in orders:
        _process_order(order)

def _process_order(order: Order) -> None:
    if not order.is_valid:
        return
    for item in order.items:
        _process_item(item)

def _process_item(item: Item) -> None:
    if not item.in_stock:
        return
    try:
        result = process(item)
    except ProcessingError:
        log.exception("Processing failed for item id=%s", item.id)
        return
    if result.ok:
        persist(result)
```

Profundidade máxima agora: 2. Cada função tem responsabilidade clara, cada `return` cedo elimina um caminho da cabeça do leitor.

**Quando NÃO se aplica:**

- **Parsers / state machines** onde a hierarquia natural do problema espelha a hierarquia do código (parser de YAML aninhado, walker de AST). Documente.
- **Context managers que naturalmente aninham** (`with file:` dentro de `with lock:` dentro de `with transaction:`) — `with` blocks contam baixo na carga cognitiva porque não introduzem branching. Considere usar `ExitStack` ou helper se ficar ruidoso.

---

## Como o reviewer mede

Para todas as três regras, o reviewer:

1. Lê o trecho via `get_file_slice`.
2. Conta paths/níveis/linhas manualmente sobre o código observado — modelo é confiável para funções até ~150 linhas.
3. Cita `CLAUDE_MD:<slug>` com a métrica observada e o threshold da regra.

Acima de 150 linhas o modelo tende a subestimar — mas função de 150 linhas já viola `function-length` antes de chegar à contagem manual, então o gate cai antes.
