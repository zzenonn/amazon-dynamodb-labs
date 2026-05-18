---
title: "Inverted index - Find order by ID"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

The `inverted-index` GSI reverses the table's key schema: it uses `sk` as the partition key and `pk` as the sort key. This enables you to look up any item by its sort key value without knowing which partition it belongs to.

## The use case

In a REST API, you often need to look up an order by its ID: `GET /orders/ord-aaa-001`. On the base table, orders are stored under the user's partition key (`pk = #USER#alice`), so you would need to know the user first. The inverted index solves this by letting you query directly with the order's sort key.

## Query the inverted index

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

In this GSI, the order's sort key (`#ORDER#ord-aaa-001`) becomes the partition key. The query finds the order regardless of which user's partition it lives in on the base table.

## GSI eventual consistency

GSI queries are always eventually consistent. When you write an item to the base table, DynamoDB asynchronously replicates it to the GSI. In most cases, this happens within milliseconds, but there is no guarantee of immediate consistency. You cannot use `ConsistentRead: true` with a GSI query.

## Test the inverted index query

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

	fmt.Println("Looking up order by ID using inverted-index GSI...")
	order, err := repo.GetOrderByID(ctx, "ord-aaa-001")
	if err != nil {
		log.Fatalf("Failed to get order: %v", err)
	}

	fmt.Printf("Order: %s\n", order.ID)
	fmt.Printf("User: %s\n", order.UserID)
	fmt.Printf("Status: %s\n", order.Status)
}
```

Run:
```bash
go run .
```

Expected output:
```text
Looking up order by ID using inverted-index GSI...
Order: ord-aaa-001
User: alice
Status: pending
```

The inverted index found the order and also told you which user it belongs to (from the `pk` attribute, which is the sort key in the GSI).
