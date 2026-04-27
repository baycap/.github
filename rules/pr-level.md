# PR-level — Convenções de branch, título e escopo

Regras aplicáveis a qualquer PR, independente da linguagem ou arquivos tocados. O reviewer avalia esses aspectos no contexto do PR (título, tamanho, branch), não em linhas específicas do diff.

## 1. Título do PR

Formato obrigatório:

```
[IORQ-<NOTION_TASK_ID>] <FEAT|FIX|REFACTOR>: <título em inglês>
```

- **ID do Notion** entre colchetes (ex.: `[IORQ-1934]`) — a automação do board atualiza status automaticamente.
- **Tipo em UPPERCASE**. Principais: `FEAT`, `FIX`, `REFACTOR`. Também aceitos quando apropriado: `CHORE`, `DOCS`, `TEST`, `PERF`, `STYLE`, `CI`, `BUILD`.
- **Título curto e descritivo** em inglês, reflete a mudança principal.

Exemplos válidos:

```
[IORQ-1934] FEAT: add acquisition total sum card
[IORQ-2010] FIX: convert floats to Decimal before save
[IORQ-2221] REFACTOR: acquisitions with ds
```

**Antipadrão:** título sem ID do Notion, ou com tipo em lowercase, ou sem `:` após o tipo, ou descrição em português.

## 2. Nome da branch

```
<tipo>/IORQ-<NOTION_TASK_ID>-<descrição-kebab-case>
```

- `tipo`: `feat`, `fix`, `refactor` principalmente. Outros aceitos: `chore`, `docs`, `test`, `perf`, `style`, `ci`, `build`.
- `IORQ-XXXX`: ID da task no Notion.
- `descrição`: 3–5 palavras em kebab-case, inglês, sem acentos.

### Branches sucessoras da mesma task

Quando uma task do Notion for entregue em mais de uma branch/PR (por tamanho, risco ou dependência), as sucessoras herdam o nome da primeira e recebem sufixo incremental `-<N>` começando em `-2`:

```
refactor/IORQ-2221-acquisitions-with-ds       # 1ª branch
refactor/IORQ-2221-acquisitions-with-ds-2     # 2ª
refactor/IORQ-2221-acquisitions-with-ds-3     # 3ª
```

Nunca usar `-1` — a primeira é implícita.

## 3. Tamanho do PR

- **Alvo**: 200–300 linhas de diff (adições + remoções).
- PRs maiores devem ser quebrados em sub-PRs incrementais usando o sufixo `-<N>`.
- Quebrar preferencialmente por **camadas** (schema → usecase → route) ou **features isoladas**, não por arquivos aleatórios.
- Se não for possível quebrar sem inviabilizar o review, justificar no corpo do PR.

## 4. Testes acompanham o código

PRs de `feat`, `fix` e `refactor` **não separam código e testes**:

- Testes novos ou atualizados para a mudança vão no **mesmo PR** do código.
- Não abrir PR `test/IORQ-XXXX-...` separado para cobrir código já mergeado.
- Exceção: refator puramente mecânico coberto pelos testes existentes — declarar explicitamente no PR.

## 5. Uma responsabilidade por PR

Um PR resolve um problema. Se ao revisar você encontra duas mudanças desconexas (ex.: refator + nova feature), é sinal para quebrar.

## 6. Operações git proibidas sem aprovação explícita

- `git push --force`, `git push -f`
- `git add -f`
- `git reset --hard`
- Qualquer operação destrutiva em branches compartilhadas

## 7. Rastreabilidade Notion → Branch → PR

Toda mudança de código deve ser rastreável a uma task no Notion (board Delivery). A exceção é infraestrutura interna pura (CI, linting, docs de processo) que não representa entrega de produto — pode dispensar task, mas documentar no PR.
