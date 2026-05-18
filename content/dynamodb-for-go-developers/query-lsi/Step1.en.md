---
title: "Query orders by status and date"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

The `status-date-index` LSI shares the same partition key (`pk`) as the base table but uses `status_date` as the sort key. The `status_date` attribute is a composite string in the format `<status>#<date>`, for example `pending#2024-01-10`.

## LSI vs GSI

Local Secondary Indexes differ from Global Secondary Indexes in several important ways:

| Feature | LSI | GSI |
|---------|-----|-----|
| Partition key | Same as base table | Can be different |
| Consistent reads | Yes (strongly consistent available) | No (always eventually consistent) |
| Created | Only at table creation time | Any time |
| Storage limit | 10 GB per partition key value | No limit |
| Write capacity | Shares with base table | Has its own |

Use an LSI when you want an alternate sort order within the same partition and need strongly consistent reads.

## Query a user's pending orders

Add this function to `repository.go`:

```go
func (r *Repository) GetUserOrdersByStatus(ctx context.Context, userID string, status OrderStatus) ([]*Order, error) {
	result, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:              aws.String(r.tableName),
		IndexName:              aws.String("status-date-index"),
		KeyConditionExpression: aws.String("pk = :pk AND begins_with(status_date, :status_prefix)"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":pk":            &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", userID)},
			":status_prefix": &types.AttributeValueMemberS{Value: string(status) + "#"},
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
				if len(skStr.Value) > 7 {
					order.ID = skStr.Value[7:] // Remove "#ORDER#" prefix
				}
			}
		}
		orders = append(orders, &order)
	}

	return orders, nil
}
```

The key condition uses `begins_with(status_date, :status_prefix)` with a value like `"pending#"`. Because the `status_date` attribute has the format `pending#2024-01-10`, this returns all pending orders for the user, sorted chronologically by date.

## Query with a date range

You can also query for orders within a specific status and date range using the `BETWEEN` operator:

```go
func (r *Repository) GetUserOrdersByStatusDateRange(ctx context.Context, userID string, status OrderStatus, startDate, endDate string) ([]*Order, error) {
	result, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:              aws.String(r.tableName),
		IndexName:              aws.String("status-date-index"),
		KeyConditionExpression: aws.String("pk = :pk AND status_date BETWEEN :start AND :end"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":pk":    &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", userID)},
			":start": &types.AttributeValueMemberS{Value: fmt.Sprintf("%s#%s", status, startDate)},
			":end":   &types.AttributeValueMemberS{Value: fmt.Sprintf("%s#%s", status, endDate)},
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

This is useful for queries like "get alice's pending orders from January 2024."

## Strongly consistent reads on LSI

Unlike GSIs, LSIs support strongly consistent reads. This is because the LSI data is stored in the same partition as the base table data:

```go
result, err := r.client.Query(ctx, &dynamodb.QueryInput{
	TableName:              aws.String(r.tableName),
	IndexName:              aws.String("status-date-index"),
	KeyConditionExpression: aws.String("pk = :pk AND begins_with(status_date, :prefix)"),
	ExpressionAttributeValues: map[string]types.AttributeValue{
		":pk":     &types.AttributeValueMemberS{Value: "#USER#alice"},
		":prefix": &types.AttributeValueMemberS{Value: "pending#"},
	},
	ConsistentRead: aws.Bool(true),
})
```

Use this when you need to see the most recent status change immediately after writing it.

## Test the LSI query

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

	fmt.Println("Querying alice's pending orders using status-date-index LSI...")
	pendingOrders, err := repo.GetUserOrdersByStatus(ctx, "alice", OrderStatusPending)
	if err != nil {
		log.Fatalf("Failed to query LSI: %v", err)
	}
	fmt.Printf("Found %d pending order(s) for alice:\n", len(pendingOrders))
	for _, o := range pendingOrders {
		fmt.Printf("  Order: %s  Created: %s\n", o.ID, o.CreatedAt.Format("2006-01-02"))
	}

	fmt.Println("\nQuerying alice's shipped orders using status-date-index LSI...")
	shippedOrders, err := repo.GetUserOrdersByStatus(ctx, "alice", OrderStatusShipped)
	if err != nil {
		log.Fatalf("Failed to query LSI: %v", err)
	}
	fmt.Printf("Found %d shipped order(s) for alice:\n", len(shippedOrders))
	for _, o := range shippedOrders {
		fmt.Printf("  Order: %s  Created: %s\n", o.ID, o.CreatedAt.Format("2006-01-02"))
	}
}
```

Run:
```bash
go run .
```

Expected output:
```text
Querying alice's pending orders using status-date-index LSI...
Found 1 pending order(s) for alice:
  Order: ord-aaa-001  Created: 2024-01-10

Querying alice's shipped orders using status-date-index LSI...
Found 1 shipped order(s) for alice:
  Order: ord-aaa-003  Created: 2024-01-08
```

The LSI allows you to efficiently filter and sort a user's orders by status without scanning all of their orders. Combined with the GSIs from the previous module, you now have complete coverage of all six access patterns defined in the data model.
