以下に、**「Spring Security で LogoutFilter を使わず、Controller にあるログアウト業務ロジックを Filter から強制実行する方法」**について、
これまでの議論を **Markdown形式で体系的にまとめた完全版**を示します。

---

# Spring Security のログアウト処理を Controller に持ちながら

# Filter から強制的にログアウトするための設計指針

## 📌 背景

Spring Security では通常、ログアウトは以下の仕組みで処理される。

* `LogoutFilter`
* `LogoutHandler`
* `LogoutSuccessHandler`

しかし今回の要件は、

* **ログアウト処理が Controller に存在し、そこに業務ロジックが含まれている**
* **LogoutFilter／LogoutHandler を使えない（業務ロジックを統合できない）**
* **Filter 側から強制的にログアウト処理を起こしたい**

というもの。

この構造では、Spring Security 標準の LogoutFilter を利用することはできない。
では、どうやって Filter から Controller のログアウト処理を実行するのか？

---

## 🎯 結論（最重要）

### ✔ Controller にあるログアウト処理を **Service に抽出する**

### ✔ Filter からその Service の `logout()` を直接呼び出す！

**これ以外の方法は存在しない。
これが唯一の正攻法であり、最も安全で壊れない設計である。**

---

## 🧨 なぜ LogoutFilter/LogoutHandler を使えないのか？

* Controller にすでに存在する業務ロジックは **LogoutFilter に注入できない**
* LogoutFilter は URL ベースのため **Filter から任意タイミングで呼べない**
* Controller を Filter から直接呼ぶことはできない（Servlet の階層が違う）
* Session／SecurityContext の状態遷移が壊れる危険性が高い

→ だから、**ログアウト処理を Controller から切り離す必要がある**。

---

## 🧩 設計ステップ

### ### Step 1: Controller にあるログアウト業務ロジックを整理

**Before（Controller に直書きされている状態）**

```java
@PostMapping("/logout")
public ResponseEntity<?> logout(HttpServletRequest req) {

    // 業務ロジック
    auditService.write(...);
    businessService.unlock(...);

    // 春のログアウト処理
    req.getSession().invalidate();
    SecurityContextHolder.clearContext();

    return ResponseEntity.ok("logout");
}
```

---

## ✔ Step 2: 共通の LogoutService を作成する

Controller と Filter の両方が呼べるようにする。

```java
@Service
public class LogoutService {

    private final AuditService auditService;
    private final BusinessService businessService;

    public void logout(HttpServletRequest request, Authentication auth) {

        if (auth != null) {
            // ①業務ロジック
            businessService.unlock(auth.getName());
            auditService.writeLogout(auth.getName());
        }

        // ②SecurityContext のクリア
        request.getSession(false)?.invalidate();
        SecurityContextHolder.clearContext();
    }
}
```

---

## ✔ Step 3: Controller は LogoutService を呼ぶだけにする

```java
@PostMapping("/logout")
public ResponseEntity<?> logout(HttpServletRequest req) {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    logoutService.logout(req, auth);
    return ResponseEntity.ok("logout");
}
```

---

## ✔ Step 4: Filter からも LogoutService を直接呼び出す

```java
@Component
public class ForceLogoutFilter extends OncePerRequestFilter {

    private final LogoutService logoutService;

    public ForceLogoutFilter(LogoutService logoutService) {
        this.logoutService = logoutService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
        throws IOException, ServletException {

        if (shouldForceLogout(request)) {

            Authentication auth =
                SecurityContextHolder.getContext().getAuthentication();

            // Controller と完全に同じログアウト処理
            logoutService.logout(request, auth);

            response.setStatus(200);
            response.getWriter().write("{\"forced\": true}");
            return;
        }

        filterChain.doFilter(request, response);
    }

    private boolean shouldForceLogout(HttpServletRequest req) {
        return "1".equals(req.getHeader("X-Force-Logout"));
    }
}
```

---

## 🎉 これで満たされる要件

### ✔ Controller と Filter の両方からログアウト処理を一元的に呼べる

### ✔ Controller にあった業務ロジックはそのまま活かせる

### ✔ Spring Security の標準 LogoutFilter に一切依存しない

### ✔ 「強制ログアウト」「条件付きログアウト」が自然に実装できる

### ✔ セッション無効化・SecurityContext クリアも安全に行える

---

## ❌ 逆にやってはいけないこと

### ✕ Filter から Controller を直接呼ぶ

不可能。Spring MVC と Servlet フィルタは階層が違う。

### ✕ LogoutFilter を内部呼び出しして業務ロジックと統合

LogoutFilter は業務ロジックを持てず URL ベースなので適用不能。

### ✕ Service 抽出せずに Controller と Filter の両方へ複製コピペ

ロジックが二重管理になり破綻する。

---

## 📝 まとめ

| 項目                          | 結論                  |
| --------------------------- | ------------------- |
| LogoutFilter は使える？          | ❌ 業務ロジックがあるなら使えない   |
| Filter → Controller の直接呼び出し | ❌ 不可能               |
| 正しいアーキテクチャ                  | ✔ LogoutService を作る |
| Controller からのログアウト         | ✔ Service を呼ぶ       |
| Filter からのログアウト             | ✔ Service を呼ぶ       |

---

# 🔥 最終結論

### **ログアウト処理を Controller に持ったまま Filter から強制実行したい場合、

Service 化して Controller と Filter がその Service を呼ぶ構造しかない。**

この方式は Spring Security のアーキテクチャにも完全に適合し、
ログアウト処理の一元化・安全性・テスト容易性のすべてを満たす。

---

必要なら以下も提供できます：

* LogoutService の完全テンプレート
* 業務ロジック（DB更新・二重ログイン解除など）の組み込み例
* Filter の優先順位設計（Before/After）
* JWT + セッション併用環境のログアウト設計ガイド

どこまでまとめますか？
