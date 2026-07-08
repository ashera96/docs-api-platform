---
title: "Monetize an AI API with Stripe"
description: "Charge for an Anthropic Claude AI API without writing billing code: publish an AI proxy with a Pay-as-you-go plan, cap LLM cost with token-based rate limiting, and let WSO2 API Platform sync per-request usage to Stripe for metering and payments."
canonical_url: https://wso2.com/api-platform/docs/guides/monetization/ai-api-monetization/
md_url: https://wso2.com/api-platform/docs/guides/monetization/ai-api-monetization.md
tags:
  - guides
  - monetization
  - stripe
  - ai-gateway
  - anthropic
author: WSO2 API Platform Documentation Team
last_updated: 2026-07-06
content_type: "tutorial"
---

# Monetize an AI API with Stripe

## Overview

This guide shows you how to charge consumers for access to an LLM without writing any billing code. You front the Anthropic Claude API with a WSO2 API Platform AI proxy, connect Stripe, and publish a paid subscription plan. WSO2 API Platform syncs usage to Stripe, handles payment processing, enforces subscription access, and gives consumers a self-service billing portal — while a token-based rate limit protects you from runaway LLM cost.

By the end, you'll have a published Claude AI proxy with a Pay-as-you-go plan, a token quota on every subscription, and per-request Stripe metering on every authenticated call.

---

## Prerequisites

- A WSO2 API Platform account. Sign up for free at [console.bijira.dev](https://console.bijira.dev).
- An Anthropic API key for the backend Claude API. Create one in the [Anthropic Console](https://console.anthropic.com/).
- A [Stripe account](https://dashboard.stripe.com/) with your **Publishable Key** and **Secret Key**. These are available in the Stripe Dashboard under **Developers > API Keys**.
- `curl` for testing.

---

## Architecture

```text
AI API consumers (developers / apps)

    |  HTTPS + OAuth2 bearer token
    v

+-----------------------------------+
|         WSO2 AI Gateway            |
|  auth · token rate limit · audit  |
+-----------------------------------+

    |  HTTPS (backend API key)        |  usage records
    v                                 v

Anthropic Claude API           +---------------------------+
                               |          Stripe           |
                               |  metering · invoicing     |
                               |  payment · portal         |
                               +---------------------------+
```

Consumers subscribe to a plan in the Developer Portal, pay via Stripe checkout, and call the AI API with an OAuth2 bearer token. The gateway injects the backend Anthropic key, enforces the token-based rate limit, and reports usage records to Stripe, which handles invoicing, payment collection, and the consumer billing portal. WSO2 API Platform never touches payment details, and consumers never see your Anthropic key.

---

## Step 1: Create and deploy the Claude AI proxy

Create an AI proxy for the Anthropic Claude provider so the gateway fronts Claude and injects your backend key.

Follow [Create an API Proxy for an Anthropic Claude AI API](../../cloud/create-api-proxy/third-party-apis/claude.md) to create the proxy, configure the backend endpoint and Anthropic API key, and deploy it to Production. Return here once the **Production** deployment status shows **Active**.

!!! note
    Monetization is applied on the AI proxy, not on the backend. Because the gateway authenticates every consumer and meters every call, you can bill each consumer accurately without exposing your Anthropic key or changing the backend.

---

## Step 2: Cap LLM cost with a token-based rate limit

Unlike a REST API, an LLM charges you per token, so a single expensive prompt can cost far more than a single cheap one. Attach a token-based rate limit to the AI proxy so each subscription is bounded by a token quota, not just a request count.

1. In the left navigation menu of your Claude AI proxy, click **Manage**, then click **Policies**.
2. Attach the **Token-Based Rate Limit** policy.
3. Set a per-subscription token quota and time unit — for example, `100000` tokens per **Month**.
4. Click **Save** and redeploy the proxy if prompted.

**Expected result:** The gateway counts prompt and completion tokens per subscription and blocks calls once the quota is exhausted.

!!! tip
    Pair the token quota with the request-count limit in your subscription plan (Step 4). The request count controls how *often* a consumer can call the API; the token limit controls how *much* LLM work each consumer can consume — the real driver of your Anthropic bill. For details, see [Token-Based Rate Limit for AI APIs](../../cloud/create-api-proxy/third-party-apis/token-ratelimit.md).

---

## Step 3: Connect Stripe

Connect your Stripe account at the organization level. The platform uses these credentials to create Stripe products, prices, and usage records on your behalf — you don't write any billing code.

1. In the API Platform Console header, go to the **Organization** list and select your organization.
2. In the left navigation menu, click **Admin**, then click **Settings**.
3. Click the **Credentials** tab, then the **Stripe Credentials** sub-tab.
4. Click **Add Stripe Credentials**.
5. Enter your **Secret Key** and **Publishable Key** from the [Stripe Dashboard](https://dashboard.stripe.com/).
6. Click **Save**.

**Expected result:** Your Stripe account is connected and ready to process payments for AI API subscriptions.

!!! note
    You can use a Standard secret key or a Restricted secret key. For Restricted keys, see [Getting Started with API Monetization](../../monetization/getting-started.md) for the minimum permissions required.

---

## Step 4: Create a subscription plan

Create a Pay-as-you-go subscription plan that charges consumers per API request.

1. In the left navigation menu, click **Admin**, then click **Settings**.
2. Click the **Subscription Plans** tab, then click **+ Create**.
3. Enter the following details:

    | Field | Value |
    |---|---|
    | **Name** | Pay-as-you-go |
    | **Description** | Pay $0.05 per Claude request, billed monthly |
    | **Request Count** | 1000 |
    | **Request Count Time Unit** | Minute |
    | **Pricing Model** | Unit |
    | **Currency** | USD |
    | **Billing Period** | Monthly |
    | **Unit Amount** | 0.05 |

4. Click **Create**.

**Expected result:** The Pay-as-you-go plan appears in the Subscription Plans list.

!!! note
    In usage-based models, a unit equals one API request. With the Unit model, the total charge is request count × unit price. For tiered pricing, use Volume or Graduated instead. Set the unit price above your expected per-call Anthropic cost so each billed request stays profitable.

---

## Step 5: Enable the plan on the AI API

Enable the Pay-as-you-go plan on the Claude AI proxy so consumers can discover and subscribe to it in the Developer Portal.

1. In the left navigation menu of your Claude AI proxy, click **Manage**, then click **Monetize**.
2. Enable the toggle for **Pay-as-you-go**.
3. Click **Save**.

**Expected result:** The Pay-as-you-go plan is active on the AI API and appears in the Developer Portal subscription dialog.

!!! tip
    You can enable multiple plans on the same AI API to offer tiered pricing — for example, a free tier with a small token quota alongside this pay-as-you-go tier — without managing separate proxies.

---

## Step 6: Publish the AI API to the Developer Portal

Publish the AI API so consumers can discover it, review available plans, and subscribe.

1. In the left navigation menu, click **Manage**, then click **Lifecycle**.
2. Click **Publish**.
3. In the **Publish API** dialog, confirm the details and click **Confirm**.

**Expected result:** The lifecycle state changes to **Published** and the AI API appears in the Developer Portal.

---

## Step 7: Subscribe to the paid plan

As an API consumer, subscribe to the Pay-as-you-go plan through the Developer Portal.

1. Go to [devportal.bijira.dev](https://devportal.bijira.dev) and sign in.
2. In the sidebar, click **APIs**.
3. Find the Claude AI API and click **Subscribe**.
4. In the **Choose Your Subscription Plan** dialog, select an application from the dropdown, or create one named `Claude App`.
5. Click **Subscribe** on the **Pay-as-you-go** plan.
6. A Stripe checkout window opens. Enter your payment details and complete the payment.

**Expected result:** The application is subscribed to the Claude AI API under the Pay-as-you-go plan. The Stripe checkout confirmation is shown.

!!! note
    Payments are processed by Stripe, and WSO2 API Platform doesn't store your payment details. If your Stripe account is in test mode, use the test card `4242 4242 4242 4242` with any future expiry date and any CVC to complete checkout.

---

## Step 8: Generate API credentials

Generate OAuth2 credentials for your application so you can make authenticated AI API calls.

1. In the Developer Portal, click **Applications** and open **Claude App**.
2. Click **Manage Keys**.
3. On the **Production** tab, click **Generate Key**.
4. Copy the **Consumer Key** and **Consumer Secret**.
5. Click **Generate** to generate an access token and copy it.

**Expected result:** You have an access token ready to use in AI API calls.

!!! note
    Bearer tokens expire after 3600 seconds by default. To extend the lifetime, click **Modify** on the Manage Keys page and increase the **Application Access Token Expiry Time** before generating.

---

## Step 9: Invoke the AI API

Make your first authenticated call to the monetized Claude AI API using the access token. The gateway authenticates the request, injects your backend Anthropic key, and forwards it to Claude.

```bash
curl -X POST https://<your-gateway-url>/<proxy-context>/v1/messages \
  -H "Authorization: Bearer <your-token>" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 256,
    "messages": [
      { "role": "user", "content": "In one sentence, what is API monetization?" }
    ]
  }'
```

Replace `<your-gateway-url>` and `<proxy-context>` with the endpoint from the AI API's **Overview** page in the Developer Portal, and `<your-token>` with the access token you just copied.

**Expected result:** `HTTP 200` with a JSON message from Claude. Each successful call increments your usage meter in Stripe and counts against your token quota.

---

## Step 10: View billing and usage

After making AI API calls, check your usage, upcoming charges, and invoices from the Developer Portal.

1. In the Developer Portal, click your profile icon in the top right corner.
2. Click **Billing**.
3. On the **Usage Summary** tab, review your **Total Billed API Calls**, **Active Subscriptions**, and **Estimated Cost** for the current period.
4. Click the **Invoices** tab to view and download past invoices.
5. Click the **Billed Subscriptions** tab to see subscription details and click **Manage** to update or cancel.

**Expected result:** Your Claude AI API calls appear in the usage breakdown with the Pay-as-you-go plan, and the estimated cost reflects your request count × $0.05.

!!! note
    Allow a few minutes for usage to appear in the Billing page after the first request.

---

## Verify

1. Confirm authenticated calls are billed. Call the AI API with your token and then check the Usage Summary — the call count should increment.

    **Expected result:** The request count in **Billing > Usage Summary** increases by the number of calls made.

2. Confirm the token quota is enforced. Make repeated calls until the token quota configured in Step 2 is exhausted.

    **Expected result:** The gateway returns `HTTP 429` Too Many Requests once the token limit for the subscription is reached, protecting you from further LLM cost.

3. Confirm unauthenticated requests are rejected. Call the AI API without an Authorization header.

    ```bash
    curl -v -X POST https://<your-gateway-url>/<proxy-context>/v1/messages
    ```

    **Expected result:** `HTTP 401` Unauthorized.

4. Confirm subscription enforcement. In the Developer Portal, cancel the Pay-as-you-go subscription under **Billing > Billed Subscriptions > Manage > Cancel**. Then attempt to call the AI API with the same token.

    **Expected result:** `HTTP 403` Forbidden — the gateway revokes access when the subscription lapses.

---

## Troubleshooting

| Symptom | Resolution |
|---|---|
| Stripe Credentials save fails | Confirm your Secret Key and Publishable Key are from the same Stripe account and that the secret key has the required permissions. |
| AI API calls return `HTTP 401` from the backend | Confirm the backend Anthropic API key is configured on the proxy and is valid in the Anthropic Console. |
| Plans don't appear in Developer Portal subscription dialog | Confirm the plan toggles are enabled in **Manage > Monetize** and the AI API is in the **Published** lifecycle state. |
| Stripe checkout doesn't open | Confirm your Stripe account is active and not in restricted mode. Check the Stripe Dashboard for any pending account verification. |
| `HTTP 401` on every call after subscribing | Confirm your access token is not expired. Generate a new token in **Manage Keys** and update your request. |
| `HTTP 403` after subscribing | Confirm the subscription is active in **Billing > Billed Subscriptions**. If payment failed, the subscription may be inactive. |
| `HTTP 429` sooner than expected | Confirm the token quota in the Token-Based Rate Limit policy matches your intended per-subscription budget. Large prompts and completions consume tokens quickly. |
| Usage not appearing in Billing page | Allow a few minutes after making calls. Usage reporting to Stripe is near-real-time but not instantaneous. |

---

## What you learned

- Fronted the Anthropic Claude API with a WSO2 API Platform AI proxy so the gateway authenticates consumers and injects the backend key.
- Capped per-subscription LLM cost with a token-based rate limit — the metering dimension that matters most for AI APIs.
- Connected a Stripe account so the platform manages all billing infrastructure without custom code.
- Created a Unit-priced Pay-as-you-go plan, then enabled and published it to the Developer Portal for consumer self-service.
- Invoked a monetized AI API using OAuth2 credentials, with each request automatically metered and reported to Stripe.
- Viewed real-time usage, invoices, and subscription management from the Developer Portal Billing page.

---

## Next steps

- [Token-Based Rate Limit for AI APIs](../../cloud/create-api-proxy/third-party-apis/token-ratelimit.md) — tune per-subscription token quotas to balance consumer access against your LLM cost.
- [Manage paid subscription plans](../../monetization/manage-paid-subscription-plans.md) — edit, version, or deactivate your plans as your pricing evolves.
- [Monetize a REST API with Stripe](api-monetization.md) — compare the monetization flow for a standard REST API.
- [Monetize MCP Tools with the Self-Hosted Gateway](mcp-tool-monetization.md) — bill AI agents per MCP tool call with the self-hosted AI gateway and Moesif.
