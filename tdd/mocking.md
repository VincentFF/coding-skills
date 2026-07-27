# When to Mock

Mock at **system boundaries** only:

- External APIs (payment, email, etc.)
- Databases (sometimes - prefer test DB)
- Time/randomness
- File system (sometimes)

Don't mock:

- Your own types/packages
- Internal collaborators
- Anything you control

## Designing for Mockability

At system boundaries, define small interfaces from the consumer's point of view. Production clients satisfy those interfaces; tests can replace only the external boundary.

**1. Use dependency injection**

Pass external dependencies in rather than creating them internally:

```go
// Easy to mock
type PaymentClient interface {
    Charge(ctx context.Context, amount int) (Receipt, error)
}

func ProcessPayment(ctx context.Context, order Order, paymentClient PaymentClient) (Receipt, error) {
    return paymentClient.Charge(ctx, order.Total)
}

// Hard to mock
func ProcessPaymentWithStripe(ctx context.Context, order Order) (Receipt, error) {
    client := NewStripeClient(os.Getenv("STRIPE_KEY"))
    return client.Charge(ctx, order.Total)
}
```

**2. Prefer SDK-style interfaces over generic fetchers**

Create specific functions for each external operation instead of one generic function with conditional logic:

```go
// GOOD: Each method is independently mockable
type StoreAPI interface {
    GetUser(ctx context.Context, id string) (*User, error)
    GetOrders(ctx context.Context, userID string) ([]Order, error)
    CreateOrder(ctx context.Context, data OrderData) (*Order, error)
}

type HTTPStoreAPI struct {
    baseURL string
    client  *http.Client
}

func (api *HTTPStoreAPI) GetUser(ctx context.Context, id string) (*User, error) {
    // Call GET /users/{id}.
    return nil, nil
}

func (api *HTTPStoreAPI) GetOrders(ctx context.Context, userID string) ([]Order, error) {
    // Call GET /users/{userID}/orders.
    return nil, nil
}

func (api *HTTPStoreAPI) CreateOrder(ctx context.Context, data OrderData) (*Order, error) {
    // Call POST /orders.
    return nil, nil
}

// BAD: Mocking requires conditional logic inside the mock
type StoreFetcher interface {
    Fetch(ctx context.Context, endpoint string, options RequestOptions) ([]byte, error)
}
```

The SDK approach means:
- Each mock returns one specific shape
- No conditional logic in test setup
- Easier to see which endpoints a test exercises
- Type safety per endpoint
