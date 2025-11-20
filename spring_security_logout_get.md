Spring Security の **ログアウト処理をカスタマイズする方法**を、
**Spring Boot 3.4 / Spring Security 6.x（最新アーキテクチャ）対応**で、
**実装コード中心 + 実務でよく使うパターン全部**まとめて解説します。

---

# ✅ Spring Security のログアウトはどう動いているか

Spring Security のデフォルトでは

* `/logout` を `POST` で叩くとログアウト
* `SecurityContext` をクリア
* セッション無効化
* `/login?logout` へリダイレクト

という処理が自動的に入っています。

**カスタマイズは `HttpSecurity.logout()` の設定で行う。**

---

# ✅ 基本：ログアウトのエンドポイントを変更

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .logout(logout -> logout
            .logoutUrl("/api/logout")   // デフォルト: /logout
        );
    return http.build();
}
```

---

# ✅ ログアウト成功時の挙動を変更（リダイレクト・JSON返却など）

### ● リダイレクト先を変更

```java
.logout(logout -> logout
    .logoutSuccessUrl("/goodbye")
)
```

### ● JSON を返す（SPA / API 用）

```java
.logout(logout -> logout
    .logoutSuccessHandler((request, response, auth) -> {
        response.setStatus(HttpServletResponse.SC_OK);
        response.setContentType("application/json");
        response.getWriter().write("{\"message\": \"logout success\"}");
    })
)
```

---

# ✅ セッション削除の挙動をカスタム

Spring Security はデフォルトで `session.invalidate()` を呼ぶが、変更可能。

```java
.logout(logout -> logout
    .invalidateHttpSession(true)        // デフォルト true
    .clearAuthentication(true)          // 認証情報クリア
    .deleteCookies("JSESSIONID")        // セッションIDクッキー削除
)
```

APIではクッキー削除が重要。

---

# ✅ ログアウト前に独自処理を入れたい（ログアウトハンドラー）

例：DB に audit log を書きたい、JWT blacklist に追加したいなど。

```java
.logout(logout -> logout
    .addLogoutHandler((request, response, authentication) -> {
        if (authentication != null) {
            System.out.println("Logout user: " + authentication.getName());
        }
    })
)
```

複数個 add 可能。

---

# ✅ ログアウト成功後に別ページを返したい（Handler）

```java
.logout(logout -> logout
    .logoutSuccessHandler(new LogoutSuccessHandler() {
        @Override
        public void onLogoutSuccess(
                HttpServletRequest request,
                HttpServletResponse response,
                Authentication authentication) throws IOException {
            response.sendRedirect("/custom-logout-page");
        }
    })
)
```

またはラムダで十分。

---

# ✅ カスタムログアウト用 HTML フォーム（CSRF 対応）

Spring Security は **ログアウトは POST** が必要（CSRF 対策）。

```html
<form action="/logout" method="post">
  <input type="hidden" name="_csrf" value="${_csrf.token}">
  <button type="submit">Logout</button>
</form>
```

`GET /logout` を許可したい場合：

```java
.logout(logout -> logout
    .logoutRequestMatcher(new AntPathRequestMatcher("/logout", "GET"))
)
```

※ セキュリティ的には推奨されない。

---

# ✅ API（SPA / モバイル向け）は CSRF を切ってログアウト

REST API の場合、クッキーではなく JWT を使うため CSRF 無効化が一般的。

```java
http
    .csrf(csrf -> csrf.disable())
    .logout(logout -> logout
         .logoutUrl("/api/logout")
         .logoutSuccessHandler((req, res, auth) -> {
             res.setStatus(200);
             res.getWriter().write("{\"status\":\"logout\"}");
         })
    );
```

---

# ✅ JWT のログアウトでやるべきこと（重要）

JWT は stateless なので「ログアウト」という概念がない。

**方法：**

1. フロントから JWT を削除
2. サーバ側の blacklist に入れる（必要なら）
3. `SecurityContextHolder.clearContext()` を呼ぶ

例：

```java
.addLogoutHandler((request, response, auth) -> {
    String token = request.getHeader("Authorization");
    if (token != null) {
        jwtBlacklistService.add(token); // DB or Redis へ
    }
})
```

---

# 🔥 実用的なフルサンプル（Spring Boot 3.4）

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .logout(logout -> logout
                .logoutUrl("/api/logout")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
                .deleteCookies("JSESSIONID")
                .addLogoutHandler((request, response, authentication) -> {
                    System.out.println("Logout: " +
                        (authentication != null ? authentication.getName() : "anonymous"));
                })
                .logoutSuccessHandler((request, response, authentication) -> {
                    response.setStatus(HttpServletResponse.SC_OK);
                    response.setContentType("application/json");
                    response.getWriter().write("{\"result\":\"success\"}");
                })
            );

        return http.build();
    }
}
```

---

# 📌 まとめ

SpringSecurity のログアウトは

### **ポイントは全部 `http.logout(...)` の設定でカスタム可能**

| やりたいこと    | 設定                        |
| --------- | ------------------------- |
| パス変更      | `logoutUrl()`             |
| リダイレクト変更  | `logoutSuccessUrl()`      |
| JSON返却    | `logoutSuccessHandler()`  |
| 独自処理追加    | `addLogoutHandler()`      |
| セッション削除制御 | `invalidateHttpSession()` |
| Cookie削除  | `deleteCookies()`         |
| GETでログアウト | `logoutRequestMatcher()`  |
| JWTログアウト  | 独自 handler + Blacklist    |

---

# 必要ならもっと詳しく書けます

* **LogoutHandler と LogoutSuccessHandler の違い**
* **セッション方式 vs JWT方式のログアウト設計整理**
* **Cookie/JWT 混在環境でのベストプラクティス**
* **ローカル or CloudFront 配下でのログアウト挙動**

どこを深掘りする？
