# Generating E2E Test Specs

Generate Playwright Test spec files (.spec.ts) from interactive browser sessions for CI pipelines.

## Workflow

### Step 1: Interactive Exploration

Use playwright-cli to walk through the flow. Every command generates Playwright code:

```bash
playwright-cli -s=qa open http://localhost:5173/login
playwright-cli -s=qa snapshot
playwright-cli -s=qa fill e1 "doctor@hospital.com"
# Output: await page.getByRole('textbox', { name: 'Email' }).fill('doctor@hospital.com');
playwright-cli -s=qa fill e2 "password123"
# Output: await page.getByRole('textbox', { name: 'Password' }).fill('password123');
playwright-cli -s=qa click e3
# Output: await page.getByRole('button', { name: 'Sign In' }).click();
```

### Step 2: Collect Generated Code

Each playwright-cli action outputs the equivalent Playwright TypeScript code. Collect these into a test file.

### Step 3: Write the Spec File

Create spec in `e2e/` directory:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Login Flow', () => {
  test('successful login redirects to dashboard', async ({ page }) => {
    await page.goto('/login');

    // Generated from playwright-cli session
    await page.getByRole('textbox', { name: 'Email' }).fill('doctor@hospital.com');
    await page.getByRole('textbox', { name: 'Password' }).fill('password123');
    await page.getByRole('button', { name: 'Sign In' }).click();

    // Assertions (add manually)
    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
  });

  test('invalid credentials show error', async ({ page }) => {
    await page.goto('/login');
    await page.getByRole('textbox', { name: 'Email' }).fill('wrong@email.com');
    await page.getByRole('textbox', { name: 'Password' }).fill('wrongpass');
    await page.getByRole('button', { name: 'Sign In' }).click();

    await expect(page.getByText('Invalid credentials')).toBeVisible();
  });
});
```

### Step 4: Add Playwright Config

If the project doesn't have `playwright.config.ts`:

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30000,
  retries: 1,
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  webServer: {
    command: 'pnpm dev',
    port: 5173,
    reuseExistingServer: true,
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
  ],
});
```

## Auth Setup for Test Specs

Create a shared auth state to avoid re-login in every test:

```typescript
// e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test';
import path from 'path';

const authFile = path.join(__dirname, '.auth/user.json');

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByRole('textbox', { name: 'Email' }).fill('test@example.com');
  await page.getByRole('textbox', { name: 'Password' }).fill('password');
  await page.getByRole('button', { name: 'Sign In' }).click();
  await expect(page).toHaveURL(/.*dashboard/);
  await page.context().storageState({ path: authFile });
});
```

Then in config:
```typescript
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  {
    name: 'chromium',
    dependencies: ['setup'],
    use: { storageState: 'e2e/.auth/user.json' },
  },
],
```

## Test Organization

```
e2e/
  auth.setup.ts           # Login once, share state
  .auth/
    user.json             # Saved auth state (gitignored)
  patients/
    patient-list.spec.ts  # Patient listing tests
    patient-search.spec.ts
    patient-create.spec.ts
  dashboard/
    dashboard.spec.ts
  auth/
    login.spec.ts
    logout.spec.ts
```

## Package.json Scripts

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:headed": "playwright test --headed",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:report": "playwright show-report"
  }
}
```

## Best Practices

1. **Use role-based locators** — `getByRole`, `getByLabel`, `getByText` over CSS selectors
2. **One assertion per concern** — don't assert 10 things in one test
3. **Independent tests** — each test should work in isolation
4. **Descriptive names** — `test('doctor can search patients by MRN')` not `test('search')`
5. **Page Object Model** for large projects — encapsulate page interactions
