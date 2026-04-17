# Review Rules

Regras canônicas de code review compartilhadas entre os repositórios da organização. Este folder é consumido pelo agente de review **Beethoven** e pode ser referenciado por ferramentas de dev (Cursor, Claude Code) via wrappers nos repos consumidores.

## Estrutura

```
rules/
├── MANIFEST.yml      # mapeamento canônico: globs → arquivos de regra
├── pr-level.md       # regras que valem para qualquer PR
├── python.md         # convenções Python (backend)
├── frontend.md       # convenções TypeScript/Next.js
├── design-system.md  # convenções para Design Systems (iorq-ds)
└── testing.md        # convenções de teste
```

## Como funciona

1. O **reviewer (Beethoven)** lê `MANIFEST.yml` no momento do review, cruza os globs com os paths tocados pelo PR, e concatena os arquivos de regra relevantes ao contexto do LLM.
2. Cada repo pode ter seu próprio `.ai/rules/MANIFEST.yml` com regras adicionais ou mais específicas — o reviewer lê **baseline (org) + local (repo)** em ordem.
3. Ferramentas de dev (Cursor, Claude Code) podem referenciar estes arquivos via wrappers no repo local (ver README do repo consumidor).

## Princípios de conteúdo

- **Detectáveis no diff**: regras devem descrever padrões que podem ser identificados em código. Guidelines aspiracionais ficam em outros docs.
- **Padrão + contraexemplo**: cada regra mostra o padrão preferido e o antipadrão, como em `python-try-except`.
- **Sem PII de incidentes**: não referenciar números de PR internos, dashboards, quantidades específicas de ocorrências em apps — isso fica nos repos privados.
- **Português**: regras são escritas em pt-BR por padrão, seguindo convenção dos repos consumidores.

## Edição

- PRs neste folder requerem review antes de merge (CODEOWNERS pode ser configurado depois).
- Mudanças propagam imediatamente para reviews (Beethoven lê `main`).
- Adicionar nova regra: criar arquivo em `rules/<nome>.md` e incluir entry no `MANIFEST.yml`.
