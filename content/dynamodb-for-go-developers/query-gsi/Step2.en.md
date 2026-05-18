---
title: "Sparse index - Get pending orders"
date: 2021-04-21T07:33:04-05:00
weight: 20
---

The `placed-index` GSI is a sparse index. Only items that have the `placed_id` attribute appear in this index. In your data model, orders have `placed_id` set only when their status is `pending` or `confirmed`. Once an order is shipped or delivered, the attribute is removed, and the order disappears from the index.

## Why sparse indexes?

Sparse indexes are useful when you need to query a subset of items efficiently. Instead of scanning the entire table and filtering, you query an index that contains only the items you care about. This is faster and consumes fewer Read Request Units.

Common use cases:
- Active/pending orders (this workshop)
- Items flagged for review
- Users with incomplete profiles
- Resources that need attention

## Query pending orders

Add this function to `repository.go`:

```go
func (r *Repository) GetPendingOrders(ctx context.Context) ([]*Order, error) {
	result, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:              aws.String(r.tableName),
		IndexName:              aws.String("placed-index"),
		KeyConditionExpression: aws.String("placed_id = :placed_id"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":placed_id": &types.AttributeValueMemberS{Value: string(OrderStatusPending)},
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
		orders = append(orders, &order)
	}

	return orders, nil
}
```

This query hits the `placed-index` GSI with `placed_id = "pending"`. It returns all orders across all users that are currently pending. Because this is a sparse index, the query only reads items that are actually pending — not every order in the system.

## Query confirmed orders

You can query for confirmed orders the same way:

```go
func (r *Repository) GetConfirmedOrders(ctx context.Context) ([]*Order, error) {
	result, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:              aws.String(r.tableName),
		IndexName:              aws.String("placed-index"),
		KeyConditionExpression: aws.String("placed_id = :placed_id"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":placed_id": &types.AttributeValueMemberS{Value: string(OrderStatusConfirmed)},
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
		orders = append(orders, &order)
	}

	return orders, nil
}
```

## GSI write costs

Every time you write an item to the base table, DynamoDB also writes it to each GSI where the item's attributes match the GSI key schema. This means:
- Adding `placed_id` to an order causes a write to the `placed-index` GSI
- Removing `placed_id` causes a delete from the GSI
- The `inverted-index` GSI receives a write for every item (since every item has `pk` and `sk`)

Keep this write amplification in mind when designing indexes. Sparse indexes are cost-efficient because they only contain the subset of items that have the index key attribute.

## Test the sparse index query

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

	fmt.Println("Querying placed-index for all pending orders...")
	orders, err := repo.GetPendingOrders(ctx)
	if err != nil {
		log.Fatalf("Failed to get pending orders: %v", err)
	}

	fmt.Printf("Found %d pending orders:\n", len(orders))
	for _, o := range orders {
		fmt.Printf("  Order: %s  User: %s\n", o.ID, o.UserID)
	}
}
```

Run:
```bash
go run .
```

Expected output:
```text
Querying placed-index for all pending orders...
Found 3 pending orders:
  Order: ord-aaa-001  User: alice
  Order: ord-bbb-001  User: bob
  Order: ord-ccc-001  User: carol
```

The sparse index returned only the pending orders across all users. Orders with status `shipped` or `delivered` do not appear because they lack the `placed_id` attribute.

In the next module, you query the Local Secondary Index to sort a user's orders by status and date.
