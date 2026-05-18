---
title: "2. The Data Model"
date: 2021-04-21T07:33:04-05:00
weight: 20
chapter: true
description: "Understand single table design and how entities map to DynamoDB keys."
---

In this chapter, you learn the single table design pattern and understand how multiple entity types are stored in one DynamoDB table. You learn about composite keys, access patterns, and index design.

The data model supports three main entities:
- **Users**: Customer profiles with addresses
- **Orders**: Order tracking with status management
- **Order Items**: Individual items within orders

::children{depth=1}
