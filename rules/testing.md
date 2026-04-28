# Testing — Convenções de Teste

Aplicáveis a arquivos sob `tests/`, com nome `test_*.py`, `*_test.py`, `*.test.{ts,tsx}` ou `*.spec.{ts,tsx}`.

## Nomenclatura

- **Arquivos Python**: `test_<unidade>.py` ou `<unidade>_test.py` (siga o padrão do repo)
- **Funções/métodos**: `test_should_<comportamento>_when_<condição>`
- **Diretórios**: `tests/unit/`, `tests/integration/`, `tests/services/` (espelhando a estrutura de `src/` ou `system/`)

```python
# ✅
def test_should_return_token_when_credentials_valid(self): ...
def test_should_raise_when_amount_is_negative(self): ...

# ❌
def test_login(self): ...
def test_1(self): ...
def test_negative_amount(self): ...   # falta o "should/when"
```

## Mocks

### `spec=Class` sempre

Mock sem `spec` aceita qualquer atributo/método silenciosamente. Em refator, o teste continua passando enquanto produção quebra.

```python
# ❌
mock_repo = Mock()
mock_repo.find_by_inexistent_method()  # passa, mas método não existe

# ✅
from system.ports.order_repository import OrderRepositoryPort
mock_repo = Mock(spec=OrderRepositoryPort)
mock_repo.find_by_inexistent_method()  # AttributeError — testa contrato
```

Para async, `AsyncMock(spec=...)`.

### Verificar call count E argumentos

```python
# ❌ — só verifica que foi chamado, não COMO
mock_repo.save.assert_called()

# ✅
mock_repo.save.assert_called_once_with(expected_order)
```

Para múltiplas chamadas: `mock.assert_has_calls([call(a), call(b)], any_order=False)`.

### Verificar return value real, não só interações

```python
# ❌ — só verifica mocks, não o resultado
result = await use_case.execute(input)
mock_repo.save.assert_called_once_with(...)

# ✅
result = await use_case.execute(input)
assert result.status == OrderStatus.ACTIVE
mock_repo.save.assert_called_once_with(expected_order)
```

## Geração de dados — Polyfactory

**Nunca** criar dados de teste manualmente. Instâncias hardcoded ficam desatualizadas em refator e ocultam edge cases.

```python
# ❌
order = Order(id=UUID("12345678-1234-5678-1234-567812345678"), amount=Decimal("100"))

# ✅
class OrderGenerator(ModelFactory[Order]):
    __model__ = Order
    __faker_locale__ = "pt_BR"
    document_number = Use(fn=lambda: "".join(random.choices(string.digits, k=11)))
    status = Use(fn=lambda: random.choice(list(OrderStatus)))

order = OrderGenerator.build()
```

Generators ficam em `tests/generators/`. Instanciados em fixtures/setUp compartilhado.

## Estrutura — Arrange / Act / Assert

```python
async def test_should_return_token_when_email_valid(self):
    # Arrange
    email = self.email_generator.build()
    self.mock_sso.verify.return_value = True

    # Act
    result = await self.use_case.execute(email)

    # Assert
    assert result.token == "expected-token"
    self.mock_sso.verify.assert_called_once_with(email)
```

Para integration tests, adicionar comentários **Given / When / Then**:

```python
async def test_should_create_user_when_payload_valid(self):
    # Given: a valid user payload
    payload = self.user_generator.build()

    # When: the create endpoint is called
    response = self.client.post("/users", json=payload.model_dump())

    # Then: a 201 Created is returned with the user id
    assert response.status_code == status.HTTP_201_CREATED
    assert response.json()["id"] is not None
```

## Unit vs Integration

### Unit tests
- Mockar **todos** os adapters/ports via construtor
- Testar a lógica do use case / serviço de domínio
- Nunca tocar banco, fila, HTTP real
- Rápidos (centenas por segundo)

### Integration tests
- **Não** mockar adapter classes — exercitar adapters reais contra **technology mocks**
- Database: `testing.postgresql` (ou equivalente in-memory)
- AWS: `moto` (mock layer, não SDK mock)
- HTTP externo: `respx` ou `httpx-mock`
- Docker é último recurso (custo de execução; só quando não há double fiel)

## O que NÃO testar

- Atribuição direta de campos (`obj.x = 1; assert obj.x == 1`)
- Valores default de Pydantic/dataclass
- Igualdade básica de Python
- Serialização padrão sem transformação custom

Critério: "se eu remover este teste, algum bug de domínio passaria despercebido?" Se a resposta é não, o teste não vale o custo de manutenção.

## O que TESTAR

- Validators custom (`@field_validator`, `@model_validator`)
- Computed properties (`@property`, `@computed_field`)
- Métodos de domínio com regra de negócio
- Conversões entre modelos com lógica de transformação
- Use cases — orquestração e fluxos de erro
- Invariantes de domínio (estado válido vs inválido)
- Edge cases: zero, negativo, vazio, máximo, concorrência

## Frontend (TS/TSX)

- **Vitest** ou **Jest** dependendo do projeto (siga o que o repo já usa)
- **@testing-library/react** para render
- **@testing-library/user-event** sobre `fireEvent` (mais realista)
- Queries por role/label, não por classname (acessibilidade + resiliência)
- Coverage threshold do projeto (geralmente 80% para componentes de DS)

```tsx
// ✅
const user = userEvent.setup();
render(<Button onClick={handleClick}>Submit</Button>);
await user.click(screen.getByRole('button', { name: /submit/i }));
expect(handleClick).toHaveBeenCalledOnce();
```

## Imports no topo, sempre

Vale para tests também. Imports dentro de função quebram análise estática e escondem dependências do teste.
