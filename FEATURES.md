# Features

Code-derived inventory of what this repo implements. Bullets and key file paths —
the mechanism lives in `docs/how-it-works.md`, the walkthrough in `docs/demo-script.md`.

_Last generated: 2026-09-02 by feature-doc._

A commercetools Connect connector (`connect.yaml`) that integrates Avalara AvaTax for
US/Canada sales tax calculation and compliance, private-labeled for the Monotype
Fonts demo (`f622981 feat: private Avalara connector for Monotype Fonts (AWS
us-east-2)`). It ships as three deployAs applications — `service` (a commercetools
API Extension for real-time tax), `event` (a Pub/Sub subscriber for order-lifecycle
transaction recording), and `mc-app` (a Merchant Center Custom Application for
configuration) — built on the certified Avalara/commercetools connector base rather
than one of the storefront starters (`b2c-starter`, `b2b-starter`, etc.), so there is
no starter fork base and bullets below carry no provenance tags.

## Real-Time Tax Calculation (`service`)

- Cart-update API Extension triggers on `cart` `Create`/`Update` whenever
  `shippingAddress`, `shippingInfo` and non-empty `lineItems` are present
  (`service/src/connector/actions.ts`, `createCartUpdateExtension`), and only runs
  tax calc for countries enabled in the `taxCalculation` setting (`none` / `US` /
  `CA` / `USCA`) (`service/src/controllers/cart.controller.ts`)
- Calls the AvaTax `getTax` endpoint for line items, custom line items and shipping,
  then applies the result to the cart via `setLineItemTaxAmount`,
  `setShippingMethodTaxAmount` and `setCartTotalTax`, switching the cart's `taxMode`
  to `externalAmount` (`service/src/avalara/requests/actions/get.tax.ts`,
  `service/src/avalara/requests/postprocess/postprocess.get.tax.ts`)
- Cart hashing (MD5 over a key-order-invariant JSON of customer, shipping address,
  line items and shipping info) skips a redundant AvaTax call when nothing relevant
  to tax has changed since the last calculation (`service/src/utils/hash.utils.ts`)
- Handles mixed taxable/non-taxable line items and tax-inclusive pricing
  (`includedInPrice`) in the same cart

## Address Validation

- `/service/check-address` validates an address against AvaTax's `resolveAddress`
  API and returns AvaTax's suggested/corrected address plus any error messages
  (`service/src/controllers/check.address.controller.ts`)
- Used from the mc-app to validate the merchant's origin address, with a "Check
  address" button in the settings UI that surfaces AvaTax's suggested corrections
  inline (`mc-app/src/components/settings/AvalaraOriginAddress.tsx`)
- Also callable from a storefront/frontend to validate a shipping address; the
  endpoint accepts a JWT signed with `FRONTEND_API_KEY` for non-Merchant-Center
  callers, separate from the Merchant-Center JWKS-verified path
  (`service/src/middleware/auth.middleware.ts`)
- Returns `addressValidation: false` without calling AvaTax at all when the merchant
  has turned address validation off in settings

## Transaction Management (`event`)

- Subscribes to a Google Cloud Pub/Sub topic for `OrderCreated`, `OrderStateChanged`,
  `OrderStateTransition` and `OrderReturnShipmentStateChanged` order messages
  (`event/src/connector/actions.ts`, `createOrderSubscription`); only US/CA shipping
  addresses are processed (`event/src/controllers/event.controller.ts`)
- Commits an AvaTax transaction on order creation, or on a configurable order state
  (state IDs picked in the mc-app), per the `commitOnOrderCreation` /
  `commitOrderStates` settings (`event/src/controllers/event.controller.ts`)
- Voids an unlocked AvaTax transaction on cancellation, or automatically falls back
  to a full refund transaction when AvaTax reports the transaction is already locked
  (filed for returns) via `CannotModifyLockedTransaction`
  (`event/src/avalara/index.ts`, `voidOrRefundTransaction`)
- Partial refunds: on `OrderReturnShipmentStateChanged`, builds a refund-transaction-
  lines call scoped to the specific returned line items, guarding against
  re-refunding an already-fully-refunded order
  (`event/src/avalara/index.ts`, `partiallyRefundTransaction`;
  `event/src/avalara/helpers/refund.lines.helpers.ts`)
- Order Edit recalculation: on `OrderEditApplied` with no `taxedPrice` set yet,
  recomputes tax via AvaTax and applies the result back to the order as a
  commercetools Order Edit (`changeTaxMode`, `setLineItemTaxAmount`,
  `setCustomLineItemTaxAmount`, `setShippingMethodTaxAmount`, `setOrderTotalTax`),
  computing an effective blended tax rate for orders with mixed taxable/non-taxable
  amounts (`event/src/avalara/helpers/order.edit.helpers.ts`)
- All AvaTax calls and errors are logged and always return HTTP 200 to Pub/Sub
  (to avoid redelivery loops); only a missing Avalara configuration returns 400
  (`event/src/controllers/event.controller.ts`)

## Flexible Tax Configuration

- Product-level Avalara tax codes via a configurable product attribute (default
  `avatax-code`), shared by both `service` and `event` via
  `AVATAX_PRODUCT_ATTRIBUTE_NAME` (`service/src/utils/hash.utils.ts`)
- Category-level and shipping-method-level tax codes stored as custom fields
  (`avalaraTaxCode`) on connector-provisioned custom types
  (`service/src/connector/actions.ts`)
- Customer entity-use (tax exemption) codes stored as an `avalaraEntityUseCode`
  custom field on the customer custom type, applied to AvaTax calls for exemption
  handling (`service/src/connector/actions.ts`,
  `event/src/avalara/helpers/entity.use.code.helpers.ts`)
- Custom-type keys/names for every one of these fields (category, shipping method,
  customer, order, custom line item) are configurable in `connect.yaml`, and the
  connector only creates a type if one with that key doesn't already exist — so a
  merchant can share one custom type across several connectors
  (`connect.yaml`, `service/src/connector/actions.ts`)

## Merchant Center Configuration UI (`mc-app`)

- Self-service settings screen covering logging, tax-calculation mode, address
  validation and document-recording toggles, all persisted to a single custom
  object (`avalara-connector-settings`)
  (`mc-app/src/components/settings/AvalaraConfiguration.tsx`)
- "Test Connection" button that pings AvaTax with the deployed credentials and
  reports whether they authenticate, without leaving the Merchant Center
  (`mc-app/src/components/settings/AvalaraCredentials.tsx`,
  `service/src/controllers/test.connection.controller.ts`)
- Origin-address form with live "Check address" validation against AvaTax, showing
  which specific address fields AvaTax would correct
  (`mc-app/src/components/settings/AvalaraOriginAddress.tsx`)
- Transaction Management screen: pulls the project's actual order-state machine
  (built-in + custom workflow states, localized) into a data table so a merchant
  can pick exactly which order states trigger an AvaTax commit vs. a
  void/cancel, with mutual-exclusion validation preventing the same state being
  picked for both (`mc-app/src/components/settings/AvalaraTransactionManagement.tsx`)
- Toggle to enable/disable return-driven partial refunds (`activateReturns`) and to
  disable AvaTax document recording entirely (`disableDocRec`)
- Configurable AvaTax SDK logging level (Error/Warn/Info/Debug), surfaced through
  the connector's own deployment logs rather than a separate log viewer

## Self-Provisioning Deploy

- `connector:post-deploy` (service) creates the cart-update API Extension pointed
  at the deployed service URL, creates all five Avalara custom-field types
  (customer, shipping method, category, order, custom line item), and then runs a
  live AvaTax credential check, logging a warning if the credentials are invalid
  (`service/src/connector/post-deploy.ts`)
- `connector:post-deploy` (event) creates the Google Cloud Pub/Sub order
  subscription for the four order message types the transaction manager handles
  (`event/src/connector/actions.ts`)
- `connector:pre-undeploy` on both applications removes the extension/subscription
  and rolls back the field definitions it added (or deletes the whole custom type
  if that was its only field), so undeploying leaves the commercetools project
  clean (`service/src/connector/pre-undeploy.ts`, `event/src/connector/pre-undeploy.ts`)
- `mc-app` deploy config defaults `CLOUD_IDENTIFIER` to `aws-us` (corrected from an
  invalid `aws-us-east-2` — `connect.yaml`, `ddf6b2f`)

## Security

- Merchant Center requests to `/test-connection` and `/check-address` are verified
  against the Merchant Center's own JWKS-issued JWT (`iss` checked against known MC
  API URLs); non-Merchant-Center callers (e.g. a storefront) instead verify against
  a shared `FRONTEND_API_KEY` secret, so the same two endpoints serve both callers
  under different trust boundaries (`service/src/middleware/auth.middleware.ts`)
- Avalara and commercetools credentials live only in the connector's secured
  configuration (`AVALARA_USERNAME/PASSWORD/COMPANY_CODE`, `CTP_CLIENT_ID/SECRET`),
  never in the mc-app or in cart/order data

## Demo Tooling

- `dev/scripts/ngrok.sh` for exposing a local `service`/`event` instance during
  development against a live commercetools project
- `dev/scripts/update-dependencies.sh` for bumping the connector's own dependency set
