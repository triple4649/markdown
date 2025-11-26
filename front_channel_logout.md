了解しました。
あなたが求めているのは **「別のサービスでログアウトしたとき、こちらのサービスも自動的にログアウトされる（シングルサインオンの“フロントチャネル連鎖ログアウト”）」** の実装例です。

つまり：

* **Aサービス**でログアウト
  　↓
* 中央ログインサーバーが
  　「他のすべてのサービス（Bサービス、Cサービスなど）」に
  　**ブラウザ経由で “ログアウト要求” を送りつける**
  　↓
* 各サービスはその要求を受けて **自動ログアウト処理を実行する**

これが **フロントチャネルログアウト（連鎖ログアウト）** の本質です。

そしてこれは、
あなたの認識の通り **利用者向けサービス → 中央ログインサーバー ではなく**、

> **中央ログインサーバー → ブラウザ → 各サービス**
> にログアウトメッセージが飛んでくる仕組み

です。

---

# 🟥 重要：Spring Security “単体では” フロントチャネル連鎖ログアウトを自動で実装しない

Spring Security が提供するのは、

* **自分のサービスから中央ログインサーバーへログアウトする仕組み**
  （`OidcClientInitiatedLogoutSuccessHandler`）

だけです。

しかしあなたの求める **「別のサービスでログアウトしたときに、こちらも自動的にログアウトされる」** の機能は、

* 中央ログインサーバー（Keycloak など）が
  　**各利用者向けサービスの “Front-Channel Logout URL” に iframe を張る**

* 利用者向けサービス側は
  　**そのURLでログアウト処理を行えるようにしておく**

という2段構成になります。

---

# ✅ フロントチャネル連鎖ログアウトの“正しい動作フロー”

ここではサービスA・サービスBの2つのサービスがシングルサインオンされているとします。

---

## ▼ 1. ユーザーが「サービスA」でログアウトする

```
https://serviceA.example.com/logout
```

サービスA
→ 中央ログインサーバー
→ ログインセッションが破棄される

---

## ▼ 2. 中央ログインサーバーは「ログアウト用HTML」を返す

（重要：このHTMLの中に iframe が複数埋め込まれる）

例（概念図）：

```html
<html>
 <body>
   <iframe src="https://serviceA.example.com/front-channel-logout"></iframe>
   <iframe src="https://serviceB.example.com/front-channel-logout"></iframe>
   <iframe src="https://serviceC.example.com/front-channel-logout"></iframe>
 </body>
</html>
```

この iframe を **ブラウザが読み込む** ことで、
各サービスに対して「ログアウトせよ」という通知が飛ぶ。

---

## ▼ 3. Bサービスは iframe のアクセスを受けてログアウト処理を実行

```
GET https://serviceB.example.com/front-channel-logout
```

サービスBは “このリクエストが来たらセッションを破棄すれば良い”。

---

# 🟩 Spring Securityで必要なのはこれだけ

## ★ 利用者向けサービス側に

　**front-channel-logout 用の URL を用意してセッションを削除する**

中央ログインサーバーが iframe でアクセスするだけなので、
認証のチェックも不要、
CSRF も不要、
とにかく「来たらセッションを切る」だけでよい。

---

# 🔧 **Spring Boot / Spring Security の実装例（略語なし）**

## 1. Front-Channel Logout のURLを作る

```java
@RestController
public class FrontChannelLogoutController {

    @GetMapping("/front-channel-logout")
    public ResponseEntity<String> frontChannelLogout(HttpServletRequest request,
                                                     HttpServletResponse response) {
        // セッションを破棄
        var session = request.getSession(false);
        if (session != null) session.invalidate();

        // SecurityContextも削除
        SecurityContextHolder.clearContext();

        // 中央ログインサーバーの要求なので何か返す必要はない
        return ResponseEntity.ok("OK");
    }
}
```

---

## 2. セキュリティ設定：/front-channel-logout を全許可する

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/front-channel-logout").permitAll()
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/front-channel-logout")
            );

        return http.build();
    }
}
```

---

# 🟦 これでどう動く？

1. ユーザーが **サービスA** でログアウト
2. 中央ログインサーバーは「ログアウト完了画面」を返す
   → 中に **サービスA・B・C の front-channel-logout への iframe** が入っている
3. ブラウザが **サービスB/C** に自動で GET

   ```
   https://serviceB.example.com/front-channel-logout
   ```
4. B/Cはセッション破棄
5. 次に B/C を操作しようとすると、ログイン画面に戻る

---

# 📌 まとめ（あなたが求めていた内容）

| 内容                          | 説明                                 |
| --------------------------- | ---------------------------------- |
| 別ページでログアウトされたらこちらもログアウト？    | 可能                                 |
| Spring Security が自動でやってくれる？ | いいえ。 **利用者サービス側はURLを1つ用意するだけ**     |
| 中央ログインサーバーは？                | iframe を張るHTMLを返し、各サービスを自動的に呼ぶ     |
| 必要な実装                       | `/front-channel-logout` のセッション破棄のみ |

---

# さらに必要なら

* Keycloak の設定（Front-Channel Logout URL の設定方法）
* Azure AD / Auth0 / Cognito での front-channel logout の設定
* back-channel logout との比較
* SSOログイン＋フロントチャネルログアウトの動作検証方法
* ブラウザのキャッシュ問題の回避方法

なども提供できますので言ってください。

