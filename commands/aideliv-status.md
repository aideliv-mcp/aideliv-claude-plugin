---
description: Show the status and current lowest price of AiDeliv delivery orders (auctions)
argument-hint: [order id]
---

Show the customer where their AiDeliv delivery orders stand. Follow the `aideliv-orders` skill.

- Without an order id: list their orders with `aideliv_list_orders` (newest first) and say what each status means for them.
- With an order id: show `aideliv_get_order` and, if bidding has started, `aideliv_list_rates`. While bidding is open, say it is the current lowest price and may still go down. Once bidding has closed, it is the final price: remind the customer to confirm it in the AiDeliv cabinet before the deadline, or a new auction will be needed. Quote prices and delivery times exactly as returned.

Order id: $ARGUMENTS
