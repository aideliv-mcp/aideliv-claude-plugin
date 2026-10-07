---
description: Show the status and carriers' rates of AiDeliv orders
argument-hint: [order id]
---

Show the customer where their AiDeliv orders stand. Follow the `aideliv-orders` skill.

- Without an order id: list their orders with `aideliv_list_orders` (newest first) and say what each status means for them.
- With an order id: show `aideliv_get_order` and, if the order has rates, `aideliv_list_rates`. Quote prices and delivery times exactly as returned. Remind the customer that choosing a rate is done in the AiDeliv cabinet.

Order id: $ARGUMENTS
