# n8n WooCommerce order processing

A simple, ready-to-import [n8n](https://n8n.io) workflow that processes every new **WooCommerce** order automatically.

## What it does

When a new order is created in your shop:

1. **Logs the order** in a Google Sheet (one row per order: number, date, status, customer, email, phone, items, total, payment method, shipping address).
2. **Checks whether the order is paid** (WooCommerce status `processing` or `completed`).
3. **Paid order:**
   - sends an order confirmation email to the customer (items, total, shipping address);
   - sends an alert email to the shop owner;
   - tags the order in WooCommerce with a hidden `_n8n_processed_at` meta field, so you can see it was handled.
4. **Unpaid order** (`pending` or `on-hold`, for example bank transfer or cheque): sends an alert to the shop owner to check the payment before shipping.

No credentials or personal data are included in this repository.

## Files

| File | Purpose |
|---|---|
| `woocommerce-order-processing.json` | The n8n workflow to import |
| `orders-template.csv` | The column headers for the Google Sheet |

## Requirements

- An n8n instance reachable from the internet over **HTTPS** (WooCommerce must be able to call its webhook)
- A WooCommerce shop (WordPress) with the REST API enabled
- A Google account (Google Sheets)
- An SMTP account to send emails (your host's mailbox, Brevo, Mailgun, Gmail SMTP, etc.)

## Setup

### 1. Create the Google Sheet

1. Create a Google Sheet named, for example, **WooCommerce Orders**, with a tab named **Orders**.
2. Paste these headers in row 1 (or import `orders-template.csv`):

```
order_id | order_number | date | status | customer | email | phone | items | total | currency | payment_method | shipping_address
```

### 2. Create WooCommerce API keys

1. In WordPress, go to **WooCommerce → Settings → Advanced → REST API → Add key**.
2. Give it a description (for example *n8n*), choose your user and set **Permissions: Read/Write**.
3. Click **Generate API key** and copy the **Consumer key** and **Consumer secret**.

Read/Write is needed because the workflow creates the webhook in WooCommerce and tags processed orders.

### 3. Import the workflow into n8n

In n8n, click **Workflows → Add workflow → … menu → Import from File** and choose `woocommerce-order-processing.json` (or **Import from URL** with the raw GitHub link of the file).

### 4. Connect your accounts

Open each node with a warning icon and create or select its credential:

| Node | Credential |
|---|---|
| New WooCommerce order | **WooCommerce API**: consumer key, consumer secret, and your shop URL (for example `https://myshop.com`) |
| Mark order as processed by n8n | the same WooCommerce credential |
| Log order in Google Sheets | **Google Sheets OAuth2**, then pick your spreadsheet and the **Orders** tab |
| Send confirmation to customer, Notify shop owner (2 nodes) | **SMTP**: host, port, user, password |

### 5. Set your email addresses

In the three email nodes, replace the example addresses:

- `My Shop <shop@example.com>`: the sender (must be allowed by your SMTP account)
- `owner@example.com`: the address that receives the shop alerts

You can also adapt the subject and the HTML text of each email to your shop and language.

### 6. Test, then activate

1. Click **Execute workflow**: n8n waits for a test order.
2. Place a test order in your shop (for example with the *Cash on delivery* or *Direct bank transfer* method).
3. Check the Google Sheet row and the emails.
4. When everything works, switch the workflow to **Active** (Publish). n8n creates the WooCommerce webhook automatically; you can see it in **WooCommerce → Settings → Advanced → Webhooks**.

## How it works

```
New WooCommerce order ─► Prepare order data ─► Log order in Google Sheets ─► Is the order paid?
                                                                              ├─ yes ─► Confirmation to customer ─► Alert to owner ─► Tag order as processed
                                                                              └─ no  ─► Alert to owner: awaiting payment
```

## Customising

- **Different paid statuses:** edit the `is_paid` field in *Prepare order data*.
- **Notifications on Slack, Telegram or WhatsApp:** replace or duplicate the owner email nodes with the matching n8n node.
- **Automatically complete virtual or downloadable orders:** in *Mark order as processed by n8n*, add the update field **Status = completed** (only if all your products are digital).
- **More columns in the sheet:** add a field in *Prepare order data*, add the column in the sheet and map it in *Log order in Google Sheets*.

## Limitations

- The workflow reacts to **new orders** (`order.created`). An order paid later (for example a bank transfer validated the next day) only gets the "awaiting payment" alert. To also handle that case, duplicate the workflow with the `order.updated` event and add a check on the status change.
- WooCommerce already sends its own order emails. If you use this confirmation email, you may want to disable the default *Processing order* email in **WooCommerce → Settings → Emails** to avoid sending two.
- Your n8n instance must use HTTPS, or WooCommerce will refuse to deliver the webhook.

## License

[MIT](LICENSE)

## Author

Chafik Lounici — ITC Consulting
