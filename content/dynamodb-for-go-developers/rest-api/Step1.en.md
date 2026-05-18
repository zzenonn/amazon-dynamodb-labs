---
title: "HTTP handlers"
date: 2021-04-21T07:33:04-05:00
weight: 10
---

In this step, you create HTTP handlers that call the repository functions. The handlers translate HTTP requests into DynamoDB operations and return JSON responses.

## Install the HTTP router

You use the `chi` router for clean URL parameter handling:

```bash
go get github.com/go-chi/chi/v5
```

## Create the handlers file

Create a file named `handlers.go`:

```go
package main

import (
	"encoding/json"
	"net/http"
	"time"

	"github.com/go-chi/chi/v5"
	"github.com/google/uuid"
)

type API struct {
	repo *Repository
}

func NewAPI(repo *Repository) *API {
	return &API{repo: repo}
}

func (api *API) CreateUser(w http.ResponseWriter, r *http.Request) {
	var user User
	if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
		http.Error(w, "Invalid JSON", http.StatusBadRequest)
		return
	}

	if user.Username == "" {
		http.Error(w, "Username is required", http.StatusBadRequest)
		return
	}

	if err := api.repo.CreateUser(r.Context(), user); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(user)
}

func (api *API) GetUser(w http.ResponseWriter, r *http.Request) {
	username := chi.URLParam(r, "username")

	user, err := api.repo.GetUser(r.Context(), username)
	if err != nil {
		http.Error(w, err.Error(), http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(user)
}

func (api *API) CreateOrder(w http.ResponseWriter, r *http.Request) {
	var req struct {
		UserID     string      `json:"user_id"`
		AddressKey string      `json:"address_key"`
		Items      []OrderItem `json:"items"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid JSON", http.StatusBadRequest)
		return
	}

	if req.UserID == "" {
		http.Error(w, "user_id is required", http.StatusBadRequest)
		return
	}

	order := &Order{
		ID:         uuid.New().String(),
		UserID:     req.UserID,
		Status:     OrderStatusPending,
		AddressKey: req.AddressKey,
		CreatedAt:  time.Now(),
		UpdatedAt:  time.Now(),
	}

	// Assign item IDs if not provided
	for i := range req.Items {
		if req.Items[i].ItemID == "" {
			req.Items[i].ItemID = uuid.New().String()
		}
		req.Items[i].OrderID = order.ID
	}

	// Use transaction to create order + items atomically
	if len(req.Items) > 0 {
		if err := api.repo.PlaceOrder(r.Context(), order, req.Items); err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
	} else {
		if err := api.repo.CreateOrder(r.Context(), order); err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(order)
}

func (api *API) GetOrder(w http.ResponseWriter, r *http.Request) {
	orderID := chi.URLParam(r, "orderid")

	order, err := api.repo.GetOrderByID(r.Context(), orderID)
	if err != nil {
		http.Error(w, err.Error(), http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(order)
}

func (api *API) GetUserOrders(w http.ResponseWriter, r *http.Request) {
	username := chi.URLParam(r, "username")

	// Check for status filter (uses LSI)
	status := r.URL.Query().Get("status")
	if status != "" {
		orders, err := api.repo.GetUserOrdersByStatus(r.Context(), username, OrderStatus(status))
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(orders)
		return
	}

	// No filter — query primary table
	orders, err := api.repo.GetOrdersByUserID(r.Context(), username)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(orders)
}

func (api *API) UpdateOrderStatus(w http.ResponseWriter, r *http.Request) {
	orderID := chi.URLParam(r, "orderid")

	var req struct {
		Status OrderStatus `json:"status"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid JSON", http.StatusBadRequest)
		return
	}

	if err := api.repo.UpdateOrderStatus(r.Context(), orderID, req.Status); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{"status": "updated"})
}

func (api *API) GetPendingOrders(w http.ResponseWriter, r *http.Request) {
	orders, err := api.repo.GetPendingOrders(r.Context())
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(orders)
}

func (api *API) GetOrderItems(w http.ResponseWriter, r *http.Request) {
	orderID := chi.URLParam(r, "orderid")

	items, err := api.repo.GetOrderItems(r.Context(), orderID)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(items)
}
```

Each handler follows the same pattern: parse the request, call the repository, and encode the response. The `GetUserOrders` handler demonstrates how a query parameter (`?status=pending`) triggers the LSI query instead of the base table query.
