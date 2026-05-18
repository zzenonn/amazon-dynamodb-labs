---
title: "TransactGetItems"
date: 2021-04-21T07:33:04-05:00
weight: 20
---

`TransactGetItems` reads up to 100 items atomically, returning a consistent snapshot across all items. This guarantees you see all items as they existed at the same point in time.

## Use case: fetch a complete order

When displaying an order to a user, you want to show the order details and all items in a consistent state. A regular `GetItem` + `Query` sequence could return results from different points in time if a write happens between the two calls.

## Write the GetOrderSnapshot function

Add this function to `repository.go`:

```go
func (r *Repository) GetOrderSnapshot(ctx context.Context, userID, orderID string) (*Order, []OrderItem, error) {
	// First, get the item IDs (we need to know them for TransactGetItems)
	orderItems, err := r.GetOrderItems(ctx, orderID)
	if err != nil {
		return nil, nil, err
	}

	var transactItems []types.TransactGetItem

	// Get the order
	transactItems = append(transactItems, types.TransactGetItem{
		Get: &types.Get{
			TableName: aws.String(r.tableName),
			Key: map[string]types.AttributeValue{
				"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", userID)},
				"sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
			},
		},
	})

	// Get each order item
	for _, item := range orderItems {
		transactItems = append(transactItems, types.TransactGetItem{
			Get: &types.Get{
				TableName: aws.String(r.tableName),
				Key: map[string]types.AttributeValue{
					"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
					"sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ITEM#%s", item.ItemID)},
				},
			},
		})
	}

	result, err := r.client.TransactGetItems(ctx, &dynamodb.TransactGetItemsInput{
		TransactItems: transactItems,
	})
	if err != nil {
		return nil, nil, err
	}

	// Parse the order (first response)
	var order Order
	if len(result.Responses) > 0 && result.Responses[0].Item != nil {
		if err := attributevalue.UnmarshalMap(result.Responses[0].Item, &order); err != nil {
			return nil, nil, err
		}
		order.UserID = userID
		order.ID = orderID
	}

	// Parse the items (remaining responses)
	var items []OrderItem
	for _, resp := range result.Responses[1:] {
		if resp.Item != nil {
			var item OrderItem
			if err := attributevalue.UnmarshalMap(resp.Item, &item); err != nil {
				continue
			}
			items = append(items, item)
		}
	}

	return &order, items, nil
}
```

`TransactGetItems` returns the results in the same order as the request. The first response is the order, and the remaining responses are the items.

## When to use TransactGetItems vs Query

For this specific example, a single `Query` on `pk = #ORDER#<id>` would be simpler and more efficient for retrieving items. `TransactGetItems` is most valuable when you need to read items from **different partitions** atomically:

- Reading a user profile AND an order from different partitions
- Reading multiple orders from different users simultaneously
- Any time you need a guaranteed point-in-time snapshot across partitions

## Test the transactional read

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

	fmt.Println("Fetching order snapshot with TransactGetItems...")
	order, items, err := repo.GetOrderSnapshot(ctx, "alice", "ord-txn-001")
	if err != nil {
		log.Fatalf("Transaction read failed: %v", err)
	}

	fmt.Printf("\nOrder: %s\n", order.ID)
	fmt.Printf("Status: %s\n", order.Status)
	fmt.Printf("Items (%d):\n", len(items))
	for _, item := range items {
		fmt.Printf("  %s - $%.2f x %d\n", item.Name, item.Price, item.Quantity)
	}
}
```

Run:
```bash
go run .
```

Expected output:
```text
Fetching order snapshot with TransactGetItems...

Order: ord-txn-001
Status: pending
Items (2):
  Desk Lamp - $45.99 x 1
  Notebook - $12.99 x 3
```

All data was read at a consistent point in time. In the next module, you learn about scanning the entire table.
