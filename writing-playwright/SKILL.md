---
name: writing-playwright
description: Use when writing or modifying Playwright E2E tests, page object models, or test helpers in the frontend/e2e/ directory
license: MIT
metadata:
    language: typescript
    framework: playwright
---

## Core principles

- Tests live in `frontend/e2e/` mirroring feature structure.
- Every page has a Page Object Model — tests never call `page.getBy*` directly.
- Locators use `data-testid` attributes exclusively — no `getByRole`, `getByText`, `getByLabel`.
- Use `test.step()` blocks instead of comments to organize test logic.
- Never use `let` to capture network requests — use inline `waitForRequest`/`waitForResponse`.
- All Playwright commands run via `npx playwright` directly.

## File conventions

- Test files: `frontend/e2e/<feature>/<feature>.spec.ts`
- Page objects: `frontend/e2e/<feature>/<page>.po.ts` (e.g., `recipe-list.po.ts`)
- Helpers: `frontend/e2e/<feature>/helpers.ts` for shared mocks/fixtures.
- Test file naming: kebab-case, `.spec.ts` suffix.

## Page Object Models

Every page under test gets a PO class. The PO owns all locators and interactions — tests call PO methods only:

```typescript
import { type Page, type Locator } from "@playwright/test";

export class RecipeListPage {
    readonly heading: Locator;
    readonly recipeRows: Locator;
    readonly addButton: Locator;

    constructor(private readonly page: Page) {
        this.heading = page.getByTestId("recipe-list-heading");
        this.recipeRows = page.getByTestId("recipe-row");
        this.addButton = page.getByTestId("add-recipe-btn");
    }

    async goto() {
        await this.page.goto("/recipes");
    }

    async getRecipeNames(): Promise<string[]> {
        return this.recipeRows.allTextContents();
    }

    async clickAdd() {
        await this.addButton.click();
    }
}
```

Tests instantiate the PO and interact through it:

```typescript
test("shows recipes", async ({ page }) => {
    const recipeList = new RecipeListPage(page);

    await recipeList.goto();

    await test.step("verify recipe list is rendered", async () => {
        await expect(recipeList.heading).toBeVisible();
        await expect(recipeList.recipeRows).toHaveCount(2);
    });
});
```

## Locators — `data-testid` only

Every interactive or assertable element must have a `data-testid` attribute in the Angular template:

```html
<h1 data-testid="recipe-list-heading">Recipes</h1>
<button data-testid="add-recipe-btn">Add</button>
<div data-testid="recipe-row" *ngFor="let r of recipes">{{ r.name }}</div>
```

In page objects, always use `page.getByTestId('...')`. Never use `getByRole`, `getByText`, `getByLabel`, or CSS selectors.

## Network mocking with `page.route`

Use `page.route` to intercept and mock API responses. Set up routes before navigating. Always call the route handler's `fulfill` with explicit status and JSON body:

```typescript
import { test, expect } from "@playwright/test";
import { RecipeListPage } from "./recipe-list.po";

test("displays recipes from API", async ({ page }) => {
    await page.route("**/api/recipes", (route) =>
        route.fulfill({
            status: 200,
            contentType: "application/json",
            body: JSON.stringify([
                { id: 1, name: "Pasta" },
                { id: 2, name: "Salad" },
            ]),
        }),
    );

    const recipeList = new RecipeListPage(page);
    await recipeList.goto();

    await test.step("verify mocked recipes are visible", async () => {
        await expect(recipeList.recipeRows).toHaveCount(2);
        await expect(recipeList.recipeRows.nth(0)).toHaveText("Pasta");
        await expect(recipeList.recipeRows.nth(1)).toHaveText("Salad");
    });
});
```

## Verifying request payloads with `page.waitForRequest`

Use `page.waitForRequest` as an inline expression — never assign to a variable. Chain assertions directly on the resolved request:

```typescript
test("saves recipe on form submit", async ({ page }) => {
    const recipeForm = new RecipeFormPage(page);
    await recipeForm.goto();

    await recipeForm.fillName("Soup");
    await recipeForm.clickSave();

    await expect(
        page.waitForRequest(
            (req) =>
                req.url().includes("/api/recipes") && req.method() === "POST",
        ),
    ).resolves.toSatisfy(async (req) => {
        const body = req.postDataJSON();
        expect(body.name).toBe("Soup");
        return true;
    });
});
```

For simpler payload checks, use `test.step` to await and assert:

```typescript
test("creates recipe with correct payload", async ({ page }) => {
    const recipeForm = new RecipeFormPage(page);
    await recipeForm.goto();
    await recipeForm.fillName("Soup");

    const [request] = await Promise.all([
        page.waitForRequest("**/api/recipes"),
        recipeForm.clickSave(),
    ]);

    await test.step("verify request payload", async () => {
        const body = request.postDataJSON();
        expect(body.name).toBe("Soup");
        expect(body.variants).toHaveLength(1);
    });
});
```

> **Note:** The `Promise.all` pattern above is the one exception to the no-`let` rule — it's idiomatic Playwright for capturing the request that a click triggers.

## `test.step` structure

Organize every test into `test.step` blocks. No comments — steps serve as documentation:

```typescript
test("edits existing recipe", async ({ page }) => {
    await page.route("**/api/recipes/1", (route) =>
        route.fulfill({
            status: 200,
            contentType: "application/json",
            body: JSON.stringify({ id: 1, name: "Pasta", variants: [] }),
        }),
    );

    await page.route("**/api/recipes/1", (route) =>
        route.fulfill({ status: 200 }),
    );

    const detail = new RecipeDetailPage(page);

    await test.step("load recipe detail page", async () => {
        await detail.goto(1);
        await expect(detail.recipeName).toHaveText("Pasta");
    });

    await test.step("update the recipe name", async () => {
        await detail.fillName("Updated Pasta");
    });

    await test.step("save and verify API call", async () => {
        const [request] = await Promise.all([
            page.waitForRequest("**/api/recipes/1"),
            detail.clickSave(),
        ]);

        expect(request.method()).toBe("PUT");
        expect(request.postDataJSON().name).toBe("Updated Pasta");
    });
});
```

## Assertions

- Use Playwright's built-in `expect` — not Node's assert, not Vitest's.
- Prefer web-first assertions that auto-retry: `toBeVisible`, `toHaveText`, `toHaveValue`, `toHaveCount`.
- Use `toBeEnabled` / `toBeDisabled` for button state checks.
- Use `toHaveAttribute` for link/HREF verification.

## Running tests

**ALWAYS START WITH `cd frontend`** — all Playwright commands must be run from the `frontend/` directory:

```bash
cd frontend
npx playwright test
npx playwright test --grep "displays recipes"
npx playwright test --ui
```

Start `npx ng serve` separately before running tests.

## Recording traces

Traces are enabled by default in the config (`trace: 'on-first-retry'`). To record traces on every test run:

```bash
npx playwright test --trace on
```

## Reading traces

Use `playwright-trace-analyzer` from `uv` (it is a python package) to inspect trace files (found in `frontend/test-results/`):

```bash
# Get a high-level summary of the trace
playwright-trace-analyzer summary trace.zip

# View all actions executed during the test
playwright-trace-analyzer actions trace.zip
playwright-trace-analyzer actions trace.zip --errors-only

# Extract console messages and warnings
playwright-trace-analyzer console trace.zip
playwright-trace-analyzer console trace.zip --level error

# Check failed network requests
playwright-trace-analyzer network trace.zip --failed-only

# Extract screenshots from the trace
playwright-trace-analyzer screenshots trace.zip -o ./screenshots

# Extract screenshots with deduplication control
playwright-trace-analyzer screenshots trace.zip --dedupe-threshold 0.01  # default: drops visually identical frames
playwright-trace-analyzer screenshots trace.zip --dedupe-threshold 0     # disable deduplication

# View trace metadata
playwright-trace-analyzer metadata trace.zip
```

## Config notes

- `playwright.config.ts` sets `testDir: './e2e'`.
- Base URL is not set in config — use `page.goto('/path')` for relative navigation.
- Trace collection is on first retry (`trace: 'on-first-retry'`).

## What NOT to do

- No `let` for capturing requests — prefer `Promise.all` pattern or inline `waitForRequest`.
- No `page.waitForTimeout` — use auto-retrying assertions or `waitForSelector`.
- No `page.$` / `page.$$` — use `getByTestId` only.
- No `getByRole`, `getByText`, `getByLabel`, or CSS selectors — `getByTestId` exclusively.
- No calling `page.getBy*` directly in tests — always go through a Page Object.
- No comments — `test.step` blocks replace them.
- No running Playwright via Docker/Podman — always use `npx playwright` directly.
- No `test.slow()` — fix the test instead.

## Common rationalizations

| Excuse                                | Reality                                                              |
| ------------------------------------- | -------------------------------------------------------------------- |
| "getByRole is more semantic"          | `data-testid` only. Consistency over elegance.                       |
| "A comment would clarify this step"   | Use `test.step()` blocks instead. They serve as documentation.       |
| "I'll capture the request with a let" | Use inline `waitForRequest` or `Promise.all` pattern.                |
| "getByText is fine for this label"    | `data-testid` exclusively. No exceptions.                            |
| "I need to run Playwright via Docker" | Always use `npx playwright` directly from the `frontend/` directory. |
