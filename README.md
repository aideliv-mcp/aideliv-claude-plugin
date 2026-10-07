# AiDeliv plugin for Claude

Create freight orders on [AiDeliv](https://aideliv.com), follow the rates carriers send, and add shipping details, from Claude.

AiDeliv is a freight platform: a customer creates an order, carriers send rates (bids) during a short window, and the customer chooses a rate in the AiDeliv cabinet. There is no price before rates arrive, so the plugin never estimates one.

## What is in the plugin

| Part | Name | What it does |
|---|---|---|
| MCP server | `aideliv` | Connects Claude to `https://aideliv.com/mcp` (Streamable HTTP, OAuth) |
| Skill | `aideliv-orders` | Explains how to create orders, follow rates and add shipping details without guessing data or spending points before the customer confirms |
| Command | `/aideliv-order` | Starts an order from a description or from an Amazon FBA shipment |
| Command | `/aideliv-status` | Shows the status and rates of the customer's orders |

## Install

Add the connection:

```
claude mcp add --transport http aideliv https://aideliv.com/mcp
```

Or run the whole plugin (connection, skill and commands) from a clone of this repository:

```
git clone https://github.com/aideliv-mcp/aideliv-claude-plugin
claude --plugin-dir ./aideliv-claude-plugin
```

Then open `/mcp`, choose `aideliv` and sign in with your AiDeliv customer account. Only customer accounts can use the connector.

## Tools

| Tool | Access | Purpose |
|---|---|---|
| `aideliv_get_account_context` | read | Account status, verification, points balance, whether an order can be created now |
| `aideliv_get_reference_data` | read | Ids for countries, shipping types, shipment types, categories and units |
| `aideliv_list_orders` | read | The customer's orders, newest first |
| `aideliv_get_order` | read | One order with its addresses and cargo totals |
| `aideliv_list_rates` | read | Rates carriers offered on one order |
| `aideliv_prepare_order` | write | Validates order data and saves a draft. Creates no order, charges nothing |
| `aideliv_create_order` | write | Creates the order from a confirmed draft and spends points |
| `aideliv_set_delivery_details` | write | Adds Amazon shipments or a private address to an order whose rate was chosen |

## Permissions (OAuth scopes)

| Scope | Allows |
|---|---|
| `mcp:use` | Connect to AiDeliv through MCP |
| `aideliv:read` | View your account, orders and rates |
| `aideliv:orders:write` | Create orders and add shipping details on your behalf |

You can connect with `mcp:use` and `aideliv:read` only. Then Claude can look at your orders and rates but cannot create or change anything.

## Data the plugin receives and sends

The plugin has no code of its own and no other servers. Everything goes between Claude and `https://aideliv.com/mcp`, under the signed-in customer's account.

**Received from AiDeliv**
- Account status, verification state and points balance.
- Reference lists (countries, shipping types, units, categories).
- The customer's own orders: status, countries, pick-up and delivery addresses, cargo totals and deadlines.
- Rates carriers offered on those orders: price in USD, delivery time, covered addresses. Carriers are anonymous.

**Sent to AiDeliv**
- Order data the customer gave: shipping type, countries, pick-up and delivery addresses, cargo lines (description, category, quantity, weight, volume, dimensions, HS code), invoice number.
- Shipping details for an order: Amazon shipment ids, Amazon reference ids and number of boxes, or a private delivery address with city, zip code, street, contact person, and an email or a phone.
- Drafts and confirmation codes returned by the server, to confirm an order.

**Not collected**
- The plugin does not read the customer's Amazon account. If the customer has a separate Amazon connector, Claude may read a shipment there and send only the values an order needs to AiDeliv. Amazon tokens and credentials are never copied anywhere.
- The plugin does not read local files and does not send data to any third party.

**Storage and logging on the AiDeliv side**
- Orders and shipping details are stored in the customer's AiDeliv account, as when they are created in the web cabinet.
- Unconfirmed drafts expire after 30 minutes.
- A server activity log records identifiers and outcomes of MCP calls (for example the order id and the kind of action), not addresses, contacts or prices.
- Access tokens are issued through OAuth with PKCE. An access token lasts one hour and a refresh token 30 days. Sign out of the connector in Claude to stop access.

## Safety rules built into the skill

- Prices, delivery times and fees are only quoted as the server returns them.
- Order data is never invented: Claude asks the customer for anything missing.
- An order is created, and points are spent, only after the customer has seen the server's summary and confirmed it explicitly.
- Choosing a rate is done by the customer in the AiDeliv cabinet. Claude cannot accept a rate.

## Support

[https://aideliv.com](https://aideliv.com)

## License

MIT. See [LICENSE](LICENSE).
