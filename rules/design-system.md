# Design System — Convenções para `iorq-ds` e similares

Aplicáveis quando o PR toca código sob `apps/*-ds/**`, `services/*-ds/**` ou `packages/*-ds/**`. Estas regras **substituem** as de `frontend.md` para componentes do DS — o DS tem requisitos próprios.

## Estrutura

```
stories/<ComponentName>/
├── <ComponentName>.tsx
├── <ComponentName>.stories.tsx
├── <ComponentName>.component.test.tsx
└── components/
    └── <SubComponent>/
        └── <SubComponent>.tsx

utils/<utilName>/
├── <utilName>.ts
└── <utilName>.unit.test.ts
```

- **PascalCase** para componentes: `Button/Button.tsx`, `DateRangePicker/DateRangePicker.tsx`
- **camelCase** para utilities: `mask/mask.ts`, `zodResolver/zodResolver.ts`
- Storybook, testes e sub-componentes **co-localizados** com o componente

## Implementação de componentes

### Wrapping de HeroUI

Use HeroUI primitives como base. Wrap/narrow a API via `extends Omit<HeroUIProps, '...'>`. Nunca recriar o que HeroUI já provê.

### Named exports + tipo de props exportado

```tsx
// ✅
export interface ButtonProps extends Omit<HeroUIButtonProps, 'color' | 'variant'> {
  variant?: ButtonVariant;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>((props, ref) => {
  // ...
});
Button.displayName = 'Button';
```

- **Sempre** named export — `default` proibido em DS
- Tipo de props exportado junto com o componente
- `forwardRef` quando o componente envolve elemento DOM ou HeroUI primitive
- **`displayName` obrigatório em `forwardRef`** — sem ele o componente aparece como `Anonymous` em devtools e Storybook

### `'use client'` apenas quando necessário

Adicione a directive só se o componente realmente requer client-side (event handlers, state, refs). Componentes puramente visuais não precisam.

### Discriminated union para variantes

```tsx
export type Variant = 'primary' | 'secondary' | 'tertiary';
export type Size = 'sm' | 'md' | 'lg';
```

### Controlled vs Uncontrolled split

Para componentes integráveis a forms:

```tsx
export type InputProps<T extends FieldValues = FieldValues> =
  | ControlledInputProps<T>
  | UncontrolledInputProps;

export const Input = forwardRef<HTMLInputElement, InputProps>((props, ref) => {
  if ('control' in props && props.control && props.name) {
    return <ControlledInput {...props} ref={ref} />;
  }
  return <UncontrolledInput {...props} ref={ref} />;
});
```

### Componentes do DS são UI primitives — sem business logic

```tsx
// ❌ — proibido em DS
import { useQuery } from '@tanstack/react-query';

export const FeatureList = () => {
  const { data } = useQuery({ queryKey: ['features'], queryFn: fetchFeatures });
  return <Card>{/* ... */}</Card>;
};
```

DS components NÃO devem conter:
- Data fetching ou API calls
- Business logic ou domain rules
- React Query hooks (`useQuery`, `useMutation`)

State é limitado a concerns de UI: open/close, focus, controlled values.

## Callback prop dentro de `setState` updater (BUG)

**Esta é a regra mais valiosa do DS.** Chamar callback prop (`onToggle`, `onChange`, `onSelect`) dentro de um `setState` updater dispara o erro do React: *"Cannot update a component while rendering a different component"*. O `setState` do parent dispara durante a render do child.

```tsx
// ❌ — callback dentro do updater
const handleToggle = useCallback((): void => {
  setIsOpen((prev) => {
    const newState = !prev;
    onToggle?.(newState); // setState do parent durante render do child!
    return newState;
  });
}, [onToggle]);

// ❌ — callback síncrono junto com setState (ainda pode conflitar com useLocalStorage / useSyncExternalStore)
const handleToggle = useCallback((): void => {
  const newState = !isOpen;
  setIsOpen(newState);
  onToggle?.(newState);
}, [isOpen, onToggle]);
```

```tsx
// ✅ — useEffect + useRef
const [isOpen, setIsOpen] = useState(startOpen);
const onToggleRef = useRef(onToggle);
onToggleRef.current = onToggle;
const isInitialMount = useRef(true);

useEffect(() => {
  if (isInitialMount.current) {
    isInitialMount.current = false;
    return;
  }
  onToggleRef.current?.(isOpen);
}, [isOpen]);

const handleToggle = useCallback((): void => {
  setIsOpen((prev) => !prev);
}, []);
```

Regras-chave:
- Callback armazenado em `useRef` (atualizado a cada render) evita stale closures e re-runs desnecessários do effect
- Guard `isInitialMount` para pular o mount inicial
- Event handler só chama `setLocalState`, nunca o callback do parent diretamente

## Styling

### Tailwind direto, sem CVA

```tsx
// ❌ — class-variance-authority proibida no DS
import { cva } from 'class-variance-authority';
const button = cva('rounded-lg', { variants: { size: { sm: 'px-2', md: 'px-4' } } });

// ✅ — Record<Variant, string> com tw()
import { tw } from '@utils/tw/tw';

const sizeStyles: Record<Size, string> = {
  sm: tw('px-2 py-1 text-sm'),
  md: tw('px-4 py-2 text-base'),
  lg: tw('px-6 py-3 text-lg'),
};
```

### `tw()` para IntelliSense

```tsx
// ✅
const baseClasses = [tw('flex items-center gap-2'), tw('rounded-lg px-4')].join(' ');

// ❌ — string crua
const baseClasses = 'flex items-center gap-2 rounded-lg px-4';
```

### `cn()` do HeroUI para merge condicional

```tsx
import { cn } from '@heroui/react';
const classes = cn(baseStyles, isDisabled && disabledStyles, className);
```

Nunca importar `cn` ou `clsx` local — o do HeroUI é o padrão.

### Design tokens, não cores hardcoded

```tsx
// ❌
className="text-[#1a1a1a] bg-[#fff]"

// ✅
className="text-content-primary bg-primary"
className="ds-text-small ds-text-medium"
```

### Data attributes para state styling

```tsx
className="data-[invalid=true]:border-danger data-[selected=true]:bg-primary"
```

### Ordem de classes Tailwind é auto-sortida

`prettier-plugin-tailwindcss` reordena. Não tente ordenar manualmente.

## Acessibilidade

- Confie em HeroUI / React Aria para roles e ARIA attributes — não duplique o que a lib já provê
- `aria-label` explícito em botões só com ícone (ou outros controles sem texto visível)
- Tests verificam atributos ARIA-chave (`aria-selected`, `aria-disabled`, `aria-current`) e usam queries por role como estratégia primária

## Tests obrigatórios

Todo componente novo **DEVE** ter `*.component.test.tsx` colocated. Toda utility nova **DEVE** ter `*.unit.test.ts`. Padrões detalhados em `testing.md`. Coverage threshold do projeto (geralmente 80%) deve ser respeitado.

## Exports e API pública

- Todo componente público **DEVE** estar no barrel `index.ts`
- Exportar tanto o componente quanto seu props type:

```ts
export type { ButtonProps } from './stories/Button/Button';
export { Button } from './stories/Button/Button';
```

- Sub-componentes/internos **NÃO** vão para o barrel
- `export type { ... }` para type-only exports (tree-shaking)
- Agrupar exports no barrel com section comments (Action, Input, Display, Layout, Navigation, Utils)

## Storybook

Todo componente novo **DEVE** ter `*.stories.tsx` em formato CSF3 com `tags: ['autodocs']`. Use `fn()` de `@storybook/test` para action callbacks. Documentação completa do CSF3 fica fora do escopo desta regra.

## Dependências

- **Novas dependências de runtime são desencorajadas.** O DS deve ter o mínimo de runtime deps
- `react`, `react-dom`, `@heroui/react`, `framer-motion` são **peerDependencies** e devem ser **external** no tsup
- Toda nova dependência externa deve ser quase sempre `peerDependency` para proteger consumers de bundle bloat e conflitos de versão
