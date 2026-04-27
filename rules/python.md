# Python — Convenções de Backend

Aplicáveis a todo arquivo `**/*.py` (exceto `tests/**`, que tem regras próprias em `testing.md`).

## Exception Handling

### Sem `except Exception` genérico

```python
# ❌ EVITAR — captura tudo, esconde bugs, destrói traceback
try:
    do_work()
except Exception as e:
    raise Exception(f"Error: {str(e)}")
```

```python
# ✅ PREFERIR — capturar exceções específicas de infraestrutura externa
try:
    s3_client.put_object(...)
except ClientError as e:
    logger.exception("S3 upload failed for key=%s", key)
    raise UploadFailedError(key) from e
```

**Quando o `except Exception` é aceitável:**

- **Batch processing por item**: para que uma falha não interrompa o lote inteiro. Logar com `logger.exception` e persistir estado por item.
- **Handlers de fila (SQS, EventBridge)**: a borda do sistema legitimamente captura tudo para permitir retry/DLQ.

### Let-it-crash para falhas de infraestrutura

Banco indisponível, credenciais expiradas, fila inacessível: **deixa propagar**. Entrypoints (handlers, routers FastAPI) não devem capturar exceções de infra. O framework e a observabilidade lidam com o resto.

### Sem leak de `str(e)` em HTTPException

```python
# ❌ CRÍTICO — vaza stack trace, nomes de tabela, queries SQL para o cliente
except Exception as e:
    raise HTTPException(status_code=500, detail=str(e))
```

```python
# ✅ — remover o try/except inteiro; FastAPI já retorna 500 para exceções não tratadas
do_work()
```

Se a operação realmente precisa transformar a exceção em um status HTTP específico, usar exceção de domínio + handler global, não `str(e)` direto na resposta.

### Status codes do enum, não inteiros literais

```python
# ❌
raise HTTPException(status_code=403, detail="Forbidden")
return Response(status_code=200)

# ✅
from fastapi import status
raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Forbidden")
return Response(status_code=status.HTTP_200_OK)
```

### Semântica correta de status

- Auth/autorização → `403 FORBIDDEN`, não `500`
- API externa indisponível → `502 BAD_GATEWAY`, não `500`
- Recurso não existe → `404 NOT_FOUND`, não `400`

### Sem `logger.error(...) + raise` redundante

```python
# ❌ — log duplicado, handler superior já loga
try:
    do_work()
except SomeError as e:
    logger.error(f"Error: {e}")
    raise
```

```python
# ✅ — deixa propagar
do_work()
```

Exceção: se houver cleanup (rollback, close de recurso), preferir `try/finally` ou context manager.

## Sem `print()` em produção

`print` não tem nível, não carrega metadata, não vai para CloudWatch/Datadog/observabilidade.

```python
# ❌
print(f"Processing message: {msg}")
print(f"[Auth Error] no token")

# ✅
logger.info("Processing message: %s", msg)
logger.error("Auth failed: no token")
```

Mapeamento de níveis: erro → `error`, warning → `warning`, fluxo normal → `info`, debug → `debug` (ou remover).

## Type Hints

### Obrigatórios em toda assinatura

Inclui `-> None` quando aplicável:

```python
async def process_order(order: Order) -> None:
    ...

def get_total(items: list[Item]) -> Decimal:
    ...
```

### Modern typing — sem `typing.List`/`Optional`/`Union`

```python
# ❌
from typing import List, Optional, Dict, Tuple, Union

def get_users(ids: List[int]) -> Optional[Dict[str, Any]]: ...
def parse(data: Union[str, bytes]) -> Tuple[bool, str]: ...
```

```python
# ✅
from typing import Any

def get_users(ids: list[int]) -> dict[str, Any] | None: ...
def parse(data: str | bytes) -> tuple[bool, str]: ...
```

| Deprecated (`typing`) | Moderno | Disponível desde |
|---|---|---|
| `List[X]` | `list[X]` | Python 3.9 |
| `Dict[K, V]` | `dict[K, V]` | Python 3.9 |
| `Tuple[X, Y]` | `tuple[X, Y]` | Python 3.9 |
| `Set[X]` | `set[X]` | Python 3.9 |
| `Type[X]` | `type[X]` | Python 3.9 |
| `Optional[X]` | `X \| None` | Python 3.10 |
| `Union[X, Y]` | `X \| Y` | Python 3.10 |

`Any`, `TypeVar`, `Protocol`, `Callable`, `Literal`, `TypeAlias`, `TypeGuard`, `ClassVar`, `Final` ainda vêm de `typing` — não têm equivalente nativo.

## Pydantic

### `BaseModel` para DTOs e dados estruturados

Nunca usar dicts genéricos onde poderia haver um modelo. Validações, serialização e type-checking dependem disso.

### Todo campo DEVE usar `Field(description=...)`

```python
# ❌ — bare annotation
class Order(BaseModel):
    id: UUID
    amount: Decimal

# ✅
class Order(BaseModel):
    id: UUID = Field(description="Unique order identifier")
    amount: Decimal = Field(description="Order total in BRL")
```

Description é obrigatória — vira documentação OpenAPI e contexto para LLMs/integrações.

## IDs e Valores

### `Decimal` para valores monetários

```python
# ❌
amount: float = 100.50  # arredondamento binário, perda de precisão

# ✅
from decimal import Decimal
amount: Decimal = Decimal("100.50")
```

Conversões para float só na borda (display, serialização específica), nunca no domínio ou cálculo.

### `datetime.now(UTC)` para timestamps

```python
# ❌
from datetime import datetime
now = datetime.utcnow()    # deprecated, naive
now = datetime.now()        # local timezone, frágil em produção

# ✅
from datetime import datetime, UTC
now = datetime.now(UTC)
```

### `StrEnum` para conjuntos finitos de valores

```python
# ❌
STATUS = {"pending", "active", "closed"}
status: str  # qualquer string passa

# ✅
from enum import StrEnum

class OrderStatus(StrEnum):
    PENDING = "pending"
    ACTIVE = "active"
    CLOSED = "closed"

status: OrderStatus
```

### IDs ordenáveis por tempo

`uuid6.uuid7()` quando o ID precisa ser ordenável temporalmente (chaves de log, ULIDs alternativos). `uuid.uuid4()` quando ordem não importa.

## Estilo

### `async/await` para I/O

Operações de banco, HTTP, fila — sempre async. Nunca chamar I/O síncrono de dentro de função async (bloqueia event loop).

### Imports absolutos

```python
# ❌
from .repository import OrderRepository
from ..domain.order import Order

# ✅
from system.repositories.order_repository import OrderRepository
from system.entities.order import Order
```

(O source root depende do projeto: `system/`, `src/`, etc. Use o que o repo já adota.)

### Imports no topo do arquivo

Stdlib → third-party → local, ordenados pelo ruff/isort. Imports dentro de função são **antipadrão** salvo casos específicos (lazy load para evitar circular import documentado, condicional por feature flag).

### `re.compile` no nível do módulo

```python
# ❌ — recompila a regex em cada chamada
def is_cnpj(value: str) -> bool:
    return bool(re.match(r"\d{14}", value))

# ✅
_CNPJ_RE = re.compile(r"\d{14}")

def is_cnpj(value: str) -> bool:
    return bool(_CNPJ_RE.match(value))
```

### Identificadores em inglês

Nomes de variáveis, funções, classes em inglês. Valores de string internos (mensagens de UI, descrições) podem ser no idioma do domínio.

### OOP sobre funcional

Classe é o padrão default no projeto — funções standalone só quando há claro ganho (utilities puras sem estado, transformações pequenas). Isso facilita injeção de dependência via construtor.

### Sem código especulativo

Não escrever código para cenários que o contrato garante que não acontecem. `if` para casos impossíveis, validação dupla, fallbacks redundantes — tudo lixo. Se a invariante muda, o teste pega.
