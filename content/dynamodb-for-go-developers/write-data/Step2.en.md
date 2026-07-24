---
title: "BatchWriteItem - Bulk load data"
date: 2021-04-21T07:33:04-05:00
weight: 20
---

The `BatchWriteItem` operation lets you write up to 25 items in a single API call. This is more efficient than calling `PutItem` repeatedly when you need to load multiple items.

In this step, you implement the bulk-load path and load the sample data you use for the rest of the workshop.

## Your turn: implement BatchWriteItems

Find the `BatchWriteItems` stub in `repository.go` and complete it, following the `TODO(lab)` comment. Two patterns matter here:

1. **Chunking into batches of 25** - DynamoDB limits `BatchWriteItem` to 25 items per request, so process the input slice in chunks of at most 25.

2. **Retrying unprocessed items** - if DynamoDB cannot process all items (due to throughput limits), it returns them in `UnprocessedItems`. Retry until all items are written. In a production application, you would add exponential backoff to this retry loop.

::::expand{header="Expand this to see the solution for BatchWriteItems"}
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

		for len(output.UnprocessedItems) > 0 {
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
::::

## Your turn: implement SeedData

The project's demo harness (`demo.go`) already builds the sample dataset as typed model objects - three users, six orders in various states, and six order items - and exposes a `load-data` command that calls `SeedData`. You implement `SeedData`.

Find the `SeedData` stub in `repository.go` and complete it. It should marshal every user, order, and order item with the helpers you wrote in the previous step (`marshalUser`, `marshalOrder`, `marshalOrderItem`), collect them into one slice, and hand that slice to `BatchWriteItems`.

Because `SeedData` reuses your marshaling helpers, the sample dataset is built from the same models as `CreateUser`/`CreateOrder`/`CreateOrderItem` - no hand-written attribute maps anywhere.

::::expand{header="Expand this to see the solution for SeedData"}
```go
func (r *Repository) SeedData(ctx context.Context, users []User, orders []Order, orderItems []OrderItem) error {
	var items []map[string]types.AttributeValue

	for _, u := range users {
		item, err := marshalUser(u)
		if err != nil {
			return err
		}
		items = append(items, item)
	}
	for _, o := range orders {
		item, err := marshalOrder(o)
		if err != nil {
			return err
		}
		items = append(items, item)
	}
	for _, oi := range orderItems {
		item, err := marshalOrderItem(oi.OrderID, oi)
		if err != nil {
			return err
		}
		items = append(items, item)
	}

	return r.BatchWriteItems(ctx, items)
}
```
::::

::alert[Both stubs' `// TODO(lab):` comments describe the exact structure. If you get stuck, see the full reference solution as described in :link[Set up the Go project]{href="/dynamodb-for-go-developers/setup/step1"}.]{type="info"}

## Run the bulk load

```bash
go run . load-data
```

Expected output:
```text
Loading sample data...
Successfully loaded 15 items.
```

If you instead see a `TODO(lab): ... not implemented` error, the message names the function still missing an implementation - fill it in and re-run.

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

You should see a count that includes all users, orders, and order items you loaded.

You now have sample data in the table that supports all the query patterns you explore in the next modules.
