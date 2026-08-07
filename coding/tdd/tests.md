# Good and Bad Tests

## Good Tests

**Integration-style**: Test through real interfaces, not mocks of internal parts.

```go
// GOOD: Tests observable behavior
func TestUserCanCheckoutWithValidCart(t *testing.T) {
    cart := CreateCart()
    product := Product{SKU: "book", Price: 10}
    cart.Add(product)

    paymentMethod := PaymentMethod{Token: "tok_test"}
    result, err := Checkout(cart, paymentMethod)
    if err != nil {
        t.Fatalf("Checkout() error = %v", err)
    }
    if result.Status != "confirmed" {
        t.Errorf("Checkout().Status = %q, want %q", result.Status, "confirmed")
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
    if _, err := Checkout(cart, mockPayment); err != nil {
        t.Fatalf("Checkout() error = %v", err)
    }
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
    if _, err := CreateUser(UserInput{Name: "Alice"}); err != nil {
        t.Fatalf("CreateUser() error = %v", err)
    }

    var id string
    if err := db.QueryRow("SELECT id FROM users WHERE name = ?", "Alice").Scan(&id); err != nil {
        t.Fatalf("expected database row to exist: %v", err)
    }
}

// GOOD: Verifies through interface
func TestCreateUserMakesUserRetrievable(t *testing.T) {
    user, err := CreateUser(UserInput{Name: "Alice"})
    if err != nil {
        t.Fatalf("CreateUser() error = %v", err)
    }
    retrieved, err := GetUser(user.ID)
    if err != nil {
        t.Fatalf("GetUser() error = %v", err)
    }
    if retrieved.Name != "Alice" {
        t.Errorf("GetUser().Name = %q, want %q", retrieved.Name, "Alice")
    }
}
```

**Tautological tests**: Expected value restates the implementation, so the test passes by construction.

```go
// BAD: Expected value is recomputed the way the code computes it
func TestCalculateTotalSumsLineItems(t *testing.T) {
    items := []LineItem{{Price: 10}, {Price: 5}}
    want := 0
    for _, i := range items {
        want += i.Price
    }
    if got := CalculateTotal(items); got != want {
        t.Errorf("CalculateTotal() = %d, want %d", got, want)
    }
}

// GOOD: Expected value is an independent, known literal
func TestCalculateTotalSumsLineItemsFromKnownExample(t *testing.T) {
    got := CalculateTotal([]LineItem{{Price: 10}, {Price: 5}})
    if got != 15 {
        t.Errorf("CalculateTotal() = %d, want 15", got)
    }
}
```
