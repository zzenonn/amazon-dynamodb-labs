---
title: "Set up the Go project"
date: 2021-04-21T07:33:04-05:00
weight: 30
---

In this step, you create the Go project and install the AWS SDK dependencies you need for the rest of the workshop.

## Initialize the project

Create a new directory for the project and initialize the Go module:

```bash
mkdir dynamodb-for-go-developers && cd dynamodb-for-go-developers
go mod init dynamodb-for-go-developers
```

## Install dependencies

Install the AWS SDK for Go v2 packages. You need the core SDK, the DynamoDB client, the DynamoDB attribute value marshaler, and the config loader:

```bash
go get github.com/aws/aws-sdk-go-v2
go get github.com/aws/aws-sdk-go-v2/config
go get github.com/aws/aws-sdk-go-v2/service/dynamodb
go get github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue
go get github.com/google/uuid
```

## Create the main file

Create a file named `main.go` with the initial project structure. This file initializes the AWS client and serves as the entry point for all operations you build in this workshop.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/dynamodb"
)

func main() {
	region := os.Getenv("AWS_REGION")
	if region == "" {
		region = "us-east-1"
	}

	cfg, err := config.LoadDefaultConfig(context.TODO(),
		config.WithRegion(region),
	)
	if err != nil {
		log.Fatalf("Failed to load AWS config: %v", err)
	}

	client := dynamodb.NewFromConfig(cfg)
	fmt.Printf("DynamoDB client initialized for region: %s\n", region)

	_ = client // You will use this in the next module
}
```

## Define the data models

Create a file named `models.go`. This defines the Go structs for the three entity types you store in DynamoDB: users, orders, and order items.

```go
package main

import "time"

type OrderStatus string

const (
	OrderStatusPending   OrderStatus = "pending"
	OrderStatusConfirmed OrderStatus = "confirmed"
	OrderStatusShipped   OrderStatus = "shipped"
	OrderStatusDelivered OrderStatus = "delivered"
	OrderStatusCancelled OrderStatus = "cancelled"
)

type Address struct {
	Street  string `json:"street" dynamodbav:"street"`
	State   string `json:"state,omitempty" dynamodbav:"state,omitempty"`
	Country string `json:"country" dynamodbav:"country"`
}

type User struct {
	Username  string             `json:"username" dynamodbav:"-"`
	FullName  string             `json:"full_name,omitempty" dynamodbav:"full_name,omitempty"`
	Email     string             `json:"email,omitempty" dynamodbav:"email,omitempty"`
	Addresses map[string]Address `json:"addresses,omitempty" dynamodbav:"addresses,omitempty"`
}

type Order struct {
	ID         string      `json:"id" dynamodbav:"order_id"`
	UserID     string      `json:"user_id" dynamodbav:"user_id"`
	Status     OrderStatus `json:"status" dynamodbav:"status"`
	AddressKey string      `json:"address_key" dynamodbav:"address_key"`
	CreatedAt  time.Time   `json:"created_at" dynamodbav:"created_at"`
	UpdatedAt  time.Time   `json:"updated_at" dynamodbav:"updated_at"`
}

type OrderItem struct {
	OrderID     string  `json:"order_id" dynamodbav:"order_id"`
	ItemID      string  `json:"item_id" dynamodbav:"item_id"`
	Name        string  `json:"name" dynamodbav:"name"`
	Description string  `json:"description" dynamodbav:"description"`
	Price       float64 `json:"price" dynamodbav:"price"`
	Quantity    int     `json:"quantity" dynamodbav:"quantity"`
}
```

Notice the `dynamodbav` struct tags. These control how the AWS SDK marshals and unmarshals Go structs to and from DynamoDB attribute values. The `User.Username` field uses `dynamodbav:"-"` because it is encoded in the partition key rather than stored as a separate attribute.

## Verify the setup

Run the program to confirm everything compiles and the AWS client initializes:

```bash
go run .
```

Expected output:
```text
DynamoDB client initialized for region: us-east-1
```

If you see an error about credentials, verify your AWS CLI configuration with `aws sts get-caller-identity`.

Your project is ready. In the next module, you learn about the data model and how entities map to a single DynamoDB table.
