---
title: "LGOD: DynamoDB for Go Developers"
chapter: true
description: "200 level: Hands-on with DynamoDB APIs, single table design, and indexes using Go."
weight: 25
---

In this workshop, you learn how to use [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) with the AWS SDK for Go v2. You build a complete inventory management system using single table design, working through each DynamoDB API operation step by step.

Here's what this workshop includes:

::children{depth=1}

### Target audience

This workshop is designed for Go developers who want to learn DynamoDB from the ground up by building a real application. You will write Go code that creates tables, writes data, queries with indexes, and performs transactions — running each operation directly from `go run .`.

### Requirements

#### Go programming experience
- Familiarity with Go structs, interfaces, and error handling
- Basic understanding of Go modules (`go mod`)
- Comfort with the command line

#### Basic knowledge of AWS services
- An AWS account with permissions to create DynamoDB tables
- AWS CLI installed and configured with credentials
- Among other services this lab will guide you through the use of [Amazon DynamoDB](https://aws.amazon.com/dynamodb/)

#### Basic understanding of DynamoDB
- If you're not familiar with DynamoDB, consider reviewing the documentation on "[What is Amazon DynamoDB?](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)"

### What you'll build

A complete inventory management system featuring:
- **Users** with profiles and multiple addresses
- **Orders** with status tracking and lifecycle management
- **Order Items** with pricing and quantities
- **Single table design** with multiple access patterns

### What you'll learn

- Creating and managing DynamoDB tables with the Go SDK
- Writing items with `PutItem` and `BatchWriteItem`
- Reading items with `GetItem` and `Query`
- Querying Global Secondary Indexes (GSI) for cross-partition lookups
- Querying Local Secondary Indexes (LSI) for alternate sort orders
- Updating items with expressions and conditions
- Deleting items with safeguards
- Atomic operations with `TransactWriteItems` and `TransactGetItems`
- Scanning tables with pagination and parallel segments
