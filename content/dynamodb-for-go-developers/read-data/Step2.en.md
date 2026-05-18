---
title: "Query - Retrieve collections"
date: 2021-04-21T07:33:04-05:00
weight: 20
---

The `Query` operation retrieves multiple items that share the same partition key. You can filter on the sort key using a key condition expression. This is how you retrieve hierarchically related data — for example, all orders belonging to a user.

## Get all orders for a user

Add the following function to `repository.go`:

```go
func (r *Repository) GetOrdersByUserID(ctx context.Context, userID string) ([]*Order, error) {
	result, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:              aws.String(r.tableName),
		KeyConditionExpression: aws.String("pk = :pk AND begins_with(sk, :sk_prefix)"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":pk":        &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", userID)},
			":sk_prefix": &types.AttributeValueMemberS{Value: "#ORDER#"},
		},
	})
	if err != nil {
		return nil, err
	}

	var orders []*Order
	for _, item := range result.Items {
		var order Order
		if err := attributevalue.UnmarshalMap(item, &order); err != nil {
			continue
		}
		order.UserID = userID
		if skValue, ok := item["sk"]; ok {
			if skStr, ok := skValue.(*types.AttributeValueMemberS); ok {
				order.ID = skStr.Value[7:] // Remove "#ORDER#" prefix
			}
		}
		orders = append(orders, &order)
	}

	return orders, nil
}
```

The `KeyConditionExpression` has two parts:
- `pk = :pk` — matches the exact partition key for this user
- `begins_with(sk, :sk_prefix)` — matches only items whose sort key starts with `#ORDER#`

This excludes the user's `PROFILE` item (which has `sk = "PROFILE"`) and returns only order items within that partition.

## Get all items in an order

```go
func (r *Repository) GetOrderItems(ctx context.Context, orderID string) ([]OrderItem, error) {
	result, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:              aws.String(r.tableName),
		KeyConditionExpression: aws.String("pk = :pk AND begins_with(sk, :sk_prefix)"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":pk":        &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
			":sk_prefix": &types.AttributeValueMemberS{Value: "#ITEM#"},
		},
	})
	if err != nil {
		return nil, err
	}

	var items []OrderItem
	for _, item := range result.Items {
		var orderItem OrderItem
		if err := attributevalue.UnmarshalMap(item, &orderItem); err != nil {
			continue
		}
		items = append(items, orderItem)
	}

	return items, nil
}
```

The same pattern applies. Order items have `pk = #ORDER#<id>` and `sk` starting with `#ITEM#`. A single query retrieves all items belonging to an order.

## Sort order and limits

By default, Query returns items in ascending sort key order. You can reverse this with `ScanIndexForward`:

```go
result, err := r.client.Query(ctx, &dynamodb.QueryInput{
	TableName:              aws.String(r.tableName),
	KeyConditionExpression: aws.String("pk = :pk AND begins_with(sk, :sk_prefix)"),
	ExpressionAttributeValues: map[string]types.AttributeValue{
		":pk":        &types.AttributeValueMemberS{Value: "#USER#alice"},
		":sk_prefix": &types.AttributeValueMemberS{Value: "#ORDER#"},
	},
	ScanIndexForward: aws.Bool(false), // Descending order
	Limit:           aws.Int32(5),     // Return at most 5 items
})
```

Setting `ScanIndexForward` to `false` returns the most recent orders first (assuming sort key values are ordered). `Limit` caps the number of items returned in a single response.

## Pagination

When a query result exceeds 1 MB or you set a `Limit`, DynamoDB returns a `LastEvaluatedKey`. You use this as `ExclusiveStartKey` in the next request to continue reading:

```go
func (r *Repository) GetAllOrdersPaginated(ctx context.Context, userID string, pageSize int32) ([]*Order, error) {
	var allOrders []*Order
	var lastKey map[string]types.AttributeValue

	for {
		input := &dynamodb.QueryInput{
			TableName:              aws.String(r.tableName),
			KeyConditionExpression: aws.String("pk = :pk AND begins_with(sk, :sk_prefix)"),
			ExpressionAttributeValues: map[string]types.AttributeValue{
				":pk":        &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", userID)},
				":sk_prefix": &types.AttributeValueMemberS{Value: "#ORDER#"},
			},
			Limit: aws.Int32(pageSize),
		}
		if lastKey != nil {
			input.ExclusiveStartKey = lastKey
		}

		result, err := r.client.Query(ctx, input)
		if err != nil {
			return nil, err
		}

		for _, item := range result.Items {
			var order Order
			if err := attributevalue.UnmarshalMap(item, &order); err != nil {
				continue
			}
			allOrders = append(allOrders, &order)
		}

		lastKey = result.LastEvaluatedKey
		if lastKey == nil {
			break
		}
	}

	return allOrders, nil
}
```

This loop continues until `LastEvaluatedKey` is nil, meaning all matching items have been read.

## Test the Query functions

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

	// Get alice's orders
	fmt.Println("Orders for alice:")
	orders, err := repo.GetOrdersByUserID(ctx, "alice")
	if err != nil {
		log.Fatalf("Failed to get orders: %v", err)
	}
	for _, o := range orders {
		fmt.Printf("  Order: %s  Status: %s\n", o.ID, o.Status)
	}

	// Get items for the first order
	if len(orders) > 0 {
		fmt.Printf("\nItems in order %s:\n", orders[0].ID)
		items, err := repo.GetOrderItems(ctx, orders[0].ID)
		if err != nil {
			log.Fatalf("Failed to get items: %v", err)
		}
		for _, item := range items {
			fmt.Printf("  %s - $%.2f x %d\n", item.Name, item.Price, item.Quantity)
		}
	}
}
```

Run:
```bash
go run .
```

Expected output:
```text
Orders for alice:
  Order: ord-aaa-001  Status: pending
  Order: ord-aaa-002  Status: confirmed
  Order: ord-aaa-003  Status: shipped

Items in order ord-aaa-001:
  Laptop - $1299.99 x 1
  Mouse - $29.99 x 2
```

You have now retrieved related data using the primary table's key structure. In the next module, you use Global Secondary Indexes to query across partitions.
