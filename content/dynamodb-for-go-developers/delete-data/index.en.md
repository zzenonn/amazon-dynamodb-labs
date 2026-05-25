---
title: "6. Delete Data"
date: 2021-04-21T07:33:04-05:00
weight: 60
description: "Delete items with conditions and handle related data cleanup."
---

The `DeleteItem` operation removes a single item from the table by its primary key. DynamoDB does not have foreign keys or cascade deletes, so cleaning up related items is your responsibility.

## Delete a single item

Add this function to `repository.go`:

```go
func (r *Repository) DeleteOrderItem(ctx context.Context, orderID, itemID string) error {
	_, err := r.client.DeleteItem(ctx, &dynamodb.DeleteItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
			"sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ITEM#%s", itemID)},
		},
	})
	return err
}
```

`DeleteItem` is idempotent — deleting an item that doesn't exist does not produce an error.

## Conditional delete

You can protect deletes with conditions. For example, only allow cancelling an order that is still pending:

```go
func (r *Repository) CancelOrder(ctx context.Context, orderID string) error {
	order, err := r.GetOrderByID(ctx, orderID)
	if err != nil {
		return err
	}

	_, err = r.client.DeleteItem(ctx, &dynamodb.DeleteItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", order.UserID)},
			"sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
		},
		ConditionExpression: aws.String("#status = :expected"),
		ExpressionAttributeNames: map[string]string{
			"#status": "status",
		},
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":expected": &types.AttributeValueMemberS{Value: string(OrderStatusPending)},
		},
		ReturnValues: types.ReturnValueAllOld,
	})
	return err
}
```

`ReturnValues: AllOld` returns the attributes of the item as it was before deletion. This is useful for logging or confirmation purposes.

## Delete an order and its items

Because DynamoDB has no cascade delete, you must explicitly query for related items and delete them. Here is a function that deletes an order and all its items:

```go
func (r *Repository) DeleteOrderWithItems(ctx context.Context, orderID string) error {
	// First, delete all order items
	items, err := r.GetOrderItems(ctx, orderID)
	if err != nil {
		return err
	}

	for _, item := range items {
		if err := r.DeleteOrderItem(ctx, orderID, item.ItemID); err != nil {
			return fmt.Errorf("failed to delete item %s: %w", item.ItemID, err)
		}
	}

	// Then find and delete the order itself
	order, err := r.GetOrderByID(ctx, orderID)
	if err != nil {
		return err
	}

	_, err = r.client.DeleteItem(ctx, &dynamodb.DeleteItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", order.UserID)},
			"sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
		},
	})
	return err
}
```

This approach has a weakness: it is not atomic. If the process crashes between deleting items and deleting the order, you are left in an inconsistent state. In the next module, you learn how transactions solve this problem.

## Test deletes

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

	// Show items before delete
	fmt.Println("Items in ord-bbb-001 before delete:")
	items, _ := repo.GetOrderItems(ctx, "ord-bbb-001")
	for _, item := range items {
		fmt.Printf("  %s - %s\n", item.ItemID, item.Name)
	}

	// Delete one item
	fmt.Println("\nDeleting item-004 from ord-bbb-001...")
	if err := repo.DeleteOrderItem(ctx, "ord-bbb-001", "item-004"); err != nil {
		log.Fatalf("Failed to delete: %v", err)
	}
	fmt.Println("Item deleted.")

	// Show items after delete
	fmt.Println("\nItems in ord-bbb-001 after delete:")
	items, _ = repo.GetOrderItems(ctx, "ord-bbb-001")
	for _, item := range items {
		fmt.Printf("  %s - %s\n", item.ItemID, item.Name)
	}
}
```

Run:
```bash
go run .
```

Expected output:
```text
Items in ord-bbb-001 before delete:
  item-004 - Monitor
  item-005 - USB Cable

Deleting item-004 from ord-bbb-001...
Item deleted.

Items in ord-bbb-001 after delete:
  item-005 - USB Cable
```

In the next module, you learn how to use transactions to perform multiple operations atomically.
