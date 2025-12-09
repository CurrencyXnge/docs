<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 🧪 Testing Strategy & Standards

This document defines the testing pyramid and the standards for writing testable code in CurrencyEx.

## 1. The Test Pyramid

We aim for the following distribution:
1.  **Unit Tests (70%)**: Fast, isolated tests for domain logic and utilities.
2.  **Integration Tests (20%)**: Testing DB queries, API handlers with real DB, and external adapters.
3.  **E2E / Application Tests (10%)**: Full user flows (e.g., "Login -> Send Money -> Check Balance").

---

## 2. Unit Testing (Go Standard Lib + Testify)

### Principles
*   **Table Driven Tests**: Use slice of structs for test cases.
*   **Mocks Generation**: Use `mockery` to generate mocks for interfaces.
*   **No IO**: Unit tests should never touch the disk, network, or real DB.

**Example: Currency Converter Logic**
```go
func TestConvertCurrency(t *testing.T) {
    tests := []struct {
        name     string
        amount   float64
        rate     float64
        expected float64
    }{
        {"Normal", 100.0, 1.5, 150.0},
        {"Zero", 0.0, 1.5, 0.0},
        {"HighPrecision", 10.1234, 2.0, 20.2468},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Convert(tt.amount, tt.rate)
            assert.InDelta(t, tt.expected, got, 0.0001)
        })
    }
}
```

---

## 3. Integration Testing (Docker Compose)

Integration tests spin up real dependencies using **Testcontainers-go** or Docker Compose.
*   We test Repositories effectively here.
*   We test HTTP Handlers using `httptest.NewRecorder`.

**Example: Postgres Repository Test**
```go
func TestUserRepo_Create(t *testing.T) {
    // Defines a cleanup to run after test
    defer testDB.TruncateTable("users")
    
    repo := NewUserRepo(testDB)
    user := &domain.User{Email: "test@example.com"}

    err := repo.Create(context.Background(), user)
    assert.NoError(t, err)

    saved, _ := repo.FindByEmail(context.Background(), "test@example.com")
    assert.Equal(t, user.Email, saved.Email)
}
```

---

## 4. E2E Testing (Postman / Newman)

Full API workflows are defined in a **Postman Collection**.
The CI pipeline runs these using `newman`.

**Scenario: Remittance Flow**
1.  **POST `/auth/login`**: Extract token -> Set `{{token}}` variable.
2.  **GET `/rates/live`**: Verify status 200, extract `rate_id`.
3.  **POST `/remit/quote`**: Lock rate using `rate_id`.
4.  **POST `/remit/send`**: Submit txn, assert status `CREATED`.
5.  **GET `/wallet/balance`**: Assert balance decreased.

---

## 5. Mocks & Stubs Strategy

Interfaces are key. Everything in `internal/domain` should be an interface.

**Command to generate mocks**:
```bash
mockery --all --dir internal/domain --output internal/domain/mocks
```

**Usage in Service Test**:
```go
repoMock := new(mocks.UserRepository)
repoMock.On("FindByEmail", "test@a.com").Return(validUser, nil)

svc := NewAuthService(repoMock)
token, err := svc.Login("test@a.com", "pass")
```

---

## 6. Code Coverage Gates

The CI pipeline enforces minimum coverage.
- **Fail Build if Coverage < 70%**

```bash
go test ./... -coverprofile=coverage.out
go tool cover -func=coverage.out
```
