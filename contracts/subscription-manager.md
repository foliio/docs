# Subscription Manager

### Design Considerations

The role of the subscription manager is to generate invoices, validate invoice payments, and mint or extend subscriptions. Under the hood, we use Chainlink to get up to date prices when creating an invoice.&#x20;

### Subscription Activation

**Limitations**&#x20;

* An account can only activate a subscription if it does not have one.
* An account must request an invoice prior to activating a subscription&#x20;

1. An account invokes createInvoice and specifies the payment method (either ONE or 1USDC), the subscription type to be quoted for, and a duration in months \[1,12]
   * By default the created invoice will expire in 20 minutes from its creation. If not paid before expiration, a new invoice will need to be created&#x20;
2. The account calls getInvoice() which returns the generated invoice. A web3 client will need this to format an activate request&#x20;
   * When an invoice is paid, it must match the quote. Ie. if you create an invoice using ONE and then try to pay with 1USDC, the transaction will be reverted
3. Paying with ONE
   1. The account calls activate and sends ONE in the msg.value
4. Paying with 1USDC
   1. The account must approve the SubscriptionManager to spend 1USDC
   2. The account calls activateWith and specifies the amount it is paying&#x20;
5. SubscriptionManager validates the transaction and reverts if the payment does not match the invoice. Otherwise a new subscription is minted and sent to the caller.

```
  struct Invoice {
    // The address of the payment token. An address of (0x0) implies base currency
    address paymentToken;

    // block.timestamp when the invoice expires 
    uint256 expires;

    // How much is expected upon activation/extension. 18 decimals
    uint256 quantity;

    // Total duration in seconds that will be applied to the subscription
    uint256 quotedDuration;

    // Subscription type to be activated or extended
    uint256 subscriptionType;
  }
```
