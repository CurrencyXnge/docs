<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 🔐 Authentication & Authorization (Casbin + JWT)

This document details the security model where Users verify via Email/Password, but Staff roles (Teller/Cashier) are strictly managed by Admins.

## 1. Authentication Flow

We enforce a strict separation between **Customers** (B2C) and **Staff** (Internal Ops). They use different login portals and identifiers.

### A. Customer Flow (Public App)
*   **Portal**: `https://app.currencyex.com/login`
*   **Identifier**: Email Address.
*   **Method**: Self-Service Signup -> Email Verification.

### B. Staff Flow (Secure Workstation)
*   **Portal**: `https://admin.currencyex.com/login` (Restricted IP typically)
*   **Identifier**: **System-Generated User ID** (e.g., `TELLER-8821`, `CASH-9002`).
*   **Control**: strictly **Admin-Managed**. Staff cannot sign up themselves.

#### Staff Onboarding Lifecycle (Super Admin Only)
1.  **Create**: Admin creates entry: `Name: John Doe`, `Role: TELLER`, `Branch: Mall Branch`.
2.  **Generate**: System auto-generates a unique **User ID** (`88214`) and a **Temporary Strong Password**.
3.  **Distribute**: Admin prints/hands over credentials physically to the staff member.
4.  **First Login**: Staff logs in with User ID + Temp Password -> **Forced** to set new Password + Setup MFA (TOTP).
5.  **Lockout**: Admin can "Freeze" this ID instantly, killing all active sessions.

---

## 2. JWT Structure & Claims

The token payload differs slightly to enforce the specific constraints.

**Staff Token (`access_token`)**:
```json
{
  "sub": "uuid-5678-abcd",       // Internal UUID
  "x_user_id": "TELLER-8821",   // The Human Readable ID
  "role": "TELLER",
  "branch_id": "branch-999",     // Security Boundary
  "scope": "internal_console",   // CANNOT be used on Public App
  "iss": "currencyex-auth-svc",
  "exp": 1735689600
}
```

**Security Enforcement**:
*   A token with `scope: internal_console` will be **REJECTED** if used on `api.currencyex.com/public/*`.
*   A token with `scope: public_app` (Customers) will be **REJECTED** on `api.currencyex.com/admin/*`.

---

## 3. Authorization (RBAC with Casbin)

We use **Casbin** for fine-grained permission control. The policy is loaded from CSV or Database.

### 3.1. Casbin Model (`auth_model.conf`)
```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

### 3.2. Policy Rules (`auth_policy.csv`)

| Subject (Role) | Object (Resource) | Action | Description |
| :--- | :--- | :--- | :--- |
| `SUPER_ADMIN` | `*` | `*` | God Mode |
| `BRANCH_MANAGER` | `/api/v1/staff` | `create` | Can onboarding Tellers |
| `BRANCH_MANAGER` | `/api/v1/reports` | `read` | Branch-level reports |
| `TELLER` | `/api/v1/exchange` | `execute` | Can perform currency swap |
| `TELLER` | `/api/v1/drawer` | `read` | Can view own drawer balance |
| `CASHIER` | `/api/v1/remit` | `execute` | Can send remittance |
| `CUSTOMER` | `/api/v1/wallet` | `read` | Can view own wallet |

---

## 4. API Implementation (Middleware)

```go
// middleware/auth.go

func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. Validate JWT
        claims, err := jwtService.ParseToken(c.GetHeader("Authorization"))
        if err != nil {
            c.AbortWithStatus(401)
            return
        }

        // 2. Set Context
        c.Set("user_id", claims.Sub)
        c.Set("role", claims.Role)
        c.Next()
    }
}

func CasbinMiddleware(enforcer *casbin.Enforcer) gin.HandlerFunc {
    return func(c *gin.Context) {
        role := c.GetString("role") // "TELLER"
        obj := c.Request.URL.Path   // "/api/v1/exchange"
        act := c.Request.Method     // "POST" -> "execute" mapped

        // 3. Check Policy
        ok, _ := enforcer.Enforce(role, obj, act)
        if !ok {
            c.AbortWithStatusJSON(403, gin.H{"error": "Forbidden"})
            return
        }
        c.Next()
    }
}
```

---

## 5. Security Best Practices

1.  **Password Storage**: Argon2id (Salted & Hashed).
2.  **Session Mgmt**:
    *   **Access Token**: 15 min Validity.
    *   **Refresh Token**: 7 Days Validity (Stored in HttpOnly Cookie).
3.  **Branch Isolation**:
    *   Middleware checks `branch_id` in JWT for Tellers.
    *   A Teller from "Branch A" cannot access "Branch B" drawer even if they tweak the API request ID.
