# Data Access e Batching

Princípios universais sobre eficiência de IO — banco de dados, APIs externas, sistemas de arquivo. Aplicam-se a qualquer linguagem com ORM, cliente HTTP, ou IO em loop.

A regra mestre: **round-trip é caro**. Cada query, request, leitura tem custo fixo (latência, conexão, parsing). Multiplicar round-trips pela cardinalidade da entrada degrada de forma não-linear em produção.

> Diferença em relação a `algorithmic-complexity.md`: este arquivo trata da **camada de IO** (queries, requests, persistência). O outro trata de complexidade **computacional pura** (operações em memória).

---

## CLAUDE_MD:n-plus-one-queries

**Princípio:** se você itera sobre uma coleção fazendo uma query/request por item, o custo é N+1 round-trips. Carregue tudo em uma única operação agregada — `IN (...)` no SQL, eager loading no ORM, batch endpoint na API.

**Severidade:** `major` (`PERF:n-plus-one`). `blocker` quando o N é input externo sem cap (cliente controla quantos itens vão entrar no loop).

**Aplica em:** qualquer ORM (SQLAlchemy, Django ORM, Prisma, TypeORM, GORM, Active Record), qualquer cliente HTTP em loop, leitura de S3/storage em loop.

### ❌ Python — SQLAlchemy clássico

```py
students = session.query(Student).all()
for student in students:
    grades = (
        session.query(Grade)
        .filter(Grade.student_id == student.id)
        .all()
    )
    process(student, grades)
```

### ✅ Python — eager load com `selectinload`

```py
students = (
    session.query(Student)
    .options(selectinload(Student.grades))
    .all()
)
for student in students:
    process(student, student.grades)
```

### ✅ Python — quando o relacionamento não está mapeado, faça uma query agregada

```py
student_ids = [s.id for s in students]
grades_by_student = defaultdict(list)
for grade in session.query(Grade).filter(Grade.student_id.in_(student_ids)).all():
    grades_by_student[grade.student_id].append(grade)

for student in students:
    process(student, grades_by_student[student.id])
```

### ❌ TypeScript — Prisma sem `include`

```ts
const students = await prisma.student.findMany();
for (const student of students) {
  const grades = await prisma.grade.findMany({
    where: { studentId: student.id },
  });
  process(student, grades);
}
```

### ✅ TypeScript — `include` ou `select`

```ts
const students = await prisma.student.findMany({
  include: { grades: true },
});
for (const student of students) {
  process(student, student.grades);
}
```

### ❌ Genérico — request HTTP por item em loop

```py
for user_id in user_ids:
    profile = await api_client.get_profile(user_id)
    process(profile)
```

### ✅ Genérico — endpoint de batch ou `asyncio.gather`

```py
# Se a API tem batch endpoint, use:
profiles = await api_client.get_profiles_batch(user_ids)

# Senão, paralelize com cap de concorrência:
semaphore = asyncio.Semaphore(10)
async def fetch(uid: str) -> Profile:
    async with semaphore:
        return await api_client.get_profile(uid)

profiles = await asyncio.gather(*[fetch(uid) for uid in user_ids])
```

**Quando NÃO se aplica:**

- Lista com **cardinalidade pequena e cravada** (< ~10) onde a query agregada custaria mais por causa de um join muito largo. Raro, mas existe. Documente com comentário.
- O loop **decide condicionalmente** se faz a query (early break, filtro que descarta a maioria) — eager loading carregaria tudo desnecessariamente. Avalie caso a caso; muitas vezes a refatoração ainda compensa porque o ORM hidrata barato após uma query única.

---

## CLAUDE_MD:bulk-batch-processing

**Princípio:** ao inserir, atualizar ou apagar múltiplas linhas/registros, faça uma única operação bulk em vez de N operações individuais. Cada `session.add(...)` + `commit()` por item, ou `INSERT` por item em loop, paga overhead de round-trip + flush + transação.

**Severidade:** `major` (`PERF:non-bulk-mutation`).

**Aplica em:** persistência (DB), envio de eventos (SNS/SQS publish em loop), uploads (S3 put em loop), qualquer mutação externa.

### ❌ Python — insert individual em loop

```py
for row in csv_rows:
    record = ProcessedRow(**row)
    session.add(record)
    session.commit()  # commit por linha: catástrofe
```

### ❌ Python — variante quase tão ruim (sem commit por item, mas ainda N flushes)

```py
for row in csv_rows:
    session.add(ProcessedRow(**row))
session.commit()
```

Funciona, mas SQLAlchemy ainda faz N `INSERT` separados. Para volumes acima de algumas centenas, é gargalo.

### ✅ Python — bulk insert via `session.bulk_save_objects` ou `Core insert`

```py
records = [ProcessedRow(**row) for row in csv_rows]
session.bulk_save_objects(records)
session.commit()

# Ou com SQLAlchemy Core para máxima performance:
session.execute(insert(ProcessedRowTable), [row.model_dump() for row in records])
session.commit()
```

### ✅ Python — bulk com chunks (memória controlada)

```py
def chunks(seq, size):
    for i in range(0, len(seq), size):
        yield seq[i : i + size]

for batch in chunks(records, 1000):
    session.execute(insert(ProcessedRowTable), [r.model_dump() for r in batch])
    session.commit()
```

### ❌ TypeScript — Prisma create em loop

```ts
for (const row of csvRows) {
  await prisma.processedRow.create({ data: row });
}
```

### ✅ TypeScript — `createMany`

```ts
await prisma.processedRow.createMany({ data: csvRows });
```

### ❌ Python — SNS publish em loop

```py
for event in events:
    sns_client.publish(TopicArn=topic, Message=json.dumps(event))
```

### ✅ Python — `publish_batch` (até 10 por chamada)

```py
for batch in chunks(events, 10):
    sns_client.publish_batch(
        TopicArn=topic,
        PublishBatchRequestEntries=[
            {"Id": str(i), "Message": json.dumps(e)}
            for i, e in enumerate(batch)
        ],
    )
```

**Quando NÃO se aplica:**

- Mutação única (apenas um registro) — `create` direto está correto.
- Mutações que **dependem do retorno** da anterior (ex.: cada item gera ID que alimenta o próximo) — não é candidato a bulk natural; reavaliar arquitetura.
- Batch que precisa de **comportamento transacional individual** (cada item commita ou aborta independente). Bulk não dá; faça batch com try/except por item + DLQ.

---

## CLAUDE_MD:pre-processing-before-heavy-loop

**Princípio:** antes de um loop sobre coleção grande, organize os dados em estruturas que respondem em O(1) (dict, set) em vez de procurar em listas a cada iteração. O custo de construir o índice uma vez é amortizado pelo loop — o custo de não construir é multiplicado por ele.

**Severidade:** `major` (`PERF:missing-pre-processing`).

**Aplica em:** qualquer loop que precisa, a cada iteração, achar algo numa outra coleção. Padrão clássico em processamento de arquivos grandes (CSV, JSON Lines, eventos em lote).

### ❌ Python — busca linear dentro do loop

```py
def enrich_orders(orders: list[Order], customers: list[Customer]) -> list[Order]:
    enriched = []
    for order in orders:
        customer = next((c for c in customers if c.id == order.customer_id), None)
        if customer:
            enriched.append(order.with_customer(customer))
    return enriched
```

Se `orders` tem N itens e `customers` tem M, o custo é O(N·M). Com N=200k e M=10k vira inviável.

### ✅ Python — pré-processa em dict

```py
def enrich_orders(orders: list[Order], customers: list[Customer]) -> list[Order]:
    customers_by_id = {c.id: c for c in customers}
    return [
        order.with_customer(customers_by_id[order.customer_id])
        for order in orders
        if order.customer_id in customers_by_id
    ]
```

Custo agora é O(N + M).

### ❌ Genérico — verificação de existência em lista

```py
for item in big_list:
    if item.tag in allowed_tags_list:  # O(K) por iteração; K = len(allowed_tags_list)
        process(item)
```

### ✅ Genérico — verificação em set

```py
allowed_tags = set(allowed_tags_list)  # O(K) uma vez
for item in big_list:
    if item.tag in allowed_tags:  # O(1) por iteração
        process(item)
```

### ❌ TypeScript — `find` dentro do loop

```ts
for (const order of orders) {
  const customer = customers.find(c => c.id === order.customerId);
  if (customer) enriched.push({ ...order, customer });
}
```

### ✅ TypeScript — `Map` pré-construído

```ts
const customersById = new Map(customers.map(c => [c.id, c]));
const enriched = orders
  .filter(o => customersById.has(o.customerId))
  .map(o => ({ ...o, customer: customersById.get(o.customerId)! }));
```

**Quando NÃO se aplica:**

- Loop pequeno e infrequente (cardinalidade total < ~100) onde a leitura linear é mais clara que o índice.
- A "lista auxiliar" não cabe em memória (caso raro — quase sempre o problema é arquitetural, não tem solução com índice em memória).
