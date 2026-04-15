# Barbapapazes Vue snippet patterns

## Shared structure

The component snippets consistently use:
- a top `<script lang="ts">` block
- a `tv({ slots: { base: '' } })` recipe
- exported interfaces for `Props`, `Emits`, and `Slots`
- `class?: any` in props
- `ui?: Partial<typeof recipe.slots>` in props
- a `<script lang="ts" setup>` block
- `const props = defineProps<...>()`
- `defineEmits<...>()`
- `defineSlots<...>()`
- `const ui = computed(() => recipe())`

## Base component pattern

The full component snippet starts very light:
- define the `tv` recipe
- export the interfaces
- create `props`, emits, slots, and computed `ui`
- leave the template mostly open for the caller to fill in

## Wrapper component pattern

The modal and slideover variants add:
- `title` and `description` placeholders
- wrapper usage such as `UModal` or `USlideover`
- class application using `ui.base({ class: [props.ui?.base, props.class] })`
- user content inside the wrapper body slot

## `tv` import rule of thumb

The source snippet shows `tv(...)` usage but does not itself demonstrate the import decision. For generated components, use this order of precedence:

1. Match the local repository convention if examples already exist.
2. If local files import `tv` from `tailwind-variants`, do the same.
3. If local files use `tv` without an import, assume an auto-import or global setup.
4. If no evidence exists, prefer an explicit `import { tv } from 'tailwind-variants'` unless the project clearly documents auto-imports.

## Vue import rule of thumb

Apply the same convention-sensitive logic to `computed` and any other helpers:
- import explicitly in standard Vue/Vite codebases
- omit imports when the project uses auto-imported Vue APIs

## Biases to avoid

Do not:
- assume Nuxt auto-imports unless the project suggests it
- add variants, compound variants, or slots the user did not ask for
- replace local typing conventions just because the snippet uses `any`
- invent extra business logic in a scaffold component
