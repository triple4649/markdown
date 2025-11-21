以下に、これまであなたと議論した **OIDC（OpenID Connect）ログイン・ログアウトの仕組みを総合的にまとめた Markdown** を整理して提供します。
**略語なし（OpenID Provider / Relying Party）で統一** しています。

---

# OIDC（OpenID Connect）のログイン・ログアウトの仕組みまとめ

（Spring Security を前提とした技術者向け解説）

## 1. OIDC の基本構造

OIDC は **OAuth2 の上に「認証機能」を追加した仕様**であり、
認証の中核は **OpenID Provider（認証サーバ）** が担い、
アプリケーション（Relying Party）は認証結果を委譲して受け取る。

### 主な登場人物

* **ユーザー**
* **Relying Party（アプリケーション）**
* **OpenID Provider（認証サーバ）**

---

## 2. OIDC ログイン（認証）の仕組み

### フロー概要（Authorization Code Flow + PKCE）

1. Relying Party → OpenID Provider へログイン要求（ブラウザリダイレクト）
2. OpenID Provider がユーザーをログインさせる
   → すでにログイン済みなら画面を出さず認証成功（ここが SSO の本質）
3. 認可コード（code）が Relying Party へ返される（通常 GET）
4. Relying Party → OpenID Provider へトークン交換（POST）
5. OpenID Provider から

   * ID Token（署名付き JWT：認証証明）
   * Access Token
     が返る
6. Relying Party がユーザーをログイン済みとしてセッション作成

---

## 3. OIDC におけるログアウトは3層構造

### 1. **Relying Party のログアウト**

* Relying Party が持つアプリケーションセッションを破棄するのみ
* **SSO は切れない（OpenID Provider のセッションは残る）**

### 2. **OpenID Provider のログアウト（SSO破棄）**

アプリケーション側が OpenID Provider のログアウトURLへリダイレクトする：

* `id_token_hint`
* `post_logout_redirect_uri`

を付けてリダイレクトすると、OpenID Provider が SSO のセッションを破棄。

### 3. **OpenID Provider → Relying Party のログアウト連携（Single Logout）**

OpenID Provider がログアウトした時、Relying Party にもログアウトを通知する仕組み：

* **Front-Channel Logout**（ブラウザの iframe 経由）
* **Back-Channel Logout**（サーバ間通信。より確実）

---

## 4. Spring Security におけるログアウトの正しい実装

### 通常の `/logout` はアプリ側のセッション破棄のみ

Spring Security のデフォルト `/logout` は：

* Relying Party のセッションのみ削除
* OpenID Provider のログイン状態はそのまま残る
  → SSO は切れない

---

## 5. OpenID Provider のログアウトを行う仕組み

### `OidcClientInitiatedLogoutSuccessHandler`

このハンドラをログアウト成功時に設定することで：

1. Relying Party のセッション破棄
2. **OpenID Provider のログアウトURLへ自動リダイレクト**
3. OpenID Provider が SSOセッションを破棄
4. `post_logout_redirect_uri` に戻る

という **完全ログアウトフロー** を実現できる。

#### 設定例

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .oauth2Login()
        .and()
        .logout(logout ->
            logout.logoutSuccessHandler(
                new OidcClientInitiatedLogoutSuccessHandler(clientRegistrationRepository)
            )
        );
    return http.build();
}
```

---

## 6. `clientRegistrationRepository` には何が格納されるか？

### ✔ OIDC 全クライアント設定（ClientRegistration）が格納される

各 ClientRegistration には：

* clientId
* clientSecret
* issuer（OpenID Provider の URL）
* authorizationUri
* tokenUri
* jwkSetUri
* redirectUri
* scope（openid, profile, email など）

などの **OpenID Provider と通信するための全設定情報** が含まれる。

### `OidcClientInitiatedLogoutSuccessHandler` がこれを使う理由

* どの OpenID Provider にログアウト要求を出すか判定
* ログアウトURLの組み立て
* post_logout_redirect_uri の生成
  に必要だから。

---

## 7. SSO 完全ログアウトのフロー（Spring Security版）

1. ユーザーが `/logout` にアクセス
2. Relying Party のセッション破棄
3. `OidcClientInitiatedLogoutSuccessHandler` が起動
4. clientRegistrationRepository から設定を取得
5. OpenID Provider の logout エンドポイントへ
   `id_token_hint` と `post_logout_redirect_uri` を付けてリダイレクト
6. OpenID Provider が SSO セッションを破棄
7. OpenID Provider → Relying Party へ戻る
8. 完全な OIDC ログアウト完了

---

## 8. まとめ（本質）

* **SSO の本体は OpenID Provider のセッション**
* Relying Party のログアウトだけでは SSO は切れない
* OIDC 正規のログアウトは
  **Relying Party → OpenID Provider へログアウト要求を出すことが必須**
* Spring Security では
  **`OidcClientInitiatedLogoutSuccessHandler` を設定することで
  OIDC の正しいログアウトが実現できる**

---

必要であれば、次の追加資料もまとめられます：

* OIDC ログアウトフローの図解
* Keycloak / Azure AD / Google それぞれの logout URL 設定例
* SPA + Spring Boot の OIDC 完全ログアウト設計
* Back-Channel Logout の Spring 実装例

どれを追記しますか？



はい、**Back-Channel Logout は必ず POST です。**

理由も含めて正確に説明します。

---

# ✔ Back-Channel Logout は POST 固定（仕様で決まっている）

OpenID Connect Back-Channel Logout の RFC（OpenID Connect Back-Channel Logout 1.0）では、
**認証サーバ（OpenID Provider）→ アプリケーション（Relying Party）の通知は POST で送る** と明記されています。

### ◆ 通信形式（必須）

```
POST /backchannel-logout
Content-Type: application/x-www-form-urlencoded

logout_token=<JWT>
```

ここで送られるものは **logout_token（JWT）** であり、
サーバ間のバックチャネル通信のため **GET などは絶対に使ってはいけない** とされています。

---

# ✔ なぜ POST なのか？

## 1. **JWT（logout_token）が長いため GET に向かない**

Query parameter では URL 長の制限にかかる可能性がある。

## 2. **セキュリティ仕様上、バックチャネルは「API 呼び出し」として扱うため**

サーバ間通信は HTTP API として扱うべき、という設計思想。

## 3. **認証サーバが iframe を使わず直接サーバに通知するため**

Front-Channel のようにブラウザを介さないので GET を使う意味がない。

---

# ✔ Back-Channel Logout で Relying Party がやること

1. 認証サーバからの POST を受け取る
2. `logout_token`（JWT）を verify（署名検証）
3. `sub` や `sid` を取り出す
4. そのセッションを破棄する

Spring Security では現状、**この部分は自前実装が必要**。

---

# ✔ Front-Channel と比較（確実に理解するため）

| 方式                       | 通信経路           | メソッド         | 用途       |
| ------------------------ | -------------- | ------------ | -------- |
| **Front-Channel Logout** | ブラウザ経由（iframe） | GET          | 手軽だが不確実  |
| **Back-Channel Logout**  | サーバ間通信         | **POST（必須）** | 確実なログアウト |

---

# ✔ まとめ

* **Back-Channel Logout = POST 固定**
* Content-Type は `application/x-www-form-urlencoded`
* payload は `logout_token=<JWT>`
* Relying Party は POST を受け、JWT を検証しセッション削除する

---

必要なら：

* Back-Channel Logout の Spring Boot 実装例
* logout_token（JWT）の中身の例
* セッションID（sid）管理の注意点
* Front/Back を併用する構成例

どれをまとめますか？

