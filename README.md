# AiDeliv plugin for Claude

Launch freight auctions on [AiDeliv](https://aideliv.com) from Claude: prepare a delivery order, follow the current lowest price while carriers bid, and add delivery details after you confirm the final price.

AiDeliv is a technology marketplace built on a patent-pending auction engine. Each delivery order runs as an auction:

- Pre-vetted carriers bid against each other for 9 hours.
- You always see the current lowest price, and it updates whenever a carrier bids lower.
- When bidding closes, the lowest bid is your final price. You confirm it in your AiDeliv account within 24 hours; if you don't, the auction closes and a new auction is needed.

There is no price before the auction, so the plugin never estimates one.

Ship from China, Vietnam, India, Taiwan or Bangladesh to the US, Canada, the UK, Europe and more. Deliver to Amazon FBA and AWD, Walmart Fulfillment Services, a 3PL warehouse, or your own warehouse or business address. Shipments within the US are not available at the moment.

## What is in the plugin

| Part | Name | What it does |
|---|---|---|
| MCP server | `aideliv` | Connects Claude to `https://aideliv.com/mcp` (Streamable HTTP, OAuth) |
| Skill | `aideliv-orders` | Explains how to create delivery orders, follow auctions and add delivery details without guessing data or spending points before the customer confirms |
| Command | `/aideliv-order` | Starts a delivery order from a description or from an Amazon FBA shipment |
| Command | `/aideliv-status` | Shows the status and current lowest price of the customer's orders |

## Install

Add the plugin from the Claude directory, or add the connection only:

```
claude mcp add --transport http aideliv https://aideliv.com/mcp
```

Or run the whole plugin (connection, skill and commands) from a clone of this repository:

```
git clone https://github.com/aideliv-mcp/aideliv-claude-plugin
claude --plugin-dir ./aideliv-claude-plugin
```

Then open `/mcp`, choose `aideliv` and sign in with your AiDeliv shipper account. Only shipper accounts can use the connector.

## Tools

| Tool | Title | Access | Purpose |
|---|---|---|---|
| `aideliv_get_account_context` | Get account status | read | Account status, verification, points balance, whether an order can be created now |
| `aideliv_get_reference_data` | Get reference data | read | Ids for countries, shipping types, shipment types, categories, units and land routes |
| `aideliv_list_orders` | List delivery orders (auctions) | read | The customer's orders, newest first |
| `aideliv_get_order` | Get delivery order (auction) details | read | One order with its addresses, cargo totals and bidding deadline |
| `aideliv_list_rates` | Get current lowest price (auction) | read | The current lowest bid while bidding is open; the final price once it closes |
| `aideliv_show_progress_statistics` | Show delivery progress statistics | read | Live "Progress statistics" panel (MCP Apps): tracking step and dates per delivery address of active and completed orders |
| `aideliv_upload_file` | Upload a file for an order | write | Downloads a photo, certificate or invoice from a public https link and returns a `file_id` to attach to a draft |
| `aideliv_prepare_order` | Prepare delivery order draft | write | Validates order data and saves a draft. Creates no order, charges nothing |
| `aideliv_create_order` | Create delivery order and start auction | write | Creates the order from a confirmed draft, starts the auction and spends points |
| `aideliv_accept_rate` | Accept a bid and start the delivery | write | Accepts the customer's chosen bid and moves the order to active, once the factory contact is given and the server has the Amazon check of every delivery address |
| `aideliv_set_delivery_details` | Add delivery details to an order | write | Adds Amazon shipments or a private address after the final price is confirmed |

## Permissions (OAuth scopes)

| Scope | Allows |
|---|---|
| `mcp:use` | Connect to AiDeliv through MCP |
| `aideliv:read` | View your account, orders and prices |
| `aideliv:orders:write` | Create orders and add delivery details on your behalf |

You can connect with `mcp:use` and `aideliv:read` only. Then Claude can look at your orders and prices but cannot create or change anything.

## Data the plugin receives and sends

The plugin has no code of its own and no other servers. Everything goes between Claude and `https://aideliv.com/mcp`, under the signed-in customer's account.

**Received from AiDeliv**
- Account status, verification state and points balance.
- Reference lists (countries, shipping types, units, categories).
- The customer's own orders: status, countries, pick-up and delivery addresses, cargo totals and deadlines.
- The current lowest price of each auction: price in USD, delivery time, covered addresses. Carriers are anonymous, and higher bids are not shown.

**Sent to AiDeliv**
- Order data the customer gave: shipping type, countries, pick-up and delivery addresses, cargo lines (description, category, quantity, weight, volume, dimensions, HS code), invoice number.
- Links to photos, certificates, packing lists or invoices the customer wants attached to an order (for example from an Amazon listing). AiDeliv downloads each file itself from the https link; the plugin does not read or send the file contents.
- Delivery details for an order: Amazon shipment ids, Amazon reference ids and number of boxes, or a private delivery address with city, zip code, street, contact person, and an email or a phone.
- Drafts and confirmation codes returned by the server, to confirm an order.

**Not collected**
- The plugin does not read the customer's Amazon account. If the customer has also connected the Amazon Selling Partner connector in Claude, Claude may read a shipment there and send only the values an order needs to AiDeliv. Sales, inventory and buyer orders are not read. Amazon tokens and credentials are never copied anywhere.
- The plugin does not read local files and does not send data to any third party.

**Storage and logging on the AiDeliv side**
- Orders and delivery details are stored in the customer's AiDeliv account, as when they are created in the web cabinet.
- Unconfirmed drafts expire after 30 minutes.
- An uploaded file that is not attached to an order within 24 hours is deleted.
- A server activity log records identifiers and outcomes of MCP calls (for example the order id and the kind of action), not addresses, contacts or prices.
- Access tokens are issued through OAuth with PKCE. An access token lasts one hour and a refresh token 30 days. Sign out of the connector in Claude to stop access.

## Safety rules built into the skill

- Prices, delivery times and fees are quoted only as the server returns them.
- Order data is never invented: Claude asks the customer for anything missing.
- Photos and certificates are attached only if they exist: Claude uses links taken from the page, never invents a document, and tells the customer when a file comes from an unfamiliar source.
- An auction is started only when the cargo is ready to ship.
- An order is created, and points are spent, only after the customer has seen the server's summary and confirmed it explicitly.
- The final price is confirmed by the customer in the AiDeliv account. Claude cannot confirm it.

## Works alongside Amazon

Works alongside the Amazon Selling Partner connector in Claude: if you have connected it, Claude can read your FBA shipment details and fill in the delivery order for you.

Amazon and all related marks are trademarks of Amazon.com, Inc. or its affiliates. Walmart is a trademark of Walmart Inc. AiDeliv is not affiliated with or endorsed by Amazon or Walmart.

## Support

[https://aideliv.com](https://aideliv.com) · support@aideliv.com

## License

MIT. See [LICENSE](LICENSE).
