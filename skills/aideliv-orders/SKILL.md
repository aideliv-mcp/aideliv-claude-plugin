---
name: aideliv-orders
description: Create and follow AiDeliv delivery orders (freight auctions) through the AiDeliv MCP tools. Use when the customer wants to ship goods with AiDeliv, put a shipment (including an Amazon FBA shipment from a supplier abroad) up for carriers to bid on, check an auction's current lowest price or status, or add Amazon shipment ids or a delivery address after confirming the final price.
---

# AiDeliv delivery orders (auctions)

AiDeliv is a technology marketplace built on a patent-pending auction engine. Each delivery order runs as a freight auction:

1. The customer creates the order. Bidding opens and runs for 9 hours.
2. While bidding is open, the customer sees the **current lowest price**. It updates whenever a pre-vetted carrier bids lower. Higher bids are not shown.
3. When bidding closes, the lowest bid becomes the **final price**. The customer confirms it within 24 hours, in the AiDeliv cabinet or by telling you to accept it. If it is not confirmed in time, the auction closes and a new auction is needed.

There is no price before the auction. You work only for the connected customer, through the `aideliv_*` tools, and everything you report comes from those tools.

## Rules that never bend

1. **Never estimate, invent or recompute a price, delivery time or fee.** Quote only what `aideliv_list_rates` returns, as returned (USD, final for the customer).
2. **Never invent order data.** If a value is missing (weight, dimensions, address, category, quantity), ask the customer. A guess that turns into a real order costs them points and time.
3. **Start an auction only when the cargo is ready to ship.** Ask the customer whether the goods are ready for pick-up now. If they are not, do not create the order; offer to prepare it again when the cargo is ready. An early auction is only an empty price check.
4. **Creating an order spends the customer's points.** Call `aideliv_create_order` only after the customer has seen the summary from `aideliv_prepare_order` and said yes in plain words. "Ship it" before they saw the summary is not a confirmation.
5. **Show the server's summary as it is.** Do not retell it from memory: addresses, weights after the minimum, point cost, when bidding opens and closes, and the deadline to confirm the price.
6. **Accept a bid only on the customer's explicit yes.** Call `aideliv_accept_rate` only after the customer has seen the price from `aideliv_list_rates` and said in plain words that they accept it. Never accept on your own, and never pick a bid for them. The customer can also confirm in the AiDeliv cabinet; tell them where and by when.
7. **Carriers are anonymous.** Never try to find out who a carrier is.
8. **Shipments within the US are not available** at the moment. Do not offer them. If the pick-up address is in the US, explain this and stop.
9. Reply in the customer's language. Keep ids and codes exactly as returned.

## Creating an order

1. `aideliv_get_account_context`. If `can_create_order` is false, explain `restrictions` and stop:
   - `INSUFFICIENT_POINTS`: they need more points first;
   - `EMAIL_NOT_VERIFIED`, `ACCOUNT_PENDING_REVIEW`, `ACCOUNT_REJECTED`: the account must be verified or approved first. An order from an account that is not yet approved is created as `approving`, waits for approval and spends no points until then.
2. `aideliv_get_reference_data` once, to get ids. Never guess an id. Land shipping is possible only on the routes in `land_shipping_routes`.
3. Confirm the cargo is ready to ship (rule 3). Then collect what the order needs and ask for anything missing:
   - shipping type (sea, air, land), country from and to;
   - pick-up address in the origin country, as separate fields: street address, city, district (optional), state/province/region and ZIP/postal code, the same fields as an Amazon address. Ask for any that are missing. Put only the address in them: never the supplier's name, phone or e-mail;
   - one or more delivery addresses in the destination country, each different. Use the full warehouse address, for example the exact Amazon fulfillment center: the auction prices each address;
   - for every delivery, one or more cargo lines: what the goods are, category, quantity, weight and volume (with their units), dimension unit, and optionally dimensions, gross weight and HS code.
4. Photos and certificates are optional. If the customer wants them attached (for example from an Amazon listing), upload each one with `aideliv_upload_file` using a direct https link taken from the page. Never guess a link or invent a document: if the listing has none, skip this step. Use `type` `cargo_photo`, `certificate` or `invoice` (packing lists too); `wechat_qr` is only for a factory's WeChat QR when accepting a bid. The `file_id` is valid for 24 hours. If the result has `warnings`, tell the customer where the file came from.
   - `FILE_TOO_LARGE`, `FILE_TYPE_NOT_ALLOWED`, `URL_NOT_ALLOWED`, `DOWNLOAD_FAILED` come from the upload: tell the customer what happened, then skip the file or try another link.
5. `aideliv_prepare_order`. It creates no order and charges nothing. Give the `file_id`s as `{"id": "..."}` in `photo_cargo`, `certificate` or `invoice_files` of the delivery they belong to.
   - On `VALIDATION_FAILED`, read `field_errors`, fix those fields with the customer and call again.
   - To change a draft, call again with its `draft_id` and the **complete** data. The old confirmation code stops working.
6. Show the `summary` (including `attachments`: the files attached to each address) and ask the customer to confirm. Point out anything they may not expect:
   - `minimum_weight_applied`: the platform charges at least `minimum_weight_kg`, so the weight was topped up;
   - `points.cost` and `points.charged_on_creation` (false means the order waits for account approval);
   - `bidding.opens_at` and `closes_at`, and `decision.until`: the deadline to confirm the final price.
7. After an explicit yes: `aideliv_create_order` with the `draft_id`, `draft_version` and `confirmation_code` of that draft.
   - `DRAFT_EXPIRED` or `DRAFT_STALE`: prepare again and let the customer confirm the new summary.
   - `INSUFFICIENT_POINTS`: tell them how many points are missing.
   - Calling it again for the same draft returns the same order (`already_created`), never a second one.
8. Tell the customer the order id (`formatted_id`), that the auction is running, and when bidding closes.

## Following an order

- `aideliv_list_orders` (filter by `status`) and `aideliv_get_order` (`order_id`). Statuses:
  - `approving`: waiting for account approval;
  - `processed`: bidding is open until `rate_deadline`;
  - `finished`: bidding closed; the customer must confirm the final price in the cabinet before `archives_at` (`decision.until`), or a new auction is needed;
  - `active`: the final price was confirmed, delivery is in progress;
  - `completed`: delivered; `archived`: closed without confirmation (or declined).
- `aideliv_list_rates` returns the current lowest price per shipping type: price, transit time, whether pick-up is included and the delivery addresses it covers.
  - While `processed`, say it is the current lowest price and may still go down until bidding closes.
  - Once `finished`, it is the final price: remind the customer to confirm it in the cabinet before the deadline.
- `aideliv_show_progress_statistics` shows where the cargo of `active` and `completed` orders is: the tracking step each delivery address has reached and the date of every step. In clients that support MCP Apps it renders as a live panel that refreshes itself; do not repeat its contents in text, just add a short summary. A step without a date has not happened yet.

## Accepting a bid

`aideliv_accept_rate` (`order_id`, `rate_id` from `aideliv_list_rates`, `suppliers`, `address_checks`) accepts the bid and moves the order to `active`, so the delivery starts. The server decides; you only report what the customer told you and what you checked:

1. Call it after the customer's yes. The server answers what is still missing, one thing at a time:
   - `SUPPLIERS_REQUIRED`: the factory contact is mandatory, as in the cabinet popup. Ask the customer for it: name (factory and contact person), an email or a phone in international format (`+86...`), optionally the factory's pick-up address (in the origin country) and a WeChat contact: `wechat.id` as text, or a QR image (upload it with `aideliv_upload_file`, type `wechat_qr`, jpg or png, from a link the customer gives, and pass its `file_id` as `wechat.file_id`; not both). If `details.suggested_supplier` is given, show it and let the customer confirm or correct it; never fill contacts in yourself.
   - `ADDRESS_CHECK_REQUIRED` with `details.addresses_to_check` (`delivery_address_id` and `address`): the delivery addresses the bid covers.
2. Check **each** of those addresses in Amazon with the customer's Amazon connector. Do not guess and do not skip one. If Amazon cannot be reached, tell the customer and stop: an unchecked address is never "valid".
3. Call again with `suppliers` and `address_checks`: one `address_checks` entry per address, `is_valid` true only if Amazon confirms it; for false add a `reason`. Keep sending the same `suppliers` on every call.
4. `ADDRESS_INVALID`: the bid was not accepted. Tell the customer which address is wrong and why. Do not retry with the same result and do not change the address yourself. `VALIDATION_FAILED` on `suppliers.N.*`: fix those fields with the customer.
5. On success the order is `active` and the contacts are saved with it; the next step is `aideliv_set_delivery_details` for each delivery address. Calling it again for the same bid returns `already_chosen: true` and changes nothing.
6. Other answers: `AUCTION_NOT_ACCEPTABLE` (bidding is not open for a choice, or the decision time is over: a new auction is needed), `RATE_ALREADY_CHOSEN` (another bid was accepted), `OVERDUE_INVOICE` (the customer must pay overdue invoices first, list them from `details.invoice_numbers`).

## Delivery details after the price is confirmed

When the order is `active`, `aideliv_set_delivery_details` adds details to one delivery address (`delivery_address_id` comes from `aideliv_get_order`):

- `address_type: "amazon"` with `shipments`: `shipment_id`, `amazon_reference_id`, `number_of_boxes`. Sending the list again replaces the earlier one for that address, so it is safe to repeat.
- `address_type: "private"` with city, zip code, street, contact person and an email or a phone.

Take every value from the customer or from their Amazon data. If the order is not `active`, the tool answers `ORDER_NOT_ACTIVE`.

If the tool answers `WAREHOUSE_MISMATCH`, the shipment points to a different warehouse than the one the auction priced, for example because Amazon reassigned it. Do not retry with other values. Explain the mismatch and tell the customer to contact AiDeliv support: the carrier will confirm or adjust the price, or a new auction is needed.

## Starting from Amazon data

If an Amazon connector is available, use it to read the customer's FBA shipment instead of retyping. Read only what the order needs: no sales, inventory or customer orders. A typical mapping, to confirm with the customer:

- pick-up address: the source (ship-from) address of the shipment, which must be outside the US. Copy its address fields (address line, city, district, state/province/region, ZIP/postal code) one by one; leave out the contact name, phone and e-mail;
- delivery addresses: one per destination fulfillment center, with its full address;
- cargo lines: from the boxes (units per box, weight, dimensions), with a plain description of the product from the listing;
- the order's category, HS code and anything Amazon does not give: ask.

Shipments Amazon creates itself, such as AWD-to-FBA replenishment (ship-from is an Amazon facility), are not for AiDeliv: skip them.

Never copy Amazon tokens or credentials anywhere, and never put them in an order.

## Errors you may meet

`FORBIDDEN` (the connection lacks permission), `ACCOUNT_BLOCKED`, `RATE_LIMITED` (wait `retry_after_seconds`), `NOT_FOUND` (also used for another customer's order), `MCP_DISABLED` (try later). Explain them plainly; do not retry blindly.
