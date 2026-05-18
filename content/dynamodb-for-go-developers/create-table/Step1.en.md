---
title: "Create the DynamoDB table"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

In this step, you write the Go code to create the `simple-inventory` table with all its indexes.

## Understanding CreateTable

The `CreateTable` API requires you to define:
- **AttributeDefinitions** — only the attributes used in key schemas (primary key and indexes)
- **KeySchema** — the partition key and sort key for the table
- **GlobalSecondaryIndexes** — indexes with a different partition key than the base table
- **LocalSecondaryIndexes** — indexes that share the base table's partition key but use a different sort key
- **BillingMode** — how you pay for read/write capacity

DynamoDB is schemaless beyond the key attributes. You do not declare non-key attributes in the table definition — they are added dynamically when you write items.

## Write the CreateTable function

Create a file named `repository.go`. This file holds all DynamoDB operations. Start with the `Repository` struct and the `CreateTable` function:

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
)

type Repository struct {
	client    *dynamodb.Client
	tableName string
}

func NewRepository(client *dynamodb.Client, tableName string) *Repository {
	return &Repository{
		client:    client,
		tableName: tableName,
	}
}

func (r *Repository) CreateTable(ctx context.Context) error {
	input := &dynamodb.CreateTableInput{
		TableName: aws.String(r.tableName),
		KeySchema: []types.KeySchemaElement{
			{AttributeName: aws.String("pk"), KeyType: types.KeyTypeHash},
			{AttributeName: aws.String("sk"), KeyType: types.KeyTypeRange},
		},
		AttributeDefinitions: []types.AttributeDefinition{
			{AttributeName: aws.String("pk"), AttributeType: types.ScalarAttributeTypeS},
			{AttributeName: aws.String("sk"), AttributeType: types.ScalarAttributeTypeS},
			{AttributeName: aws.String("status_date"), AttributeType: types.ScalarAttributeTypeS},
			{AttributeName: aws.String("placed_id"), AttributeType: types.ScalarAttributeTypeS},
		},
		GlobalSecondaryIndexes: []types.GlobalSecondaryIndex{
			{
				IndexName: aws.String("inverted-index"),
				KeySchema: []types.KeySchemaElement{
					{AttributeName: aws.String("sk"), KeyType: types.KeyTypeHash},
					{AttributeName: aws.String("pk"), KeyType: types.KeyTypeRange},
				},
				Projection: &types.Projection{ProjectionType: types.ProjectionTypeAll},
			},
			{
				IndexName: aws.String("placed-index"),
				KeySchema: []types.KeySchemaElement{
					{AttributeName: aws.String("placed_id"), KeyType: types.KeyTypeHash},
				},
				Projection: &types.Projection{ProjectionType: types.ProjectionTypeAll},
			},
		},
		LocalSecondaryIndexes: []types.LocalSecondaryIndex{
			{
				IndexName: aws.String("status-date-index"),
				KeySchema: []types.KeySchemaElement{
					{AttributeName: aws.String("pk"), KeyType: types.KeyTypeHash},
					{AttributeName: aws.String("status_date"), KeyType: types.KeyTypeRange},
				},
				Projection: &types.Projection{ProjectionType: types.ProjectionTypeAll},
			},
		},
		BillingMode: types.BillingModePayPerRequest,
	}

	_, err := r.client.CreateTable(ctx, input)
	return err
}
```

Let's walk through the key pieces:

**KeySchema** defines the primary key. `pk` is the partition key (HASH) and `sk` is the sort key (RANGE). Together they uniquely identify every item.

**AttributeDefinitions** declares only the four attributes used in key schemas: `pk`, `sk`, `status_date`, and `placed_id`. You don't declare `email`, `full_name`, or other non-key attributes here.

**inverted-index GSI** swaps `sk` as the partition key and `pk` as the sort key. This enables looking up any item by its sort key value across the entire table — for example, finding an order by its ID regardless of which user placed it.

**placed-index GSI** uses `placed_id` as its partition key. This is a sparse index: only items that have a `placed_id` attribute appear in it. Orders in `pending` or `confirmed` status have this attribute; shipped and delivered orders do not.

**status-date-index LSI** shares the table's partition key (`pk`) but uses `status_date` as the sort key. This lets you query a specific user's orders sorted by status and date.

**BillingMode** is set to `PayPerRequest` (on-demand). You pay per read/write request with no capacity planning required.

## Update main.go

Update your `main.go` to call `CreateTable`:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
)

import (
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
)

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

	fmt.Printf("Creating table '%s' in region '%s'...\n", tableName, region)
	if err := repo.CreateTable(ctx); err != nil {
		log.Fatalf("Failed to create table: %v", err)
	}
	fmt.Println("Table created successfully!")
}
```

## Run the code

```bash
go run .
```

Expected output:
```text
Creating table 'simple-inventory' in region 'us-east-1'...
Table created successfully!
```

## Verify the table

Creating a table is asynchronous. You can check its status with the AWS CLI:

```bash
aws dynamodb describe-table --table-name simple-inventory --query "Table.TableStatus"
```

Expected output:
```text
"ACTIVE"
```

If the status shows `CREATING`, wait a few seconds and run the command again. Once the status is `ACTIVE`, the table is ready for use.

You can also verify the indexes were created:

```bash
aws dynamodb describe-table --table-name simple-inventory \
  --query "Table.[GlobalSecondaryIndexes[].IndexName, LocalSecondaryIndexes[].IndexName]"
```

Expected output:
```json
[
    ["inverted-index", "placed-index"],
    ["status-date-index"]
]
```

Your table is ready. In the next module, you write data to it.
