# Payment State Handling and Webhook Reliability

## Problem

Payment systems introduce asynchronous state changes through external providers.

In our case, Razorpay sends webhook events to notify the backend about payment success or failure. However, user actions such as closing the payment window mid-flow can leave the system in an inconsistent state.

We encountered a real issue where:

- The subscription plan was updated  
- Feature access was not enabled  

This resulted in a partially applied state, where the system did not reflect a valid payment outcome.

---

## Approach

### 1. Explicit Payment State Management

To handle incomplete or interrupted flows, we introduced clear state transitions:

- Payments are marked as `pending` when an order is created  
- A timeout window allows users to complete the payment  
- If the payment is not completed within this window, it is moved to a `cancelled` state  
- Subscription activation and feature access are enabled only after confirmed payment  

This ensures that no user-visible state is updated unless the payment is fully completed.

---

### 2. Idempotent Order Creation

To prevent duplicate orders:

- If a valid pending order already exists for the same plan, it is reused  
- The system verifies the order status with Razorpay before returning it  

This avoids multiple orders being created due to repeated user actions.

---

### 3. Webhook Logging and Deduplication

Webhook events are stored in a dedicated collection with the following properties:

- Each event includes an `eventId` and `eventType`  
- A unique index is enforced on `(eventId, eventType)`  

This ensures that duplicate webhook deliveries from Razorpay do not result in repeated processing.

Additional fields such as `processingStartedAt`, `processingEndedAt`, `handled`, and `success` are tracked to monitor processing lifecycle and failures.

---

## Why this approach

- Prevents inconsistent states caused by interrupted user flows  
- Ensures subscription activation is tied strictly to confirmed payments  
- Handles duplicate webhook deliveries safely through database-level constraints  
- Improves observability and debugging through structured webhook logs  
- Avoids duplicate order creation under repeated user actions  

---

## Tradeoffs

- Adds additional state management complexity  
- Requires careful handling of edge cases such as timeouts and retries  
- Relies on webhook delivery for final confirmation of payment state  
- Database-level idempotency prevents duplicates but does not fully eliminate the need for careful processing logic  

---

## Notes

Webhook idempotency is enforced using unique event identifiers at the database level. Duplicate events are safely ignored through this constraint.

This decision focuses on ensuring that application state remains consistent and reflects only completed and valid payment outcomes, even in the presence of retries, delays, or interrupted user actions.
