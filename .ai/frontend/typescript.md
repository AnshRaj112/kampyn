# Frontend TypeScript Standards

## Purpose

This document defines the TypeScript standards for the KAMPYN frontend, built with Next.js, React, TanStack Query, Zustand, Zod, and Tailwind CSS.

TypeScript is the primary language for frontend application code. Its purpose is to provide strong compile-time guarantees, explicit contracts, predictable data flow, and maintainable interfaces across KAMPYN's modules.

These standards apply to:
- React components and hooks.
- Next.js pages, layouts, route handlers, and server actions.
- API clients and response types.
- TanStack Query and Zustand.
- Forms and Zod schemas.
- Shared UI components and design-system primitives.
- Authentication and tenant context.
- Real-time communication.
- Testing, configuration, and frontend utilities.

The objective is to catch errors early, make invalid states difficult to represent, reduce runtime defects, and ensure frontend code remains scalable as KAMPYN grows.

---

## 1. Core Principles

### 1.1 Strict Type Safety

TypeScript must be configured in strict mode.

- Avoid `any`.
- Prefer explicit and narrow types.
- Use `unknown` for untrusted values.
- Use discriminated unions for mutually exclusive states.
- Use generics where they improve reuse and type safety.
- Avoid unnecessary type assertions.
- Validate external data at runtime.
- Keep types aligned with actual application contracts.

TypeScript must strengthen correctness, not merely silence compiler errors.

### 1.2 Type at System Boundaries

All data entering the application from outside a trusted, typed function boundary must be treated as untrusted until validated.

Examples include:
- API responses.
- URL parameters.
- Form submissions.
- Browser storage.
- WebSocket messages.
- Third-party integrations.
- Environment variables.
- User-generated content.
- Imported files.

Use Zod or another explicitly approved runtime validation mechanism at appropriate boundaries.

Compile-time types alone do not validate runtime data.

### 1.3 Prefer Inference Where Clear

Do not annotate every variable or expression unnecessarily.

Prefer inference when the type is obvious and stable:

```ts
const itemCount = items.length;
const isAvailable = itemCount > 0;
```

Use explicit annotations for:
- Public function parameters and return types.
- Exported contracts.
- Complex state.
- Generic boundaries.
- Callback contracts where inference is insufficient.
- Values where explicit typing materially improves clarity.

### 1.4 Model Domain Concepts Explicitly

Use meaningful types that reflect KAMPYN's domain.

Prefer:

```ts
type OrderId = string;
type TenantId = string;
type UserId = string;

interface OrderSummary {
  id: OrderId;
  status: OrderStatus;
  createdAt: string;
}
```

Avoid generic, ambiguous contracts such as:

```ts
interface Data {
  id: string;
  status: string;
  value: any;
}
```

Domain types must represent real application concepts rather than speculative abstractions.

### 1.5 Make Invalid States Difficult to Represent

Prefer type structures that constrain invalid combinations.

Use discriminated unions, literal types, and explicit optionality rather than loosely related booleans or nullable fields.

Types must help developers understand what states are possible and which operations are valid in each state.

---

## 2. TypeScript Configuration

The frontend must use strict TypeScript configuration.

Recommended baseline:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "es2022"],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "noFallthroughCasesInSwitch": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts"
  ],
  "exclude": ["node_modules"]
}
```

This configuration is a baseline and must be reconciled with the actual Next.js and TypeScript versions used by the repository.

### 2.1 Strict Compiler Options

The following options are required unless an explicit compatibility exception is approved:

- `strict`
- `noImplicitOverride`
- `noFallthroughCasesInSwitch`
- `noUncheckedIndexedAccess`
- `exactOptionalPropertyTypes`
- `forceConsistentCasingInFileNames`

`noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` may require additional handling in legacy code. Any exception must be narrowly scoped, documented, and tracked for removal.

### 2.2 Avoid Weakening Configuration

Do not:
- Disable strict mode to accommodate individual files.
- Use broad compiler exclusions to hide errors.
- Suppress type errors without a documented reason.
- Add `@ts-ignore` as a routine workaround.
- Change global compiler options solely to make an implementation compile.

Prefer fixing the type model or isolating a genuine third-party compatibility issue.

---

## 3. Type Definitions

### 3.1 Interface vs Type

Use `interface` primarily for object contracts that are expected to be extended or implemented.

Use `type` for:
- Unions.
- Intersections.
- Primitive aliases.
- Tuples.
- Function types.
- Mapped and conditional types.
- Discriminated unions.
- Composed domain types.

Example:

```ts
interface UserProfile {
  id: string;
  displayName: string;
}

type UserRole = "student" | "vendor" | "staff" | "admin";

type UserWithRole = UserProfile & {
  role: UserRole;
};
```

Do not enforce a mechanical rule that all declarations must use one form. Choose the form that best communicates the contract.

### 3.2 Avoid Duplicate Types

Reuse shared domain contracts where appropriate.

Do not create multiple incompatible representations of the same entity across:
- API functions.
- Components.
- Hooks.
- Stores.
- Forms.
- Tests.

However, do not force distinct concepts into one type simply because their fields currently look similar.

For example, an API response, an editable form draft, and a table row may represent different contracts and should be modelled accordingly.

### 3.3 Domain Types

Domain types should be cohesive and feature-owned.

Suggested structure:

```text
src/
└── features/
    └── orders/
        ├── types/
        │   ├── order.ts
        │   ├── order-status.ts
        │   └── order-filters.ts
        ├── schemas/
        │   └── order.schema.ts
        ├── api/
        └── ...
```

Avoid one enormous global `types.ts` file containing unrelated application contracts.

Shared types belong in shared modules only when they are genuinely reused and have clear ownership.

### 3.4 Branded Types

Branded types may be used to distinguish values that share the same underlying representation but must not be mixed accidentally.

Example:

```ts
type Brand<T, Name extends string> = T & {
  readonly __brand: Name;
};

type TenantId = Brand<string, "TenantId">;
type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
```

Use branded types when the distinction prevents meaningful classes of bugs.

Do not introduce branding for every string or numeric field. Conversion from untrusted data must be validated before assigning a branded type.

A branded type is a compile-time aid, not a security boundary.

### 3.5 Literal Types and Enums

Prefer literal unions for simple finite sets:

```ts
type BookingStatus =
  | "pending"
  | "confirmed"
  | "cancelled"
  | "completed";
```

Use TypeScript enums only where the project has an established, justified convention or interoperability requirement.

Do not use broad `string` types for known finite domain states.

### 3.6 Optional and Nullable Properties

Optional and nullable values are different concepts.

- Optional means the property may be absent.
- Nullable means the property may be present with a `null` value.

Represent the actual contract accurately.

```ts
interface User {
  id: string;
  middleName?: string;
  profileImageUrl: string | null;
}
```

Do not use optional properties as a substitute for understanding the API contract.

With `exactOptionalPropertyTypes`, explicitly model `undefined` when it is a meaningful value in the contract.

---

## 4. Avoiding `any`

The use of `any` is prohibited in production code unless a narrow, documented interoperability exception is approved.

Avoid:

```ts
function processOrder(order: any) {
  return order.status;
}
```

Prefer:

```ts
function processOrder(order: Order) {
  return order.status;
}
```

For unknown external values:

```ts
function handleExternalPayload(payload: unknown) {
  const result = externalPayloadSchema.safeParse(payload);

  if (!result.success) {
    return;
  }

  processPayload(result.data);
}
```

### 4.1 Exceptions

A narrow `any` exception may be permitted when:
- An external library exposes unusable or fundamentally untyped APIs.
- A temporary migration requires an isolated compatibility layer.
- The exception is reviewed and has a defined removal path.

Do not let `any` escape into domain models, exported APIs, stores, or reusable components.

Prefer `unknown`, generics, type guards, or adapter types wherever possible.

---

## 5. Functions and Return Types

### 5.1 Function Signatures

Public and exported functions must have clear parameter and return contracts.

```ts
export async function getOrder(
  orderId: OrderId,
  signal?: AbortSignal,
): Promise<Order> {
  // Implementation
}
```

Explicit return types are particularly important for:
- Exported functions.
- API clients.
- Hooks.
- Store actions.
- Shared utilities.
- Complex asynchronous operations.
- Public SDK-facing contracts.

Internal functions may rely on inference when the result is straightforward.

### 5.2 Avoid Overly Broad Return Types

Avoid returning `object`, `Function`, or `Record<string, any>` where a specific type is available.

Prefer:

```ts
type ApiResult<T> = {
  data: T;
  requestId: string;
};
```

Use `void` for functions that intentionally return no meaningful value.

### 5.3 Async Functions

Async functions must express their result type clearly.

- Use `Promise<T>` for asynchronous return contracts.
- Handle expected failures through the established error model.
- Do not return inconsistent result shapes.
- Avoid unhandled promises.
- Do not convert all errors into successful-looking fallback values.

### 5.4 Function Complexity

Functions must remain small, cohesive, and easy to test.

- Keep one primary responsibility per function.
- Extract complex transformations into named helpers.
- Avoid deeply nested conditions.
- Avoid excessive callback nesting.
- Use explicit domain operations instead of generic mutation utilities when clarity improves.

Follow the project-wide 200-line file limit and relevant complexity standards.

---

## 6. Generics

Generics should improve type safety and reusable behavior.

Suitable use cases:
- Reusable UI components.
- Typed API responses.
- Shared pagination structures.
- Generic table helpers.
- Reusable state utilities.
- Typed event handlers.
- Data transformation utilities.

Example:

```ts
interface PaginatedResponse<T> {
  items: T[];
  page: number;
  pageSize: number;
  total: number;
}
```

### 6.1 Generic Design Rules

- Use meaningful type parameter names.
- Prefer constrained generics when operations require specific properties.
- Avoid unnecessary generic parameters.
- Avoid deeply nested conditional types unless they solve a concrete problem.
- Do not create generic frameworks for one-off operations.
- Keep exported generic APIs understandable.

Prefer:

```ts
function getById<T extends { id: string }>(
  items: readonly T[],
  id: string,
): T | undefined {
  return items.find((item) => item.id === id);
}
```

Avoid abstractions that hide simple logic or make inference unpredictable.

---

## 7. Union Types and Exhaustiveness

Use discriminated unions to represent mutually exclusive states.

Example:

```ts
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: AppError };
```

This prevents invalid combinations such as a successful state with no data or an error state without an error.

### 7.1 Exhaustive Switches

Handle every member of a finite union.

```ts
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${String(value)}`);
}

function getBookingLabel(status: BookingStatus): string {
  switch (status) {
    case "pending":
      return "Pending";

    case "confirmed":
      return "Confirmed";

    case "cancelled":
      return "Cancelled";

    case "completed":
      return "Completed";

    default:
      return assertNever(status);
  }
}
```

Exhaustiveness checks must be used for domain states where missing a case could lead to incorrect UI or behavior.

Do not silently map unknown domain states to a valid state.

If the backend contract intentionally supports forward-compatible unknown values, model that behavior explicitly and render a safe fallback.

---

## 8. Type Narrowing and Type Guards

Prefer native TypeScript narrowing and well-defined type guards over unsafe assertions.

Example:

```ts
function isOrder(value: unknown): value is Order {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "status" in value
  );
}
```

For complex data, prefer runtime schema validation over manually implemented incomplete type guards.

### 8.1 Null and Undefined Safety

- Handle nullable values explicitly.
- Avoid non-null assertions (`!`) unless a proven invariant justifies them.
- Prefer optional chaining and nullish coalescing where semantically correct.
- Avoid truthiness checks when valid values may be `0`, `false`, or an empty string.
- Validate array indexes and object access when values may be absent.

Avoid:

```ts
const firstOrder = orders[0]!;
```

Prefer:

```ts
const firstOrder = orders[0];

if (!firstOrder) {
  return null;
}
```

### 8.2 Type Assertions

Type assertions must be rare and justified.

Avoid:

```ts
const order = response.data as Order;
```

This does not validate the runtime payload.

Prefer validating at the boundary:

```ts
const order = orderSchema.parse(response.data);
```

When an assertion is genuinely necessary, keep it localized and document the invariant that makes it safe.

Do not use double assertions such as `value as unknown as Target` to bypass a mismatched contract without a reviewed adapter or boundary.

---

## 9. Zod and Runtime Validation

Zod is the preferred runtime validation library for external and user-provided data.

Use it for:
- API response validation where runtime guarantees are required.
- Form validation.
- URL and query parameter parsing.
- Environment variable parsing.
- Persisted client state.
- WebSocket event validation.
- Third-party integration payloads.
- File metadata and import boundaries.

### 9.1 Schema and Type Alignment

Infer TypeScript types from schemas where the schema is the canonical runtime contract.

```ts
import { z } from "zod";

export const orderStatusSchema = z.enum([
  "pending",
  "confirmed",
  "cancelled",
  "completed",
]);

export const orderSchema = z.object({
  id: z.string().min(1),
  status: orderStatusSchema,
  createdAt: z.string().datetime(),
});

export type Order = z.infer<typeof orderSchema>;
```

Avoid maintaining a separate handwritten type that duplicates the same schema without a clear reason.

### 9.2 Input and Output Types

Use `z.input` and `z.output` when transformations create different input and parsed output types.

```ts
const quantitySchema = z
  .string()
  .transform(Number)
  .pipe(z.number().int().positive());

type QuantityInput = z.input<typeof quantitySchema>;
type Quantity = z.output<typeof quantitySchema>;
```

Do not assume that raw form input and validated domain values are identical.

### 9.3 Parsing Strategy

Choose between `parse` and `safeParse` based on the error-handling contract.

- Use `parse` when invalid data should fail the current operation through an exception.
- Use `safeParse` when validation failure is an expected result that should be handled explicitly.

Do not silently discard invalid data without an intentional fallback strategy.

### 9.4 Schema Ownership

Schemas should live close to the domain or boundary they validate.

- Form schemas belong to their feature or shared form contract.
- API schemas belong to the relevant API/domain boundary.
- Shared schemas must represent genuinely shared contracts.
- Avoid a single enormous global schema file.
- Avoid circular imports between types, schemas, and feature modules.

### 9.5 Backend Contract Authority

Frontend Zod schemas improve runtime safety but do not replace backend validation.

The backend remains authoritative for business rules, authorization, pricing, inventory, bookings, payments, and other domain operations.

Where API contracts are generated from a shared specification, prefer that source for compile-time contracts and use runtime validation where it provides meaningful protection.

---

## 10. API and Data Contracts

### 10.1 API Response Types

API responses must use explicit contracts.

```ts
interface ApiError {
  code: string;
  message: string;
  requestId?: string;
}

interface ApiResponse<T> {
  data: T;
  requestId: string;
}
```

The exact response envelope must follow KAMPYN's canonical API contract. Do not introduce a new envelope in an individual feature.

### 10.2 Separate API, Domain, and UI Models

Do not assume every API response is directly suitable for every component.

Separate models when they have genuinely different responsibilities:

- API DTO: represents the backend contract.
- Domain model: represents frontend domain behavior where required.
- Form model: represents editable input.
- View model: represents data prepared for a specific UI.

Avoid creating multiple layers of types when they provide no meaningful distinction.

### 10.3 Avoid Leaking Backend Internals

Components should not depend unnecessarily on:
- Database-specific fields.
- Internal storage structures.
- Backend-only implementation details.
- Raw persistence documents.
- Internal service metadata.

Use a defined API boundary and adapt data where required.

### 10.4 Shared Contracts

Shared API contracts should have explicit ownership and versioning.

If KAMPYN provides an SDK or generated client:
- Reuse canonical public contracts.
- Avoid manually recreating generated types.
- Validate runtime inputs at trust boundaries.
- Keep frontend-only presentation types separate.
- Handle API version compatibility deliberately.

---

## 11. React Component Typing

### 11.1 Props

All reusable components must have explicit, meaningful prop contracts.

```tsx
interface FoodItemCardProps {
  item: FoodItem;
  onSelect?: (itemId: string) => void;
  disabled?: boolean;
}

export function FoodItemCard({
  item,
  onSelect,
  disabled = false,
}: FoodItemCardProps) {
  // Component implementation
}
```

Avoid:
- Broad `object` props.
- `any` props.
- Unnecessary optional properties.
- Ambiguous boolean combinations.
- Prop contracts that expose internal implementation details.

### 11.2 Children

Use `React.ReactNode` when a component accepts arbitrary renderable children.

```tsx
interface CardProps {
  children: React.ReactNode;
}
```

Use narrower types when the component expects a specific element or render function.

Do not use `React.FC` by default. Prefer ordinary function declarations or arrow functions with explicit prop types, following the repository's convention.

### 11.3 Event Types

Use React's event types when explicit annotation is necessary.

```tsx
function handleChange(
  event: React.ChangeEvent<HTMLInputElement>,
) {
  setValue(event.currentTarget.value);
}
```

Prefer contextual typing when JSX already infers the event type.

Avoid using generic browser event types where React-specific types are required.

### 11.4 Polymorphic Components

Polymorphic components must preserve the type relationship between the selected element and its props.

Do not build polymorphic APIs unless they solve a real reuse requirement.

Prefer simple explicit variants when they provide clearer contracts and better accessibility.

### 11.5 Refs

Use the appropriate ref type for the target element.

```tsx
const inputRef = useRef<HTMLInputElement>(null);
```

Avoid unsafe ref assertions or sharing refs between incompatible element types.

---

## 12. Hooks and Custom Hook Typing

Custom hooks must expose stable, predictable return contracts.

Example:

```ts
interface UseOrdersResult {
  orders: Order[];
  isPending: boolean;
  isError: boolean;
  refetch: () => Promise<unknown>;
}
```

Where practical, infer the return type from TanStack Query or other established APIs rather than duplicating their entire type structures.

Custom hooks must:
- Follow React's Rules of Hooks.
- Have descriptive names beginning with `use`.
- Expose only necessary values and actions.
- Avoid leaking internal implementation details.
- Preserve correct generic types.
- Keep side effects explicit.
- Avoid unnecessary wrapper hooks that add no abstraction value.

Do not suppress hook-related TypeScript errors to bypass incorrect hook usage.

---

## 13. TanStack Query Typing

TanStack Query must use precise query, mutation, and key types.

### 13.1 Query Functions

Query functions must return a known data type.

```ts
async function getOrders(
  filters: OrderFilters,
  signal?: AbortSignal,
): Promise<Order[]> {
  const response = await apiClient.get("/orders", {
    params: filters,
    signal,
  });

  return orderListSchema.parse(response.data);
}
```

### 13.2 Query Keys

Use typed query-key factories that accurately reflect the query parameters and identity context.

```ts
export const orderKeys = {
  all: ["orders"] as const,

  tenant: (tenantId: TenantId) =>
    [...orderKeys.all, tenantId] as const,

  list: (
    tenantId: TenantId,
    filters: OrderFilters,
  ) => [...orderKeys.tenant(tenantId), "list", filters] as const,

  detail: (tenantId: TenantId, orderId: OrderId) =>
    [...orderKeys.tenant(tenantId), "detail", orderId] as const,
};
```

Do not use query-key typing as a substitute for backend access control.

### 13.3 Mutation Variables

Mutation variables must be explicit and cohesive.

```ts
interface CancelOrderVariables {
  tenantId: TenantId;
  orderId: OrderId;
  reason?: string;
}
```

Avoid passing loosely structured objects or multiple positional parameters when a named contract improves clarity.

### 13.4 Mutation Results

Mutation return types must reflect the backend result.

Do not type a mutation as successful if the backend only accepted a request for asynchronous processing.

Distinguish:
- Request accepted.
- Operation pending.
- Operation confirmed.
- Operation failed.
- Result requiring verification.

### 13.5 Cache Types

When updating the query cache:
- Use correct data types.
- Preserve immutable update behavior.
- Avoid unsafe casts.
- Validate incoming event data before cache updates.
- Handle missing cache entries explicitly.
- Avoid creating invalid partial representations of domain entities.

---

## 14. Zustand Typing

Zustand stores must have explicit state and action contracts.

```ts
interface CartState {
  items: CartDraftItem[];
  addItem: (item: CartDraftItem) => void;
  removeItem: (itemId: string) => void;
  clear: () => void;
}
```

### 14.1 Store Type Safety

- Type state and actions.
- Keep action parameters narrow.
- Use readonly types for values that must not be mutated by consumers.
- Avoid `any` in selectors and middleware.
- Keep persisted state types distinct from runtime-only state when appropriate.
- Validate hydrated state at runtime.

### 14.2 Selectors

Selectors must preserve the type of the selected value.

```ts
const items = useCartStore((state) => state.items);
```

Avoid returning overly broad types or exposing internal mutable references where they can be modified outside store actions.

### 14.3 Middleware

Middleware configuration must preserve the store's inferred types.

When using persistence, devtools, or other middleware:
- Follow the installed Zustand version's supported typing patterns.
- Avoid casting the entire store to bypass middleware type errors.
- Validate persisted values before they enter runtime state.
- Keep middleware composition centralized and consistent.

---

## 15. Type-Safe Forms

Form types must accurately represent the lifecycle of user input.

- Use Zod schemas for runtime validation where appropriate.
- Infer form input and parsed output types from schemas.
- Distinguish raw input from normalized domain values.
- Type field errors and server validation errors.
- Avoid generic form state that erases field-level type information.
- Keep form state separate from authoritative server data.

Example:

```ts
const bookingFormSchema = z.object({
  resourceId: z.string().min(1),
  startTime: z.string().datetime(),
  endTime: z.string().datetime(),
});

type BookingFormInput = z.input<typeof bookingFormSchema>;
type BookingFormData = z.output<typeof bookingFormSchema>;
```

Validation must not imply that a booking is available or authorized. The backend must confirm those conditions.

---

## 16. Error Typing

Errors must have a consistent, explicit representation at application boundaries.

Example:

```ts
type AppErrorCode =
  | "UNAUTHENTICATED"
  | "FORBIDDEN"
  | "VALIDATION_FAILED"
  | "NOT_FOUND"
  | "CONFLICT"
  | "RATE_LIMITED"
  | "NETWORK_ERROR"
  | "INTERNAL_ERROR";

interface AppError {
  code: AppErrorCode;
  message: string;
  requestId?: string;
  fieldErrors?: Record<string, string[]>;
}
```

This is an illustrative shape. Use the canonical KAMPYN API error contract if one is defined.

### 16.1 Unknown Errors

JavaScript exceptions may contain arbitrary values.

Handle caught values as `unknown`:

```ts
try {
  await performAction();
} catch (error: unknown) {
  const message =
    error instanceof Error
      ? error.message
      : "An unexpected error occurred.";

  reportError(message);
}
```

Do not assume every caught value is an `Error`.

### 16.2 Safe Error Presentation

Do not expose:
- Stack traces.
- Internal service names.
- Database errors.
- Secrets.
- Raw infrastructure messages.
- Sensitive request payloads.

Map internal errors to safe user-facing messages through the established error-handling layer.

---

## 17. Utility and Shared Types

Shared types must be introduced only when they have clear, reusable meaning.

Suitable examples:
- Pagination metadata.
- Sort descriptors.
- Standardized API error contracts.
- Shared identifier primitives.
- Generic result wrappers.
- Reusable form field contracts.

Avoid:
- Giant shared type registries.
- Premature generic frameworks.
- Duplicate aliases for identical primitive values without benefit.
- Utility types that obscure simple contracts.
- Circular dependencies between feature modules.

### 17.1 Utility Types

Use TypeScript utility types where they accurately express intent.

Examples:
- `Pick`
- `Omit`
- `Partial`
- `Required`
- `Readonly`
- `Record`
- `ReturnType`
- `Parameters`
- `Awaited`

Do not use `Partial<T>` indiscriminately for update contracts. Define specific update DTOs when only certain fields may change.

### 17.2 Readonly Types

Use `readonly` or `Readonly<T>` when consumers should not mutate data.

Use `ReadonlyArray<T>` or `readonly T[]` for immutable collection contracts where appropriate.

Readonly is a compile-time constraint and does not deeply freeze runtime objects.

---

## 18. Type-Safe Environment Configuration

Environment variables must be validated and typed at application boundaries.

Do not scatter direct `process.env` access across feature components.

Use a centralized configuration module that:
- Validates required values.
- Distinguishes server-only and public variables.
- Rejects missing or malformed configuration early.
- Avoids exposing secrets to client bundles.
- Documents environment-specific requirements.

Example:

```ts
const publicEnvSchema = z.object({
  NEXT_PUBLIC_API_BASE_URL: z.string().url(),
});

export const publicEnv = publicEnvSchema.parse({
  NEXT_PUBLIC_API_BASE_URL:
    process.env.NEXT_PUBLIC_API_BASE_URL,
});
```

Server-only secrets must be validated in a server-only module and must never be imported into client components.

Never use TypeScript declarations to pretend an environment variable exists without runtime validation.

---

## 19. Type-Safe URL and Search Parameters

URL parameters are untrusted strings and must be parsed.

- Validate required parameters.
- Normalize optional parameters.
- Validate pagination and sorting values.
- Handle invalid enum values safely.
- Avoid unsafe numeric conversions.
- Use Zod or explicit parsing helpers for complex parameter sets.

Example:

```ts
const orderFiltersSchema = z.object({
  status: z
    .enum(["pending", "confirmed", "cancelled", "completed"])
    .optional(),
  page: z.coerce.number().int().positive().default(1),
});
```

The schema must match the actual route contract and avoid silently accepting invalid business states.

---

## 20. Real-Time and Event Typing

WebSocket, server-sent event, and third-party event payloads must be validated at runtime.

Use explicit event discriminators:

```ts
type OrderEvent =
  | {
      type: "order.updated";
      tenantId: string;
      orderId: string;
      status: OrderStatus;
    }
  | {
      type: "order.cancelled";
      tenantId: string;
      orderId: string;
    };
```

Do not trust an event merely because its TypeScript interface exists.

At the receiving boundary:
- Validate the event shape.
- Check supported event types.
- Handle unknown events safely.
- Scope events to the active tenant and user context.
- Avoid unsafe cache updates.
- Handle duplicate and out-of-order events.

The backend remains the source of truth for event authorization and durable state.

---

## 21. Next.js Server and Client Type Boundaries

TypeScript must support clear server/client separation.

### 21.1 Server Components

Server components may access approved server-side services and data sources according to the architecture.

- Keep server-only types and modules out of client bundles.
- Avoid importing server-only dependencies into client components.
- Use serializable props when passing data across the server/client boundary.
- Validate external data before rendering or passing it onward.
- Avoid leaking secrets through props or serialized output.

### 21.2 Client Components

Client components must:
- Use browser APIs only in valid client execution contexts.
- Receive appropriately typed, serializable props when crossing the boundary.
- Avoid relying on server-only modules.
- Keep client state types distinct from server implementation details.

### 21.3 Server Actions and Route Handlers

Server Actions and Route Handlers must:
- Validate incoming values at runtime.
- Use explicit input and output contracts.
- Apply authentication and authorization checks.
- Return safe, predictable results.
- Avoid using client-supplied types as proof of valid input.

Server Actions must not duplicate backend domain logic where KAMPYN's backend is authoritative.

### 21.4 Serialization

Do not pass unsupported runtime values across serialization boundaries without deliberate transformation.

Be careful with:
- `Date` objects.
- `Map` and `Set`.
- Class instances.
- Functions.
- BigInt.
- Database-specific types.
- Undefined values.
- Circular structures.

Use explicit DTOs and serialization helpers where needed.

---

## 22. Type Safety for Security and Multi-Tenancy

Types can reduce accidental misuse but cannot enforce authorization.

- Use explicit tenant and user context types.
- Keep tenant-scoped identifiers distinct where helpful.
- Avoid optional tenant context in operations that require it.
- Represent permission data using known contracts.
- Avoid broad string-based permission checks scattered across components.
- Validate all externally supplied tenant and identity values.
- Never trust client-side roles or permissions as an authorization decision.

Example:

```ts
interface TenantContext {
  tenantId: TenantId;
  displayName: string;
}
```

The existence of a `TenantContext` object does not prove the current user is authorized to access that tenant. The backend must validate access for every protected operation.

---

## 23. Performance and Type Complexity

Type safety must not create unnecessary maintenance or compilation overhead.

- Prefer simple, understandable types.
- Avoid excessive conditional and recursive types.
- Avoid deeply nested generic abstractions.
- Limit unnecessary declaration merging.
- Avoid duplicating large mapped types.
- Keep module dependency graphs clear.
- Avoid importing large type-only dependency trees into unrelated features.

Use `import type` for type-only imports where appropriate:

```ts
import type { Order } from "@/features/orders/types/order";
```

With `verbatimModuleSyntax`, preserve the distinction between runtime imports and type-only imports.

Do not sacrifice meaningful runtime validation or API clarity solely to reduce type complexity.

---

## 24. TypeScript and Code Quality

All TypeScript code must follow KAMPYN's general engineering standards.

- No redundant code.
- Prefer reuse where it improves cohesion.
- Keep modules small and focused.
- Keep production files at or below 200 lines where practical.
- Use descriptive names.
- Avoid deeply nested control flow.
- Avoid unnecessary mutation.
- Use immutable transformations where appropriate.
- Avoid inefficient repeated scans and nested loops for large datasets.
- Document non-obvious contracts and invariants.
- Keep public APIs stable and intentional.

Types should clarify implementation rather than conceal complexity.

### 24.1 Comments

Comments must explain:
- Non-obvious invariants.
- Important domain constraints.
- Workarounds and their removal conditions.
- Complex generic behavior.
- Security or serialization boundaries.

Do not add comments that merely restate the type or function name.

### 24.2 Deprecation

When a type or exported contract is being replaced:
- Mark the old contract as deprecated where appropriate.
- Provide a migration path.
- Update consumers.
- Remove obsolete types after migration.
- Avoid maintaining multiple competing contracts indefinitely.

---

## 25. Linting and Static Analysis

TypeScript code must pass the repository's configured:
- TypeScript compiler.
- ESLint rules.
- React and Next.js lint rules.
- Import and module boundary checks.
- Formatting rules.
- Type-aware linting where configured.

Avoid:
- Disabling lint rules at file scope without justification.
- Using `@ts-ignore` to silence real type errors.
- Using `@ts-expect-error` without a clear reason and relevant test.
- Broad lint suppressions.
- Unchecked casts at API boundaries.

If an exception is necessary:
- Keep it as narrow as possible.
- Explain why it is safe.
- Include a follow-up or test where applicable.

---

## 26. Testing Requirements

TypeScript contracts must be supported by tests where behavior or runtime validation is involved.

### 26.1 Unit Tests

Test:
- Domain transformations.
- Type guard behavior.
- Runtime schema validation.
- URL parsing and normalization.
- Error mapping.
- Reducer and state transitions.
- Generic utilities with representative type cases.

### 26.2 Compile-Time Tests

Where public generic APIs or SDK contracts are complex, use compile-time tests to verify:
- Valid usage is accepted.
- Invalid usage is rejected.
- Generic inference behaves as expected.
- Public types remain stable.

Do not rely on compile-time tests for runtime data validation.

### 26.3 Integration Tests

Test:
- API request and response contracts.
- Query and mutation boundaries.
- Form input parsing.
- Server action validation.
- Real-time event parsing.
- Authentication and tenant context propagation.
- Persistence hydration and migration.

### 26.4 Regression Tests

When fixing a type-related runtime bug:
- Add a regression test for the behavior.
- Add schema validation where the issue occurred at a trust boundary.
- Add a compile-time test if the issue involved an exported contract and compile-time rejection is feasible.

---

## 27. Anti-Patterns

The following are prohibited unless an explicit, reviewed exception exists:

- Using `any` as a shortcut.
- Using type assertions to bypass incorrect contracts.
- Treating compile-time types as runtime validation.
- Using broad `string` types for finite domain states.
- Using `object` or `Function` instead of meaningful types.
- Duplicating schemas and types without clear ownership.
- Creating one giant global type file.
- Overusing generic or conditional types.
- Using `Partial<T>` for every update contract.
- Ignoring `null` or `undefined` possibilities.
- Using non-null assertions without a proven invariant.
- Using `@ts-ignore` to suppress errors.
- Casting API responses directly to trusted domain types.
- Trusting browser storage or real-time events based on TypeScript interfaces.
- Exposing database-specific models directly to UI components.
- Passing non-serializable data across server/client boundaries without transformation.
- Importing server-only modules into client components.
- Treating client-side tenant or role types as authorization.
- Weakening global TypeScript settings to accommodate individual files.
- Building abstractions that make types harder to understand than the code they protect.

---

## 28. Code Review Checklist

Before approving TypeScript changes, verify:

- [ ] Strict TypeScript settings are preserved.
- [ ] Types reflect real domain and API contracts.
- [ ] No unnecessary `any` exists.
- [ ] Untrusted inputs are treated as `unknown`.
- [ ] Runtime validation is applied at relevant boundaries.
- [ ] Zod schemas and inferred types have clear ownership.
- [ ] Public functions and exports have clear contracts.
- [ ] Nullable and optional values are modelled accurately.
- [ ] Discriminated unions represent mutually exclusive states.
- [ ] Exhaustive handling is used for important finite states.
- [ ] Assertions are rare, localized, and justified.
- [ ] API DTOs are separated from UI models where needed.
- [ ] React props and hooks have appropriate types.
- [ ] TanStack Query keys, functions, and mutations are typed.
- [ ] Zustand stores and selectors are typed.
- [ ] Form input and parsed output types are distinguished where necessary.
- [ ] Errors follow the established application contract.
- [ ] URL and environment inputs are validated.
- [ ] Server/client boundaries are respected.
- [ ] Tenant and user context types are not treated as security boundaries.
- [ ] Type complexity remains maintainable.
- [ ] Linting and static analysis pass.
- [ ] Relevant runtime and compile-time tests are included.
- [ ] Files follow repository organization and size standards.

---

## 29. Definition of Done

A TypeScript implementation is complete when:

- All relevant code compiles under strict TypeScript settings.
- Types represent the actual domain and system contracts.
- Untrusted data is validated at appropriate runtime boundaries.
- No unjustified `any`, unsafe assertion, or suppression remains.
- Public interfaces are clear, stable, and maintainable.
- Server/client boundaries and serialization requirements are respected.
- State, API, and form contracts are consistent.
- Tenant and user isolation assumptions are explicit.
- Relevant tests cover runtime behavior and validation.
- Linting and static analysis pass.
- Documentation is updated for meaningful public contract changes.
- The implementation follows KAMPYN's modularity, reuse, and file-size standards.

**Final principle:** TypeScript must make KAMPYN's contracts explicit, catch mistakes before runtime, and make unsafe assumptions visible—not hide them behind assertions, broad types, or compiler suppressions.