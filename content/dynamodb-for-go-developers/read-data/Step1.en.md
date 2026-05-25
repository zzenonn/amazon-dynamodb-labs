---
title: "GetItem and Query"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

The two primary read operations on the base table are `GetItem` (single item by key) and `Query` (multiple items sharing a partition key).

## GetItem — single item retrieval

The `GetItem` operation retrieves a single item by its full primary key (partition key + sort key). It is the most efficient read operation in DynamoDB — it goes directly to the partition that holds the item.

Add the following function to `repository.go`:

```go
func (r *Repository) GetUser(ctx context.Context, username string) (*User, error) {
	result, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", username)},
			"sk": &types.AttributeValueMemberS{Value: "PROFILE"},
		},
	})
	if err != nil {
		return nil, err
	}

	if result.Item == nil {
		return nil, fmt.Errorf("user not found: %s", username)
	}

	var user User
	if err := attributevalue.UnmarshalMap(result.Item, &user); err != nil {
		return nil, err
	}
	user.Username = username
	return &user, nil
}
```

`GetItem` requires the complete primary key. Because you know both the partition key (`#USER#alice`) and the sort key (`PROFILE`) for a user, you can fetch the exact item directly.

The `attributevalue.UnmarshalMap` function converts the DynamoDB attribute map back into the Go struct, using the `dynamodbav` tags to map attribute names to struct fields.

### Consistent reads

By default, `GetItem` uses eventually consistent reads. If you need to read the most recent write immediately, you can request a strongly consistent read:

```go
result, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
	TableName:      aws.String(r.tableName),
	Key:            key,
	ConsistentRead: aws.Bool(true),
})
```

Strongly consistent reads cost twice as many Read Request Units (RRUs) as eventually consistent reads. Use them only when your application requires it.

### Projection expressions

If you only need certain attributes, use a projection expression to reduce the data transferred:

```go
result, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
	TableName: aws.String(r.tableName),
	Key: map[string]types.AttributeValue{
		"pk": &types.AttributeValueMemberS{Value: "#USER#alice"},
		"sk": &types.AttributeValueMemberS{Value: "PROFILE"},
	},
	ProjectionExpression: aws.String("full_name, email"),
})
```

This returns only the `full_name` and `email` attributes. The item still consumes the same RRUs (DynamoDB reads the full item internally), but you reduce the payload size over the network.

## Query — retrieve collections

The `Query` operation retrieves multiple items that share the same partition key. You can filter on the sort key using a key condition expression. This is how you retrieve hierarchically related data — for example, all orders belonging to a user.

### Get all orders for a user

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

### Get all items in an order

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

### Sort order and limits

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

### Pagination

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

## Test GetItem and Query

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

	// GetItem: fetch a user
	user, err := repo.GetUser(ctx, "alice")
	if err != nil {
		log.Fatalf("Failed to get user: %v", err)
	}
	fmt.Printf("User: %s (%s)\n", user.Username, user.Email)

	// Query: get alice's orders
	fmt.Println("\nOrders for alice:")
	orders, err := repo.GetOrdersByUserID(ctx, "alice")
	if err != nil {
		log.Fatalf("Failed to get orders: %v", err)
	}
	for _, o := range orders {
		fmt.Printf("  Order: %s  Status: %s\n", o.ID, o.Status)
	}

	// Query: get items for the first order
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
User: alice (alice@example.com)

Orders for alice:
  Order: ord-aaa-001  Status: pending
  Order: ord-aaa-002  Status: confirmed
  Order: ord-aaa-003  Status: shipped

Items in order ord-aaa-001:
  Laptop - $1299.99 x 1
  Mouse - $29.99 x 2
```
