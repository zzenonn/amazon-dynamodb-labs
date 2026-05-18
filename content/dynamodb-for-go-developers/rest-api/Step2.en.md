---
title: "Wire up routes and test"
date: 2021-04-21T07:33:04-05:00
weight: 20
---

In this step, you wire the handlers into an HTTP server and test every endpoint using `curl`.

## Update main.go for API mode

Replace `main.go` with the final version that serves the API:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware"
)

func main() {
	region := os.Getenv("AWS_REGION")
	if region == "" {
		region = "us-east-1"
	}

	tableName := os.Getenv("DYNAMODB_TABLE_NAME")
	if tableName == "" {
		tableName = "simple-inventory"
	}

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	cfg, err := config.LoadDefaultConfig(context.TODO(),
		config.WithRegion(region),
	)
	if err != nil {
		log.Fatalf("Failed to load AWS config: %v", err)
	}

	client := dynamodb.NewFromConfig(cfg)
	repo := NewRepository(client, tableName)
	api := NewAPI(repo)

	r := chi.NewRouter()
	r.Use(middleware.Logger)
	r.Use(middleware.Recoverer)

	// User routes
	r.Post("/users", api.CreateUser)
	r.Get("/users/{username}", api.GetUser)
	r.Get("/users/{username}/orders", api.GetUserOrders)

	// Order routes
	r.Post("/orders", api.CreateOrder)
	r.Get("/orders/{orderid}", api.GetOrder)
	r.Put("/orders/{orderid}/status", api.UpdateOrderStatus)
	r.Get("/orders/pending", api.GetPendingOrders)

	// Order item routes
	r.Get("/orders/{orderid}/items", api.GetOrderItems)

	fmt.Printf("Starting server on port %s...\n", port)
	fmt.Printf("Table: %s | Region: %s\n\n", tableName, region)
	fmt.Println("Endpoints:")
	fmt.Println("  POST   /users                        Create user")
	fmt.Println("  GET    /users/{username}             Get user profile")
	fmt.Println("  GET    /users/{username}/orders      Get user's orders")
	fmt.Println("  GET    /users/{username}/orders?status=pending  Filter by status (LSI)")
	fmt.Println("  POST   /orders                       Place order (transaction)")
	fmt.Println("  GET    /orders/{orderid}             Get order by ID (GSI)")
	fmt.Println("  PUT    /orders/{orderid}/status      Update order status")
	fmt.Println("  GET    /orders/pending               Get pending orders (sparse GSI)")
	fmt.Println("  GET    /orders/{orderid}/items       Get order items")

	log.Fatal(http.ListenAndServe(":"+port, r))
}
```

## Start the server

```bash
go run .
```

Expected output:
```text
Starting server on port 8080...
Table: simple-inventory | Region: us-east-1

Endpoints:
  POST   /users                        Create user
  GET    /users/{username}             Get user profile
  GET    /users/{username}/orders      Get user's orders
  GET    /users/{username}/orders?status=pending  Filter by status (LSI)
  POST   /orders                       Place order (transaction)
  GET    /orders/{orderid}             Get order by ID (GSI)
  PUT    /orders/{orderid}/status      Update order status
  GET    /orders/pending               Get pending orders (sparse GSI)
  GET    /orders/{orderid}/items       Get order items
```

Leave the server running and open a **new terminal** for the following tests.

## Test the API

### Create a user

```bash
curl -s -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{
    "username": "dave",
    "full_name": "Dave Wilson",
    "email": "dave@example.com",
    "addresses": {
      "home": {"street": "789 Oak Ave", "state": "NY", "country": "USA"}
    }
  }' | jq .
```

### Get a user profile (primary table)

```bash
curl -s http://localhost:8080/users/alice | jq .
```

### Place an order with items (transaction)

```bash
curl -s -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "alice",
    "address_key": "home",
    "items": [
      {"name": "Webcam", "price": 89.99, "quantity": 1},
      {"name": "USB Hub", "price": 24.99, "quantity": 2}
    ]
  }' | jq .
```

Note the order ID in the response — you need it for the next commands. Set it as a variable:

```bash
ORDER_ID="<paste-order-id-here>"
```

### Get an order by ID (inverted-index GSI)

```bash
curl -s http://localhost:8080/orders/$ORDER_ID | jq .
```

### Get order items (primary table query)

```bash
curl -s http://localhost:8080/orders/$ORDER_ID/items | jq .
```

### Get all of a user's orders (primary table query)

```bash
curl -s http://localhost:8080/users/alice/orders | jq .
```

### Get a user's pending orders (status-date-index LSI)

```bash
curl -s "http://localhost:8080/users/alice/orders?status=pending" | jq .
```

### Get all pending orders (placed-index sparse GSI)

```bash
curl -s http://localhost:8080/orders/pending | jq .
```

### Update order status

```bash
curl -s -X PUT http://localhost:8080/orders/$ORDER_ID/status \
  -H "Content-Type: application/json" \
  -d '{"status": "confirmed"}' | jq .
```

### Verify the order left the pending list

```bash
curl -s http://localhost:8080/orders/pending | jq .
```

## How each endpoint maps to a DynamoDB pattern

| Endpoint | DynamoDB Operation | Index |
|----------|-------------------|-------|
| `POST /users` | PutItem | Primary |
| `GET /users/{id}` | GetItem | Primary |
| `GET /users/{id}/orders` | Query (begins_with) | Primary |
| `GET /users/{id}/orders?status=X` | Query (begins_with) | LSI: status-date-index |
| `POST /orders` | TransactWriteItems | Primary |
| `GET /orders/{id}` | Query | GSI: inverted-index |
| `PUT /orders/{id}/status` | UpdateItem | Primary |
| `GET /orders/pending` | Query | GSI: placed-index |
| `GET /orders/{id}/items` | Query (begins_with) | Primary |

You now have a complete REST API backed by DynamoDB with single table design, exercising all the access patterns and indexes defined at the start of the workshop.
