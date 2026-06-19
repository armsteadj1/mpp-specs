# Using Stripe SPTs with the Generic SPT Method

This example shows how a server can advertise the generic `spt` payment
method while a client enabler fulfills the challenge with a Stripe Shared
Payment Token.

The important boundary is that Stripe-specific objects stay inside the
client-enabler and server-enabler adapters. The HTTP Payment challenge and
credential use the generic SPT shape.

## Flow

1. Server returns a `402 Payment Required` challenge with `method="spt"`.
2. The decoded request names Stripe as one supported processor and offers a
   Stripe-backed payment handler.
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
    "payeeId": "profile_merchant_123",
    "sessionId": "checkout_abc123",
    "usageCount": 1,
    "expiresAt": "2026-06-19T19:30:00Z"
  },
  "methodDetails": {
    "processors": [
      {
        "id": "stripe",
        "origin": "https://api.stripe.com",
        "environment": "production"
      }
    ],
    "payee": {
      "id": "profile_merchant_123",
      "displayName": "Example Merchant",
      "origin": "https://api.merchant.example",
      "country": "US",
      "category": "digital-services"
    },
    "paymentHandlers": [
      {
        "id": "stripe-spt-card-or-link",
        "processorId": "stripe",
        "usesDelegatedPayment": true,
        "credentialTypes": ["shared-payment-token"],
        "instrumentTypes": ["card", "wallet"],
        "pciScope": "token-only",
        "requiredInterventions": ["step-up-if-required"]
      }
    ],
    "acceptedInstrumentTypes": ["card", "wallet"],
    "tokenBinding": {
      "required": [
        "challenge-id",
        "amount",
        "currency",
        "payee",
        "expires"
      ],
      "recommended": ["realm", "request", "resource-origin"]
    },
    "assurance": {
      "payerInteraction": "step-up-if-required",
      "delegation": "payer-policy"
    }
  }
}
~~~

## Client Enabler: Create a Stripe SPT

The client enabler maps the generic allowance to Stripe SPT creation.
The exact Stripe API shape may change; this is intentionally illustrative.

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
    "paymentHandlerId": "stripe-spt-card-or-link",
    "tokenType": "shared-payment-token",
    "allowanceReference": "spt_1N4Zv32eZvKYlo2CPhVPkJlW",
    "clientReference": "client_attempt_456",
    "instrumentType": "card"
  }
}
~~~

## Server Enabler: Redeem Through Stripe Adapter

The server validates the generic challenge and credential first. Only after
that does it call the Stripe adapter.

~~~ javascript
const settlementPolicy = getTrustedSettlementPolicy({
  externalId: "order_12345",
  payeeId: "profile_merchant_123"
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
  "payeeId": "profile_merchant_123",
  "instrumentType": "card"
}
~~~

## Notes

The generic SPT method does not standardize Stripe objects. It standardizes the
challenge, credential, allowance, handler-selection, verification, settlement,
and receipt contracts around an opaque processor-issued SPT.

Other processors can use the same HTTP shape with their own token issuance and
redemption APIs.
