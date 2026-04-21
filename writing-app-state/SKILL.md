---
name: writing-app-state
description: Use when creating or modifying Angular view-models, managing component state with signals/computed, or implementing action methods
license: MIT
metadata:
  language: typescript
  framework: angular
---

## View-Model pattern

### Structure

A view-model is an `@Injectable` class that holds:
1. **Signals** for mutable state (private writable, exposed as readonly).
2. **Computed signals** for derived state.
3. **Action methods** that update state and trigger side effects.
4. **Data-fetching state**: `loading`, `error`, `data` — always explicit.

### Provisioning

View-models are **not** root-scoped. Provide them at the route or page component level:

```typescript
@Component({
  selector: 'app-recipe-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  providers: [RecipeListViewModel],
  template: `...`,
})
export class RecipeListComponent {
  protected readonly vm = inject(RecipeListViewModel);
}
```

Or at the route level for shared state across child routes:

```typescript
// recipes.routes.ts
export const ROUTES: Routes = [
  {
    path: '',
    providers: [RecipeListViewModel],
    component: RecipeListComponent,
  },
];
```

### Rules

- View-models are `@Injectable()` with no `providedIn`.
- Provide at the component via `providers: [...]` or at the route level.
- **Reads** use `httpResource` — no manual loading/error signals needed. See writing-data-fetching skill.
- **Writes** use `HttpClient` via service layer — manage loading/error manually.
- Every view-model that fetches data must expose `loading`, `error`, and the data signal. The template must handle all three states.
- Use `signal.set()` for replacing values, `signal.update()` for transforming. Never use `signal.mutate()`.
- Prefer `computed()` over manual derivation in methods.

## Route params

Read route params directly in the view-model using `toSignal` — no `setId()` methods, no component-to-VM param passing:

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { ActivatedRoute } from '@angular/router';
import { map } from 'rxjs/operators';

@Injectable()
export class RecipeDetailViewModel {
  private readonly route = inject(ActivatedRoute);
  private readonly recipeService = inject(RecipeService);

  private readonly id = toSignal(this.route.paramMap.pipe(map(p => p.get('id')!)));

  private readonly recipeResource = httpResource<Recipe>(() => `/api/recipes/${this.id()}`);

  readonly recipe = computed(() => this.recipeResource.value() ?? null);
  readonly loading = this.recipeResource.isLoading;
  readonly error = computed(() => this.recipeResource.error() as string | null);
}
```

- `id` is a signal derived from the route — when the param changes, `httpResource` re-fetches automatically.
- The view-model is self-contained: it reads its own params, no external wiring needed.

## Signals — always

- **All state is signals.** No plain class properties for template-bound state.
- `signal<T>(initialValue)` for writable state.
- `computed(() => expr)` for derived state.
- `effect(() => { ... })` for side effects (rare — prefer explicit action methods).
- Read with `mySignal()`, write with `.set()` or `.update()`.
- **Never RxJS** in new code. No `Observable`, `Subject`, `BehaviorSubject`, `async pipe`, `subscribe()`.
- Exception: `firstValueFrom` in services, `toSignal` + `paramMap` for route params.

## Example: list view-model

```typescript
@Injectable()
export class RecipeListViewModel {
  private readonly recipeService = inject(RecipeService);

  private readonly recipesResource = httpResource<Recipe[]>(() => '/api/recipes');

  readonly recipes = computed(() => this.recipesResource.value() ?? []);
  readonly loading = this.recipesResource.isLoading;
  readonly error = computed(() => this.recipesResource.error() as string | null);

  readonly hasRecipes = computed(() => this.recipes().length > 0);
  readonly recipeCount = computed(() => this.recipes().length);

  private readonly deleting = signal(false);
  readonly deleting = this.deleting.asReadonly();

  async deleteRecipe(id: string): Promise<void> {
    this.deleting.set(true);
    try {
      await this.recipeService.delete(id);
    } finally {
      this.deleting.set(false);
    }
  }
}
```

## Example: detail view-model (route-param driven)

See `writing-data-fetching` for the full read+write integration pattern. Key structure:

```typescript
@Injectable()
export class RecipeDetailViewModel {
  private readonly route = inject(ActivatedRoute);
  private readonly recipeService = inject(RecipeService);
  private readonly router = inject(Router);

  // Read — driven by route param
  private readonly id = toSignal(this.route.paramMap.pipe(map(p => p.get('id')!)));
  private readonly recipeResource = httpResource<Recipe>(() => `/api/recipes/${this.id()}`);

  readonly recipe = computed(() => this.recipeResource.value() ?? null);
  readonly loading = this.recipeResource.isLoading;
  readonly error = computed(() => this.recipeResource.error() as string | null);

  // Write — manual state for mutation
  private readonly deleting = signal(false);
  readonly deleting = this.deleting.asReadonly();

  async delete(): Promise<void> {
    if (!this.recipe()) return;
    this.deleting.set(true);
    try {
      await this.recipeService.delete(this.recipe()!.id);
      this.router.navigate(['/recipes']);
    } finally {
      this.deleting.set(false);
    }
  }
}
```

## Write operation state

For mutations, manage loading/error in the view-model. See `writing-data-fetching` for full examples of submit state, error handling, and template integration.

```typescript
private readonly submitting = signal(false);
private readonly submitError = signal<string | null>(null);

async save(data: CreateRecipeRequest): Promise<void> {
  this.submitting.set(true);
  this.submitError.set(null);
  try {
    const recipe = await this.recipeService.create(data);
    this.savedRecipe.set(recipe);
  } catch (e) {
    this.submitError.set(e instanceof Error ? e.message : 'Failed to save');
  } finally {
    this.submitting.set(false);
  }
}
```

## Local UI state

For transient UI state (modals, selections, toggles), signals are sufficient:

```typescript
@Injectable()
export class RecipeListViewModel {
  private readonly selectedId = signal<string | null>(null);
  private readonly filterText = signal('');

  readonly selectedId = this.selectedId.asReadonly();
  readonly filterText = this.filterText.asReadonly();

  readonly filteredRecipes = computed(() => {
    const filter = this.filterText().toLowerCase();
    return this.recipes().filter(r => r.name.toLowerCase().includes(filter));
  });

  select(id: string | null): void {
    this.selectedId.set(id);
  }

  setFilter(text: string): void {
    this.filterText.set(text);
  }
}
```

## Cross-feature state

Avoid global stores. When state must be shared:

1. **Root-scoped service** — `@Injectable({ providedIn: 'root' })` with signals.
2. **Input binding** — pass state down via `input()`.
3. **Route params** — read via `toSignal` + `ActivatedRoute.paramMap`.

```typescript
@Injectable({ providedIn: 'root' })
export class AuthState {
  private readonly user = signal<User | null>(null);
  readonly user = this.user.asReadonly();
  readonly isAuthenticated = computed(() => this.user() !== null);

  setUser(user: User | null): void {
    this.user.set(user);
  }
}
```

## View-model lifecycle

- View-models are created when the component is created, destroyed when the component is destroyed.
- `httpResource` is automatically cleaned up on destroy.
- No manual subscription management needed — signals handle reactivity.
- Route params are read via `toSignal` in the constructor — no component-to-VM wiring.

## What NOT to do

- No global state management libraries (NgRx, Akita, Elf).
- No `BehaviorSubject` or `Subject` for state — use signals.
- No storing Observables as state — convert to signals immediately.
- No `signal.mutate()`.
- No `setId()` methods — use route params via `toSignal` instead.
- No business logic in components — it goes in the view-model.

## Common rationalizations

| Excuse | Reality |
|--------|---------|
| "I'll use BehaviorSubject for this" | Use signals. No RxJS for state. |
| "A setId() method is simpler" | Read params via `toSignal` + `paramMap` in the view-model. |
| "This state needs to be global" | Root-scoped service with signals, or route-level providers. |
| "I'll manage loading manually" | `httpResource` handles loading/error for reads automatically. |
