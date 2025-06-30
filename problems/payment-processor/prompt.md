# Payment Processor

Payment processing is the act of receiving payment requests, forwarding
those requests to downstream processors, and then returning responses:

```
+-----------+         +------------------+         +-----------------+
| Merchant  |         | Payment          |         | Downstream      |
|           |<----->  | Processor        | <-----> | Processor       |
|           | hold    | (Your design     |  hold   |                 |
|           | request | here)            | request |                 |
|           |<----->  |                  | <-----> |                 |
|           | charge  |                  |  batch  |                 |
|           | request |                  | file    |                 |
+-----------+         +------------------+         +-----------------+
```


1. **Hold*** Receive a request for an account with an account number (e.g. a credit card number) and amount. This request is then forwarded to a downstream processor.
   - The downstream processor then has the choice to approve or deny the hold request. If the request is approved, the money in the account is held for some short period of time.
2. **Charge*** If the hold request is approved, the customer has the opportunity to sign (and potentially tip) and send a charge request.
   - If no charge request is received, eventually the held funds are released.
   - The charge requests are gathered up into a large batch file grouped by downstream processor. Every evening @ 10pm, that file is sent and the funds are wire transferred in a big batch.

---

**For example**, a customer might tap their American Express credit card at a coffee shop.

1. **Hold*** The coffee shop's card machine sends a hold request to the processor (us) and we forward the request to the downstream processor (in this case American Express) to hold funds for the payment with the credit card number and amount.
   - American Express might approve that hold based on a number of factors including if there is sufficient credit on the account.
2. **Charge*** At a later time, the retailer sends a charge request which is batched by the processor (us) and sent in a batch file with all the American Express transactions to initiate a large transfer for all of the funds.

**Note.** This is an intentionally simplified view of a payments system to fit in the time period of the interview. Many real world payment systems have additional complexity.