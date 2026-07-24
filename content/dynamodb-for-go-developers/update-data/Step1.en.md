---
title: "UpdateItem with expressions"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

The `UpdateItem` operation modifies an existing item's attributes without replacing the entire item. This is more efficient than `PutItem` when you only need to change a few fields, because DynamoDB only writes the changed attributes.

## Update expressions

Update expressions define what attributes to change. The four clauses are:

| Clause | Purpose | Example |
|--------|---------|---------|
| `SET` | Add or change attributes | `SET email = :email` |
| `REMOVE` | Delete attributes | `REMOVE placed_id` |
| `ADD` | Increment numbers or add to sets | `ADD quantity :inc` |
| `DELETE` | Remove elements from a set | `DELETE tags :old_tags` |

You can combine clauses in a single expression, but each keyword may appear **only once** - all your `SET` assignments go in one `SET` clause.

## Your turn: update order status

Updating an order's status is a good exercise because it touches multiple attributes and interacts with the sparse index. Find the `UpdateOrderStatus` stub in `repository.go` and implement it, following the `TODO(lab)` comment. The function should:

1. **Look up the order first** with `r.GetOrderByID(ctx, orderID)` to learn its `UserID` (needed for the base-table key).
2. **Build a `SET` clause** for `status`, `status_date` (`<newStatus>#<today>`), and `updated_at`. `status` is a reserved word, so alias every attribute name via `ExpressionAttributeNames`.
3. **Manage the sparse index attribute:**
   - If the new status is `pending` or `confirmed`, fold `#placed_id = :placed_id` into the **same** `SET` clause (putting the order in the sparse `placed-index`).
   - Otherwise, append ` REMOVE #placed_id` (taking the order out of the sparse index).
4. **Call `UpdateItem`** with the key `pk = #USER#<UserID>`, `sk = #ORDER#<orderID>`.

You will need to add the `"time"` import to `repository.go`: `time.Now().Format("2006-01-02")` builds the `status_date` date and `time.RFC3339` formats `updated_at`.

::::expand{header="Expand this to see the solution for UpdateOrderStatus"}
```go
func (r *Repository) UpdateOrderStatus(ctx context.Context, orderID string, newStatus OrderStatus) error {
	order, err := r.GetOrderByID(ctx, orderID)
	if err != nil {
		return err
	}

	statusDate := fmt.Sprintf("%s#%s", newStatus, time.Now().Format("2006-01-02"))

	// An UpdateExpression may use each keyword (SET/REMOVE) only once, so the
	// placed_id change is folded into the same SET or REMOVE clause rather than
	// appended as a second SET.
	setExpr := "SET #status = :status, #status_date = :status_date, #updated_at = :updated_at"
	exprNames := map[string]string{
		"#status":      "status",
		"#status_date": "status_date",
		"#updated_at":  "updated_at",
	}
	exprValues := map[string]types.AttributeValue{
		":status":      &types.AttributeValueMemberS{Value: string(newStatus)},
		":status_date": &types.AttributeValueMemberS{Value: statusDate},
		":updated_at":  &types.AttributeValueMemberS{Value: time.Now().Format(time.RFC3339)},
	}

	var updateExpr string
	if newStatus == OrderStatusPending || newStatus == OrderStatusConfirmed {
		// Active order: set placed_id so it appears in the sparse placed-index.
		setExpr += ", #placed_id = :placed_id"
		exprNames["#placed_id"] = "placed_id"
		exprValues[":placed_id"] = &types.AttributeValueMemberS{Value: string(newStatus)}
		updateExpr = setExpr
	} else {
		// Inactive order: drop placed_id so it falls out of the sparse index.
		exprNames["#placed_id"] = "placed_id"
		updateExpr = setExpr + " REMOVE #placed_id"
	}

	_, err = r.client.UpdateItem(ctx, &dynamodb.UpdateItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]types.AttributeValue{
			"pk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#USER#%s", order.UserID)},
			"sk": &types.AttributeValueMemberS{Value: fmt.Sprintf("#ORDER#%s", orderID)},
		},
		UpdateExpression:          aws.String(updateExpr),
		ExpressionAttributeNames:  exprNames,
		ExpressionAttributeValues: exprValues,
	})
	return err
}
```
::::

::alert[Because an `UpdateExpression` may use `SET` only once, the `placed_id` assignment must be part of the same `SET` clause - not a second one. The `TODO(lab)` comment shows how to fold it in. If you get stuck, see the full reference solution as described in :link[Set up the Go project]{href="/dynamodb-for-go-developers/setup/step1"}.]{type="info"}

## Check your work

Run the demo. The core walkthrough confirms an order (active status keeps `placed_id`) and then ships it (inactive status removes `placed_id`):

```bash
go run . demo
```

Expected fragment:
```text
== UpdateItem: confirm an order (active status -> keeps placed_id) ==
  ord-aaa-001  status=confirmed (now appears in confirmed placed-index)

== UpdateItem: ship an order (inactive status -> removes placed_id) ==
  ord-aaa-001  status=shipped (dropped from sparse placed-index)
```

By removing the `placed_id` attribute when an order becomes inactive, the order is automatically removed from the sparse GSI - no separate index maintenance required.
