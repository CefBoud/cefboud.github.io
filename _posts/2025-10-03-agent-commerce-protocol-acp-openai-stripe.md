---
title: "Exploring the Agentic Commerce Protocol (ACP)"
author: cef
date: 2025-10-04
categories: [Technical Writing, Open Source]
tags: [ACP, AI, Agentic Commerce Protocol, Agentic Commerce, OpenAI, Stripe]
render_with_liquid: false
description: "Explore how the Agentic Commerce Protocol (ACP) lets AI agents shop on your behalf, from product discovery to secure checkout, using shared payment tokens and merchant integrations. This post breaks down the workflow, key components of AI-driven e-commerce."
---

> In case you're curious about how coding agents work, check out [this post](https://cefboud.com/posts/coding-agents-internals-opencode-deepdive/) where I explore an open-source agent internals.

<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/aciN7k0oY-k"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>

An AI Agent can complete tasks by calling tools(functions). It can look up the weather or perform a web search. Or as of late, make purchases on behalf of the user. Why? Because it's more convenient. Think: "Hey ChatGPT, buy me an X, my budget is Y, and I want something durable. Oh, and I prefer brands A and B." When it works well, it can be pretty handy.

![ACP Overview](/assets/chatgpt5-acp.png)


For e-commerce sellers, this translates to more sales. For AI companies and Payment Service Providers (PSPs) like Stripe and PayPal, it's a new opportunity for profit.

To streamline this whole process, OpenAI and Stripe partnered to release the Agentic Commerce Protocol (ACP), the subject of this very post. Shopify and Etsy joined the party on the merchant side of things. Here is the [announcement video](https://x.com/OpenAI/status/1972708279043367238).

I was pretty curious to understand how it actually works. So I took some time to go through the docs: [OpenAI](https://developers.openai.com/commerce/guides/get-started), [Stripe](https://docs.stripe.com/agentic-commerce), [GitHub](https://github.com/agentic-commerce-protocol/agentic-commerce-protocol/tree/main).


## The Challenge
There are many things that need to be solved for this to work. The agent/LLM needs to know about the products (description, shipping, availability, return policy, etc.) and stay up to date with changes.

The agent needs to index the product info and fetch the relevant details when the user requests them. Then a checkout session needs to be started with the merchant and updated based on the user's choices (shipping preferences, product options, etc.). Finally, the user's payment information needs to be handled securely, and their purchasing intent must be limited to the current checkout session. Updates about the order status also need to be communicated to the user (order confirmed, shipped, cancelled, etc.).

ACP strives to solve all of these in a flexible and secure manner.

## Protocol Overview
![ACP Overview](/assets/ACP.png)

### Product Feed

It all starts with the [Product Feed](https://developers.openai.com/commerce/specs/feed). OpenAI asks merchants to register at chatgpt.com/merchants to sync their products with ChatGPT. It's done via HTTPS with an initial load, followed by frequent refreshes (every 15 minutes per the doc). The product schema is pretty rich and described in this [Field Reference](https://developers.openai.com/commerce/specs/feed#field-reference). Here is a JSON-formatted subset:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "OpenAI Product (Trimmed)",
  "type": "object",
  "properties": {
    "enable_search": { "type": "string", "enum": ["true", "false"], "description": "Controls whether the product can be surfaced in ChatGPT search results" },
    "enable_checkout": { "type": "string", "enum": ["true", "false"], "description": "Allows direct purchase inside ChatGPT. Must be 'true' only if enable_search is also 'true'." },
    "id": { "type": "string", "maxLength": 100, "description": "Merchant product ID (unique, stable)" },
    "title": { "type": "string", "maxLength": 150 },
    "description": { "type": "string", "maxLength": 5000 },
    "link": { "type": "string", "format": "uri" },
    "image_link": { "type": "string", "format": "uri" },
    "price": { "type": "string", "description": "e.g. '79.99 USD'" },
    "sale_price": { "type": "string" },
    "inventory_quantity": { "type": "integer" },
    "product_category": { "type": "string" },
    "brand": { "type": "string", "maxLength": 70 },
    "weight": { "type": "string", "description": "e.g. '1.5 lb'" },
    "condition": { "type": "string", "enum": ["new", "refurbished", "used"] },
    "color": { "type": "string", "maxLength": 40 },
    "size": { "type": "string", "maxLength": 20 },
    "gender": { "type": "string", "enum": ["male", "female", "unisex"] },
    "seller_name": { "type": "string", "maxLength": 70 },
    "seller_url": { "type": "string", "format": "uri" },
    "return_policy": { "type": "string", "format": "uri" },
    "return_window": { "type": "integer", "minimum": 0 },
    "product_review_count": { "type": "integer" },
    "product_review_rating": { "type": "number" },
    "store_review_count": { "type": "integer" },
    "store_review_rating": { "type": "number" },
    "q_and_a": { "type": "string" },
    "raw_review_data": { "type": "string" }
  }
}
```

Basically, anything relevant to the shopping experience is there. `raw_review_data`, `q_and_a`, and other review-related fields are particularly interesting. This really brings in the personal shopper feel. Since the LLM can try to align user intent with product reviews, if (or when) AI shopping grows in use, there will be a great deal of incentive to game this review system. Having good reviews on a product or service would become even more impactful. Verified and authentic reviews would be even more valuable than they are today.

Anyway, the merchant makes a list of their products available in the above format to ChatGPT, which then proceeds to store and index this data.

### Checkout Session

When a user, while chatting with the LLM, expresses a desire to buy something, the model suggests a few products based on the indexed data. If the user picks one, the LLM starts a [checkout session](https://developers.openai.com/commerce/specs/checkout#checkout-session) with the merchant in what's dubbed “Agentic Checkout” (makes it sound so… interesting).

Request to `/checkout_sessions` from ChatGPT to the merchant:

```json
{
  "items":[{"id":"item_456","quantity":1}],
  "fulfillment_address":{"name":"test","line_one":"1234 Chat Road","line_two":"Apt 101","city":"San Francisco","state":"CA","country":"US","postal_code":"94131"}
}
```

Response:

```json
{
  "id": "checkout_session_123",
  "payment_provider": { "provider": "stripe", "supported_payment_methods": ["card"] },
  "status": "ready_for_payment",
  "currency": "usd",
  "line_items": [
    { "id": "line_item_456", "item": { "id": "item_456", "quantity": 1 }, "base_amount": 300, "discount": 0, "subtotal": 0, "tax": 30, "total": 330 }
  ],
  "fulfillment_address": { "name": "test", "line_one": "1234 Chat Road", "line_two": "Apt 101", "city": "San Francisco", "state": "CA", "country": "US", "postal_code": "94131" },
  "fulfillment_option_id": "fulfillment_option_123",
  "totals": [
    { "type": "items_base_amount", "display_text": "Item(s) total", "amount": 300 },
    { "type": "subtotal", "display_text": "Subtotal", "amount": 300 },
    { "type": "tax", "display_text": "Tax", "amount": 30 },
    { "type": "fulfillment", "display_text": "Fulfillment", "amount": 100 },
    { "type": "total", "display_text": "Total", "amount": 430 }
  ],
  "fulfillment_options": [
    { "type": "shipping", "id": "fulfillment_option_123", "title": "Standard", "subtitle": "Arrives in 4-5 days", "carrier": "USPS", "earliest_delivery_time": "2025-10-12T07:20:50.52Z", "latest_delivery_time": "2025-10-13T07:20:50.52Z", "subtotal": 100, "tax": 0, "total": 100 },
    { "type": "shipping", "id": "fulfillment_option_456", "title": "Express", "subtitle": "Arrives in 1-2 days", "carrier": "USPS", "earliest_delivery_time": "2025-10-09T07:20:50.52Z", "latest_delivery_time": "2025-10-10T07:20:50.52Z", "subtotal": 500, "tax": 0, "total": 500 }
  ],
  "messages": [],
  "links": [
    { "type": "terms_of_use", "url": "https://www.testshop.com/legal/terms-of-use" }
  ]
}
```

The merchant's response includes more information such as tax amounts, shipping options, etc. The LLM presents this information to the user in a friendlier manner (UI, text, etc.), and based on the user's choices (e.g., shipping option), it updates the session by calling `POST /checkout_sessions/{checkout_session_id}`.

### Payment Token

The user (now buyer) enters their card information into ChatGPT. When the buyer wants to pull the trigger on a purchase, the LLM will issue what is called a [`Shared Payment Token`](https://developers.openai.com/commerce/specs/payment) from the PSP (Payment Service Provider) such as Stripe. This is basically a token that represents the user's intent to buy from the seller. It works with that specific seller and has an amount limit (it can't be used for more than the purchase price), and it also expires after some time. This grants extra security — the token has a limited blast radius.

ChatGPT sends [the following request](https://github.com/agentic-commerce-protocol/agentic-commerce-protocol/blob/main/examples/examples.delegate_payment.json) to the PSP (Stripe):

`POST /agentic_commerce/delegate_payment`

```json
{
  "payment_method": {
    "type": "card", "card_number_type": "fpan", "virtual": false, "number": "4242424242424242",
    "exp_month": "11", "exp_year": "2026", "name": "Jane Doe", "cvc": "223",
    "checks_performed": ["avs", "cvv"], "iin": "424242",
    "display": { "brand": "visa", "last4": "4242", "funding_type": "credit", "wallet_type": "apple_pay" },
    "metadata": { "issuing_bank": "temp" }
  },
  "allowance": {
    "reason": "one_time", "max_amount": 2000, "currency": "usd",
    "checkout_session_id": "csn_01HV3P3XYZ9ABC", "merchant_id": "testshop",
    "expires_at": "2025-10-09T07:20:50.52Z"
  },
  "risk_signals": [{ "type": "card_testing", "score": 10, "action": "manual_review" }],
  "metadata": { "campaign": "q4", "source": "chatgpt_checkout" }
}
```

Notice the `allowance` field, where we put constraints on the generated token. This token is basically a temporary permission to spend up to the `max_amount` with `merchant_id` before `expires_at` during `checkout_session_id`.

Response:

```json
{
  "id": "vt_01J8Z3WXYZ9ABC",
  "created": "2025-09-29T11:00:00Z",
  "metadata": {
    "source": "agent_checkout",
    "merchant_id": "testshop",
    "idempotency_key": "idem_abc123"
  }
}
```

This is the happy path. There are also requests for rejected payments and errors. Based on these, the LLM would inform the buyer so they can address the problem.

From OpenAI and Stripe's docs, it's unclear whether the id (`vt_01J8Z3WXYZ9ABC`) is the actual token or if there's some mechanism to retrieve the token using that ID.

Anyway, with the token in hand (figuratively, of course), the LLM then completes the checkout by sending a POST request to `/checkout_sessions/checkout_session_123/complete` and providing the shared token.

```json
{
  "buyer": {
    "first_name": "John",
    "last_name": "Smith",
    "email": "johnsmith@mail.com",
    "phone_number": "15552003434"
  },
  "payment_data": {
    "token": "spt_123",
    "provider": "stripe",
    "billing_address": {
      "name": "test",
      "line_one": "1234 Chat Road",
      "line_two": "Apt 101",
      "city": "San Francisco",
      "state": "CA",
      "country": "US",
      "postal_code": "94131"
    }
  }
}
```

The merchant can [redeem the token](https://docs.stripe.com/agentic-commerce/concepts/shared-payment-tokens#use-a-sharedpaymenttoken) by creating a `PaymentIntent`, which is the same way they handle other non-agentic payments with their PSP.

### Webhook

As the order status changes, the merchant sends requests (webhook events) to an [endpoint exposed by OpenAI](https://developers.openai.com/commerce/specs/checkout#webhooks) so the customer receives updates on their order (`confirmed`, `shipped`, `cancelled`, etc.).

Refunds are also notified via this webhook.
The [PSP](https://docs.stripe.com/agentic-commerce/concepts/shared-payment-tokens#webhooks-deactivated) (Stripe) also sends events to the agent when the token has been used, so it can notify the user of the payment processing.



# Closing Thoughts
The protocol is rather simple and elegant. It's really quite interesting how an LLM's ability to call tools/functions imbues it with so many capabilities, and once people develop protocols and standards, great things can be achieved — such as an AI that shops on people's behalf. I'm really looking forward to seeing how this plays out once it's mass deployed.

Another thing to note is that the PSP acts as the actual source of trust, issuing shared tokens that are limited in scope and risk. One can imagine a different decentralized, trustless layer. The future will be full of surprises!


If you want to get in touch, you can find me on Twitter/X at [@moncef_abboud](https://x.com/moncef_abboud).  
