# Algorithmic Complexity

Princípios universais sobre complexidade computacional **em memória**. Aplicam-se a qualquer linguagem.

A regra mestre: **Big-O importa quando a entrada cresce.** Código que roda em 10ms para 100 itens pode levar 100s para 100k itens se o algoritmo for O(n²). Em fintech (volumes de transações, processamento de arquivos, reconciliação), a entrada cresce sem aviso.

> Diferença em relação a `data-access-and-batching.md`: aquele cobre **IO** (queries, requests). Este cobre **operações puramente em memória** (loops aninhados, lookups em lista, recomputação dentro de loop).

---

## CLAUDE_MD:nested-loops-quadratic-on-collection

**Princípio:** `for` aninhado sobre coleções (com cardinalidade dependente de input) cria custo O(n·m) — quadrático quando as coleções têm tamanho parecido. A regra geral: se o loop interno está procurando algo no externo, há um índice (`dict`, `set`, `Map`) que reduz para O(n+m).

**Severidade:** `major` (`PERF:nested-loops`). `blocker` quando o input externo controla a cardinalidade sem cap (risco de DoS / degradação em produção).

**Aplica em:** qualquer linguagem. O padrão é universal — só a sintaxe muda.

### ❌ Python — match O(n·m) em loop aninhado

```py
def reconcile(internal: list[Transaction], external: list[Transaction]) -> list[Match]:
    matches = []
    for tx in internal:
        for ext in external:
            if tx.reference == ext.reference and tx.amount == ext.amount:
                matches.append(Match(internal=tx, external=ext))
                break
    return matches
```

Com 50k transações de cada lado, são 2.5 bilhões de comparações.

### ✅ Python — index por chave composta

```py
def reconcile(internal: list[Transaction], external: list[Transaction]) -> list[Match]:
    external_by_key = {(ext.reference, ext.amount): ext for ext in external}
    return [
        Match(internal=tx, external=external_by_key[(tx.reference, tx.amount)])
        for tx in internal
        if (tx.reference, tx.amount) in external_by_key
    ]
```

Custo: O(n+m). 100k operações em vez de 2.5 bilhões.

### ❌ Python — deduplicação por busca linear

```py
def unique_by_id(items: list[Item]) -> list[Item]:
    result = []
    for item in items:
        if item.id not in [x.id for x in result]:  # O(k) por iteração
            result.append(item)
    return result
```

### ✅ Python — set de IDs vistos

```py
def unique_by_id(items: list[Item]) -> list[Item]:
    seen: set[str] = set()
    result: list[Item] = []
    for item in items:
        if item.id not in seen:
            seen.add(item.id)
            result.append(item)
    return result
```

### ❌ TypeScript — `find` dentro de `forEach`

```ts
const matches: Match[] = [];
internal.forEach(tx => {
  const ext = external.find(e => e.reference === tx.reference && e.amount === tx.amount);
  if (ext) matches.push({ internal: tx, external: ext });
});
```

### ✅ TypeScript — `Map` pré-construído

```ts
const externalByKey = new Map(
  external.map(e => [`${e.reference}|${e.amount}`, e]),
);
const matches = internal
  .filter(tx => externalByKey.has(`${tx.reference}|${tx.amount}`))
  .map(tx => ({
    internal: tx,
    external: externalByKey.get(`${tx.reference}|${tx.amount}`)!,
  }));
```

### Sinais auxiliares que o reviewer deve flagar

Mesmo que o padrão exato não seja "for-in-for clássico", os seguintes cheiros indicam complexidade ruim:

- **Chamada de método com custo linear dentro de loop**: `list.index(...)`, `list.count(...)`, `list.remove(...)`, `"x" in big_list` repetido.
- **Recomputação de derivado dentro de loop**: `for x in items: total = sum(other_items)` — `sum` recalcula a cada iteração; mova para fora do loop.
- **Sort dentro de loop**: `for x in items: candidates.sort()` — ordenar é O(k log k), e fazer N vezes vira O(N · k log k). Ordene uma vez antes.

**Quando NÃO se aplica:**

- Coleções com **cardinalidade pequena e cravada** (< ~50) onde o loop aninhado é mais legível que o índice. A constante 50 é heurística — quando em dúvida, prefira o índice; é raro o código ficar pior.
- Loop aninhado com **early termination cedo** garantido (`break` no primeiro hit, condições raras) — avaliar caso a caso. Se a entrada for adversarial, o pior caso ainda é quadrático.
