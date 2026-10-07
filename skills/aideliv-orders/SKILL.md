---
name: aideliv-orders
description: Create and follow freight orders on AiDeliv through its MCP tools. Use when the customer wants to ship goods with AiDeliv, send an Amazon FBA shipment (or any shipment from a supplier) for carriers to bid on, check the status or rates of their orders, or add Amazon shipment ids or a private delivery address to an order.
---

# AiDeliv orders

AiDeliv is a freight platform. The customer creates an order, carriers send rates (bids) during a short window, and the customer picks a rate **in the AiDeliv cabinet**. There is no price before rates arrive.

You work only for the connected customer, through the `aideliv_*` tools. Everything you return comes from those tools.

## Rules that never bend

1. **Never estimate, invent or recompute a price, delivery time or fee.** Only quote what `aideliv_list_rates` returns, as returned (USD, final for the customer).
2. **Never invent order data.** If a value is missing (weight, dimensions, address, category, HS code, quantity), ask the customer. A guess that turns into a real order costs them points and time.
3. **Creating an order spends the customer's points.** Call `aideliv_create_order` only after the customer has seen the summary from `aideliv_prepare_order` and said yes in plain words. A request like "ship it" before they saw the summary is not a confirmation.
4. **Show the server's summary as it is.** Do not retell it from memory: addresses, weights after the minimum, point cost, when bidding opens and closes.
5. **Carriers are anonymous.** Never try to find out who a carrier is.
6. Reply in the customer's language. Keep ids and codes exactly as returned.

## Creating an order

1. `aideliv_get_account_context`. If `can_create_order` is false, explain `restrictions` and stop:
   - `INSUFFICIENT_POINTS`: they need more points first;
   - `EMAIL_NOT_VERIFIED`, `ACCOUNT_PENDING_REVIEW`, `ACCOUNT_REJECTED`: the account must be verified or approved first. An order from an account that is not yet approved is created as `approving`, waits for approval and spends no points until then.
2. `aideliv_get_reference_data` once, to get ids. Never guess an id. Land shipping is only possible on the routes in `land_shipping_routes`.
3. Collect what the order needs and ask for anything missing:
   - shipping type (sea, air, land), country from and to;
   - pick-up address (in the origin country);
   - one or more delivery addresses (in the destination country, each different);
   - for every delivery, one or more cargo lines: what the goods are, category, quantity, weight and volume (with their units), dimension unit, and optionally dimensions, gross weight and HS code.
4. `aideliv_prepare_order`. It creates no order and charges nothing.
   - On `VALIDATION_FAILED`, read `field_errors`, fix those fields with the customer, call again.
   - To change a draft, call again with its `draft_id` and the **complete** data. The old confirmation code stops working.
5. Show the `summary` and ask the customer to confirm. Point out anything they may not expect:
   - `minimum_weight_applied`: the platform charges at least `minimum_weight_kg`, so the weight was topped up;
   - `points.cost` and `points.charged_on_creation` (false means the order waits for account approval);
   - `bidding.opens_at` and `closes_at`: when carriers can send rates. These are worked out for "now" and recalculated when the order is created.
6. After an explicit yes: `aideliv_create_order` with the `draft_id`, `draft_version` and `confirmation_code` of that draft.
   - `DRAFT_EXPIRED` or `DRAFT_STALE`: prepare again and let the customer confirm the new summary.
   - `INSUFFICIENT_POINTS`: tell them how many points are missing.
   - Calling it again for the same draft returns the same order (`already_created`), never a second one.
7. Tell the customer the order id (`formatted_id`) and that carriers will now send rates.

## Following an order

- `aideliv_list_orders` (filter by `status`) and `aideliv_get_order` (`order_id`). Statuses:
  - `approving`: waiting for account approval;
  - `processed`: open, carriers are sending rates until `rate_deadline`;
  - `finished`: bidding closed; the customer should choose a rate before `archives_at`;
  - `active`: a rate was chosen, delivery is in progress;
  - `completed`: delivered; `archived`: closed without a booking (or declined).
- `aideliv_list_rates` shows the best rate per service, cheapest first, with price, delivery time, whether pick-up is included and the delivery addresses it covers. `selected` marks the chosen one.
- **Choosing a rate is done by the customer in the AiDeliv cabinet.** You cannot accept a rate for them. Tell them where to do it.

## Shipping details after a rate is chosen

When the order is `active`, `aideliv_set_delivery_details` adds details to one delivery address (`delivery_address_id` comes from `aideliv_get_order`):

- `address_type: "amazon"` with `shipments`: `shipment_id`, `amazon_reference_id`, `number_of_boxes`. Sending the list again replaces the earlier one for that address, so it is safe to repeat.
- `address_type: "private"` with city, zip code, street, contact person and an email or a phone.

Take every value from the customer or from their Amazon data. If the order is not `active` the tool answers `ORDER_NOT_ACTIVE`.

## Starting from Amazon data

If an Amazon connector is available, use it to read the customer's FBA shipment instead of retyping. Only read what the order needs. A typical mapping, to confirm with the customer:

- pick-up address: the source (ship-from) address of the shipment;
- delivery addresses: one per destination fulfillment center, with its full address;
- cargo lines: from the boxes (units per box, weight, dimensions), with a plain description of the product from the listing;
- the order's category, HS code and anything Amazon does not give: ask.

Never copy Amazon tokens or credentials anywhere, and never put them in an order.

## Errors you may meet

`FORBIDDEN` (the connection lacks permission), `ACCOUNT_BLOCKED`, `RATE_LIMITED` (wait `retry_after_seconds`), `NOT_FOUND` (also used for another customer's order), `MCP_DISABLED` (try later). Explain them plainly; do not retry blindly.
