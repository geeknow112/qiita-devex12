---
title: 【コピペOK】k6で負荷テスト - すぐ使えるスクリプト集
tags:
  - JavaScript
  - 負荷テスト
  - performance
  - コピペ
  - k6
private: false
updated_at: '2026-06-09T22:48:13+09:00'
id: 9db39b938712182ed613
organization_url_name: null
slide: false
ignorePublish: false
---

## この記事の使い方

k6のスクリプトをシーン別にまとめました。コピペして`k6 run script.js`で実行できます。

## インストール

```bash
# Mac
brew install k6

# Windows
choco install k6

# Docker（インストール不要）
docker run --rm -i grafana/k6 run - <script.js
```

## 基本テンプレート

### とりあえず動かしたい

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 10,         // 同時ユーザー数
  duration: '30s', // 実行時間
};

export default function () {
  const res = http.get('https://your-api.example.com/health');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
  
  sleep(1);
}
```

**実行:**
```bash
k6 run script.js
```

## シーン別スクリプト


### 1. スモークテスト（デプロイ後の動作確認）

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 1,
  duration: '1m',
  thresholds: {
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://your-api.example.com/health');
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

**用途:** デプロイ直後に「とりあえず動くか」を確認

---

### 2. 負荷テスト（本番想定）

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // 2分で50ユーザーまで増加
    { duration: '5m', target: 50 },   // 5分間維持
    { duration: '2m', target: 0 },    // 2分で終了
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95%が500ms以内
    http_req_failed: ['rate<0.01'],   // エラー率1%未満
  },
};

export default function () {
  http.get('https://your-api.example.com/api/users');
  sleep(1);
}
```

**用途:** 本番想定の負荷で耐えられるか確認

---

### 3. ストレステスト（限界を探る）

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '3m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '3m', target: 200 },
    { duration: '2m', target: 300 },
    { duration: '3m', target: 300 },
    { duration: '2m', target: 0 },
  ],
};

export default function () {
  http.get('https://your-api.example.com/api/users');
  sleep(1);
}
```

**用途:** どこで限界が来るか確認

---

### 4. 認証付きAPIテスト

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

const BASE_URL = 'https://your-api.example.com';

export const options = {
  vus: 20,
  duration: '5m',
};

export function setup() {
  // ログインしてトークン取得
  const res = http.post(`${BASE_URL}/auth/login`, JSON.stringify({
    email: 'test@example.com',
    password: 'password123',
  }), {
    headers: { 'Content-Type': 'application/json' },
  });
  return { token: res.json('access_token') };
}

export default function (data) {
  const res = http.get(`${BASE_URL}/api/users`, {
    headers: { 'Authorization': `Bearer ${data.token}` },
  });
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

**用途:** JWT認証が必要なAPI

---

### 5. ECサイト風シナリオ

```javascript
import http from 'k6/http';
import { check, sleep, group } from 'k6';

const BASE_URL = 'https://your-site.example.com';

export const options = {
  vus: 30,
  duration: '10m',
};

export default function () {
  group('トップページ', () => {
    http.get(`${BASE_URL}/`);
    sleep(2);
  });

  group('商品検索', () => {
    http.get(`${BASE_URL}/search?q=laptop`);
    sleep(1);
  });

  group('商品詳細', () => {
    http.get(`${BASE_URL}/products/123`);
    sleep(3);
  });

  group('カート追加', () => {
    http.post(`${BASE_URL}/cart`, JSON.stringify({ productId: 123 }), {
      headers: { 'Content-Type': 'application/json' },
    });
    sleep(1);
  });
}
```

**用途:** ユーザーの行動をシミュレート

---

## 結果の出力

```bash
# JSON形式で出力
k6 run --out json=results.json script.js

# CSV形式で出力
k6 run --out csv=results.csv script.js
```

## よく使うオプション

```bash
# ユーザー数と時間をコマンドラインで指定
k6 run --vus 100 --duration 5m script.js

# 結果をJSON出力
k6 run --out json=result.json script.js

# 環境変数を渡す
k6 run -e BASE_URL=https://staging.example.com script.js
```

## GitHub Actionsで自動実行

```yaml
name: Load Test

on:
  push:
    branches: [main]

jobs:
  k6:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run k6 test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: load-test.js
```

## 参考

- [k6公式](https://k6.io/docs/)
- [k6 GitHub](https://github.com/grafana/k6)
