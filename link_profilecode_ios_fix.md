# iOS アプリ起動後のストア遷移問題の修正

## 🐛 問題の症状

- **Android**: ✅ 正常動作（アプリ起動 or ストア遷移）
- **iOS（アプリ未インストール）**: ✅ 正常動作（ストア遷移）
- **iOS（アプリインストール済み）**: ❌ アプリが起動するが、すぐにストアに遷移してしまう

## 🔍 原因分析

### アプリ側の確認結果

**アプリ側は問題ありません**:
- `AppDelegate.swift` でUniversal Linksを正しく処理
- `deep_link_handler.dart` でストアへのリダイレクト処理は**存在しない**
- アプリ側はストアに遷移していない

### ウェブ側の問題

**問題の原因**: `link.profilecode.codes` 側の実装

現在の実装（問題あり）:
```javascript
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
```

**問題点**:
- Universal Linksでアプリが起動しても、JavaScriptのタイマーが動作し続ける
- 3秒後にカスタムスキームを試行
- さらに3秒後にApp Storeに遷移してしまう
- アプリが起動したことを検知する仕組みがない

---

## ✅ 解決策

### 修正方法: アプリ起動検知の実装

アプリが起動したことを検知し、タイマーをクリアする必要があります。

#### 修正後の実装

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

---

## 🔑 重要な修正ポイント

### 1. アプリ起動検知の実装

以下の3つのイベントでアプリ起動を検知：

```javascript
// 1. ページが非表示になったら（visibilitychange）
document.addEventListener('visibilitychange', function() {
    if (document.hidden) {
        detectAppLaunch();
    }
});

// 2. ページがアンロードされる前（pagehide）
window.addEventListener('pagehide', function() {
    detectAppLaunch();
});

// 3. ウィンドウがフォーカスを失ったら（blur）
window.addEventListener('blur', function() {
    detectAppLaunch();
});
```

### 2. タイマーのクリア

アプリが起動したことを検知したら、タイマーをクリア：

```javascript
function detectAppLaunch() {
    appLaunched = true;
    if (customSchemeTimeout) {
        clearTimeout(customSchemeTimeout);
    }
    if (appStoreTimeout) {
        clearTimeout(appStoreTimeout);
    }
}
```

---

## 📋 修正チェックリスト

### `link.profilecode.codes` 側の修正

- [ ] アプリ起動検知の実装（`visibilitychange`, `pagehide`, `blur`イベント）
- [ ] タイマーのクリア処理
- [ ] デプロイして動作確認

### 動作確認

- [ ] iOS実機でアプリが起動後、ストアに遷移しないことを確認
- [ ] iOS実機でアプリ未インストール時、ストアに遷移することを確認
- [ ] Android実機で動作確認（既存の動作が維持されているか）

---

## 🧪 テスト手順

### iOS実機テスト

1. アプリをインストールした状態で `https://link.profilecode.codes/join?token=test123` を開く
2. アプリが起動することを確認
3. **ストアに遷移しないことを確認** ✅
4. アプリをアンインストールした状態で同じURLを開く
5. ストアに遷移することを確認 ✅

---

## 📊 修正前後の比較

### 修正前

| 動作 | 結果 |
|------|------|
| iOS（アプリインストール済み） | ❌ アプリ起動 → ストア遷移 |
| iOS（アプリ未インストール） | ✅ ストア遷移 |
| Android | ✅ 正常動作 |

### 修正後

| 動作 | 結果 |
|------|------|
| iOS（アプリインストール済み） | ✅ アプリ起動（ストアに遷移しない） |
| iOS（アプリ未インストール） | ✅ ストア遷移 |
| Android | ✅ 正常動作 |

---

## ⚠️ 注意事項

### 実装時の注意点

1. **イベントリスナーの追加**: 3つのイベント（`visibilitychange`, `pagehide`, `blur`）すべてを実装する
2. **タイマーのクリア**: `clearTimeout` で確実にタイマーをクリアする
3. **フラグの管理**: `appLaunched` フラグでアプリ起動状態を管理する

### デプロイ後の確認

1. iOS実機でアプリが起動後、ストアに遷移しないことを確認
2. アプリ未インストール時、ストアに遷移することを確認
3. Androidでの動作が維持されていることを確認

---

## 🔗 参考情報

- [iOS Universal Links ドキュメント](https://developer.apple.com/documentation/xcode/supporting-universal-links-in-your-app)
- [Page Visibility API](https://developer.mozilla.org/ja/docs/Web/API/Page_Visibility_API)
- アプリ側の実装: `ios/Runner/AppDelegate.swift`, `lib/deep_link/deep_link_handler.dart`

---

**最終更新**: 2024年
**作成者**: profilecode開発チーム

