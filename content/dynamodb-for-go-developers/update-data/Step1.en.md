---
title: "UpdateItem with expressions"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

The `UpdateItem` operation modifies an existing item's attributes without replacing the entire item. This is more efficient than `PutItem` when you only need to change a few fields, because DynamoDB only writes the changed attributes.

## Update expressions

Update expressions define what attributes to change. The four clauses are:

| Clause | Purpose | Example |
|--------|---------|---------|
| `SET` | Add or change attributes | `SET email = :email` |
| `REMOVE` | Delete attributes | `REMOVE placed_id` |
| `ADD` | Increment numbers or add to sets | `ADD quantity :inc` |
| `DELETE` | Remove elements from a set | `DELETE tags :old_tags` |

You can combine multiple clauses in a single expression.

## Update a user profile

Add this function to `repository.go`:

```go
func (r *Repository) UpdateUser(ctx context.Context, username string, fullName, email string) error {
	_, err := r.client.UpdateItem(ctx, &dynamodb.UpdateItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", username)},
			"sk": &types.AttributeValueMemberS{Value: "PROFILE"},
		},
		UpdateExpression: aws.String("SET full_name = :name, email = :email"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":name":  &types.AttributeValueMemberS{Value: fullName},
			":email": &types.AttributeValueMemberS{Value: email},
		},
	})
	return err
}
```

The `SET` clause assigns new values to the `full_name` and `email` attributes. If the attributes don't exist, they are created. Other attributes on the item remain unchanged.

## Update order status

Updating an order's status is more complex because it involves multiple attributes and interacts with the sparse index:

```go
func (r *Repository) UpdateOrderStatus(ctx context.Context, orderID string, newStatus OrderStatus) error {
	// First, find the order to get the user ID
	order, err := r.GetOrderByID(ctx, orderID)
	if err != nil {
		return err
	}

	statusDate := fmt.Sprintf("%s#%s", newStatus, time.Now().Format("2006-01-02"))

	updateExpr := "SET #status = :status, #status_date = :status_date, #updated_at = :updated_at"
	exprNames := map[string]string{
		"#status":      "status",
		"#status_date": "status_date",
		"#updated_at":  "updated_at",
	}
	exprValues := map[string]types.AttributeValue{
		":status":      &types.AttributeValueMemberS{Value: string(newStatus)},
		":status_date": &types.AttributeValueMemberS{Value: statusDate},
		":updated_at":  &types.AttributeValueMemberS{Value: time.Now().Format(time.RFC3339)},
	}

	// Manage the sparse index attribute
	if newStatus == OrderStatusPending || newStatus == OrderStatusConfirmed {
		updateExpr += " SET #placed_id = :placed_id"
		exprNames["#placed_id"] = "placed_id"
		exprValues[":placed_id"] = &types.AttributeValueMemberS{Value: string(newStatus)}
	} else {
		updateExpr += " REMOVE #placed_id"
		exprNames["#placed_id"] = "placed_id"
	}

	_, err = r.client.UpdateItem(ctx, &dynamodb.UpdateItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", order.UserID)},
			"sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
		},
		UpdateExpression:          aws.String(updateExpr),
		ExpressionAttributeNames:  exprNames,
		ExpressionAttributeValues: exprValues,
	})
	return err
}
```

This function does three things:
1. **Updates the status** and `status_date` attributes
2. **Adds `placed_id`** if the new status is `pending` or `confirmed` (putting the order in the sparse index)
3. **Removes `placed_id`** if the new status is anything else (taking the order out of the sparse index)

Notice the use of `ExpressionAttributeNames` (the `#` prefixed names). These are required when attribute names conflict with DynamoDB reserved words — `status` is a reserved word.

## Test the update

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

	// Check pending orders before update
	fmt.Println("Pending orders before update:")
	pending, _ := repo.GetPendingOrders(ctx)
	for _, o := range pending {
		fmt.Printf("  %s (user: %s)\n", o.ID, o.UserID)
	}

	// Ship alice's first order
	fmt.Println("\nUpdating ord-aaa-001 to 'shipped'...")
	if err := repo.UpdateOrderStatus(ctx, "ord-aaa-001", OrderStatusShipped); err != nil {
		log.Fatalf("Failed to update order: %v", err)
	}
	fmt.Println("Order updated.")

	// Check pending orders after update
	fmt.Println("\nPending orders after update:")
	pending, _ = repo.GetPendingOrders(ctx)
	for _, o := range pending {
		fmt.Printf("  %s (user: %s)\n", o.ID, o.UserID)
	}
}
```

Run:
```bash
go run .
```

Expected output:
```text
Pending orders before update:
  ord-aaa-001 (user: alice)
  ord-bbb-001 (user: bob)
  ord-ccc-001 (user: carol)

Updating ord-aaa-001 to 'shipped'...
Order updated.

Pending orders after update:
  ord-bbb-001 (user: bob)
  ord-ccc-001 (user: carol)
```

Notice that `ord-aaa-001` disappeared from the pending orders query. By removing the `placed_id` attribute, the order was automatically removed from the sparse GSI.
