# Using Stripe SPTs with the Generic SPT Method

This example shows how a server can advertise the generic `spt` payment
method while a client enabler fulfills the challenge with a Stripe Shared
Payment Token.

The important boundary is that Stripe-specific objects stay inside the
client-enabler and server-enabler adapters. The HTTP Payment challenge and
credential use the generic SPT shape.

## Stripe Mapping Summary

This example intentionally does not copy Stripe field names into the generic
method. The mapping below shows which parts come directly from the Stripe SPT
draft and which parts are generic additions for other processors.

| Generic SPT field | Stripe SPT draft equivalent | Notes |
| --- | --- | --- |
| `amount` | `amount` | Direct mapping. Stripe uses the amount when creating the payment after SPT issuance. |
| `currency` | `currency` | Direct mapping. Also maps into `usage_limits.currency` during SPT creation. |
| `description` | `description` | Direct mapping for payer display/context. |
| `externalId` | `externalId` | Direct mapping at the challenge level. Stripe's credential payload also has optional `externalId`; the generic credential names this `clientReference` to avoid confusing server and client references. |
| `allowance.maxAmount` | `usage_limits.max_amount` | Direct mapping. This is the maximum amount the SPT may authorize. |
| `allowance.currency` | `usage_limits.currency` | Direct mapping. |
| `allowance.expiresAt` | `usage_limits.expires_at` | Direct mapping. |
| `allowance.recipientId` | `seller_details.networkId` / `methodDetails.networkId` | Stripe binds issuance to the seller's Business Network Profile ID. Generic SPT calls this recipient scope because other processors may use merchant IDs, seller IDs, account IDs, or profile IDs. |
| `allowance.reason` | No direct Stripe field | Generic addition for delegated-payment policy and future recurring, metered, or session-scoped processors. |
| `sessionId` | No direct Stripe field | Generic addition for checkout/session binding and server reconciliation. A Stripe adapter can copy it into metadata. |
| `methodDetails.processors[].id` | Implied by `method="stripe"` in Stripe draft | Generic addition so one `method="spt"` challenge can offer Stripe or another processor without creating one method per processor. |
| `methodDetails.processors[].profile` | No direct Stripe field | Generic addition for processor-declared SPT capability/profile selection. |
| `methodDetails.recipient.id` | `methodDetails.networkId` / `seller_details.networkId` | Direct conceptual mapping, but generic SPT makes it optional because some processor profiles may imply merchant/account scope. |
| `methodDetails.recipient.displayName` | No direct Stripe field | Generic optional display/safety context. |
| `methodDetails.tokenBinding` | No direct Stripe field | Generic addition to make anti-replay and challenge binding explicit across processors. |
| `payload.sharedPaymentToken` | `payload.spt` | Direct mapping with a processor-neutral name. |
| `payload.processorId` | Implied by `method="stripe"` in Stripe draft | Generic addition so the server knows which processor adapter must redeem the opaque SPT. |
| `payload.tokenType` | No direct Stripe field | Generic optional discriminator. Defaults to `shared-payment-token`. |
| `payload.allowanceReference` | Stripe SPT ID | Generic optional reference to the allowance/token issuance result. |
| `payload.clientReference` | `payload.externalId` | Direct conceptual mapping, renamed to avoid collision with server-side `externalId`. |
| `processorPayload` | No direct Stripe field | Generic extension escape hatch. Stripe-specific data should stay here or inside the adapter, not become generic wire vocabulary. |

Stripe-specific fields that the generic method intentionally does not expose:

* `paymentMethodTypes`: handled by the processor/profile. Generic SPT should not
  force the merchant to enumerate card, Link, bank account, or wallet routing.
* `payment_method`: selected inside the client/processor flow, not in the HTTP
  challenge.
* PaymentIntent and Connect settlement parameters: derived from trusted
  server-side settlement policy after the generic credential is validated.
* Payer authentication and risk details: handled by Stripe.js and Stripe. The
  generic credential MUST NOT carry raw authentication or risk values.

## Flow

1. Server returns a `402 Payment Required` challenge with `method="spt"`.
2. The decoded request names Stripe as one supported SPT processor.
3. The client enabler creates a Stripe SPT scoped to the challenge allowance.
4. The client submits the generic SPT credential containing the Stripe SPT.
5. The server enabler redeems the token through its Stripe adapter.
6. The server returns the protected resource and a generic payment receipt.

## Challenge

~~~ http
HTTP/1.1 402 Payment Required
Cache-Control: no-store
Content-Type: application/problem+json
WWW-Authenticate: Payment id="ch_spt_123",
  realm="api.merchant.example",
  method="spt",
  intent="charge",
  expires="2026-06-19T19:30:00Z",
  request="<base64url-jcs-json>"

{
  "type": "https://paymentauth.org/problems/payment-required",
  "title": "Payment Required",
  "status": 402,
  "detail": "This resource requires payment."
}
~~~

Decoded `request`:

~~~ json
{
  "amount": "5000",
  "currency": "usd",
  "description": "Premium API access for 1 month",
  "externalId": "order_12345",
  "sessionId": "checkout_abc123",
  "allowance": {
    "reason": "one-time",
    "maxAmount": "5000",
    "currency": "usd",
    "recipientId": "profile_merchant_123",
    "sessionId": "checkout_abc123",
    "expiresAt": "2026-06-19T19:30:00Z"
  },
  "methodDetails": {
    "processors": [
      {
        "id": "stripe",
        "origin": "https://api.stripe.com",
        "profile": "spt-charge-2026-06",
        "environment": "production"
      }
    ],
    "recipient": {
      "id": "profile_merchant_123",
      "displayName": "Example Merchant",
      "origin": "https://api.merchant.example",
      "country": "US",
      "category": "digital-services"
    },
    "tokenBinding": {
      "required": [
        "challenge-id",
        "amount",
        "currency",
        "recipient",
        "expires"
      ],
      "recommended": ["realm", "request", "resource-origin"]
    }
  }
}
~~~

## Client Enabler: Create a Stripe SPT

The client enabler maps the generic allowance to Stripe SPT creation.
The exact Stripe API shape may change; this is intentionally illustrative.
The generic `recipient.id` maps to Stripe's business/network profile identifier
used in `seller_details.networkId`.
Stripe decides whether additional authentication or risk checks are required
before issuing the SPT. The generic challenge does not attempt to require or
prove those processor-side decisions.

~~~ javascript
const sharedPaymentToken = await stripe.sharedPayment.issuedTokens.create({
  payment_method: "pm_123",
  usage_limits: {
    currency: "usd",
    max_amount: 5000,
    expires_at: Math.floor(Date.parse("2026-06-19T19:30:00Z") / 1000)
  },
  seller_details: {
    networkId: "profile_merchant_123"
  },
  metadata: {
    challenge_id: "ch_spt_123",
    session_id: "checkout_abc123",
    external_id: "order_12345"
  }
});
~~~

## Credential

The HTTP credential is still generic. The Stripe SPT value is carried as the
opaque `sharedPaymentToken`.

~~~ json
{
  "challenge": {
    "id": "ch_spt_123",
    "realm": "api.merchant.example",
    "method": "spt",
    "intent": "charge",
    "expires": "2026-06-19T19:30:00Z",
    "request": "<base64url-jcs-json>"
  },
  "payload": {
    "sharedPaymentToken": "spt_1N4Zv32eZvKYlo2CPhVPkJlW",
    "processorId": "stripe",
    "tokenType": "shared-payment-token",
    "allowanceReference": "spt_1N4Zv32eZvKYlo2CPhVPkJlW",
    "clientReference": "client_attempt_456"
  }
}
~~~

## Server Enabler: Redeem Through Stripe Adapter

The server validates the generic challenge and credential first. Only after
that does it call the Stripe adapter.

~~~ javascript
const settlementPolicy = getTrustedSettlementPolicy({
  externalId: "order_12345",
  recipientId: "profile_merchant_123"
});

const paymentIntent = await stripe.paymentIntents.create(
  {
    amount: 5000,
    currency: "usd",
    confirm: true,
    shared_payment_granted_token: "spt_1N4Zv32eZvKYlo2CPhVPkJlW",
    automatic_payment_methods: {
      enabled: true,
      allow_redirects: "never"
    },
    metadata: {
      challenge_id: "ch_spt_123",
      session_id: "checkout_abc123",
      external_id: "order_12345"
    }
  },
  {
    idempotencyKey: "spt-charge-ch_spt_123_<token-hash>",
    stripeAccount: settlementPolicy.stripeAccount
  }
);
~~~

The server grants access only if the adapter returns a successful settlement
outcome that satisfies the generic SPT charge rules.

## Receipt

Decoded `Payment-Receipt`:

~~~ json
{
  "method": "spt",
  "status": "success",
  "timestamp": "2026-06-19T19:28:11Z",
  "reference": "pi_3N4Zv32eZvKYlo2C0abc1234",
  "processorId": "stripe",
  "amount": "5000",
  "currency": "usd",
  "externalId": "order_12345",
  "recipientId": "profile_merchant_123"
}
~~~

## Notes

The generic SPT method does not standardize Stripe objects. It standardizes the
challenge, credential, allowance, processor-selection, verification, settlement,
and receipt contracts around an opaque processor-issued SPT.

Other processors can use the same HTTP shape with their own token issuance and
redemption APIs.
