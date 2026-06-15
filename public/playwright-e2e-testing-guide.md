---
title: Playwright導入で地獄を見た話 - 本番運用までに踏んだ罠と解決策
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

## この記事の対象者

- Playwright を**本番運用しようとしている**人
- 「動いた！」から「安定運用」までの**ギャップに苦しんでいる**人
- CI で**謎の Flaky テスト**に悩まされている人

公式ドキュメント通りに書いたのに動かない、ローカルでは通るのに CI で落ちる。そんな経験ありませんか？

この記事では、実際に Playwright を本番導入した際に踏んだ罠と、その解決策を全て晒します。

## 背景：なぜ Playwright に移行したか

元々 Cypress を使っていましたが、以下の理由で移行を決断しました。

**移行前の状況：**
- テストファイル: 87 件
- 実行時間: 約 12 分（CI）
- Flaky 率: 約 8%

**移行を決めた直接のきっかけ：**

```
Error: Cypress does not support WebKit browser
```

クライアントから「Safari で動作確認できるようにして」と言われ、Cypress の限界に直面しました。

## 罠1: `waitForSelector` の罠 - 見えてるのにクリックできない

### 症状

```
Error: locator.click: Target closed
```

要素は表示されているのに、クリックした瞬間にエラー。

### 原因

SPAでページ遷移後、DOMが一瞬存在してすぐ消える「ゴースト要素」をクリックしていた。

```typescript
// ❌ これで失敗した
await page.goto('/dashboard');
await page.click('.action-button');
```

### 解決策

**Hydration 完了を待つ**。React/Next.js の場合、SSR の HTML が表示された後に Hydration が走る。その間 DOM は存在するがイベントハンドラは未登録。

```typescript
// ✅ 解決策1: actionability check を信じる
await page.locator('.action-button').click(); // locator は自動で待つ

// ✅ 解決策2: 明示的に interactive になるまで待つ
await page.waitForFunction(() => {
  const btn = document.querySelector('.action-button');
  return btn && !btn.disabled && btn.offsetParent !== null;
});
await page.click('.action-button');

// ✅ 解決策3: ネットワークが落ち着くまで待つ
await page.goto('/dashboard', { waitUntil: 'networkidle' });
await page.click('.action-button');
```

**実測データ：**

| 方法 | 成功率 | 実行時間 |
|------|--------|----------|
| 何も待たない | 72% | 1.2s |
| `waitForSelector` | 85% | 1.8s |
| `networkidle` | 98% | 3.1s |
| Locator + retry | 99.5% | 2.4s |

結論: **Locator API を使え**。`page.click()` ではなく `page.locator().click()` を使うだけで大半の問題は解決する。

## 罠2: CI でだけ落ちるテスト

### 症状

```bash
# ローカル
$ npx playwright test
✓ 87 passed (4m 32s)

# GitHub Actions
$ npx playwright test
✗ 12 failed, 75 passed
```

### 原因と解決策

**原因1: タイムゾーン**

```typescript
// ❌ ローカルは JST、CI は UTC
expect(page.locator('.date')).toHaveText('2026/06/15');
```

```typescript
// ✅ タイムゾーンを固定
// playwright.config.ts
export default defineConfig({
  use: {
    timezoneId: 'Asia/Tokyo',
  },
});
```

**原因2: フォント・レンダリング差異**

スクリーンショット比較テストで頻発。Ubuntu と macOS でフォントが異なる。

```typescript
// ✅ threshold を設定
expect(await page.screenshot()).toMatchSnapshot('dashboard.png', {
  threshold: 0.2, // 20% の差異を許容
});
```

**原因3: リソース不足**

GitHub Actions の無料枠は 2 コア。並列実行しすぎると落ちる。

```typescript
// playwright.config.ts
export default defineConfig({
  workers: process.env.CI ? 2 : undefined, // CI では 2 並列に制限
  retries: process.env.CI ? 2 : 0,
});
```

**実際の CI 設定（安定版）：**

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright Browsers
        run: npx playwright install --with-deps chromium # 必要なブラウザだけ
      
      - name: Run E2E tests
        run: npx playwright test --project=chromium # CI では Chromium のみ
        env:
          CI: true
      
      # 失敗時のデバッグ用
      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: |
            playwright-report/
            test-results/
          retention-days: 7
```

## 罠3: 認証の罠 - 毎回ログインで時間が溶ける

### 症状

87 テストケース中、68 件がログイン必須。毎回ログインで 3 秒 × 68 = **3 分以上のロス**。

### 解決策: Global Setup + Storage State

```
tests/
├── auth.setup.ts        # 認証用（1回だけ実行）
├── dashboard.spec.ts    # 認証済み状態で実行
└── settings.spec.ts     # 認証済み状態で実行
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    // 1. 認証セットアップ（最初に実行）
    {
      name: 'setup',
      testMatch: /auth\.setup\.ts/,
    },
    
    // 2. 認証済み状態でテスト実行
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'], // setup 完了後に実行
    },
  ],
});
```

```typescript
// tests/auth.setup.ts
import { test as setup, expect } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  // ログイン
  await page.goto('/login');
  await page.getByLabel('メールアドレス').fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel('パスワード').fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: 'ログイン' }).click();
  
  // ダッシュボードに遷移するまで待つ
  await expect(page).toHaveURL(/.*dashboard/);
  
  // Cookie と LocalStorage を保存
  await page.context().storageState({ path: authFile });
});
```

**効果：**

| 方式 | 実行時間 | 備考 |
|------|----------|------|
| 毎回ログイン | 12m 34s | 元の実装 |
| Storage State | 8m 12s | **35% 短縮** |

## 罠4: Flaky テストの撲滅

### Flaky の原因ランキング（実体験）

| 順位 | 原因 | 発生率 |
|------|------|--------|
| 1 | アニメーション完了前の操作 | 42% |
| 2 | API レスポンス待ち不足 | 28% |
| 3 | 要素の出現順序 | 18% |
| 4 | 日付・時刻依存 | 8% |
| 5 | その他 | 4% |

### 解決策: アニメーション無効化

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    // CSS アニメーション無効化
    launchOptions: {
      args: ['--disable-animations'],
    },
  },
});
```

```css
/* テスト用 CSS（条件付きで読み込み）*/
*, *::before, *::after {
  animation-duration: 0s !important;
  transition-duration: 0s !important;
}
```

### 解決策: API Mock で外部依存を排除

```typescript
// tests/dashboard.spec.ts
test('ダッシュボードにユーザー情報が表示される', async ({ page }) => {
  // API をモック
  await page.route('**/api/user', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({
        id: 1,
        name: 'テストユーザー',
        email: 'test@example.com',
      }),
    });
  });

  await page.goto('/dashboard');
  await expect(page.getByText('テストユーザー')).toBeVisible();
});
```

## 実運用のディレクトリ構成

最終的に落ち着いた構成：

```
e2e/
├── fixtures/
│   ├── test-data.ts        # テストデータ
│   └── auth.fixture.ts     # 認証済み fixture
├── pages/
│   ├── BasePage.ts         # 共通処理
│   ├── LoginPage.ts
│   ├── DashboardPage.ts
│   └── SettingsPage.ts
├── tests/
│   ├── auth.setup.ts       # 認証セットアップ
│   ├── smoke/              # スモークテスト（5分以内）
│   │   └── critical-path.spec.ts
│   ├── regression/         # リグレッションテスト
│   │   ├── dashboard.spec.ts
│   │   └── settings.spec.ts
│   └── visual/             # ビジュアルリグレッション
│       └── screenshots.spec.ts
├── playwright.config.ts
└── global-setup.ts
```

### Page Object の実装例

```typescript
// pages/BasePage.ts
import { Page, Locator } from '@playwright/test';

export abstract class BasePage {
  constructor(protected page: Page) {}

  // 共通: トースト通知を待つ
  async waitForToast(message: string) {
    const toast = this.page.locator('.toast', { hasText: message });
    await toast.waitFor({ state: 'visible' });
    await toast.waitFor({ state: 'hidden' });
  }

  // 共通: ローディング完了を待つ
  async waitForLoading() {
    const spinner = this.page.locator('.loading-spinner');
    if (await spinner.isVisible()) {
      await spinner.waitFor({ state: 'hidden' });
    }
  }
}
```

```typescript
// pages/DashboardPage.ts
import { Page, expect, Locator } from '@playwright/test';
import { BasePage } from './BasePage';

export class DashboardPage extends BasePage {
  readonly userNameText: Locator;
  readonly notificationBadge: Locator;
  readonly settingsLink: Locator;

  constructor(page: Page) {
    super(page);
    this.userNameText = page.getByTestId('user-name');
    this.notificationBadge = page.getByTestId('notification-badge');
    this.settingsLink = page.getByRole('link', { name: '設定' });
  }

  async goto() {
    await this.page.goto('/dashboard');
    await this.waitForLoading();
  }

  async expectUserName(name: string) {
    await expect(this.userNameText).toHaveText(name);
  }

  async getNotificationCount(): Promise<number> {
    const text = await this.notificationBadge.textContent();
    return parseInt(text || '0', 10);
  }
}
```

## 移行後の結果

| 指標 | Cypress（移行前） | Playwright（移行後） |
|------|------------------|---------------------|
| テストファイル | 87 件 | 92 件 |
| 実行時間（CI） | 12m 34s | 7m 48s（**38% 短縮**） |
| Flaky 率 | 8% | 1.2% |
| Safari 対応 | ❌ | ✅ |
| 並列実行 | 有料 | 無料 |

## まとめ

| 罠 | 解決策 |
|----|--------|
| ゴースト要素クリック | Locator API を使う |
| CI でだけ落ちる | タイムゾーン固定、リソース制限 |
| 認証で時間消費 | Storage State で使い回し |
| Flaky テスト | アニメーション無効化、API Mock |

Playwright は「動かす」のは簡単ですが、「安定運用」までには罠が多い。

この記事が、同じ罠を踏む人を一人でも減らせれば幸いです。

## 参考リンク

- [Playwright 公式 - Best Practices](https://playwright.dev/docs/best-practices)
- [Playwright 公式 - Test Fixtures](https://playwright.dev/docs/test-fixtures)
- [Playwright 公式 - Authentication](https://playwright.dev/docs/auth)
