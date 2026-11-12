# Pipwright: Make.com and Stripe setup guide

Two ready-made Make blueprints come with the page:
- `make-scenario-A-site-webhook.json`: the site webhook. Runs the AI chatbot, saves leads, creates Stripe Checkout sessions and answers order-status checks.
- `make-scenario-B-stripe-payments.json`: listens to Stripe, verifies each event, marks orders paid and sends the setup email.

I could not test these inside a Make account. Treat them as a complete first draft: import, then open each module once and fix anything Make flags (connections, data stores, IDs). If Make refuses the import, build the same flow by hand from the module list in section 4.

## 0. Before you start
- Stripe: confirm with Stripe that selling automated forex trading software is allowed for your account and country. Trading-related businesses are sometimes restricted or need review.
- In Stripe create Products with monthly recurring Prices for Pro ($49) and Desk ($199). Copy the Price IDs (price_...).
- Create a restricted Stripe API key that can write Checkout Sessions and read Events. Use it instead of the full secret key.
- In Make create two Data Stores: `orders` and `leads`. Each needs a text key. Orders fields: plan, name, email, country, platform, status, accepted_terms, accepted_risk, utm_source, utm_campaign, gclid, stripe_session_id, stripe_customer, amount. Leads fields: name, email, experience, platform, interest, message, utm_source, utm_medium, utm_campaign, first_utm_source, region.
- Have an Anthropic API key (the chatbot calls Claude through a plain HTTP module, so it needs no extra Make app).

## 1. Import Scenario A (site webhook)
1. Make > Create a new scenario > the "..." menu at the bottom > Import Blueprint > choose `make-scenario-A-site-webhook.json`.
2. Module 1 (Custom webhook): click Add, create a webhook named `pipwright-site`, copy its URL.
3. Search the blueprint for every `REPLACE_` value and set it:
   - `REPLACE_WITH_YOUR_TOKEN`: the filter before the router. Use a long random string.
   - `REPLACE_ANTHROPIC_API_KEY`: in the chat route's HTTP module header `x-api-key`.
   - `REPLACE_STRIPE_SECRET_KEY`: your restricted Stripe key, in the Authorization header as `Bearer sk_...` or `rk_...`.
   - `price_REPLACE_PRO` and `price_REPLACE_DESK`: your Stripe Price IDs.
   - `REPLACE-YOUR-DOMAIN.com`: your site, in `success_url` and `cancel_url`.
   - In each Data Store module choose `orders` (order and status routes) or `leads` (lead route).
   - In each Email module choose your connection (Make's own email or Gmail) and set the From address.
4. Turn the webhook's "JSON pass-through" OFF. The page sends ordinary form fields, which Make reads natively and which need no browser preflight request.
5. In the Webhook response modules, change `Access-Control-Allow-Origin` from `*` to your domain once testing is done.
6. Run once, then send a test from the site to let Make learn the fields. Then switch the scenario on.

## 2. Put the webhook in the page
In index.html set:
```
window.PW_AUTOMATION = { webhook: "https://hook.eu2.make.com/xxxx", token: "the same token as the Make filter", botName: "Pip" };
```
The token is visible in the page source, so it only filters spam.

## 3. Import Scenario B (Stripe payments)
1. Import `make-scenario-B-stripe-payments.json` and create a second custom webhook named `stripe-events`.
2. In Stripe > Developers > Webhooks add an endpoint with that URL and the events `checkout.session.completed` and `customer.subscription.deleted`.
3. Set `REPLACE_STRIPE_SECRET_KEY` in the HTTP module. This module fetches the event again from Stripe by its ID, so a forged request cannot mark an order paid.
4. Choose the `orders` Data Store, your Email connection, and a Slack incoming-webhook URL (or delete the Slack module).
5. Optional: Stripe's own "Watch Events" module signs and verifies events. If you prefer it, use it as the trigger and keep the rest.

## 4. What each scenario contains (for building by hand)
Scenario A: Custom webhook > Router (filter: token equals your secret) with four routes on `type`:
- chat: HTTP POST api.anthropic.com/v1/messages with the Pipwright system prompt > Webhook response with `{{content[1].text}}`. The model must reply with only a JSON object `{ "reply", "actions", "suggestions" }`. The page sends the message and history with double quotes already removed so they are safe inside JSON.
- lead: Data Store add record > Email (only if email_consent is true) > Webhook response `{"ok":true}`.
- order: Data Store add record (status pending) > nested Router: starter (update status, welcome email, respond ok) or paid (HTTP POST api.stripe.com/v1/checkout/sessions with mode subscription, the Price ID chosen from `plan`, `client_reference_id` = order_ref, `success_url` ending #order-complete and `cancel_url` ending #order-cancelled > update record with the session ID > respond `{"ok":true,"order_id":"...","checkout_url":"..."}`).
- order_status: Data Store get record by order_id > respond `{"status":"paid|pending|failed","email":"..."}`.
Scenario B: Custom webhook > HTTP GET api.stripe.com/v1/events/{id} > Router: completed and paid (update order to paid, email setup steps, notify Slack) and subscription deleted (update order to canceled).

Fields the page sends on every request (form-encoded): type, token, session_id, page_url, sent_at, referrer, landing_page, timezone, region, language, gclid, fbclid, utm_source, utm_medium, utm_campaign, utm_term, utm_content and first_utm_* for first touch. Then per type:
- chat: message, history (a text transcript)
- lead: name, email, experience, platform, interest, message, email_consent
- order: plan, name, email, country, platform, accepted_terms, accepted_risk, order_ref
- order_status: order_id, email

## 5. Security checklist
- Stripe and Anthropic keys live only inside Make, never in index.html.
- Card details are entered only on Stripe. The page never handles them and only follows checkout links on checkout.stripe.com or buy.stripe.com.
- Prices come from Stripe Price IDs inside Make. The page sends only the plan name.
- Mark an order paid only from Scenario B. Never trust the browser returning to #order-complete.
- Add a rate limit for each session_id in Make, and set Access-Control-Allow-Origin to your own domain.
- Keep the risk warning and terms tick. They are stored with each order.

## 6. Test in order
1. Chat: ask "how much is Pro?" and check the reply and the buttons.
2. Lead: submit the form, check the Data Store row and the welcome email.
3. Starter order: check the free account email.
4. Stripe test mode, card 4242 4242 4242 4242: order goes pending, then paid, and the page shows "Payment received".
5. Cancel on the Stripe page and check #order-cancelled.
6. Switch to live keys and Price IDs and run one low-value real order.
7. In Google Tag Manager preview, check pw_page_context, begin_checkout, generate_lead and purchase arrive with the ecommerce items.
