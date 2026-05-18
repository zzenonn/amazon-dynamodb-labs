---
title: "BatchWriteItem - Bulk load data"
date: 2021-04-21T07:33:04-05:00
weight: 20
---

The `BatchWriteItem` operation lets you write up to 25 items in a single API call. This is more efficient than calling `PutItem` repeatedly when you need to load multiple items.

In this step, you load sample data that you use for the rest of the workshop.

## Write the batch load function

Add the following function to `repository.go`:

```go
func (r *Repository) BatchWriteItems(ctx context.Context, items []map[string]types.AttributeValue) error {
	for i := 0; i < len(items); i += 25 {
		end := i + 25
		if end > len(items) {
			end = len(items)
		}

		batch := items[i:end]
		var writeRequests []types.WriteRequest
		for _, item := range batch {
			writeRequests = append(writeRequests, types.WriteRequest{
				PutRequest: &types.PutRequest{Item: item},
			})
		}

		output, err := r.client.BatchWriteItem(ctx, &dynamodb.BatchWriteItemInput{
			RequestItems: map[string][]types.WriteRequest{
				r.tableName: writeRequests,
			},
		})
		if err != nil {
			return err
		}

		// Handle unprocessed items
		for len(output.UnprocessedItems) > 0 {
			fmt.Printf("  Retrying %d unprocessed items...\n", len(output.UnprocessedItems[r.tableName]))
			output, err = r.client.BatchWriteItem(ctx, &dynamodb.BatchWriteItemInput{
				RequestItems: output.UnprocessedItems,
			})
			if err != nil {
				return err
			}
		}
	}
	return nil
}
```

There are two important patterns here:

1. **Chunking into batches of 25** — DynamoDB limits `BatchWriteItem` to 25 items per request. The function processes the slice in chunks.

2. **Retrying unprocessed items** — If DynamoDB cannot process all items (due to throughput limits), it returns them in `UnprocessedItems`. The code retries until all items are written. In a production application, you would add exponential backoff to this retry loop.

## Load sample data

Update `main.go` to load sample data for the remaining exercises. This creates multiple users, orders in various states, and order items:

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

	fmt.Println("Loading sample data...")

	items := buildSampleData()
	if err := repo.BatchWriteItems(ctx, items); err != nil {
		log.Fatalf("Failed to batch write: %v", err)
	}

	fmt.Printf("Successfully loaded %d items.\n", len(items))
}

func buildSampleData() []map[string]types.AttributeValue {
	var items []map[string]types.AttributeValue

	// Users
	users := []struct {
		username string
		fullName string
		email    string
	}{
		{"alice", "Alice Smith", "alice@example.com"},
		{"bob", "Bob Johnson", "bob@example.com"},
		{"carol", "Carol Williams", "carol@example.com"},
	}

	for _, u := range users {
		items = append(items, map[string]types.AttributeValue{
			"pk":        &types.AttributeValueMemberS{Value: "#USER#" + u.username},
			"sk":        &types.AttributeValueMemberS{Value: "PROFILE"},
			"full_name": &types.AttributeValueMemberS{Value: u.fullName},
			"email":     &types.AttributeValueMemberS{Value: u.email},
		})
	}

	// Orders for alice
	orders := []struct {
		userID    string
		orderID   string
		status    string
		date      string
		placedID  string
	}{
		{"alice", "ord-aaa-001", "pending", "2024-01-10", "pending"},
		{"alice", "ord-aaa-002", "confirmed", "2024-01-12", "confirmed"},
		{"alice", "ord-aaa-003", "shipped", "2024-01-08", ""},
		{"bob", "ord-bbb-001", "pending", "2024-01-14", "pending"},
		{"bob", "ord-bbb-002", "delivered", "2024-01-05", ""},
		{"carol", "ord-ccc-001", "pending", "2024-01-15", "pending"},
	}

	for _, o := range orders {
		item := map[string]types.AttributeValue{
			"pk":          &types.AttributeValueMemberS{Value: "#USER#" + o.userID},
			"sk":          &types.AttributeValueMemberS{Value: "#ORDER#" + o.orderID},
			"order_id":    &types.AttributeValueMemberS{Value: o.orderID},
			"user_id":     &types.AttributeValueMemberS{Value: o.userID},
			"status":      &types.AttributeValueMemberS{Value: o.status},
			"status_date": &types.AttributeValueMemberS{Value: o.status + "#" + o.date},
			"address_key": &types.AttributeValueMemberS{Value: "home"},
			"created_at":  &types.AttributeValueMemberS{Value: o.date + "T10:00:00Z"},
		}
		if o.placedID != "" {
			item["placed_id"] = &types.AttributeValueMemberS{Value: o.placedID}
		}
		items = append(items, item)
	}

	// Order items
	orderItems := []struct {
		orderID string
		itemID  string
		name    string
		price   string
		qty     string
	}{
		{"ord-aaa-001", "item-001", "Laptop", "1299.99", "1"},
		{"ord-aaa-001", "item-002", "Mouse", "29.99", "2"},
		{"ord-aaa-002", "item-003", "Keyboard", "79.99", "1"},
		{"ord-bbb-001", "item-004", "Monitor", "499.99", "1"},
		{"ord-bbb-001", "item-005", "USB Cable", "9.99", "3"},
		{"ord-ccc-001", "item-006", "Headphones", "199.99", "1"},
	}

	for _, oi := range orderItems {
		items = append(items, map[string]types.AttributeValue{
			"pk":          &types.AttributeValueMemberS{Value: "#ORDER#" + oi.orderID},
			"sk":          &types.AttributeValueMemberS{Value: "#ITEM#" + oi.itemID},
			"order_id":    &types.AttributeValueMemberS{Value: oi.orderID},
			"item_id":     &types.AttributeValueMemberS{Value: oi.itemID},
			"name":        &types.AttributeValueMemberS{Value: oi.name},
			"price":       &types.AttributeValueMemberN{Value: oi.price},
			"quantity":    &types.AttributeValueMemberN{Value: oi.qty},
		})
	}

	return items
}
```

You need to add the `types` import: `"github.com/aws/aws-sdk-go-v2/service/dynamodb/types"`.

## Run the bulk load

```bash
go run .
```

Expected output:
```text
Loading sample data...
Successfully loaded 15 items.
```

## Verify the data

Count the items in the table:

```bash
aws dynamodb scan --table-name simple-inventory --select COUNT
```

Expected output:
```json
{
    "Count": 15,
    "ScannedCount": 15,
    "ConsumedCapacity": null
}
```

You should see a count that includes all users, orders, and order items you loaded. The exact count depends on whether you still have the data from the previous step (the `john` user and items). If you ran the previous step, the total will be higher.

You now have sample data in the table that supports all the query patterns you explore in the next modules.
