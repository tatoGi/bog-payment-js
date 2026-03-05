# @tatogi/bog-payment-js

JavaScript client for Bank of Georgia iPay API.

## Install

```bash
npm install @tatogi/bog-payment-js
```

## Usage

```js
import { BogPaymentClient } from "@tatogi/bog-payment-js";

const bog = new BogPaymentClient({
  clientId: process.env.BOG_CLIENT_ID,
  clientSecret: process.env.BOG_CLIENT_SECRET
});

const order = await bog.createOrder(
  {
    callback_url: "https://example.com/bog/callback",
    external_order_id: "order-1001",
    purchase_units: {
      total_amount: 100,
      currency: "GEL",
      basket: [
        {
          product_id: "product-1",
          name: "T-Shirt",
          quantity: 1,
          unit_price: 100
        }
      ]
    },
    redirect_urls: {
      success: "https://example.com/checkout/success",
      fail: "https://example.com/checkout/fail"
    }
  },
  {
    idempotencyKey: crypto.randomUUID(),
    acceptLanguage: "en"
  }
);

console.log(order._links.redirect.href);
```

## Configuration

- `clientId` (required)
- `clientSecret` (required)
- `authUrl` (optional)  
  default: `https://oauth2.bog.ge/auth/realms/bog/protocol/openid-connect/token`
- `baseUrl` (optional)  
  default: `https://ipay.ge/opay/api/v1`
- `timeoutMs` (optional)  
  default: `30000`
- `fetchImpl` (optional) for custom runtime fetch

## Methods

- `getAccessToken(forceRefresh = false)`
- `createOrder(payload, { idempotencyKey, acceptLanguage } = {})`
- `getOrderDetails(orderId)`
- `payWithSavedCard(parentOrderId, payload, { idempotencyKey, acceptLanguage } = {})`
- `saveCard(orderId, { idempotencyKey } = {})`
- `deleteSavedCard(orderId, { idempotencyKey })`
- `confirmPreAuthorization(orderId, payload = {})`
- `rejectPreAuthorization(orderId, payload = {})`

## Test

```bash
npm test
```

## Manual Testing Without BOG Keys

If you do not have real BOG credentials yet, you can test integration flow with a local mock server.

1. Create `mock-server.mjs`:

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  res.setHeader("Content-Type", "application/json");

  if (req.method === "POST" && req.url === "/oauth/token") {
    return res.end(
      JSON.stringify({
        access_token: "mock_access_token",
        token_type: "Bearer",
        expires_in: 3600
      })
    );
  }

  if (req.method === "POST" && req.url === "/opay/api/v1/checkout/orders") {
    return res.end(
      JSON.stringify({
        id: "mock_order_123",
        status: "created",
        _links: { redirect: { href: "https://example.test/mock-checkout" } }
      })
    );
  }

  if (req.method === "GET" && req.url.startsWith("/opay/api/v1/checkout/payment/")) {
    const orderId = req.url.split("/").pop();
    return res.end(JSON.stringify({ id: orderId, status: "completed", amount: 100, currency: "GEL" }));
  }

  res.statusCode = 404;
  res.end(JSON.stringify({ message: "Not found" }));
});

server.listen(8787, () => {
  console.log("Mock BOG server is running on http://127.0.0.1:8787");
});
```

2. Run the mock server:

```bash
node mock-server.mjs
```

3. Point client to mock endpoints:

```js
import { BogPaymentClient } from "@tatogi/bog-payment-js";

const bog = new BogPaymentClient({
  clientId: "mock_client",
  clientSecret: "mock_secret",
  authUrl: "http://127.0.0.1:8787/oauth/token",
  baseUrl: "http://127.0.0.1:8787/opay/api/v1"
});
```

4. Test full flow manually:
- call `createOrder(...)` and verify `redirect` link
- call `getOrderDetails("mock_order_123")` and verify `status: completed`

## Publish

```bash
npm login
npm publish --access public
```

For automated publish from GitHub Actions:
- add repository secret `NPM_TOKEN`
- push a tag like `v1.0.0`
