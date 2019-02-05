# Payment Batching

It's often possible to add more outputs to a transaction without
increasing how many inputs it contains, meaning a single transaction can
increase the number of people it pays faster than it proportionally
grows in size.  That's scalability, and this chapter describes how
high-frequency spenders can use this scaling technique of *payment
batching* to reduce transaction sizes and fees by about 75% in
practical situations.

As of February 2019, payment batching is used by multiple popular
Bitcoin services (mainly exchanges), is available as a built-in feature
of many wallets (including Bitcoin Core), and should be easy to
implement in custom wallets and payment-sending solutions.  On the
downside, use of the technique can lead to temporary unexpected behavior
for the receivers of payments and may result in a reduction of privacy.

## Transaction size per receiver

A typical Bitcoin transaction using P2WPKH inputs and outputs contains
one input from the spender of about 67 vbytes and two outputs of about
31 vbytes each, one to the receiver and one as change back to the
spender.  An additional 11 vbytes are used for transaction overhead
(version, locktime, and other fields).

![Best-case P2WPKH vbytes per payment](img/p2wpkh-batching-best-case.png)

If we add just 4 more receivers, including an additional 31 vbyte output
for each one of them, but otherwise keep the transaction the same, the
total size of the transaction becomes 264 vbytes.  Whereas the previous
transaction used all 140 vbytes to pay a single receiver, the batched
transaction uses only about 53 vbytes per receiver---a bit over 60%
savings per payment.

Extrapolating this simple best-case situation, we see that the number of
vbytes used per receiver asymptotically approaches the size of a single
output.  This makes the maximum savings possible a bit over 75%.

![Saving rates for best, typical, and tough cases of payment batching](img/p2wpkh-batching-cases-combined.png)

Realistically, the more a transaction spends, the more likely it is to
need additional inputs.  This doesn't prevent payment batching from
being useful, although it does reduce its effectiveness.  For example,
we can imagine the tough case of a gambling site that receives bets of
10 mBTC and pays out winnings of 100 mBTC, requiring at least 10 inputs
for every output added.  Here, the maximum savings from payment batching
(without optimizations described later) peak at only about 5%.

Falling between these extremes are a large number of services who
receive payments of about the same value as the payments they make, so
for every output they add, they need to add one input on average.
Savings in this typical case peak at about 30%.

Services that find themselves frequently using more than one input per
transaction may be able to increase their savings using a two-step
procedure.  In the first step, multiple small inputs are
[consolidated][chapter consolidation] into a single larger input using
slow (but low-feerate) transactions that spend the service's money back
to itself.  In the second step, the service spends from one of its
consolidated inputs using payment batching and achieves the best-case
efficiency described above.

If we assume that consolidation transactions will pay only 20% of the
feerate of a normal transaction and will consolidate 100 inputs at a
time, we can calculate the savings of using the two-step procedure for
our 10-to-1 and 1-to-1 scenarios above (while showing, for comparison,
the simple best-case scenario of already having a large input available).

![Saving rates for best, typical, and tough cases of payment batching after consolidation](img/p2wpkh-batching-after-consolidation.png)

Most notably, the savings from consolidation allows the tough case go
from worst performing to best performing.  For the typical case,
consolidation actually loses money when only making a single payment,
but when actually batching, it performs almost as well as the best case
scenario.

In addition to payment batching directly providing a fee savings,
batching also uses limited block space more efficiently by reducing the
number of vbytes per payment.  This increases the available supply of
block space and so, given constant demand, can make it more affordable.
In that way, increased use of payment batching may lower the feerate for
all Bitcoin users.

In summary, payment batching provides significant savings for services
that typically have inputs available that are 5 to 20 times larger than
their typical output.  For services not in that position, the savings
from batching alone are smaller but perhaps still worth the effort;
if the services are willing to also pre-consolidate their inputs, the
savings can be quite dramatic.

Note: the figures and plots above all assume use of P2WPKH inputs and
outputs.  We expect that to become the dominant script type on the
network in the future (until something better comes along).  However, if
you use a different script type (P2PKH, or multisig using P2SH or
P2WSH), the number of vbytes used to spend them are even larger, so the
savings rate will be higher.

## Problems

The fee-reduction benefits of payment batching do create tradeoffs and
problems that will need to be addressed by any service using the
technique.

### Delayed gratification

This is the primary problem with payment batching.  Although some
situations naturally lend themselves to payment batching (e.g.  sending
out the company payroll for every employee at the same time), many
services primarily send money to users when those users make a
withdrawal request.  In order to batch payments, the service must get
the user to accept that their payment will not be sent immediately---it
will be held for some period of time and then combined with other
withdrawal requests.

One mitigation for the problem of delayed gratification is to allow the
user to choose between an immediate payment and a delayed payment, with
a different fee provided for each option.  For example:

    [X] Free withdrawal (payment sent within 6 hours)
    [ ] Immediate withdrawal (withdrawal fee 0.123 mBTC)

Another mitigation is to use transaction replacement to append new
outputs to an existing transaction.  This allows you to send each payment
immediately, but each replacement must pay a higher feerate than the
earlier version of the transaction.  This reduces your savings rate
unless you were planning to fee bump an earlier transaction anyway.
We'll cover batching and replacement in more detail later in this
chapter.

## Reduced privacy

A second problem with payment batching is that it can make users feel
like they have less privacy.  Every user you pay in the same transaction
can reasonably assume that everyone else receiving an output from that
transaction is being paid by you.  If you had sent separate
transactions, any onchain relationship between the payments might be
less apparent or even non-existent.

![Screenshot of a possible transaction batch in a block explorer](img/batch-screenshot.png)

Note that transactions belonging to particular Bitcoin services are
often identifiable by experts even if they don't use payment
batching, so batching doesn't necessarily cause a reduction in privacy
for those cases.

It may be possible to partially mitigate this problem by sending batched
payments in a coinjoin transaction created with other users.  Depending
on the technique used, this would not necessarily reduce the efficiency
of batching and could provide significantly enhanced privacy.  However,
naive implementations of coinjoin previously provided by Bitcoin
services have had [flaws][coinjoin sudoku] that prevented them from
providing significant privacy advantages.  As of February 2019, no
currently-available coinjoin implementation is fully compatible with the
needs of payment batching.

## Possible inability to fee bump

A final problem is that you may not be able to fee bump a batched
payment.  Transaction relay nodes such as Bitcoin Core impose limits on
the transactions they relay to prevent attackers from wasting bandwidth,
CPU, and other node resources.  By yourself, you can easily avoid
reaching these limits, but the receivers of the payments you send can
respend that money in child transactions that become part of the
transaction group containing your transaction.

The closer to a limit a transaction group becomes, the less likely
you'll be able to fee bump your transaction using either
Child-Pays-for-Parent (CPFP) fee bumping or Replace-by-Fee (RBF) fee
bumping.  In addition, the more unconfirmed children a transaction has,
the more RBF fee bumping will cost as you'll have to pay for both the
increased feerate of your transaction as well as for all the potential
fees lost to miners when they remove any child transactions in order
to accept your replacement.

Note that these problems are not unique to batched payments---independent
payments can have the same problem.  However, if an independent payment
can't be fee bumped because the independent receiver spent their output,
only that user is affected.  But if a single receiver of a batched
payment spends their output to the point where fee bumping becomes
impossible, all the other receivers of that transaction are also affected.

## Implementation

Payment batching is extremely easy using certain existing wallet
implementations, such as Bitcoin Core.  Check your software
documentation for a function that allows you to send multiple payments.

```bash
bitcoin-cli sendmany "" '{
  "bc1q5c2d2ue7x38hcw2ugk5q7y4ae7nw4r6vxcptu8": 0.1,
  "bc1qztjzd7hpf2xmngr7zkgkxsvdqcv2jpyfgwgtsv": 0.2,
  "bc1qsul9emtnz0kks939egx2ssa6xnjpsvgwq9chrw": 0.3
}'
```

<!-- for max standard tx size: src/policy/policy.h:static const unsigned int MAX_STANDARD_TX_WEIGHT = 400000; -->

If using your own implementation, you are probably already creating
transactions with two outputs in most cases, so it should be easy to add
support for additional outputs.  The only notable consideration is that
Bitcoin Core nodes (and most other nodes) will refuse to accept or relay
transactions over 100,000 vbytes, so you should not attempt to send
batched payments larger than this.

## Combining batching with other opt-in scaling techniques

Payment batching works well with all the techniques described in this
guide, but some combinations are especially notable:

### Changeless transactions

Changeless transactions save 31 to 43 vbytes (for typical transaction
templates) by closely matching the total value of the inputs and outputs
in the transaction, plus fees, so that there's no need to return any
change back to the spender's wallet.  This is also a perfectly efficient
way to consolidate inputs as you decrease the total number of your
inputs by making a payment you would've made anyway.

Changeless transactions always decrease overall transaction size, but
because the efficiency of payment batching comes from spreading the
fixed costs of the transaction across multiple receivers, the benefit
of changeless transactions is weakened (in percentage terms) the more
you are able to batch.  The following plot compares the best-case
situation from earlier in this chapter, which included a change output,
to an otherwise-identical changeless transaction.

![Plot of normal best case against a changeless best
case](img/p2wpkh-batching-changeless.png)

Although the benefit may be small in percentage terms, the cost of
implementing changeless transactions may also be small.  Your goal is to
find a single input you own whose value is equal to or slightly larger
than the total value of the outputs you want to send.  A simple
algorithm would be to wait until you have a minimum number of payments
to batch and check to see if you have an input that is equal to or
slightly larger than that.  If not, wait until you have another payment
to send, combine that with the earlier payments, and try again.  Keep
trying until you reach a limit (e.g. it's time to send the oldest
payment) or the aggregate output value is larger than your largest
input.

For more information about changeless transactions, including exactly
what "slightly larger than the output value" means, see the
corresponding section in the chapter about [coin selection
strategies][].

### Replace-by-Fee (RBF)

RBF is typically promoted as a strategy for increasing the fees paid in
a transaction so that it confirms sooner, but it also allows for adding
outputs to a transaction, making it suitable for adding even more
payments to a previously-sent transaction.

However, RBF transactions must pay a higher feerate than the original
transaction.  This means replacing a large batch payment costs more than
replacing a small independent payment.  Since fee batching benefits more
from larger transactions, the cost increases of RBF act in direct
opposition to the fee savings of payment batching.

Looking at the savings rate charts above, we can consider some examples.
If you sent a single payment and want to turn that into a batch
containing two payments, you'll expect to save 40% from batching in the
best case---so paying a fee increase of anything up to 40% can save you
money overall.  Alternatively, if you already have a batch of ten
payments and are considering adding an eleventh, the extra savings is
only about 1%, so it might not be worth it if RBF requires you increase
the overall feerate of the transaction by more than 1%.

However, there is a time when it's guaranteed to be more efficient to
add new payments to an existing transaction: when you plan to fee bump
that existing transaction anyway.  If you sent a payment some time ago
and are planning to use RBF to increase its fee rate, you should also
add any additional queued payments to the replacement transaction so
that they gain from batched efficiency.

For more information about RBF, see the chapter about [fee bumping][].

## Recommendations summary

1. Try to create systems where your users and customers don't expect
   their payments immediately but are willing to waiting for some time
   (the longer the better).

2. Use low-feerate consolidations to keep some large inputs available
   for spending.

3. Within each time window, send all payments together in the same
   transaction.  Ideally, your prior consolidations should allow the
   transaction to contain only a single input.

4. Don't depend on being able to fee bump the batched payments.  This
   means using a high-enough feerate on the initial transaction to
   ensure it has a high probability of confirming within your desired
   time window. For example, use the `CONSERVATIVE` mode of Bitcoin
   Core's `estimatesmartfee` RPC.

5. Optionally, look for opportunities to send the batched payments
   without a change input.

6. Optionally, when you're already planning to attempt an RBF fee bump,
   append as many additional queued payments as possible.  (But remember
   that fee bumps are unreliable.)

[chapter consolidation]: #FIXME_not_written_yet
[coinjoin sudoku]: http://www.coinjoinsudoku.com/
[coin selection strategies]: #FIXME_not_written_yet
[fee bumping]: ../1.fee_bumping/fee_bumping.md
