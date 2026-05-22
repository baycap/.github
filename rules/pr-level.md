# PR-level — Convenções de branch, título e escopo

Regras aplicáveis a qualquer PR, independente da linguagem ou arquivos tocados. O reviewer avalia esses aspectos no contexto do PR (título, tamanho, branch), não em linhas específicas do diff.


## 1. Tamanho do PR

- **Alvo**: 200–300 linhas de diff (adições + remoções).
- PRs maiores devem ser quebrados em sub-PRs incrementais usando o sufixo `-<N>`.
- Quebrar preferencialmente por **camadas** (schema → usecase → route) ou **features isoladas**, não por arquivos aleatórios.
- Se não for possível quebrar sem inviabilizar o review, justificar no corpo do PR.

## 2. Testes acompanham o código

PRs de `feat`, `fix` e `refactor` **não separam código e testes**:

- Testes novos ou atualizados para a mudança vão no **mesmo PR** do código.
- Não abrir PR `test/IORQ-XXXX-...` separado para cobrir código já mergeado.
- Exceção: refator puramente mecânico coberto pelos testes existentes — declarar explicitamente no PR.

## 3. Uma responsabilidade por PR

Um PR resolve um problema. Se ao revisar você encontra duas mudanças desconexas (ex.: refator + nova feature), é sinal para quebrar.

## 4. Operações git proibidas sem aprovação explícita

- `git push --force`, `git push -f`
- `git add -f`
- `git reset --hard`
- Qualquer operação destrutiva em branches compartilhadas
