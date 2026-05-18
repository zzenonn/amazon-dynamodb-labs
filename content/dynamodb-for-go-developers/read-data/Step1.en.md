---
title: "GetItem - Single item retrieval"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

The `GetItem` operation retrieves a single item by its full primary key (partition key + sort key). It is the most efficient read operation in DynamoDB — it goes directly to the partition that holds the item.

## Get a user profile

Add the following function to `repository.go`:

```go
func (r *Repository) GetUser(ctx context.Context, username string) (*User, error) {
	result, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", username)},
			"sk": &types.AttributeValueMemberS{Value: "PROFILE"},
		},
	})
	if err != nil {
		return nil, err
	}

	if result.Item == nil {
		return nil, fmt.Errorf("user not found: %s", username)
	}

	var user User
	if err := attributevalue.UnmarshalMap(result.Item, &user); err != nil {
		return nil, err
	}
	user.Username = username
	return &user, nil
}
```

`GetItem` requires the complete primary key. Because you know both the partition key (`#USER#alice`) and the sort key (`PROFILE`) for a user, you can fetch the exact item directly.

The `attributevalue.UnmarshalMap` function converts the DynamoDB attribute map back into the Go struct, using the `dynamodbav` tags to map attribute names to struct fields.

## Consistent reads

By default, `GetItem` uses eventually consistent reads. If you need to read the most recent write immediately, you can request a strongly consistent read:

```go
result, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
	TableName:      aws.String(r.tableName),
	Key:            key,
	ConsistentRead: aws.Bool(true),
})
```

Strongly consistent reads cost twice as many Read Request Units (RRUs) as eventually consistent reads. Use them only when your application requires it.

## Projection expressions

If you only need certain attributes, use a projection expression to reduce the data transferred:

```go
result, err := r.client.GetItem(ctx, &dynamodb.GetItemInput{
	TableName: aws.String(r.tableName),
	Key: map[string]types.AttributeValue{
		"pk": &types.AttributeValueMemberS{Value: "#USER#alice"},
		"sk": &types.AttributeValueMemberS{Value: "PROFILE"},
	},
	ProjectionExpression: aws.String("full_name, email"),
})
```

This returns only the `full_name` and `email` attributes. The item still consumes the same RRUs (DynamoDB reads the full item internally), but you reduce the payload size over the network.

## Test the GetItem function

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

	user, err := repo.GetUser(ctx, "alice")
	if err != nil {
		log.Fatalf("Failed to get user: %v", err)
	}

	fmt.Printf("User: %s\n", user.Username)
	fmt.Printf("Name: %s\n", user.FullName)
	fmt.Printf("Email: %s\n", user.Email)
}
```

Run:
```bash
go run .
```

Expected output:
```text
User: alice
Name: Alice Smith
Email: alice@example.com
```
