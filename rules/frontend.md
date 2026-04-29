# Frontend — Convenções TypeScript / Next.js

Aplicáveis a `**/*.{ts,tsx,js,jsx}` em apps Next.js. Para apps de Design System (`apps/*-ds/**`), as regras de `design-system.md` têm prioridade.

## Estrutura de componentes

### Cada componente em sua própria pasta

```
components/
├── UploadModal/
│   ├── UploadModal.tsx
│   └── hooks/
│       └── useUploadModal.tsx
├── Filter/
│   ├── Filter.tsx
│   └── hooks/
│       └── useFilter.tsx
└── ListCard/
    ├── ListCard.tsx
    ├── hooks/
    │   └── useListCard.tsx
    └── components/
        └── Drawer/
            ├── Drawer.tsx
            └── hooks/
                └── useDrawer.tsx
```

```
// ❌ — arquivos soltos em components/, hooks/ compartilhada
components/
├── UploadModal.tsx
├── Filter.tsx
├── hooks/
│   ├── useUploadModal.tsx
│   └── useFilter.tsx
```

- **PascalCase** para nomes de componentes e arquivos: `OccurrenceApprovalCard.tsx`
- **Chip components** terminam com `Chip`: `OccurrenceTypeChip`
- Hooks específicos do componente vivem em `hooks/` dentro da pasta do próprio componente
- Sub-componentes seguem o mesmo padrão recursivamente

### Componentes são apresentacionais

Toda lógica de negócio vai para custom hook. Componente recebe dados e callbacks.

```tsx
// ✅
const ApprovalCard: FC<Props> = (props) => {
  const { handleApprove, isLoading } = useApprovalCard(props.data);
  return <Card>{/* ... */}</Card>;
};

// ❌ — lógica misturada no componente
const ApprovalCard: FC<Props> = (props) => {
  const [state, setState] = useState();
  const mutation = useMutation({ /* ... */ });
  return <Card>{/* ... */}</Card>;
};
```

### Named exports, não default

```tsx
// ✅
export const ApprovalCard: FC<Props> = (props) => { /* ... */ };

// ❌
const ApprovalCard: FC<Props> = (props) => { /* ... */ };
export default ApprovalCard;
```

`default` só quando o framework exigir (ex.: `page.tsx` do Next.js).

## Hooks

- Prefixo `use`: `useFilter`, `useRemittances`
- Colocados em `hooks/` co-localizado com a feature ou componente
- Retornam **interface tipada explícita**:

```tsx
interface UseFeature {
  data: FeatureItem[];
  isLoading: boolean;
  handleAction: () => void;
}

const useFeature = (): UseFeature => { /* ... */ };
```

- `useCallback` para event handlers (especialmente quando passados como prop)
- `useMemo` para computed values caros

## Services Pattern (data fetching)

**Toda chamada de rede vive em services**, nunca em componentes ou feature hooks.

### Estrutura

```
app/services/
└── FeatureService/
    ├── useListFeatures/
    │   ├── useListFeatures.ts
    │   └── useListFeatures.test.ts
    └── useCreateFeature/
        ├── useCreateFeature.ts
        └── useCreateFeature.test.ts
```

### Regras

- `useQuery` para GET, `useMutation` para POST/PUT/DELETE, `useInfiniteQuery` para paginadas
- Cada service exporta **dois símbolos**:
  1. Função async pura que chama `fetchApi` (testável fora de React)
  2. Hook React Query que envolve a função pura
- Componentes e feature hooks **só consomem o hook**, nunca chamam `fetchApi` diretamente
- Requests passam pelo **BFF** (`/api/...`), nunca direto para APIs externas
- **NUNCA** `fetch` nativo em código client-side
- **Sempre** passar `signal` do `queryFn` para a função de service (cancelamento de requests)

```tsx
// ✅ — componente consome service hook
import { useListFeatures } from '@services/FeatureService/useListFeatures/useListFeatures';

const useFeatureTable = (): UseFeatureTable => {
  const { data, isLoading } = useListFeatures();
  return { list: data?.data, isLoading };
};

// ❌ — fetchApi inline em hook de feature
const useFeatureTable = (): UseFeatureTable => {
  const { data } = useQuery({
    queryKey: ['features'],
    queryFn: ({ signal }) => fetchApi<ListFeaturesRes>({ url: '/api/feature', signal }),
  });
  return { list: data?.data };
};

// ❌ — fetch nativo em client
const { data } = useQuery({
  queryKey: ['features'],
  queryFn: () => fetch('/api/feature').then(res => res.json()),
});
```

## BFF Routes (`app/api/`)

### Regras

- Named exports: `export async function GET(req: NextRequest): Promise<NextResponse<T>>`
- **Use `fetchWithAuth(req, url, init)`** para chamadas autenticadas — injeta `Authorization: Bearer` e `Content-Type` automaticamente
- `fetch` nativo só em rotas **não autenticadas** (auth endpoints, S3 presign, APIs públicas como ViaCEP)
- Parse com `getResponse<T>(response)`
- Forward `x-forwarded-for` do request original
- Query params via helper `getParameters(req)`
- **Sempre forward `req.method`** para upstream em vez de hardcoded — mantém handler e upstream em sync
- Catch de `ApiError` + fallback genérico 500

```tsx
// ✅
export async function GET(
  req: NextRequest,
): Promise<NextResponse<GetFeatureResponse | ApiErrorResponse>> {
  try {
    const xForwardedFor = req.headers.get('x-forwarded-for');
    const { params } = getParameters(req);
    const url = `${process.env.NEXT_PUBLIC_API_URL}/endpoint${params}`;

    const response = await fetchWithAuth(req, url, {
      method: req.method,
      headers: { 'x-forwarded-for': xForwardedFor ?? '' },
    });

    const data = await getResponse<ApiGetFeatureResponse>(response);
    return NextResponse.json(data, { status: 200 });
  } catch (error) {
    if (error instanceof ApiError) {
      return NextResponse.json({ message: error.message }, { status: error.status });
    }
    return NextResponse.json({ message: 'Erro ao processar' }, { status: 500 });
  }
}
```

### Separação de interfaces

- **Client interfaces** (source of truth) em `app/interfaces/Feature/GetFeatureResponse.ts`
- **API interfaces** em `app/api/interfaces/Feature/ApiGetFeatureResponse.ts`
- Quando shapes idênticos, API re-exporta o tipo do cliente:

```tsx
// app/api/interfaces/Feature/ApiGetFeatureResponse.ts
import { GetFeatureResponse } from '@interfaces/Feature/GetFeatureResponse';
export type ApiGetFeatureResponse = GetFeatureResponse;
```

Quando divergem, definir tipos completos em cada arquivo + função `convertResponse` co-localizada na route.

### Query params enum co-localizado

```tsx
// app/api/feature/FeatureQueryParamsEnum.ts
export const FeatureQueryParamsEnum = {
  customerCnpj: 'customer_cnpj',
  status: 'status',
  dueDateStart: 'due_date_start',
} as const;

export type FeatureQueryParams =
  (typeof FeatureQueryParamsEnum)[keyof typeof FeatureQueryParamsEnum];
```

camelCase keys → snake_case API param values.

## Forms — react-hook-form + Zod

```tsx
const formSchema = z.object({
  state: z.string().optional(),
  search: z.string().optional(),
});
type FormSchema = z.infer<typeof formSchema>;

const form = useForm<FormSchema>({ resolver: zodResolver(formSchema) });
```

- Usar `zodResolver` do projeto (pode estar em `iorq-ds`)
- `useFormContext` para componentes aninhados, `useController` para inputs controlados
- Sync de form state para URL params via `setURLParams` quando o form é filtro

## Enums — `as const`, não TypeScript `enum`

```tsx
// ✅
export const RemittanceStatesEnum = {
  PROPOSED: 'proposed',
  ACTIVE: 'active',
} as const;

export type RemittanceStatesEnum =
  (typeof RemittanceStatesEnum)[keyof typeof RemittanceStatesEnum];

export const RemittanceStatesEnumLabels: Record<RemittanceStatesEnum, string> = {
  [RemittanceStatesEnum.PROPOSED]: 'Proposta',
  [RemittanceStatesEnum.ACTIVE]: 'Ativa',
};

// ❌
export enum RemittanceStates {
  Proposed = 'proposed',
  Active = 'active',
}
```

TypeScript `enum` gera código extra em runtime, não é tree-shakeable e tem semântica confusa. `as const` é o padrão moderno.

## Imports

- **Absolutos com `@` alias**: `@components/`, `@enum/`, `@interfaces/`, `@utils/`, `@services/`
- Auto-sorted por `simple-import-sort` — não reordene manualmente
- `import type { ... }` para imports type-only (tree-shaking)
- Preferir imports diretos sobre barrel re-exports quando possível

## TypeScript

- **Return types explícitos** em toda função e hook
- Strict typing — evite `any`, prefira `unknown` quando o tipo é genuinamente desconhecido
- `as const` para literais de objeto

## Design System (consumidor)

Se o projeto consome `iorq-ds`, **prefira componentes do DS** antes de cair em HeroUI ou outra lib:

```tsx
// ✅
import { Button, Input, DateRangePicker } from 'iorq-ds';

// ❌ — usar HeroUI quando o DS já oferece
import { Button } from '@heroui/button';
```

Se o componente não existe no DS, fallback para `@heroui/*`. Adicionar ao DS quando o uso se repete em múltiplos consumers.
