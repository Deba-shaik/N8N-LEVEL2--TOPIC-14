# Topic 14 - Sub-Workflows, Debugging & Production Readiness

## Overview
This assessment implements a modular n8n order-processing demonstration with one master workflow and two callable child workflows. Evidence was collected from the n8n screenshots provided on 23 October 2026.

## Workflows and node sequence

### 1. Sub-Workflow 1 - Tax Engine
`Execute_Workflow_Trigger -> Code_ValidateTaxInput -> Code_CalculateStateTax`

- **Execute_Workflow_Trigger:** Defines the expected order input contract (`orderId`, `customerName`, `state`, `subtotal`).
- **Code_ValidateTaxInput:** Requires `state` and a positive numeric `subtotal`; normalizes state to uppercase.
- **Code_CalculateStateTax:** Uses demonstration rates CA 8.25%, NY 8.875%, TX 6.25%, FL 6%, with 5% fallback. Calculates `taxAmount` and `grandTotal`.

### 2. Sub-Workflow 2 - Payment Gateway
`Execute_PaymentTrigger -> Code_SimulatePayment`

- **Execute_PaymentTrigger:** Defines `orderId`, `grandTotal`, `customerName`.
- **Code_SimulatePayment:** Validates positive payment amount and order ID, returns `paymentStatus: SUCCESS`, a generated `transactionId`, and `amountCharged`. This is a simulation, not a real payment integration.

### 3. Master Order Orchestrator
`Manual Trigger -> Code_PrepareOrder -> Execute_TaxEngine -> Call 'Sub-Workflow 2 - Payment Gateway deb' -> Code_FinalOrderConfirmation`

- **Code_PrepareOrder:** Produces the test order for Acme Corporation, California, subtotal 1000.
- **Execute_TaxEngine:** Calls the tax child workflow, waits for completion, and receives the calculated tax and grand total.
- **Payment sub-workflow call:** Passes `orderId`, `grandTotal`, and `customerName` to the payment child workflow and waits for the simulated response.
- **Code_FinalOrderConfirmation:** Combines the tax result and payment result, sets `orderStatus` to `CONFIRMED` for `paymentStatus: SUCCESS`.

## Observed test cases

| ID | Scenario | Observed result | Status |
|---|---|---|---|
| TC-01 | Tax Engine: CA, subtotal 1000 | Tax rate 0.0825, tax 82.50, grand total 1082.50 | PASS |
| TC-02 | Payment simulation: amount 1082.50 | Payment status SUCCESS, transaction ID generated | PASS |
| TC-03 | Master end-to-end | Five nodes green; final order status CONFIRMED | PASS |
| TC-04 | Invalid subtotal -100 | CONTRACT_VIOLATION: subtotal must be a positive number | PASS |
| TC-05 | Prevent payment after validation failure | Payment and confirmation nodes did not execute | PASS |

## Successful output example

```json
{
  "orderId": "ORD-MODULAR-901",
  "customerName": "Acme Corporation",
  "state": "CA",
  "subtotal": 1000,
  "taxAmount": 82.5,
  "grandTotal": 1082.5,
  "paymentStatus": "SUCCESS",
  "orderStatus": "CONFIRMED"
}
```

The actual workflow additionally generates a transaction ID and confirmation timestamp.

## Debugging and validation evidence

- The Tax Engine initially rejected missing input fields when the child trigger was run with schema-only sample data. Pinning realistic test data allowed isolated testing; pinned test data should be removed before normal master-workflow calls.
- Negative subtotal `-100` was submitted through the master. The Tax Engine rejected it and prevented downstream payment simulation.
- After negative-input testing, restore `subtotal: 1000`, rerun, and save the successful workflow state.

## Screenshot evidence

The companion **Topic14_Evidence_Report.pdf** includes nine screenshots: successful Tax Engine, payment input contract, payment success, tax and payment sub-executions, full master workflow, final confirmation, negative test input, and validation rejection.

## Production-readiness blueprint (proposed, not deployed)

For a real deployment, use a supported n8n Docker Compose configuration with persistent PostgreSQL storage, encrypted n8n credentials, HTTPS reverse proxy, backups, structured logs, appropriate concurrency settings, and alerting. Use real payment-provider credentials in n8n's credential store, implement idempotency, and protect order/payment workflows from duplicate charges. These controls are recommendations, **not evidenced as implemented** by the supplied screenshots.

## Evidence boundaries

- Payment processing is simulated and does not contact a payment provider.
- Screenshots verify input validation, child-workflow execution, and final confirmation.
- Screenshots do not prove Docker deployment, external monitoring, persistence, or production load testing.
