---
title: 【Go】Webサイトのセキュリティ簡易診断CLIツールを自作する
tags:
  - Go
  - Security
  - Web
  - SSL
  - CLI
private: false
updated_at: '2026-07-03T23:01:32+09:00'
id: fe240acb32715de43370
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

Webサイトを運営していると、セキュリティ面で見落としがちな問題があります。

- SSL証明書の期限切れ
- セキュリティヘッダーの未設定
- CMSのバージョン露出

これらは**公開情報のみ**で確認でき、侵入テストのような許可は不要です。本記事では、Go言語でこれらをチェックする簡易診断ツールを作ります。

## 完成イメージ

```bash
$ biz-tools scan https://example.com

=== セキュリティ診断結果 ===
対象URL: https://example.com
総合リスク: Medium (スコア: 4)

--- SSL/TLS ---
  有効: はい
  有効期限: 2026-10-25 (残り114日)
  プロトコル: TLS 1.3

--- 検出された問題 ---
1. [Medium] HSTSヘッダーなし
   → HSTSヘッダーを追加してHTTPS接続を強制してください

2. [Medium] X-Frame-Optionsなし
   → X-Frame-Options: SAMEORIGIN を設定してください
```

## チェック項目

| 項目 | 説明 |
|------|------|
| SSL証明書 | 有効性、残り日数、TLSバージョン |
| HTTPヘッダー | HSTS, X-Frame-Options, CSP等 |
| CMS検出 | WordPress等のバージョン露出 |
| サーバー情報 | Server/X-Powered-By 露出 |
| DNS | SPF/DMARC レコード |

## 実装

### 1. SSL証明書チェック

```go
func checkSSL(url string) SSLResult {
    result := SSLResult{Risk: "Info"}
    
    if strings.HasPrefix(url, "http://") {
        result.Enabled = false
        result.Risk = "Critical"
        return result
    }
    
    host := getHost(url)
    conn, err := tls.Dial("tcp", host+":443", &tls.Config{})
    if err != nil {
        result.Enabled = false
        result.Risk = "Critical"
        return result
    }
    defer conn.Close()
    
    result.Enabled = true
    certs := conn.ConnectionState().PeerCertificates
    if len(certs) > 0 {
        cert := certs[0]
        result.ValidUntil = cert.NotAfter
        result.DaysRemaining = int(time.Until(cert.NotAfter).Hours() / 24)
        result.Issuer = cert.Issuer.CommonName
        
        // TLSバージョン
        switch conn.ConnectionState().Version {
        case tls.VersionTLS13:
            result.Protocol = "TLS 1.3"
        case tls.VersionTLS12:
            result.Protocol = "TLS 1.2"
        }
        
        // 残り日数でリスク判定
        if result.DaysRemaining < 14 {
            result.Risk = "High"
        } else if result.DaysRemaining < 30 {
            result.Risk = "Medium"
        }
    }
    
    return result
}
```

### 2. HTTPセキュリティヘッダーチェック

```go
func checkHeaders(url string) HeadersResult {
    headers := HeadersResult{Risk: "Info"}
    
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Get(url)
    if err != nil {
        return headers
    }
    defer resp.Body.Close()
    
    // HSTS
    if resp.Header.Get("Strict-Transport-Security") == "" {
        headers.MissingHeaders = append(headers.MissingHeaders, "HSTS")
    }
    
    // X-Frame-Options
    if resp.Header.Get("X-Frame-Options") == "" {
        headers.MissingHeaders = append(headers.MissingHeaders, "X-Frame-Options")
    }
    
    // CSP
    if resp.Header.Get("Content-Security-Policy") == "" {
        headers.MissingHeaders = append(headers.MissingHeaders, "CSP")
    }
    
    // リスク判定
    if len(headers.MissingHeaders) >= 2 {
        headers.Risk = "Medium"
    }
    
    return headers
}
```

### 3. CMS検出

```go
func detectCMS(url string) CMSResult {
    result := CMSResult{Risk: "Info"}
    
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Get(url)
    if err != nil {
        return result
    }
    defer resp.Body.Close()
    
    body, _ := io.ReadAll(io.LimitReader(resp.Body, 100*1024))
    bodyStr := string(body)
    
    // WordPress検出
    if strings.Contains(bodyStr, "wp-content") {
        result.Detected = true
        result.Name = "WordPress"
        
        // バージョン露出チェック
        re := regexp.MustCompile(`generator.*WordPress\s*([\d.]+)`)
        if m := re.FindStringSubmatch(bodyStr); len(m) > 1 {
            result.Version = m[1]
            result.VersionExposed = true
            result.Risk = "Medium"
        }
    }
    
    return result
}
```

### 4. DNS（SPF/DMARC）チェック

```go
func checkDNS(url string) DNSResult {
    result := DNSResult{Risk: "Info"}
    domain := extractDomain(url)
    
    // MXレコード
    mx, _ := net.LookupMX(domain)
    result.HasMX = len(mx) > 0
    
    // SPFレコード
    txt, _ := net.LookupTXT(domain)
    for _, t := range txt {
        if strings.HasPrefix(t, "v=spf1") {
            result.HasSPF = true
        }
    }
    
    // DMARCレコード
    dmarc, _ := net.LookupTXT("_dmarc." + domain)
    for _, d := range dmarc {
        if strings.HasPrefix(d, "v=DMARC1") {
            result.HasDMARC = true
        }
    }
    
    // MXがあるのにSPF/DMARCがない場合はリスク
    if result.HasMX && !result.HasSPF {
        result.Risk = "High"
    }
    
    return result
}
```

## 出力フォーマット

4種類の出力形式に対応しています。

```bash
# テキスト（デフォルト）
biz-tools scan https://example.com

# Markdown
biz-tools scan https://example.com -o markdown -f report.md

# JSON
biz-tools scan https://example.com -o json

# HTML（CSS付きレポート）
biz-tools scan https://example.com -o html -f report.html
```

## 使用ライブラリ

- **cobra**: CLIフレームワーク
- 標準ライブラリ: `crypto/tls`, `net/http`, `net`, `encoding/json`

外部依存を最小限に抑えています。

## 注意事項

- このツールは**公開情報のみ**を使用した簡易診断です
- 他者のサイトをスキャンする場合は**許可を取得**してください
- 本格的な脆弱性診断には専門ツールを使用してください

## まとめ

Go標準ライブラリだけで、実用的なセキュリティ簡易診断ツールを作れます。

- SSL証明書の期限監視
- セキュリティヘッダーの設定確認
- 自社サイトの定期チェック

などに活用できます。

**GitHub**: https://github.com/geeknow112/biz-tools

## 参考

- [MDN Web Security](https://developer.mozilla.org/ja/docs/Web/Security)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
