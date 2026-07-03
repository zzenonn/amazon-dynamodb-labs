---
title: "PutItem - Create entities"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

The `PutItem` operation creates a new item or replaces an existing item with the same key. In this step, you write functions to create each entity type.

## Create the repository

All DynamoDB data-plane operations live in a `Repository` type. Create a file named `repository.go` with the struct and its constructor:

```go
package main

import (
	"context"
	"fmt"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
)

type Repository struct {
	client    *dynamodb.Client
	tableName string
}

func NewRepository(client *dynamodb.Client, tableName string) *Repository {
	return &Repository{
		client:    client,
		tableName: tableName,
	}
}
```

Notice there is no `CreateTable` function. The table was provisioned with CloudFormation in the previous module — application code only touches the data plane.

## Create a user

Add the following function to `repository.go`:

```go
func (r *Repository) CreateUser(ctx context.Context, user User) error {
	userMap, err := attributevalue.MarshalMap(user)
	if err != nil {
		return err
	}

	userMap["pk"] = &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", user.Username)}
	userMap["sk"] = &types.AttributeValueMemberS{Value: "PROFILE"}

	_, err = r.client.PutItem(ctx, &dynamodb.PutItemInput{
		TableName: aws.String(r.tableName),
		Item:      userMap,
	})
	return err
}
```

This function uses `attributevalue.MarshalMap` to convert the Go struct into a DynamoDB attribute map using the `dynamodbav` struct tags. Then it manually sets the `pk` and `sk` attributes based on the key design from the data model chapter.

Notice that `User.Username` has the tag `dynamodbav:"-"`, so it is excluded from marshaling. The username is encoded in the partition key instead of stored as a redundant attribute.

## Create an order

Add the order creation function:

```go
func (r *Repository) CreateOrder(ctx context.Context, order *Order) error {
	orderMap, err := attributevalue.MarshalMap(order)
	if err != nil {
		return err
	}

	orderMap["pk"] = &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", order.UserID)}
	orderMap["sk"] = &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", order.ID)}

	statusDate := fmt.Sprintf("%s#%s", order.Status, order.CreatedAt.Format("2006-01-02"))
	orderMap["status_date"] = &types.AttributeValueMemberS{Value: statusDate}

	if order.Status == OrderStatusPending || order.Status == OrderStatusConfirmed {
		orderMap["placed_id"] = &types.AttributeValueMemberS{Value: string(order.Status)}
	}

	_, err = r.client.PutItem(ctx, &dynamodb.PutItemInput{
		TableName: aws.String(r.tableName),
		Item:      orderMap,
	})
	return err
}
```

There are two important details here:

1. **`status_date`** is a composite attribute that combines the order status and creation date. This attribute is the sort key for the `status-date-index` LSI, enabling queries like "get all pending orders for this user, sorted by date."

2. **`placed_id`** is only set when the order is in `pending` or `confirmed` status. This is what makes the `placed-index` GSI a sparse index — only active orders appear in it.

## Create an order item

Add the order item creation function:

```go
func (r *Repository) CreateOrderItem(ctx context.Context, orderID string, item *OrderItem) error {
	itemMap, err := attributevalue.MarshalMap(item)
	if err != nil {
		return err
	}

	itemMap["pk"] = &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)}
	itemMap["sk"] = &types.AttributeValueMemberS{Value: fmt.Sprintf("#ITEM#%s", item.ItemID)}

	_, err = r.client.PutItem(ctx, &dynamodb.PutItemInput{
		TableName: aws.String(r.tableName),
		Item:      itemMap,
	})
	return err
}
```

Order items use the order ID as their partition key. This means all items belonging to the same order are co-located, which lets you fetch them all in a single query.

## Test writing data

Update `main.go` to create some sample data:

```go
func main() {
	region := os.Getenv("AWS_REGION")
	if region == "" {
		region = "us-east-1"
	}

	tableName := os.Getenv("DYNAMODB_TABLE_NAME")
	if tableName == "" {
		tableName = "simple-inventory"
	}

	cfg, err := config.LoadDefaultConfig(context.TODO(),
		config.WithRegion(region),
	)
	if err != nil {
		log.Fatalf("Failed to load AWS config: %v", err)
	}

	client := dynamodb.NewFromConfig(cfg)
	repo := NewRepository(client, tableName)
	ctx := context.Background()

	// Create a user
	user := User{
		Username: "john",
		FullName: "John Doe",
		Email:    "john@example.com",
		Addresses: map[string]Address{
			"home": {Street: "123 Main St", State: "CA", Country: "USA"},
			"work": {Street: "456 Office Blvd", State: "CA", Country: "USA"},
		},
	}

	fmt.Println("Creating user 'john'...")
	if err := repo.CreateUser(ctx, user); err != nil {
		log.Fatalf("Failed to create user: %v", err)
	}
	fmt.Println("User created.")

	// Create an order
	order := &Order{
		ID:         "order-001",
		UserID:     "john",
		Status:     OrderStatusPending,
		AddressKey: "home",
		CreatedAt:  time.Now(),
		UpdatedAt:  time.Now(),
	}

	fmt.Println("Creating order 'order-001'...")
	if err := repo.CreateOrder(ctx, order); err != nil {
		log.Fatalf("Failed to create order: %v", err)
	}
	fmt.Println("Order created.")

	// Create order items
	items := []OrderItem{
		{ItemID: "item-001", Name: "Laptop", Description: "Gaming laptop", Price: 1299.99, Quantity: 1},
		{ItemID: "item-002", Name: "Mouse", Description: "Wireless mouse", Price: 29.99, Quantity: 2},
	}

	for _, item := range items {
		fmt.Printf("Creating item '%s'...\n", item.Name)
		if err := repo.CreateOrderItem(ctx, order.ID, &item); err != nil {
			log.Fatalf("Failed to create item: %v", err)
		}
	}
	fmt.Println("All items created.")
}
```

You need to add `"time"` to your imports in `main.go`. Run the code:

```bash
go run .
```

Expected output:
```text
Creating user 'john'...
User created.
Creating order 'order-001'...
Order created.
Creating item 'Laptop'...
Creating item 'Mouse'...
All items created.
```

You can verify the data in the DynamoDB console by navigating to **Services** → **DynamoDB** → **Tables** → **simple-inventory** → **Explore table items**.
