---
title: "Scan and parallel scan"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

The `Scan` operation reads every item in the table (or index). Unlike `Query`, which targets a specific partition, `Scan` examines every item and returns those that match an optional filter expression.

## When to use Scan

Scan is appropriate for:
- **Analytics** — aggregating data across the entire table
- **Migrations** — reading all items to transform or move them
- **Small tables** — when the table has few items
- **Administrative tools** — export, backup, or reporting

Scan is **not** appropriate for normal application queries. It reads every item in the table, consuming capacity proportional to the entire table size, even if a filter discards most items.

## Basic scan with pagination

The Go SDK provides a `ScanPaginator` that handles the pagination loop automatically:

Add this function to `repository.go`:

```go
func (r *Repository) ScanAllItems(ctx context.Context) ([]map[string]types.AttributeValue, error) {
	var allItems []map[string]types.AttributeValue

	paginator := dynamodb.NewScanPaginator(r.client, &dynamodb.ScanInput{
		TableName: aws.String(r.tableName),
	})

	pageNum := 0
	for paginator.HasMorePages() {
		page, err := paginator.NextPage(ctx)
		if err != nil {
			return nil, err
		}
		pageNum++
		fmt.Printf("  Page %d: %d items\n", pageNum, len(page.Items))
		allItems = append(allItems, page.Items...)
	}

	return allItems, nil
}
```

The paginator automatically manages `LastEvaluatedKey` / `ExclusiveStartKey` across pages. Each page contains up to 1 MB of data.

## Scan with a filter expression

Filter expressions reduce the items returned to the caller, but DynamoDB still reads and charges for all items scanned:

```go
func (r *Repository) ScanOrdersByStatus(ctx context.Context, status OrderStatus) ([]map[string]types.AttributeValue, error) {
	var allItems []map[string]types.AttributeValue

	paginator := dynamodb.NewScanPaginator(r.client, &dynamodb.ScanInput{
		TableName:        aws.String(r.tableName),
		FilterExpression: aws.String("#status = :status AND begins_with(sk, :order_prefix)"),
		ExpressionAttributeNames: map[string]string{
			"#status": "status",
		},
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":status":       &types.AttributeValueMemberS{Value: string(status)},
			":order_prefix": &types.AttributeValueMemberS{Value: "#ORDER#"},
		},
	})

	for paginator.HasMorePages() {
		page, err := paginator.NextPage(ctx)
		if err != nil {
			return nil, err
		}
		allItems = append(allItems, page.Items...)
	}

	return allItems, nil
}
```

::alert[Filter expressions do NOT reduce the amount of data read from disk or the capacity consumed. They only reduce the data sent back to the client. If you find yourself filtering heavily, consider creating an index instead.]{type="warning"}

## Parallel scan

For large tables, you can split the scan across multiple goroutines to increase throughput. Each segment reads a different portion of the table:

```go
func (r *Repository) ParallelScan(ctx context.Context, totalSegments int) ([]map[string]types.AttributeValue, error) {
	type segmentResult struct {
		items []map[string]types.AttributeValue
		err   error
	}

	results := make(chan segmentResult, totalSegments)

	for segment := 0; segment < totalSegments; segment++ {
		go func(seg int) {
			var items []map[string]types.AttributeValue

			paginator := dynamodb.NewScanPaginator(r.client, &dynamodb.ScanInput{
				TableName:     aws.String(r.tableName),
				Segment:       aws.Int32(int32(seg)),
				TotalSegments: aws.Int32(int32(totalSegments)),
			})

			for paginator.HasMorePages() {
				page, err := paginator.NextPage(ctx)
				if err != nil {
					results <- segmentResult{err: err}
					return
				}
				items = append(items, page.Items...)
			}

			results <- segmentResult{items: items}
		}(segment)
	}

	var allItems []map[string]types.AttributeValue
	for i := 0; i < totalSegments; i++ {
		result := <-results
		if result.err != nil {
			return nil, result.err
		}
		allItems = append(allItems, result.items...)
	}

	return allItems, nil
}
```

Each goroutine processes one segment of the table using the `Segment` and `TotalSegments` parameters. DynamoDB divides the table's hash space evenly across segments. A good starting point is one segment per vCPU available to your application.

## Test the scan operations

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

	// Full table scan
	fmt.Println("Scanning entire table:")
	items, err := repo.ScanAllItems(ctx)
	if err != nil {
		log.Fatalf("Scan failed: %v", err)
	}
	fmt.Printf("Total items in table: %d\n", len(items))

	// Parallel scan
	fmt.Println("\nParallel scan with 4 segments:")
	items, err = repo.ParallelScan(ctx, 4)
	if err != nil {
		log.Fatalf("Parallel scan failed: %v", err)
	}
	fmt.Printf("Total items found: %d\n", len(items))
}
```

Run:
```bash
go run .
```

Expected output (item count depends on earlier steps):
```text
Scanning entire table:
  Page 1: 17 items
Total items in table: 17

Parallel scan with 4 segments:
Total items found: 17
```

Both approaches return the same total count, but parallel scan completes faster on large tables because segments are processed concurrently.

### Review

At this point you have used every major DynamoDB read and write API:
- `PutItem` / `BatchWriteItem` for writes
- `GetItem` / `Query` for targeted reads
- `UpdateItem` with expressions and conditions
- `DeleteItem` with safeguards
- `TransactWriteItems` / `TransactGetItems` for atomicity
- `Scan` for full-table reads

In the next module, you clean up the resources created during this workshop.
