---
title: "TransactWriteItems"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

`TransactWriteItems` performs up to 100 write operations atomically. If any operation fails (due to a condition check, a conflict, or insufficient capacity), the entire transaction is rolled back. No partial writes occur.

## Use case: place an order atomically

When a user places an order, you need to:
1. Verify the user exists
2. Create the order
3. Create all order items

Without transactions, a failure partway through would leave orphaned items or an order without items. Transactions guarantee all-or-nothing.

## Write the PlaceOrder function

Add this function to `repository.go`:

```go
func (r *Repository) PlaceOrder(ctx context.Context, order *Order, items []OrderItem) error {
	var transactItems []types.TransactWriteItem

	// Condition check: verify the user exists
	transactItems = append(transactItems, types.TransactWriteItem{
		ConditionCheck: &types.ConditionCheck{
			TableName: aws.String(r.tableName),
			Key: map[string]types.AttributeValue{
				"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", order.UserID)},
				"sk": &types.AttributeValueMemberS{Value: "PROFILE"},
			},
			ConditionExpression: aws.String("attribute_exists(pk)"),
		},
	})

	// Put the order
	statusDate := fmt.Sprintf("%s#%s", order.Status, order.CreatedAt.Format("2006-01-02"))
	orderItem := map[string]types.AttributeValue{
		"pk":          &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", order.UserID)},
		"sk":          &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", order.ID)},
		"order_id":    &types.AttributeValueMemberS{Value: order.ID},
		"user_id":     &types.AttributeValueMemberS{Value: order.UserID},
		"status":      &types.AttributeValueMemberS{Value: string(order.Status)},
		"status_date": &types.AttributeValueMemberS{Value: statusDate},
		"placed_id":   &types.AttributeValueMemberS{Value: string(order.Status)},
		"address_key": &types.AttributeValueMemberS{Value: order.AddressKey},
		"created_at":  &types.AttributeValueMemberS{Value: order.CreatedAt.Format(time.RFC3339)},
		"updated_at":  &types.AttributeValueMemberS{Value: order.UpdatedAt.Format(time.RFC3339)},
	}

	transactItems = append(transactItems, types.TransactWriteItem{
		Put: &types.Put{
			TableName: aws.String(r.tableName),
			Item:      orderItem,
		},
	})

	// Put each order item
	for _, item := range items {
		transactItems = append(transactItems, types.TransactWriteItem{
			Put: &types.Put{
				TableName: aws.String(r.tableName),
				Item: map[string]types.AttributeValue{
					"pk":       &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", order.ID)},
					"sk":       &types.AttributeValueMemberS{Value: fmt.Sprintf("#ITEM#%s", item.ItemID)},
					"order_id": &types.AttributeValueMemberS{Value: order.ID},
					"item_id":  &types.AttributeValueMemberS{Value: item.ItemID},
					"name":     &types.AttributeValueMemberS{Value: item.Name},
					"price":    &types.AttributeValueMemberN{Value: fmt.Sprintf("%.2f", item.Price)},
					"quantity": &types.AttributeValueMemberN{Value: fmt.Sprintf("%d", item.Quantity)},
				},
			},
		})
	}

	_, err := r.client.TransactWriteItems(ctx, &dynamodb.TransactWriteItemsInput{
		TransactItems: transactItems,
	})
	return err
}
```

The transaction contains three types of operations:

1. **ConditionCheck** — verifies the user exists without modifying anything. If the user doesn't exist, the entire transaction fails.
2. **Put (order)** — creates the order item with all attributes including the sparse index key.
3. **Put (items)** — creates each order item.

All operations succeed or all fail. There's no state where you have an order without items or items without an order.

## Transaction limits

- Maximum **100 items** per transaction (including condition checks)
- Maximum **4 MB** total request size
- All items must be in the **same region**
- No two operations can target the **same item** within a transaction

## Idempotency

You can pass a `ClientRequestToken` to make a transaction idempotent:

```go
_, err := r.client.TransactWriteItems(ctx, &dynamodb.TransactWriteItemsInput{
	TransactItems:      transactItems,
	ClientRequestToken: aws.String("order-" + order.ID),
})
```

If the same token is sent within 10 minutes, DynamoDB returns success without re-executing the transaction. This protects against duplicate order placement due to retries.

## Test the transaction

Update `main.go`:

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

	// Place an order for alice using a transaction
	order := &Order{
		ID:         "ord-txn-001",
		UserID:     "alice",
		Status:     OrderStatusPending,
		AddressKey: "home",
		CreatedAt:  time.Now(),
		UpdatedAt:  time.Now(),
	}

	items := []OrderItem{
		{ItemID: "txn-item-001", Name: "Desk Lamp", Price: 45.99, Quantity: 1},
		{ItemID: "txn-item-002", Name: "Notebook", Price: 12.99, Quantity: 3},
	}

	fmt.Println("Placing order with transaction (user: alice)...")
	if err := repo.PlaceOrder(ctx, order, items); err != nil {
		log.Fatalf("Transaction failed: %v", err)
	}
	fmt.Println("Transaction succeeded! Order and items created atomically.")

	// Verify
	fmt.Printf("\nOrder %s:\n", order.ID)
	fetched, _ := repo.GetOrderByID(ctx, order.ID)
	fmt.Printf("  Status: %s  User: %s\n", fetched.Status, fetched.UserID)

	fmt.Println("\nOrder items:")
	fetchedItems, _ := repo.GetOrderItems(ctx, order.ID)
	for _, item := range fetchedItems {
		fmt.Printf("  %s - $%.2f x %d\n", item.Name, item.Price, item.Quantity)
	}

	// Try placing an order for a non-existent user (should fail)
	fmt.Println("\nPlacing order for non-existent user 'ghost'...")
	badOrder := &Order{
		ID:         "ord-txn-002",
		UserID:     "ghost",
		Status:     OrderStatusPending,
		AddressKey: "home",
		CreatedAt:  time.Now(),
		UpdatedAt:  time.Now(),
	}
	err = repo.PlaceOrder(ctx, badOrder, items)
	if err != nil {
		fmt.Printf("Transaction failed as expected: %v\n", err)
	}
}
```

Run:
```bash
go run .
```

Expected output:
```text
Placing order with transaction (user: alice)...
Transaction succeeded! Order and items created atomically.

Order ord-txn-001:
  Status: pending  User: alice

Order items:
  Desk Lamp - $45.99 x 1
  Notebook - $12.99 x 3

Placing order for non-existent user 'ghost'...
Transaction failed as expected: operation error DynamoDB: TransactWriteItems, ...
```

The first transaction succeeded because `alice` exists. The second failed because the condition check for user `ghost` returned false, and no items were written.
