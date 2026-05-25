---
title: "8. Cleanup"
date: 2021-04-21T07:33:04-05:00
weight: 80
description: "Delete the DynamoDB table and clean up resources."
---

::alert[If you are running this workshop in your own AWS account, complete this cleanup to avoid ongoing charges.]{type="warning"}

## Delete the DynamoDB table

You can delete the table using the AWS CLI:

```bash
aws dynamodb delete-table --table-name simple-inventory
```

Expected output:
```json
{
    "TableDescription": {
        "TableName": "simple-inventory",
        "TableStatus": "DELETING"
    }
}
```

Verify deletion:
```bash
aws dynamodb describe-table --table-name simple-inventory 2>&1
```

Expected output (after a few seconds):
```text
An error occurred (ResourceNotFoundException) when calling the DescribeTable operation: Requested resource not found: Table: simple-inventory not found
```

Alternatively, you can add a delete function to your Go code:

```go
func (r *Repository) DeleteTable(ctx context.Context) error {
	_, err := r.client.DeleteTable(ctx, &dynamodb.DeleteTableInput{
		TableName: aws.String(r.tableName),
	})
	return err
}
```

## If you used CloudFormation

If you launched resources via a CloudFormation stack during setup, delete the stack:

```bash
aws cloudformation delete-stack --stack-name DynamoDBID
```

## What you learned

In this workshop you used every major DynamoDB API with the AWS SDK for Go v2:

| API | What it does | Module |
|-----|-------------|--------|
| `CreateTable` | Create table with GSIs and LSI | 2 |
| `PutItem` | Write a single item | 3 |
| `BatchWriteItem` | Write up to 25 items per call | 3 |
| `GetItem` | Read a single item by key | 4 |
| `Query` | Read items sharing a partition key | 4 |
| `Query` (GSI/LSI) | Secondary index lookups | 4 |
| `Scan` | Read the entire table | 4 |
| `UpdateItem` | Modify attributes with expressions | 5 |
| `DeleteItem` | Remove an item | 6 |
| `TransactWriteItems` | Atomic multi-item writes | 7 |
| `TransactGetItems` | Atomic multi-item reads | 7 |
| `DeleteTable` | Delete the table | 8 |

### Key design concepts applied

- **Single table design** — multiple entity types in one table
- **Composite keys with prefixes** — `#USER#`, `#ORDER#`, `#ITEM#`
- **Inverted index GSI** — cross-partition lookups by sort key
- **Sparse index GSI** — only active items appear in the index
- **Local Secondary Index** — alternate sort order within a partition
- **Condition expressions** — optimistic locking and write guards
- **Transactions** — atomic operations across multiple items

### Next steps

- Explore [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- Learn about [DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html) for change data capture
- Try the [Advanced Design Patterns](/design-patterns) workshop for more complex modeling scenarios
