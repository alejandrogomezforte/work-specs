# Playwright E2E Testing Implementation Proposal

## Context

The project currently has Jest + React Testing Library for unit/component tests but **zero E2E testing infrastructure**. The CLAUDE.md references Playwright MCP as an interactive browser tool for Claude Code, but there are no persisted E2E test files, no Playwright config, and no CI pipeline for E2E tests.

This causes two problems:
1. CSS/layout bugs (like MLID-1574) can't be tested — JSDOM doesn't compute layout
2. The Claude Code review bot recommends E2E tests that can't actually be written

This plan covers what's needed to add Playwright as a proper test framework.

---

## Cost Analysis

### Setup effort
- **Configuration**: ~2 hours (config file, scripts, auth setup, turborepo integration)
- **First tests**: ~4 hours (auth flow, smoke tests for critical pages)
- **CI integration**: ~2 hours (Bitbucket Pipelines, browser install caching)
- **Total initial investment**: ~1 day

### Ongoing cost
- **Per-test authoring**: 15-30 min per test scenario
- **CI time**: ~2-5 min per pipeline run (parallelizable)
- **Maintenance**: Tests break when UI changes — but this is the point (catching regressions)

### Dependencies to install
```bash
npm install -D @playwright/test --workspace=@repo/web
npx playwright install chromium  # ~150MB, only Chromium needed initially
```

---

## Recommended Directory Structure

```
apps/web/
├── e2e/                              # All E2E tests live here
│   ├── fixtures/                     # Shared test fixtures and page objects
│   │   ├── auth.fixture.ts           # Authentication fixture (auto-login)
│   │   └── base.fixture.ts           # Base fixture extending Playwright's test
│   ├── pages/                        # Page Object Models (POM)
│   │   ├── clinical-review.page.ts   # Clinical review page interactions
│   │   ├── patient.page.ts           # Patient page interactions
│   │   ├── intakes.page.ts           # Intakes page interactions
│   │   └── login.page.ts             # Login page interactions
│   ├── tests/                        # Test files grouped by feature
│   │   ├── auth/
│   │   │   └── login.spec.ts         # Authentication flow tests
│   │   ├── clinical-reviews/
│   │   │   ├── pdf-viewer.spec.ts    # PDF viewer scroll/overflow tests
│   │   │   └── feedback.spec.ts      # Lab result feedback workflow
│   │   ├── intakes/
│   │   │   └── review.spec.ts        # Document intake review tests
│   │   ├── patients/
│   │   │   └── search.spec.ts        # Patient search and details
│   │   └── smoke.spec.ts             # Quick smoke tests for all critical pages
│   └── utils/                        # Test utilities
│       └── test-data.ts              # Test data factories
├── playwright.config.ts              # Playwright configuration
└── ...
```

This keeps E2E tests **separate from unit tests** (which are co-located with source files) to avoid confusion and allow different CI pipelines.

---

## Suggested Configuration

### `apps/web/playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e/tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI
    ? [['html', { open: 'never' }], ['junit', { outputFile: 'e2e-results.xml' }]]
    : [['html', { open: 'on-failure' }]],
  timeout: 30_000,

  use: {
    baseURL: 'http://localhost:8080',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    // Add more browsers later if needed:
    // { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    // { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } },
  ],

  // Start dev server automatically before tests
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:8080',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000, // Next.js can be slow to start
    env: {
      NEXT_PUBLIC_DISABLE_AUTH: 'true', // Bypass Google OAuth in tests
    },
  },
});
```

### Authentication Strategy

The app already has `NEXT_PUBLIC_DISABLE_AUTH=true` which bypasses Google OAuth and grants admin role. This is the simplest path for E2E tests — no need to mock OAuth flows.

**Auth fixture** (`e2e/fixtures/auth.fixture.ts`):

```typescript
import { test as base } from '@playwright/test';

// Extend base test with auto-authenticated context
export const test = base.extend({
  // The webServer config already sets NEXT_PUBLIC_DISABLE_AUTH=true,
  // so all requests are automatically authenticated as admin.
  // If we need role-based testing later, we can add storage state fixtures.
});

export { expect } from '@playwright/test';
```

### Package.json Scripts

Add to `apps/web/package.json`:

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:headed": "playwright test --headed",
    "test:e2e:report": "playwright show-report"
  }
}
```

Add to root `package.json`:

```json
{
  "scripts": {
    "test:e2e": "turbo test:e2e --filter=@repo/web"
  }
}
```

### Turborepo Integration

Add to `turbo.json`:

```json
{
  "tasks": {
    "test:e2e": {
      "dependsOn": ["^build"],
      "outputs": ["playwright-report/**", "test-results/**"],
      "cache": false
    }
  }
}
```

E2E tests should NOT be cached — they test runtime behavior.

### Jest Exclusion

Update `jest.config.js` to exclude E2E tests:

```javascript
testPathIgnorePatterns: ['/node_modules/', '/e2e/'],
```

---

## Example Tests

### Smoke Test (`e2e/tests/smoke.spec.ts`)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Smoke Tests', () => {
  test('home page loads', async ({ page }) => {
    await page.goto('/home');
    await expect(page).toHaveTitle(/MyLocalInfusion/);
  });

  test('admin page loads for admin users', async ({ page }) => {
    await page.goto('/admin');
    await expect(page.locator('h1, h2, h3').first()).toBeVisible();
  });
});
```

### PDF Viewer Scroll Test (`e2e/tests/clinical-reviews/pdf-viewer.spec.ts`)

This is the test that the review bot wanted — and it actually makes sense as an E2E test:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Clinical Review PDF Viewer', () => {
  // This test requires a real clinical review with documents in the test DB.
  // Skip if no test data is seeded.
  test('PDF viewer scrolls internally without expanding page height', async ({ page }) => {
    // Navigate to a clinical review with a multi-page PDF
    await page.goto('/patient/{testPatientId}/clinical-reviews/{testReviewId}');

    // Wait for the PDF to load
    const pdfViewer = page.locator('[class*="previewContent"]');
    await expect(pdfViewer).toBeVisible();

    // Get the initial window scroll position
    const initialScrollY = await page.evaluate(() => window.scrollY);

    // The container should have a fixed height (not expanding with content)
    const container = page.locator('[class*="container"]').first();
    const containerBox = await container.boundingBox();
    const viewportHeight = page.viewportSize()?.height ?? 720;

    // Container should not exceed viewport height
    expect(containerBox?.height).toBeLessThanOrEqual(viewportHeight);

    // Window should not have scrolled
    expect(initialScrollY).toBe(0);
  });
});
```

---

## CI/CD Integration

### Bitbucket Pipelines

Add a step to the default pipeline in `bitbucket-pipelines.yml`:

```yaml
- step:
    name: E2E Tests
    caches:
      - node
    script:
      - curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
      - apt-get install -y nodejs
      - npm ci
      - npx playwright install --with-deps chromium
      - npm run test:e2e
    artifacts:
      - apps/web/playwright-report/**
      - apps/web/test-results/**
```

### Required Environment Variables for CI

```
NEXT_PUBLIC_DISABLE_AUTH=true
MONGODB_URI=<test-database-uri>
```

External services (WeInfuse, Azure, Spruce, etc.) should either be mocked at the API level or pointed to test/staging endpoints.

---

## Git Ignores

Add to `apps/web/.gitignore`:

```
# Playwright
/test-results/
/playwright-report/
/blob-report/
/playwright/.cache/
```

---

## Rollout Plan

### Phase 1 — Foundation (Day 1)
- [ ] Install `@playwright/test` and Chromium
- [ ] Create `playwright.config.ts` with webServer config
- [ ] Create auth fixture using `NEXT_PUBLIC_DISABLE_AUTH`
- [ ] Write smoke tests for 3-5 critical pages
- [ ] Add npm scripts and turborepo task
- [ ] Update `.gitignore` and `jest.config.js` exclusion

### Phase 2 — Critical Path Tests (Week 1-2)
- [ ] Clinical review workflow (PDF viewer, feedback, completion)
- [ ] Patient search and details page
- [ ] Document intake review flow
- [ ] Authentication flow (login/logout)

### Phase 3 — CI Integration (Week 2)
- [ ] Add Playwright step to Bitbucket Pipelines
- [ ] Configure test database for CI
- [ ] Set up test data seeding strategy
- [ ] Add artifact collection for test reports

### Phase 4 — Expand Coverage (Ongoing)
- [ ] Add tests for new features as part of TDD workflow
- [ ] Visual regression testing with `toHaveScreenshot()`
- [ ] Multi-browser testing (Firefox, mobile viewports)
- [ ] Role-based access testing (admin vs user vs auditor)

---

## CLAUDE.md Updates

Once Playwright is set up, update the CLAUDE.md to:

1. Replace the Playwright MCP section with actual Playwright test framework documentation
2. Clarify that the MCP tool is for interactive testing, while `e2e/` tests are for automated regression
3. Add CSS-only changes to the TDD exceptions list (since JSDOM can't test layout)
4. Add E2E test patterns and conventions

### Suggested TDD Exception Addition

```markdown
#### Legitimate TDD Exceptions

...

4. **Pure CSS/Style Changes**: Layout and visual-only changes
   - JSDOM does not compute CSS layout (overflow, flex, height)
   - Should be verified via Playwright E2E or manual testing
   - Must have E2E tests if Playwright infrastructure is available
```

---

## Decision Points

Before starting implementation, the team should decide:

1. **Test database strategy**: Use MongoDB Memory Server (isolated but slower) or a shared test database (faster but needs cleanup)?
2. **Test data seeding**: Create seed scripts or rely on existing dev data?
3. **CI gating**: Should E2E failures block PRs, or run as advisory only initially?
4. **Browser coverage**: Start with Chromium only, or include Firefox/Safari from day one?
5. **External service mocking**: Mock at the network level (Playwright route interception) or use test endpoints?

**Recommendation**: Start minimal — Chromium only, shared test DB, advisory CI, network-level mocking — and expand as confidence grows.
