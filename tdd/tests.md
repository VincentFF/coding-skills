# Good and Bad Tests

## Good Tests

**Integration-style**: Test through real interfaces, not mocks of internal parts.

```go
// GOOD: Tests observable behavior
func TestUserCanCheckoutWithValidCart(t *testing.T) {
    cart := CreateCart()
    cart.Add(product)
    result, err := Checkout(cart, paymentMethod)
    if err != nil {
        t.Fatalf("checkout failed: %v", err)
    }
    if result.Status != "confirmed" {
        t.Errorf("status = %q, want %q", result.Status, "confirmed")
    }
}
```

Characteristics:

- Tests behavior users/callers care about
- Uses public API only
- Survives internal refactors
- Describes WHAT, not HOW
- One logical assertion per test

## Bad Tests

**Implementation-detail tests**: Coupled to internal structure.

```go
// BAD: Tests implementation details
func TestCheckoutCallsPaymentServiceProcess(t *testing.T) {
    mockPayment := &MockPaymentService{}
    Checkout(cart, mockPayment)
    if !mockPayment.ProcessCalled {
        t.Error("expected Process to be called")
    }
}
```

Red flags:

- Mocking internal collaborators
- Testing private methods
- Asserting on call counts/order
- Test breaks when refactoring without behavior change
- Test name describes HOW not WHAT
- Verifying through external means instead of interface

```go
// BAD: Bypasses interface to verify
func TestCreateUserSavesToDatabase(t *testing.T) {
    CreateUser(UserInput{Name: "Alice"})
    row := db.Query("SELECT * FROM users WHERE name = ?", "Alice")
    if row == nil {
        t.Error("expected row to exist")
    }
}

// GOOD: Verifies through interface
func TestCreateUserMakesUserRetrievable(t *testing.T) {
    user, err := CreateUser(UserInput{Name: "Alice"})
    if err != nil {
        t.Fatalf("create user failed: %v", err)
    }
    retrieved, err := GetUser(user.ID)
    if err != nil {
        t.Fatalf("get user failed: %v", err)
    }
    if retrieved.Name != "Alice" {
        t.Errorf("name = %q, want %q", retrieved.Name, "Alice")
    }
}
```

**Tautological tests**: Expected value restates the implementation, so the test passes by construction.

```go
// BAD: Expected value is recomputed the way the code computes it
func TestCalculateTotalSumsLineItems(t *testing.T) {
    items := []LineItem{{Price: 10}, {Price: 5}}
    expected := 0
    for _, i := range items {
        expected += i.Price
    }
    if CalculateTotal(items) != expected {
        t.Errorf("total mismatch")
    }
}

// GOOD: Expected value is an independent, known literal
func TestCalculateTotalSumsLineItems(t *testing.T) {
    got := CalculateTotal([]LineItem{{Price: 10}, {Price: 5}})
    if got != 15 {
        t.Errorf("CalculateTotal() = %d, want 15", got)
    }
}
```
