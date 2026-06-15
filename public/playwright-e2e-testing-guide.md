---
title: 【2026年版】Playwright E2Eテスト入門 - Cypressから乗り換えた理由と実践ガイド
tags:
  - JavaScript
  - TypeScript
  - テスト
  - E2E
  - Playwright
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## この記事で得られること

- Playwright の**特徴と他ツールとの比較**
- プロジェクトへの**導入手順**
- 実践的な**テストコードの書き方**
- CI/CD への組み込み方法
- 実際にハマった**トラブルシューティング**

## なぜ Playwright なのか

E2E テストツールは選択肢が多いですが、2026年現在 Playwright を選ぶ理由は明確です。

### 主要ツール比較

| 項目 | Playwright | Cypress | Selenium |
|------|------------|---------|----------|
| ブラウザ対応 | Chromium, Firefox, WebKit | Chromium系のみ | 全ブラウザ |
| 実行速度 | ◎ 高速 | ○ 普通 | △ 遅い |
| 並列実行 | ◎ 標準対応 | △ 有料 | ○ 設定必要 |
| モバイル | ◎ エミュレーション | △ 限定的 | ○ Appium連携 |
| 学習コスト | ○ 中程度 | ◎ 低い | △ 高い |

### Cypress から乗り換えた理由

1. **Safari テストが必要になった** - Cypress は WebKit 非対応
2. **並列実行が無料** - Cypress Dashboard は有料
3. **API テストも統合できる** - request context が便利
4. **トレース機能が強力** - デバッグが圧倒的に楽

## 環境構築

### インストール

```bash
# 新規プロジェクト
npm init playwright@latest

# 既存プロジェクトに追加
npm install -D @playwright/test
npx playwright install
```

### 生成されるファイル構成

```
project/
├── tests/
│   └── example.spec.ts    # サンプルテスト
├── playwright.config.ts    # 設定ファイル
├── package.json
└── .github/
    └── workflows/
        └── playwright.yml  # CI設定
```

## 基本的なテストの書き方

### 最初のテスト

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('ログイン機能', () => {
  test('正常にログインできる', async ({ page }) => {
    await page.goto('https://example.com/login');
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');

    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.locator('h1')).toContainText('ダッシュボード');
  });

  test('パスワードが間違っているとエラー表示', async ({ page }) => {
    await page.goto('https://example.com/login');
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'wrong-password');
    await page.click('button[type="submit"]');

    await expect(page.locator('.error-message')).toBeVisible();
  });
});
```

### Locator の選び方

```typescript
// ✅ 推奨: role, label, placeholder
await page.getByRole('button', { name: 'ログイン' });
await page.getByLabel('メールアドレス');
await page.getByPlaceholder('example@mail.com');

// ○ 許容: data-testid
await page.getByTestId('login-button');

// △ 非推奨: CSS セレクタ（変更に弱い）
await page.locator('.btn-primary');
```

### Page Object Model

```typescript
// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('メールアドレス');
    this.passwordInput = page.getByLabel('パスワード');
    this.submitButton = page.getByRole('button', { name: 'ログイン' });
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

## 設定ファイル

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  
  reporter: [
    ['html', { open: 'never' }],
    ['json', { outputFile: 'test-results.json' }],
  ],
  
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

## 実行コマンド

```bash
# 全テスト実行
npx playwright test

# UI モード（デバッグに便利）
npx playwright test --ui

# headed モード（ブラウザ表示）
npx playwright test --headed

# レポート表示
npx playwright show-report
```

## ハマりポイント

### 1. 要素が見つからない

```typescript
// ❌ NG: 要素が表示される前にクリック
await page.click('.dynamic-button');

// ✅ OK: expect で待機を兼ねる
await expect(page.locator('.dynamic-button')).toBeVisible();
await page.locator('.dynamic-button').click();
```

### 2. 認証状態の保持

```typescript
// tests/auth.setup.ts
import { test as setup } from '@playwright/test';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.fill('input[name="email"]', 'test@example.com');
  await page.fill('input[name="password"]', 'password123');
  await page.click('button[type="submit"]');
  
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

## GitHub Actions 連携

```yaml
name: Playwright Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
```

## まとめ

| ポイント | 内容 |
|----------|------|
| ツール選定 | Safari対応、並列実行無料で Playwright |
| Locator | role, label 優先 |
| 設計 | Page Object Model で保守性向上 |
| デバッグ | `--ui` モードとトレース活用 |

## 参考リンク

- [Playwright 公式ドキュメント](https://playwright.dev/)
- [Playwright GitHub](https://github.com/microsoft/playwright)
