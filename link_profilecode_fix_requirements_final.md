# link.profilecode.codes 実装・修正要望

質問リストの回答と現在の状況に基づく、具体的な実装・修正要望です。

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

## 🚨 必須実装・修正項目（即座に対応が必要）

### 実装1: `link.profilecode.codes` の実装（最優先）

**現状**:
- `link.profilecode.codes` が未実装
- アプリ側は `link.profilecode.codes/join` を期待している

**必要な作業**:
1. `link.profilecode.codes` 用のリポジトリ/プロジェクトを作成
2. `/join` 処理を実装
3. `apple-app-site-association` ファイルを配置
4. デプロイして動作確認

**実装内容**: 下記の「実装詳細」セクションを参照

**優先度**: 🔴 **最高** - アプリ側が期待するURLを実装する必要がある

---

### 修正1: パス判定の厳密一致を修正

**問題**:
- 現在の実装: `path === "/join"`（厳密一致のみ）
- アプリ側の要件: `path.startsWith("/join")`（サブパスも対応）

**影響**:
- ❌ `/join/anything` のようなサブパスが処理されない
- ❌ アプリ側の実装と不一致

**修正箇所**: `link.profilecode.codes` の `index.html` で実装時

**実装時**:
```javascript
// path.startsWith("/join") を使用
if (path.startsWith("/join") && token) {
```

**優先度**: 🔴 **最高** - アプリ側の実装と一致させる必要がある

---

### 修正2: iOSのカスタムスキームを正しく実装

**問題**:
- `profilecode.codes` 側の実装: `profilecode://join?token=${token}`（誤り）
- アプリ側の要件: `com.profilecode.profilecode://join?token=TOKEN`

**影響**:
- ❌ アプリ側でカスタムスキームが処理されない
- ❌ iOSフォールバック処理が機能しない

**修正箇所**: `link.profilecode.codes` の `index.html` で実装時

**実装時**:
```javascript
const customScheme = `com.profilecode.profilecode://join?token=${token}`;
```

**アプリ側の設定確認**:
```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleURLSchemes</key>
<array>
    <string>com.profilecode.profilecode</string>
</array>
```

**優先度**: 🔴 **最高** - アプリが起動しない可能性がある

---

### 修正3: apple-app-site-associationのpaths設定を正しく実装

**問題**:
- `profilecode.codes` 側の実装: `["*"]`（全パスを許可）
- アプリ側の要件: `["/join*", "/auth/callback*", "/reset-password*", "/login-callback*"]`

**影響**:
- ⚠️ セキュリティ上の懸念
- ⚠️ 意図しないパスでもアプリが起動する可能性

**実装箇所**: `link.profilecode.codes` の `.well-known/apple-app-site-association`

**実装時**:
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

**優先度**: 🟡 **高** - セキュリティ上の懸念がある

---

### 実装2: `profilecode.codes/join` のリダイレクト実装

**目的**: 既存の `/join` アクセスを `link.profilecode.codes/join` にリダイレクト

**必要な作業**:
1. `profilecode.codes` 側で `join.html` を作成
2. `index.html` から `/join` 処理を削除
3. `index.html` をランディングページに変更

**実装内容**: 下記の「実装詳細」セクションを参照

**優先度**: 🟡 **高** - 既存のアクセスを新しいURLにリダイレクトする必要がある

---

## ⚠️ 要確認項目（修正後に対応）

### 確認1: apple-app-site-associationのHTTPアクセス確認

**確認項目**:
- HTTPステータス: `200 OK`
- Content-Type: `application/json`
- リダイレクトされていない（301/302ではない）

**確認コマンド**:
```bash
curl -I https://link.profilecode.codes/.well-known/apple-app-site-association
```

**期待される結果**:
```
HTTP/2 200
content-type: application/json
...
```

**対応**: GitHub Pagesを使用している場合、通常は自動的に設定されるが、確認が必要

---

### 確認2: HTTPS/SSL証明書の確認

**確認項目**:
- HTTPSでアクセス可能
- SSL証明書が有効

**確認コマンド**:
```bash
curl -I https://link.profilecode.codes/join
```

**期待される結果**:
```
HTTP/2 200
...
```

**対応**: GitHub Pagesを使用している場合、自動的にHTTPSが有効

---

### 確認3: HTTP→HTTPSリダイレクトの確認

**確認項目**:
- HTTPからHTTPSへのリダイレクトが動作する

**確認コマンド**:
```bash
curl -I http://link.profilecode.codes/join
```

**期待される結果**:
```
HTTP/1.1 301 Moved Permanently
Location: https://link.profilecode.codes/join
...
```

**対応**: GitHub Pagesを使用している場合、通常は自動的にリダイレクトされる

---

### 確認4: レスポンス時間の確認

**確認項目**:
- `/join` ページの読み込み時間が3秒以内

**確認コマンド**:
```bash
time curl -o /dev/null -s -w "%{time_total}\n" https://link.profilecode.codes/join
```

**期待される結果**: 1秒以内（静的HTMLファイルのため）

---

### 確認5: iOS実機テスト

**確認項目**:
1. Safariで `https://link.profilecode.codes/join?token=test123` を開く
2. アプリがインストール済み → アプリが起動するか確認
3. アプリ未インストール → フォールバックページが表示されるか確認

**期待される動作**:
- アプリインストール済み: アプリが起動し、ディープリンクが処理される
- アプリ未インストール: フォールバックページが表示される

---

### 確認6: Android実機テスト

**確認項目**:
1. Chromeで `https://link.profilecode.codes/join?token=test123` を開く
2. アプリがインストール済み → アプリが起動するか確認
3. アプリ未インストール → フォールバックページが表示されるか確認

**期待される動作**:
- アプリインストール済み: アプリが起動し、ディープリンクが処理される
- アプリ未インストール: フォールバックページが表示される

**代替確認方法**:
```bash
adb shell am start -a android.intent.action.VIEW \
  -d "https://link.profilecode.codes/join?token=test123"
```

---

### 確認7: メインドメインの設定確認

**確認項目**:
- `profilecode.codes/join` へのアクセス時の動作

**期待される動作**:
- `link.profilecode.codes/join` にリダイレクトされる
- または404を返す

**対応**: メインドメインのリポジトリまたはサーバー設定を確認する必要がある

---

## 📋 実装・修正チェックリスト

### 必須実装項目（`link.profilecode.codes`）

- [ ] **実装1**: `link.profilecode.codes` 用のリポジトリ/プロジェクトを作成
- [ ] **実装2**: `index.html` を実装（パス判定: `path.startsWith("/join")`）
- [ ] **実装3**: カスタムスキーム: `com.profilecode.profilecode://` を使用
- [ ] **実装4**: `.well-known/apple-app-site-association` を配置（正しい`paths`）
- [ ] **実装5**: `_config.yml` で `.well-known` をインクルード
- [ ] **実装6**: デプロイして動作確認

### 必須修正項目（`profilecode.codes`）

- [ ] **修正1**: `join.html` を作成（リダイレクト処理）
- [ ] **修正2**: `index.html` から `/join` 処理を削除
- [ ] **修正3**: `index.html` をランディングページに変更
- [ ] **修正4**: デプロイして動作確認

### 確認項目

- [ ] `apple-app-site-association`のHTTPアクセス確認
- [ ] HTTPS/SSL証明書の確認
- [ ] HTTP→HTTPSリダイレクトの確認
- [ ] レスポンス時間の確認
- [ ] iOS実機テスト
- [ ] Android実機テスト
- [ ] メインドメインの設定確認

---

## 🔧 実装手順

### ステップ1: `link.profilecode.codes` の実装（最優先、約1時間）

#### 1-1. リポジトリ/プロジェクトの作成

1. `link.profilecode.codes` 用のリポジトリ/プロジェクトを作成
2. GitHub Pagesまたは静的ホスティングサービスを設定
3. カスタムドメイン `link.profilecode.codes` を設定

#### 1-2. `index.html` の実装

`index.html` を作成し、以下のコードを実装：

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
            
            // /joinで始まるパスを処理（path.startsWith("/join")を使用）
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
                    // カスタムスキームは com.profilecode.profilecode:// を使用
                    const customScheme = `com.profilecode.profilecode://join?token=${token}`;
                    
                    // まずUniversal Linksを試す
                    window.location.href = universalLink;
                    
                    // 3秒後にアプリが起動していなければカスタムURLスキームを試す
                    setTimeout(function() {
                        window.location.href = customScheme;
                        
                        // さらに3秒後にアプリが起動していなければApp Storeに遷移
                        setTimeout(function() {
                            window.location.href = appStoreLink;
                        }, 3000);
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

**重要なポイント**:
- ✅ パス判定: `path.startsWith("/join")` を使用
- ✅ カスタムスキーム: `com.profilecode.profilecode://` を使用

#### 1-3. `.well-known/apple-app-site-association` の配置

`.well-known/apple-app-site-association` ファイルを作成：

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

#### 1-4. `_config.yml` の設定（GitHub Pages使用時）

```yaml
include:
  - .well-known
```

#### 1-5. デプロイと動作確認

1. 変更をデプロイ
2. `https://link.profilecode.codes/join?token=test123` で動作確認
3. 実機テストでアプリが起動することを確認

---

### ステップ2: `profilecode.codes/join` のリダイレクト実装（約30分）

#### 2-1. `join.html` の作成

`profilecode.codes` 側で `join.html` を作成：

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

#### 2-2. `index.html` から `/join` 処理を削除

`profilecode.codes` 側の `index.html` から以下のコードを削除：

```javascript
// 削除対象
if (path === "/join" && token) {
    // Sharelinkの場合
    window.location.href = `intent://join?token=${token}...`;
}
```

#### 2-3. `index.html` をランディングページに変更

`index.html` をランディングページのコンテンツに変更（アプリ紹介、ダウンロードリンクなど）

#### 2-4. デプロイと動作確認

1. 変更をデプロイ
2. `https://profilecode.codes/join?token=test123` が `link.profilecode.codes/join?token=test123` にリダイレクトされることを確認
3. `https://profilecode.codes/` がランディングページとして表示されることを確認

---

### ステップ3: 動作確認（約30分）

1. `link.profilecode.codes/join?token=test123` でアプリが起動することを確認
2. `profilecode.codes/join?token=test123` が正しくリダイレクトされることを確認
3. 実機テストで動作確認
4. HTTPアクセスで各種設定を確認

---

## 📊 実装前後の比較

### 実装前

| 項目 | 状態 | 問題 |
|------|------|------|
| `link.profilecode.codes` | ❌ | 未実装 |
| `profilecode.codes/join` | ⚠️ | メインドメインで処理（混在） |
| パス判定 | ❌ | 厳密一致のみ（`path === "/join"`） |
| カスタムスキーム | ❌ | `profilecode://`（誤り） |
| paths設定 | ⚠️ | 全パス許可（`["*"]`） |

### 実装後

| 項目 | 状態 | 改善 |
|------|------|------|
| `link.profilecode.codes` | ✅ | 実装済み |
| `profilecode.codes/join` | ✅ | リダイレクト（役割分離） |
| パス判定 | ✅ | サブパスも対応（`path.startsWith("/join")`） |
| カスタムスキーム | ✅ | `com.profilecode.profilecode://`（正しい） |
| paths設定 | ✅ | 特定パスのみ許可 |

---

## 🎯 期待される結果

実装完了後、以下の動作が期待されます：

1. ✅ `link.profilecode.codes/join?token=TOKEN` でアプリが起動する
2. ✅ `link.profilecode.codes/join/anything?token=TOKEN` でもアプリが起動する
3. ✅ `profilecode.codes/join?token=TOKEN` が `link.profilecode.codes/join?token=TOKEN` にリダイレクトされる
4. ✅ `profilecode.codes/` がランディングページとして表示される
5. ✅ アプリ未インストール時、カスタムスキームが正しく試行される
6. ✅ `apple-app-site-association`が正しく配信される
7. ✅ セキュリティが向上する（特定パスのみ許可）
8. ✅ 役割が明確に分離される（メインドメイン = マーケティング、サブドメイン = ディープリンク）

---

## ⚠️ 注意事項

### 実装時の注意点

1. **パス判定**: `path.startsWith("/join")` を使用（厳密一致ではない）
2. **カスタムスキーム**: `com.profilecode.profilecode://` を使用（`profilecode://` ではない）
3. **apple-app-site-association**: 正しい `paths` を設定（`["*"]` ではない）
4. **リダイレクト**: クエリパラメータとハッシュを保持する

### デプロイ後の確認

1. `link.profilecode.codes/join?token=test123` でアプリが起動することを確認
2. `profilecode.codes/join?token=test123` が正しくリダイレクトされることを確認
3. 実機テストでアプリが正しく起動することを確認
4. HTTPアクセスで各種設定を確認

---

## 📝 実装完了報告フォーマット

実装完了後、以下のフォーマットで報告をお願いします：

```
実装完了日時: YYYY-MM-DD
実装者: [名前]

必須実装項目（link.profilecode.codes）:
- [ ] リポジトリ/プロジェクトの作成
- [ ] index.htmlの実装
- [ ] apple-app-site-associationの配置
- [ ] デプロイと動作確認

必須修正項目（profilecode.codes）:
- [ ] join.htmlの作成
- [ ] index.htmlから/join処理を削除
- [ ] index.htmlをランディングページに変更
- [ ] デプロイと動作確認

確認項目:
- [ ] link.profilecode.codes/joinでアプリが起動する
- [ ] profilecode.codes/joinがリダイレクトされる
- [ ] HTTPアクセス確認
- [ ] 実機テスト（iOS）
- [ ] 実機テスト（Android）

問題点・改善点:
1. [問題点があれば記載]
2. [改善点があれば記載]
```

---

## 🔗 参考情報

- アプリ側の実装: `lib/deep_link/deep_link_handler.dart`
- iOS設定: `ios/Runner/Info.plist`, `apple-app-site-association`
- Android設定: `android/app/src/main/AndroidManifest.xml`
- 実装計画: `docs/link_profilecode_implementation_plan.md`
- 詳細な確認チェックリスト: `docs/link_profilecode_verification_checklist.md`
- 質問リスト: `docs/link_profilecode_verification_questions.md`

---

**最終更新**: 2024年
**作成者**: profilecode開発チーム
**ベース**: 質問リスト回答と現在の状況（2024年）
**ベース**: 質問リスト回答（2024年）

