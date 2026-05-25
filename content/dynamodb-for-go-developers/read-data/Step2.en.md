---
title: "Query Secondary Indexes"
date: 2021-04-21T07:33:04-05:00
weight: 20
---

Secondary indexes let you query data using different key patterns than the base table. In this step, you query both Global Secondary Indexes (GSIs) and the Local Secondary Index (LSI).

## Inverted index GSI — find order by ID

The `inverted-index` GSI reverses the table's key schema: it uses `sk` as the partition key and `pk` as the sort key. This enables you to look up any item by its sort key value without knowing which partition it belongs to.

In a REST API, you often need to look up an order by its ID: `GET /orders/ord-aaa-001`. On the base table, orders are stored under the user's partition key (`pk = #USER#alice`), so you would need to know the user first. The inverted index solves this.

Add this function to `repository.go`:

```go
func (r *Repository) GetOrderByID(ctx context.Context, orderID string) (*Order, error) {
	result, err := r.client.Query(ctx, &dynamodb.QueryInput{
		TableName:              aws.String(r.tableName),
		IndexName:              aws.String("inverted-index"),
		KeyConditionExpression: aws.String("sk = :sk"),
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
		},
		Limit: aws.Int32(1),
	})
	if err != nil {
		return nil, err
	}

	if len(result.Items) == 0 {
		return nil, fmt.Errorf("order not found: %s", orderID)
	}

	var order Order
	if err := attributevalue.UnmarshalMap(result.Items[0], &order); err != nil {
		return nil, err
	}

	// Extract username from pk attribute
	if pkValue, ok := result.Items[0]["pk"]; ok {
		if pkStr, ok := pkValue.(*types.AttributeValueMemberS); ok {
			order.UserID = pkStr.Value[6:] // Remove "#USER#" prefix
		}
	}
	order.ID = orderID
	return &order, nil
}
```

The key difference from a base table query is the `IndexName` parameter. Setting it to `"inverted-index"` tells DynamoDB to query the GSI instead of the base table.

::alert[GSI queries are always eventually consistent. You cannot use `ConsistentRead: true` with a GSI query.]{type="info"}

## Sparse index GSI — get pending orders

The `placed-index` GSI is a sparse index. Only items that have the `placed_id` attribute appear in this index. Orders have `placed_id` set only when their status is `pending` or `confirmed`. Once an order is shipped or delivered, the attribute is removed, and the order disappears from the index.

Sparse indexes are useful when you need to query a subset of items efficiently. Instead of scanning the entire table and filtering, you query an index that contains only the items you care about.

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

### GSI write costs

Every time you write an item to the base table, DynamoDB also writes it to each GSI where the item's attributes match the GSI key schema. This means:
- Adding `placed_id` to an order causes a write to the `placed-index` GSI
- Removing `placed_id` causes a delete from the GSI
- The `inverted-index` GSI receives a write for every item (since every item has `pk` and `sk`)

Sparse indexes are cost-efficient because they only contain the subset of items that have the index key attribute.

## Local Secondary Index — query by status and date

The `status-date-index` LSI shares the same partition key (`pk`) as the base table but uses `status_date` as the sort key. The `status_date` attribute is a composite string in the format `<status>#<date>`, for example `pending#2024-01-10`.

| Feature | LSI | GSI |
|---------|-----|-----|
| Partition key | Same as base table | Can be different |
| Consistent reads | Yes (strongly consistent available) | No (always eventually consistent) |
| Created | Only at table creation time | Any time |
| Storage limit | 10 GB per partition key value | No limit |

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
					order.ID = skStr.Value[7:]
				}
			}
		}
		orders = append(orders, &order)
	}

	return orders, nil
}
```

The key condition uses `begins_with(status_date, :status_prefix)` with a value like `"pending#"`. Because the `status_date` attribute has the format `pending#2024-01-10`, this returns all pending orders for the user, sorted chronologically by date.

Unlike GSIs, LSIs support strongly consistent reads because the LSI data is stored in the same partition as the base table data.

## Test all secondary index queries

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

	// GSI: inverted-index — find order by ID
	fmt.Println("Looking up order by ID (inverted-index GSI):")
	order, err := repo.GetOrderByID(ctx, "ord-aaa-001")
	if err != nil {
		log.Fatalf("Failed: %v", err)
	}
	fmt.Printf("  Order: %s  User: %s  Status: %s\n", order.ID, order.UserID, order.Status)

	// GSI: placed-index — get all pending orders
	fmt.Println("\nAll pending orders (placed-index GSI):")
	pending, err := repo.GetPendingOrders(ctx)
	if err != nil {
		log.Fatalf("Failed: %v", err)
	}
	for _, o := range pending {
		fmt.Printf("  Order: %s  User: %s\n", o.ID, o.UserID)
	}

	// LSI: status-date-index — alice's pending orders
	fmt.Println("\nAlice's pending orders (status-date-index LSI):")
	alicePending, err := repo.GetUserOrdersByStatus(ctx, "alice", OrderStatusPending)
	if err != nil {
		log.Fatalf("Failed: %v", err)
	}
	for _, o := range alicePending {
		fmt.Printf("  Order: %s\n", o.ID)
	}
}
```

Run:
```bash
go run .
```

Expected output:
```text
Looking up order by ID (inverted-index GSI):
  Order: ord-aaa-001  User: alice  Status: pending

All pending orders (placed-index GSI):
  Order: ord-aaa-001  User: alice
  Order: ord-bbb-001  User: bob
  Order: ord-ccc-001  User: carol

Alice's pending orders (status-date-index LSI):
  Order: ord-aaa-001
```
