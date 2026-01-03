# link.profilecode.codes 実装計画

現在の状況と実装計画をまとめたドキュメントです。

---

## 📊 現在の状況

### 確認された事実

1. **アプリ側の設定**
   - アプリは `link.profilecode.codes/join` を処理するように設定されている
   - テストアプリで `/join` 処理が動作していることを確認済み

2. **ウェブ側の現状**
   - `profilecode.codes/join` → メインドメインで処理されている（動作中）
   - `link.profilecode.codes/join` → **未実装**（処理できていない）

3. **問題点**
   - アプリ側は `link.profilecode.codes/join` を期待している
   - 実際には `profilecode.codes/join` で処理されている
   - `link.profilecode.codes` が未実装のため、アプリとウェブで不一致

---

## 🎯 実装目標

### 最終的な構成

```
profilecode.codes/
├── /                          # ランディングページ（メイン）
├── /join                       # link.profilecode.codes/join にリダイレクト
├── /test                       # ウェブ診断の入り口
├── /privacy-policy.html        # プライバシーポリシー（既存）
└── /eula.html                  # 利用規約（既存）

link.profilecode.codes/
└── /join?token=...             # ディープリンク処理専用
```

---

## 📋 実装ステップ

### ステップ1: `link.profilecode.codes` の実装（最優先）

**目的**: アプリ側が期待する `link.profilecode.codes/join` を実装する

**必要な作業**:
1. `link.profilecode.codes` 用のリポジトリ/プロジェクトを作成
2. `/join` 処理を実装
3. `apple-app-site-association` ファイルを配置
4. `assetlinks.json` ファイルを配置（オプション）

**実装内容**:
- `index.html` で `/join` パスを処理
- `token` パラメータを取得
- Android/iOSそれぞれでアプリ起動処理
- アプリ未インストール時のフォールバック処理

**必須修正項目**:
1. パス判定: `path.startsWith("/join")` を使用
2. カスタムスキーム: `com.profilecode.profilecode://` を使用
3. `apple-app-site-association`: 正しい `paths` を設定

---

### ステップ2: `profilecode.codes/join` のリダイレクト実装

**目的**: 既存の `/join` アクセスを `link.profilecode.codes/join` にリダイレクト

**必要な作業**:
1. `profilecode.codes` 側で `join.html` を作成
2. `index.html` から `/join` 処理を削除
3. `index.html` をランディングページに変更

**実装内容**:

#### `join.html` の作成

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Redirecting...</title>
    <script>
        // /joinへのアクセスをlink.profilecode.codesにリダイレクト
        window.onload = function() {
            const currentUrl = new URL(window.location.href);
            const queryString = currentUrl.search;
            const hash = currentUrl.hash;
            
            // link.profilecode.codes/joinにリダイレクト（クエリパラメータとハッシュを保持）
            window.location.href = `https://link.profilecode.codes/join${queryString}${hash}`;
        };
    </script>
</head>
<body>
    <p>アプリを開いています...</p>
</body>
</html>
```

#### `_config.yml` の設定

```yaml
redirect_from:
  - /join
  - /join/
```

または、`join.html` を配置して自動的に `/join` でアクセス可能にする。

---

### ステップ3: `index.html` のランディングページ化

**目的**: メインドメインをランディングページに変更

**必要な作業**:
1. `index.html` から `/join` 処理を削除
2. ランディングページのコンテンツを追加
3. アプリダウンロードリンクを追加
4. ウェブ診断への導線を追加

**削除するコード**:
```javascript
// 削除対象
if (path === "/join" && token) {
    // Sharelinkの場合
    window.location.href = `intent://join?token=${token}...`;
}
```

---

## 🔧 `link.profilecode.codes` の実装詳細

### 必須実装項目

#### 1. `index.html` の実装

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Opening Profilecode...</title>
    <script>
        (function() {
            const path = window.location.pathname;
            const urlParams = new URLSearchParams(window.location.search);
            const token = urlParams.get("token");
            
            // apple-app-site-associationファイルへのアクセスは特別処理をスキップ
            if (path === '/.well-known/apple-app-site-association') {
                return;
            }
            
            // /joinで始まるパスを処理
            if (path.startsWith("/join") && token) {
                const isAndroid = /Android/i.test(navigator.userAgent);
                const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);
                
                const playStoreLink = 'https://play.google.com/store/apps/details?id=com.profilecode.profilecode';
                const appStoreLink = 'https://apps.apple.com/app/YOUR_APP_ID';
                
                if (isAndroid) {
                    // Android Intentスキーム
                    window.location.href = `intent://join?token=${token}#Intent;scheme=https;package=com.profilecode.profilecode;S.browser_fallback_url=${encodeURIComponent(playStoreLink)};end`;
                } else if (isIOS) {
                    // iOS Universal Links → カスタムスキーム → App Store
                    const universalLink = `https://link.profilecode.codes/join?token=${token}`;
                    const customScheme = `com.profilecode.profilecode://join?token=${token}`;
                    
                    let appLaunched = false;
                    let customSchemeTimeout;
                    let appStoreTimeout;
                    
                    // アプリが起動したことを検知する関数
                    function detectAppLaunch() {
                        appLaunched = true;
                        // タイマーをクリア
                        if (customSchemeTimeout) {
                            clearTimeout(customSchemeTimeout);
                        }
                        if (appStoreTimeout) {
                            clearTimeout(appStoreTimeout);
                        }
                    }
                    
                    // ページが非表示になったらアプリが起動したと判断
                    document.addEventListener('visibilitychange', function() {
                        if (document.hidden) {
                            detectAppLaunch();
                        }
                    });
                    
                    // ページがアンロードされる前にアプリが起動したと判断
                    window.addEventListener('pagehide', function() {
                        detectAppLaunch();
                    });
                    
                    // ウィンドウがフォーカスを失ったらアプリが起動したと判断
                    window.addEventListener('blur', function() {
                        detectAppLaunch();
                    });
                    
                    // まずUniversal Linksを試す
                    window.location.href = universalLink;
                    
                    // 3秒後にアプリが起動していなければカスタムURLスキームを試す
                    customSchemeTimeout = setTimeout(function() {
                        if (!appLaunched) {
                            window.location.href = customScheme;
                            
                            // さらに3秒後にアプリが起動していなければApp Storeに遷移
                            appStoreTimeout = setTimeout(function() {
                                if (!appLaunched) {
                                    window.location.href = appStoreLink;
                                }
                            }, 3000);
                        }
                    }, 3000);
                } else {
                    // その他のデバイス
                    window.location.href = playStoreLink;
                }
            }
        })();
    </script>
</head>
<body>
    <p>アプリを開いています...</p>
</body>
</html>
```

#### 2. `.well-known/apple-app-site-association` の配置

```json
{
    "applinks": {
        "apps": [],
        "details": [
            {
                "appID": "J4SJR8W4AS.com.profilecode.profilecode",
                "paths": [
                    "/join*",
                    "/auth/callback*",
                    "/reset-password*",
                    "/login-callback*"
                ]
            }
        ]
    }
}
```

#### 3. `_config.yml` の設定（GitHub Pages使用時）

```yaml
include:
  - .well-known
```

---

## 📋 実装チェックリスト

### `link.profilecode.codes` の実装

- [ ] リポジトリ/プロジェクトを作成
- [ ] `index.html` を実装（パス判定: `path.startsWith("/join")`）
- [ ] カスタムスキーム: `com.profilecode.profilecode://` を使用
- [ ] `.well-known/apple-app-site-association` を配置
- [ ] `_config.yml` で `.well-known` をインクルード
- [ ] デプロイして動作確認

### `profilecode.codes` の修正

- [ ] `join.html` を作成（リダイレクト処理）
- [ ] `index.html` から `/join` 処理を削除
- [ ] `index.html` をランディングページに変更
- [ ] `_config.yml` で `redirect_from: /join` を設定（オプション）
- [ ] デプロイして動作確認

### 動作確認

- [ ] `link.profilecode.codes/join?token=test123` でアプリが起動する
- [ ] `profilecode.codes/join?token=test123` が `link.profilecode.codes/join?token=test123` にリダイレクトされる
- [ ] `profilecode.codes/` がランディングページとして表示される
- [ ] iOS実機テストで動作確認
- [ ] Android実機テストで動作確認

---

## 🎯 実装優先順位

### フェーズ1: 緊急対応（即座に実装）

1. **`link.profilecode.codes` の実装**
   - アプリ側が期待するURLを実装
   - 所要時間: 約1時間

### フェーズ2: 移行対応（早期に実装）

2. **`profilecode.codes/join` のリダイレクト**
   - 既存のアクセスを新しいURLにリダイレクト
   - 所要時間: 約30分

3. **`profilecode.codes` のランディングページ化**
   - メインドメインを整理
   - 所要時間: 約1時間

---

## ⚠️ 注意事項

### 実装時の注意点

1. **パス判定**: `path.startsWith("/join")` を使用（厳密一致ではない）
2. **カスタムスキーム**: `com.profilecode.profilecode://` を使用（`profilecode://` ではない）
3. **apple-app-site-association**: 正しい `paths` を設定（`["*"]` ではない）

### デプロイ後の確認

1. `link.profilecode.codes/join?token=test123` でアプリが起動することを確認
2. `profilecode.codes/join?token=test123` が正しくリダイレクトされることを確認
3. 実機テストで動作確認

---

## 🔗 参考情報

- アプリ側の実装: `lib/deep_link/deep_link_handler.dart`
- iOS設定: `ios/Runner/Info.plist`, `apple-app-site-association`
- Android設定: `android/app/src/main/AndroidManifest.xml`
- 修正要望: `docs/link_profilecode_fix_requirements_final.md`
- 確認チェックリスト: `docs/link_profilecode_verification_checklist.md`

---

**最終更新**: 2024年
**作成者**: profilecode開発チーム

