# Partial Payments

In the default case, the `Amount` field of a [Payment transaction][] in the Ripple Consensus Ledger specifies the exact amount to deliver, after charging for exchange rates and [transfer fees](concept-transfer-fees.html). The "Partial Payment" flag ([**tfPartialPayment**](reference-transaction-format.html#payment-flags)) allows a payment to succeed by reducing the amount received instead of increasing the amount sent. Partial payments are useful for [returning payments](tutorial-gateway-guide.html#bouncing-payments) without incurring additional costs to oneself.

The amount of XRP used for the [transaction cost](concept-transaction-cost.html) is always deducted from the sender’s account, regardless of the type of transaction.

Partial payments can be used to exploit naive integrations with the Ripple Consensus Ledger to steal money from exchanges and gateways. The [Partial Payments Exploit](#partial-payments-exploit) section of this document describes how this exploit works and how you can avoid it.

## Semantics

### Without Partial Payments

When sending a Payment that does not use the Partial Payment flag, the `Amount` field of the transaction specifies the exact amount to deliver, and the `SendMax` field specifies the maximum amount and currency to send. If a payment cannot deliver the full `Amount` without exceeding the `SendMax` parameter, or the full amount cannot be delivered for any other reason, the transaction fails. If the `SendMax` field is omitted from the transaction instructions, it is considered to be equal to the `Amount`. In this case, the payment can only succeed if the total amount of fees is 0.

In other words:

    Amount + (fees) = (sent amount) ≤ SendMax

In this formula, "fees" refers to [transfer fees](concept-transfer-fees.html) and currency exchange rates. The "sent amount" and the delivered amount (`Amount`) may be denominated in different currencies and converted by consuming Offers in the Ripple Consensus Ledger's decentralized exchange.

**Note:** The `Fee` field of the transaction refers to the XRP [transaction cost](concept-transaction-cost.html), which is destroyed to relay the transaction to the network. The exact transaction cost specified is always debited from the sender and is completely separate from the fee calculations for any type of payment.

### With Partial Payments

When sending a Payment that has the Partial Payment flag enabled, the `Amount` field of the transaction specifies a maximum amount to deliver. Partial payments can succeed at sending _some_ of the intended value despite limitations including fees, lack of liquidity, insufficient space in the receiving account's trust lines, or other reasons.

The optional `DeliverMin` field specifies a minimum amount to deliver. The `SendMax` field functions the same as with non-partial payments. The partial payment transaction is successful if it delivers any amount equal or greater than the `DeliverMin` field without exceeding the `SendMax` amount. If the `DeliverMin` field is not specified, a partial payment can succeed by delivering any positive amount.

In other words:

    Amount ≥ (Delivered Amount) = SendMax - (Fees) ≥ DeliverMin > 0

### Partial Payment Limitations

Partial Payments have the following limitations:

- A partial payment cannot provide the XRP to fund an address; this case returns the [result code][] `telNO_DST_PARTIAL`.
- Direct XRP-to-XRP payments cannot be partial payments; this case returns the [result code][] `temBAD_SEND_XRP_PARTIAL`.
    - However, issuance-to-XRP payments or XRP-to-issuance payments _can_ be partial payments.

[result code]: reference-transaction-format.html#transaction-results

### The `delivered_amount` Field

To help understand how much a partial payment actually delivered, the metadata of a successful Payment transaction includes a `delivered_amount` field. This field describes the amount actually delivered, in the [same format](reference-rippled.html#specifying-currency-amounts) as the `Amount` field.

For non-partial payments, the `delivered_amount` field of the transaction metadata is equal to the `Amount` field of the transaction. When a payment delivers an issued currency, the `delivered_amount` may be slightly different than the `Amount` field due to rounding.

The delivered amount is **not available** for transactions that meet **both** of the following criteria:

- Is a partial payment
- Is included in a validated ledger before 2014-01-20

If both conditions are true, then `delivered_amount` contains the string value `unavailable` instead of an actual amount. If this happens, you can only determine the actual delivered amount by reading the AffectedNodes in the transaction's metadata. If the transaction delivered an issued currency and the `issuer` of the `Amount` is the same account as the `Destination` address, the delivered amount may be divided among multiple `AffectedNodes` members representing trust lines to different counterparties.


## Partial Payments Exploit

If a financial institution's integration with the Ripple Consensus Ledger assumes that the `Amount` field of a Payment is always the full amount delivered, malicious actors may be able to exploit that assumption to steal money from the institution. This exploit can be used against gateways, exchanges, or merchants as long as those institutions' software does not process partial payments correctly.

To exploit a vulnerable financial institution, a malicious actor does something like this:

1. The malicious actor sends a Payment transaction to the institution. This transaction has a large `Amount` field and has the **tfPartialPayment** flag enabled.
2. The partial payment succeeds (result code `tesSUCCESS`) but actually delivers a very small amount of the currency specified.
3. The vulnerable institution reads the transaction's `Amount` field without looking at the `Flags` field or `delivered_amount` metadata field.
4. The vulnerable institution credits the malicious actor in an external system, such as the institution's own ledger, for the full `Amount`, despite only receiving a much smaller `delivered_amount` in the Ripple Consensus Ledger.
5. The malicious actor withdraws as much of the balance as possible to another system before the vulnerable institution notices the discrepancy.
    - Malicious actors usually prefer to convert the balance to another crypto-currency such as Bitcoin, because blockchain transactions are usually irreversible. With a withdrawal to a fiat currency system, the financial institution may be able to reverse or cancel the transaction several days after it initially executes.
    - In the case of an exchange, the malicious actor can also withdraw an XRP balance directly back into the Ripple Consensus Ledger.

In the case of a merchant, the order of operations is slightly different, but the concept is the same:

1. The malicious actor requests to buy a large amount of goods or services.
2. The vulnerable merchant invoices the malicious actor for the price of those goods and services.
3. The malicious actor sends a Payment transaction to the merchant. This transaction has a large `Amount` field and has the **tfPartialPayment** flag enabled.
4. The partial payment succeeds (result code `tesSUCCESS`) but delivers only a very small amount of the currency specified.
5. The vulnerable merchant reads the transaction's `Amount` field without looking at the `Flags` field or `delivered_amount` metadata field.
6. The vulnerable merchant treats the invoice as paid and provides the goods or services to the malicious actor, despite only receiving a much smaller `delivered_amount` in the Ripple Consensus Ledger.
7. The malicious actor uses, resells, or absconds with the goods and services before the merchant notices the discrepancy.

### Avoiding the Exploit

The simplest and most effective way to avoid partial payment exploits is to use the `delivered_amount` field of the transaction metadata, not the `Amount` field, when processing incoming transactions. This way, an institution is never mistaken about how much it _actually_ received.

Additional proactive business practices can also avoid or mitigate the likelihood of this and similar exploits. For example:

- Add additional sanity checks to your business logic for processing withdrawals. Never process a withdrawal if the total balance you hold in the Ripple Consensus Ledger does not match your expected assets and obligations.
- Follow "Know Your Customer" guidelines and strictly verify your customers' identities. You may be able to recognize and block malicious users in advance, or pursue legal action against a malicious actor who exploits your system.

{% include 'snippets/tx-type-links.md' %}