---
title: "Condition expressions"
date: 2021-04-21T07:33:04-05:00
weight: 20
---

Condition expressions let you specify requirements that must be true for an update to succeed. If the condition evaluates to false, DynamoDB rejects the write and returns a `ConditionalCheckFailedException`. This provides optimistic locking without external coordination.

## Why conditions matter

Without conditions, any write blindly overwrites the current state. Conditions prevent:
- Shipping an order that has already been cancelled
- Creating a user that already exists
- Updating a record that another process has already modified

## Conditional update: only ship pending orders

Add this function to `repository.go`:

```go
func (r *Repository) ShipOrder(ctx context.Context, orderID string) error {
	order, err := r.GetOrderByID(ctx, orderID)
	if err != nil {
		return err
	}

	statusDate := fmt.Sprintf("shipped#%s", time.Now().Format("2006-01-02"))

	_, err = r.client.UpdateItem(ctx, &dynamodb.UpdateItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", order.UserID)},
			"sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
		},
		UpdateExpression:    aws.String("SET #status = :new_status, #status_date = :status_date REMOVE #placed_id"),
		ConditionExpression: aws.String("#status = :expected_status"),
		ExpressionAttributeNames: map[string]string{
			"#status":      "status",
			"#status_date": "status_date",
			"#placed_id":   "placed_id",
		},
		ExpressionAttributeValues: map[string]types.AttributeValue{
			":new_status":      &types.AttributeValueMemberS{Value: "shipped"},
			":expected_status": &types.AttributeValueMemberS{Value: "confirmed"},
			":status_date":     &types.AttributeValueMemberS{Value: statusDate},
		},
	})
	return err
}
```

The `ConditionExpression` ensures the order's current status is `confirmed`. If someone already cancelled or shipped the order, the condition fails and the update is rejected.

## Conditional create: prevent duplicate users

You can also use conditions with `PutItem` to prevent overwriting existing items:

```go
func (r *Repository) CreateUserIfNotExists(ctx context.Context, user User) error {
	userMap, err := attributevalue.MarshalMap(user)
	if err != nil {
		return err
	}

	userMap["pk"] = &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", user.Username)}
	userMap["sk"] = &types.AttributeValueMemberS{Value: "PROFILE"}

	_, err = r.client.PutItem(ctx, &dynamodb.PutItemInput{
		TableName:           aws.String(r.tableName),
		Item:                userMap,
		ConditionExpression: aws.String("attribute_not_exists(pk)"),
	})
	return err
}
```

The `attribute_not_exists(pk)` condition ensures the item does not already exist. If a user with that username already has a profile, the write fails instead of silently overwriting it.

## Handling ConditionalCheckFailedException

In Go, you check for this error using the SDK's error types:

```go
import "errors"

var condErr *types.ConditionalCheckFailedException

if errors.As(err, &condErr) {
	fmt.Println("Condition not met — the item was not in the expected state.")
} else if err != nil {
	log.Fatalf("Unexpected error: %v", err)
}
```

## Test conditional updates

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

	// Try to ship a confirmed order (should succeed)
	fmt.Println("Shipping ord-aaa-002 (currently confirmed)...")
	err = repo.ShipOrder(ctx, "ord-aaa-002")
	if err != nil {
		var condErr *types.ConditionalCheckFailedException
		if errors.As(err, &condErr) {
			fmt.Println("  REJECTED: order is not in 'confirmed' status")
		} else {
			log.Fatalf("Unexpected error: %v", err)
		}
	} else {
		fmt.Println("  SUCCESS: order shipped")
	}

	// Try to ship a pending order (should fail — must be confirmed first)
	fmt.Println("\nShipping ord-bbb-001 (currently pending)...")
	err = repo.ShipOrder(ctx, "ord-bbb-001")
	if err != nil {
		var condErr *types.ConditionalCheckFailedException
		if errors.As(err, &condErr) {
			fmt.Println("  REJECTED: order is not in 'confirmed' status")
		} else {
			log.Fatalf("Unexpected error: %v", err)
		}
	} else {
		fmt.Println("  SUCCESS: order shipped")
	}
}
```

You need to add `"errors"` to your imports. Run:

```bash
go run .
```

Expected output:
```text
Shipping ord-aaa-002 (currently confirmed)...
  SUCCESS: order shipped

Shipping ord-bbb-001 (currently pending)...
  REJECTED: order is not in 'confirmed' status
```

The condition expression enforced the business rule: orders must be confirmed before they can be shipped. The second update was rejected because `ord-bbb-001` was still in `pending` status.
