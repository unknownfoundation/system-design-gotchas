# Timeouts, retries and circuit breakers

Prev: [Rate limiting and load shedding](../rate-limiting-and-load-shedding/README.md) · Next: [Consistency, consensus and clocks](../consistency-consensus-and-clocks/README.md) · Reviewed October 2026

Over a network a system can fail in part: some calls fail while the rest work. A retry sends a failed call again, and the guards below bound what that costs: how long a caller waits, how many times the work runs and how far the trouble spreads.

- **Percentile**: the 99.9th percentile of a call log is the latency that 99.9 percent of its calls beat.
- **Worker**: a thread or process that handles one call at a time, and stays held for as long as its call waits.
- **Idempotent**: running an operation again leaves the same result as running it once. In HTTP, PUT, DELETE and safe methods such as GET are defined as idempotent, and POST is not.
- **Cascading failure**: an overload that spreads: one part slows or fails after a deploy, a spike or clients reconnecting at once, its work falls on the rest, and workers, connections and memory run out along the chain.
- **Metastable failure**: an overload that outlives its trigger: work the failure itself created, such as retries and a growing queue, keeps the service overloaded after the trigger has gone.
- **Gray failure**: a failure that one observer sees and another does not.

A request that fails on every server does damage on each one it is retried on ([Load balancers and proxies](../load-balancers-and-proxies/README.md), "Taking out any backend with five failed requests in a minute"). A read with no timeout can hang for minutes, or for good, on a connection a firewall has forgotten, so set a read timeout on every call ([DNS, TCP and TLS](../dns-tcp-and-tls/README.md), "The first query after a quiet hour behind a firewall").

| Guard | What it bounds | What it does |
| --- | --- | --- |
| Timeout | one wait | too long holds a worker on a hung dependency, and at the median latency half of the healthy calls fail |
| Deadline | a whole request, across its services | sets one point in time at the top and carries it down; server code must still stop what it started, and per-attempt timers with retries can outlast it |
| Backoff and jitter | how soon an attempt follows the last | grows the delay after each failure; jitter randomises all or part of a delay or of a fleet's shared timer |
| Retry budget | how many attempts get added | limits retries by a count per call, a time limit, or a share of all calls |
| Idempotency key | how many times a repeat does the work | a value the caller creates for one operation and sends on each attempt; a server still holding it answers with the stored reply, and once it drops the key a repeat is new work |
| Hedged request | how long a slow read holds up its caller | sends a second copy of a read to another replica after a wait at a high percentile and takes the first reply, adding a few percent more reads on a calm day |
| Circuit breaker | calls to a dependency that keeps failing | on this page, passes calls and counts failures while closed; once open, refuses them at once for a set pause, and then passes a few trial calls |
| Bulkhead | how much of a server one dependency can hold | gives a dependency or a route its own pool of workers or connections |
| Fallback | what a caller gets when a call fails | answers from another path or with a default; a default or stale value reaches the caller as if true, so one in constant use can hide a failing call |

Clients differ in their defaults, so unless a question says otherwise, the only timers, retries, keys, breakers and fallbacks in a question are the ones it names, a server treats two calls as one only when they share a key, and a server that accepts a call works it through to the end, whether or not the caller still waits.

The trouble sits where a guard meets a failure its test never showed.

This topic lets you:

- say what a timed-out or refused call left behind at the far end, before your code sends it again or hands its connection to the next read
- limit how long a call can hold a worker on a crawling download, a far region or a chain of services
- send a call again without multiplying the load or the charge, using backoff, retry limits, keys and second copies
- keep one failing part from taking down the rest, and bring back a service that stays down, or that fails for only some of its callers

###### 1. Timing out a card payment after ten seconds

On a ticket site, your server sends a card payment to a payment provider, with a ten-second timeout on the call. The customer is watching a spinner when the timer fires. Your code has no reply from the provider and has not called it again.

What do you know about the customer's card when the timer fires?

<details><summary>A. It has not been charged</summary>

✗ A failed call in your code usually means the work did not happen, and a checkout's error path is written that way, with "payment failed, try again" on any error. The request may have reached the provider and the charge gone through with its reply lost on the way back, and then "try again" leads to a second charge.

</details>

<details><summary>B. It has been charged, and the reply is late</summary>

✗ Slow providers are the everyday cause of a timeout, and anyone who has seen a confirmation turn up long after the call has seen one finish late. The request may equally have died on its way out, and then the card is clean and no reply is ever coming.

</details>

<details><summary>C. No way to tell yet whether it was charged</summary>

✓ Right. The card may have been charged or may not, and your server can tell the customer neither "paid" nor "failed".

A timer firing leaves three cases open, since **a timeout says only that no reply came in time**. The request may never have arrived, the charge may still be running, or it may have finished with its reply lost. The card is clean in the first case and charged in the third, and your code sees the same empty wait in all of them.

Put the order into a pending state rather than a failed one and tell the customer the payment is being checked, then settle it from what the provider reports later. Any repeat of the charge carries the first attempt's idempotency key.

</details>

<details><summary>D. A charge began and was reversed when the connection closed</summary>

✗ A database can roll back a transaction left open when its connection closes, and a client can close its connection when a timer fires. The ten-second timer fires inside your client, and a provider that has accepted a charge works it through whether or not your connection stays open. A charge that went through stays on the card, and your server has no sign that anything was reversed.

</details>

---

###### 2. Retrying a shipping label after a refused connection

A warehouse system creates shipping labels by sending a POST to a carrier's API, with no idempotency key. For one parcel the client's socket reports connection refused. Your code waits a second and sends the POST again, and this time a label comes back.

How many labels does the carrier hold for that parcel?

<details><summary>A. One label, made by the retry</summary>

✓ Right. The carrier holds one label, the one the retry made, and the parcel ships with no spare label to cancel.

Connection refused means the far end turned down the connection itself, so **no byte of the POST could reach the carrier**. POST is not idempotent, and HTTP still allows a client to repeat a request it can tell was never applied.

A refused connection can be retried with no risk of a second label. After a failure that comes once the request has gone out, the outcome is unknown and the same retry can make a second label, so give those POSTs a key.

</details>

<details><summary>B. Two labels, one from each attempt</summary>

✗ POST is not idempotent, and repeating one can create the resource twice. The risk belongs to a request that may have been applied. Your first attempt stopped before there was a connection to carry it, so the carrier never read a POST and has only the retry to act on.

</details>

<details><summary>C. One label, from the first attempt, handed back again to the retry</summary>

✗ Some APIs answer a repeat with the resource they made the first time, and a failed attempt can leave work behind. The first attempt left none here: the carrier's side refused the connection, so no request existed for it to make a label from, and the POST carries no key to match a repeat by.

</details>

---

###### 3. Checking the payment's status before charging again

A ticket site charges cards through a payment provider, and the card network behind it today takes about half a minute to answer. Your server's charge call reaches the provider and times out after ten seconds, so your code looks the order up at the provider. The lookup says no payment found, and your code sends the charge again, with no key.

What does the customer's statement show for that order a minute later?

<details><summary>A. One charge, from the second attempt</summary>

✗ A lookup exists to tell you where a payment stands, and this one reported none. The first charge was at that moment waiting on the card network, and it lands on the statement in its own time, beside the charge your code added.

</details>

<details><summary>B. Two charges for that one order</summary>

✓ Right. The statement shows two charges, and the customer has paid twice for one set of tickets.

This provider records a payment only when the card network answers, and **a lookup shows only what has been recorded**. "No payment found" at ten seconds was true while the first charge still had twenty seconds to run. The lookup does find a charge that finished with only its reply lost, which is the case it is usually tested on.

Make one idempotency key when the customer presses pay and send it on both attempts: the provider answers the repeat with a conflict or with the first charge's reply, and the statement shows one charge. Use the lookup to tell the customer where things stand, and never to decide whether to charge.

</details>

<details><summary>C. One charge, from the first attempt, the second turned away</summary>

✗ One card and one amount, seconds apart, is what a duplicate means on a statement. With no key on either request the provider has nothing to match them by, so the second is a new charge to it, and the card network answers both.

</details>

<details><summary>D. No charge from either attempt</summary>

✗ During an incident a provider's calls fail in bulk, and this provider did not answer in ten seconds. Half a minute is slow, and it is a working path: the card network answers each attempt in turn, and neither is lost.

</details>

---

###### 4. Reading an export under a 30-second read timeout

A billing worker downloads an export from a partner with the code below, and nothing else in the worker limits the call. A sick proxy on the partner's side now sends a few bytes every ten seconds, and at that rate the whole export takes an hour.

```python
sock = connect_to_partner()
sock.settimeout(30)
sock.sendall(export_request)

export = b""
while True:
    chunk = sock.recv(CHUNK_SIZE)
    if not chunk:
        break
    export += chunk
```

How long does the worker hold that download open?

<details><summary>A. Thirty seconds, then a timeout error</summary>

✗ Thirty is the one time limit in this code, and client samples show that one setting as the limit on the whole call. The 30 seconds cover one read, and each chunk arrives inside ten seconds, so the read returns and the next one starts counting from zero.

</details>

<details><summary>B. Thirty seconds, then the part received passed on as the whole export</summary>

✗ A short read on a stream hands back whatever has arrived, and this loop is built to collect partial pieces. A read that timed out would end in an error and hand nothing on, and the loop hands the export on only when the partner ends the reply, so it never goes anywhere half filled.

</details>

<details><summary>C. The whole hour of the export</summary>

✓ Right. The worker holds the download for the whole hour and the 30-second timer never fires. To your code this is a healthy transfer, and it ties up one worker and one connection.

A few bytes every ten seconds end the current read, and **a timer on a socket limits each read and starts again at the next**, so the count never gets past ten. Only a gap over thirty seconds would end the download. Some clients describe their read timeout in these terms, as the wait between bytes, and some also offer a limit on the whole call, which this code does not set.

Check a limit of your own on the whole download before each read and give the socket only the time left of it. Set that limit above the time an honest export takes.

</details>

<details><summary>D. Until the partner's proxy resets it</summary>

✗ Proxies cut off transfers that crawl, when they have a limit of their own for it. The sick proxy is itself sending the bytes, every ten seconds, and the only timer on the connection never fires, so it stays up until the last of them.

</details>

---

###### 5. Setting a timeout from a week of latency logs

A storefront calls a search service in another region, about 80 ms away by round trip, over TLS connections it keeps open to each server. You set a 300 ms timeout on the whole call from last week's call log, where 300 ms is the 99.9th percentile. Today a deploy replaces all the search servers at once, and the new ones work normally.

What do the storefront's calls see right after that deploy?

<details><summary>A. One call in a thousand timing out, the same as any day</summary>

✗ One in a thousand is what a 99.9th percentile promises, and that figure comes straight from your own log. Every call that log measured ran over a connection already open, and the deploy has closed them all.

</details>

<details><summary>B. Timeouts on the slowest search queries alone</summary>

✗ A query log puts the costly searches at the top, and a timeout is there to catch slow work. The calls that cross 300 ms now are the ones that spent most of it on a lookup and handshakes, so a cheap search that opens a connection can time out while a costly one on an open connection does not.

</details>

<details><summary>C. Timeouts for customers far from the servers alone</summary>

✗ A new connection costs more the farther apart its two ends are, and customers far from a site pay that on a first visit. The connection that matters here runs from the storefront's servers to search, so a customer's location plays no part in it.

</details>

<details><summary>D. A short burst of timeouts, then the usual few</summary>

✓ Right. Some customers' searches fail after the deploy: for the first moments a good share of the calls time out against servers that are fine.

The log held calls over open connections, so its 300 ms left out what a new connection costs, and **a timer on the whole call counts the name lookup, TCP and TLS as well**. Only calls that find no idle connection pay, and right after the deploy that is most of them. Say a lookup takes 20 ms and TCP and TLS take a round trip each: setup is 180 ms, the call's own round trip makes it 260, and that leaves 40 ms of the 300 for the search itself. If the search takes 40 ms at the median, half of the calls that open a connection time out. If 5,000 calls a second go out, the burst is over within half a second as the pool of open connections fills.

Take the timeout from a log that includes new connections, or give setup a timer of its own, and replace servers a few at a time so connections close gradually.

</details>

---

###### 6. Setting one second of timeout at each of three hops

On a travel site, the gateway calls pricing, pricing spends 800 ms on its own work and then calls inventory, and inventory calls the database. Each of the three calls has its own fixed one-second timer. One evening the database stalls, and the gateway answers a customer with a 504 when its second is up.

What is the inventory service doing when the customer gets that 504?

<details><summary>A. Still waiting on the database</summary>

✓ Right. Inventory is still waiting on the database and will go on for another 800 ms. The customer already has an error page, and the system keeps working on an answer nobody will read.

**Each hop's timer starts when that hop's call starts**, so one second at every hop limits each wait and sets no limit on the whole request. Taking inventory's own work as zero, its call to the database begins 800 ms into the customer's second, and its timer fires at 1,800 ms.

Set one deadline at the gateway and pass it down, with each call limited to what is left of it: inventory then stops at 1,000 ms, as the 504 goes out. A framework that cancels the call still leaves your handler to stop the work it started.

</details>

<details><summary>B. Stopped, cancelled by the gateway's timeout</summary>

✗ Some frameworks carry a cancel down the chain when a caller gives up. This chain carries no cancel, so the gateway's timer ends the gateway's own wait, and pricing and inventory hear nothing of it.

</details>

<details><summary>C. Stopped already, its own timeout having fired first</summary>

✗ The failure starts at the database, and the call nearest to it is the first to report an error. Timers go by start time, and inventory's is the last to start, so at the 504 it still has 800 ms to run.

</details>

---

###### 7. Using exponential backoff after a shared outage

Phones running a fitness app all sync against one API. A five-second API outage fails every phone's call at the same moment, and each phone retries after 1, 2, 4 and 8 seconds, delays fixed in the app's code.

What does the API see as the phones retry?

<details><summary>A. A smooth trickle of retries, thinner each second</summary>

✗ Doubling is the usual answer to retries that bunch up, and 1, 2, 4 and 8 seconds back off hard. The doubling spaces one phone's attempts from each other. Every other phone has the same delays from the same moment of failure, so the second attempts land together as the first ones did.

</details>

<details><summary>B. A flood from every phone in every second</summary>

✗ A retry storm can keep a service down after the outage that set it off. A phone here waits between attempts and stops once one succeeds, so no phone sends anything in the gaps, and the API is back before the last delay has run.

</details>

<details><summary>C. Tall spikes at the same seconds, quiet between them</summary>

✓ Right. The API sees every phone at once, then nothing, then every phone again. Coming back from the outage, it meets its whole fleet in a single second.

Every phone here has the same schedule, and **backoff changes only each client's own schedule**. If each delay counts from the attempt before it and there are 10,000 phones, the retries fall at seconds 1, 3 and 7 with all 10,000 in each.

Use backoff and jitter together, choosing each delay at random between zero and the backoff value. The same phones then send about 1,500 in their busiest tenth of a second, where the fixed delays send 10,000.

</details>

<details><summary>D. One spike at the first retry, then ordinary traffic</summary>

✗ Phones sit on different networks with different delays, and that noise blurs a spike a little. Say the noise is a tenth of a second: that is small against delays counted in seconds, and each new delay starts from an almost identical moment, so the later retries bunch in one second as well.

</details>

---

###### 8. Allowing three attempts at each of three layers

A bank's mobile app calls an API, the API calls an accounts service, and the accounts service calls a ledger. Each of the three callers makes up to three attempts on a failed call, set in its own config. The ledger is down and refuses every call when a customer taps once.

What reaches the ledger for that one tap?

<details><summary>A. Three calls, from the accounts service</summary>

✗ The accounts service is the only caller the ledger has, and its config says three attempts. That count applies to each call it receives, and it receives several, since the API above it repeats itself and so does the app.

</details>

<details><summary>B. Three ledger calls for every attempt made above</summary>

✓ Right. One tap becomes 27 calls on a ledger that is already down, and every other customer tapping multiplies it the same way.

**Each attempt of a layer starts a full round in the layer below**, and the counts multiply: three from the app, three from the API for each of those, three from the accounts service for each of those. Every config reads 3, and no single file shows the 27.

Let one layer retry and have the others pass the failure up, which leaves the ledger 3 calls for the tap, and put a retry budget on what remains.

</details>

<details><summary>C. A few more than three, each layer adding its own attempts</summary>

✗ Three configs hold three separate counts, and a review of them adds three, three and three. The app's second attempt sets off everything its first attempt set off, so the total grows by a factor at each layer.

</details>

---

###### 9. Building a key from a hash of the order

A coffee chain builds the idempotency key in its app as a hash of customer, item and amount, so a retried tap gets the same key. The payment server keeps each key for a day. A customer orders a latte, then a second latte for a friend two minutes later.

What does the second latte's order get?

<details><summary>A. A new charge and a new receipt</summary>

✗ To the customer this is a new tap for a new drink, and keys are there for retries. The server sees no taps, only a key it already holds, because the second latte hashes to the same value, and it never reaches the step that charges.

</details>

<details><summary>B. A refusal saying the key was already used</summary>

✗ Coupon codes and one-time tokens refuse a second use, and some payment servers refuse a key reused with different contents. Two lattes match to the byte, so such a check has nothing to object to, and the server replies from its store.

</details>

<details><summary>C. The first latte's receipt, and no new charge</summary>

✓ Right. The app shows a receipt for the second latte, and it is the first latte's receipt. The shop has been paid for one drink, and the friend's coffee goes out unpaid.

Two lattes hash alike, so the server cannot tell a second order from a retry of the first: **a key has to name one intent to pay, and only the caller knows that**. Keys built from order fields pass testing, because a test retries and rarely reorders.

Create the idempotency key when the customer taps and store it with the order, for reuse only when that tap is retried. Each latte then gets its own receipt, and a retry of the first returns the first receipt.

</details>

<details><summary>D. Two charges on the card and one latte</summary>

✗ A shop and its payment server keep separate records, and those two drift apart in other failures. Here the key stops the request before either record changes, so the card shows one charge.

</details>

---

###### 10. Saving the key before charging

A payment handler at a subscription service saves the idempotency key first, to stop two copies racing, and then calls the provider. One night the server dies right after saving the key, and the client retries with the same key.

```python
def handle_payment(key, amount):
    if store.has(key):
        return {"status": "paid"}
    store.save(key)
    provider.charge(amount)
    return {"status": "paid"}
```

What does the customer end up with after the retry?

<details><summary>A. One charge, made by the retry</summary>

✗ The whole point of a key is that a retry can finish what a crash interrupted. On this retry the store already holds the key, so the handler leaves at its first check, and the line that would finish the work never runs.

</details>

<details><summary>B. A paid status and no charge</summary>

✓ Right. The client gets `paid`, no money moves, and the service now counts as paid a subscription nobody has paid for.

Recording the idempotency key and doing the charge are two steps, and **a crash between two steps leaves one without the other**. In this handler a saved key means done, and it was written before the work was. Swapping the two lines gives the opposite failure: charge first and crash before the save, and the retry charges again.

Pass the key to the provider and drop the local check, so the charge itself carries it and the same crash ends in one charge. Or keep a state per key, `started` before the call and `done` after: a retry that finds `started` gets a conflict and no charge, and your code still has to find out what happened.

</details>

<details><summary>C. Two charges, one per attempt</summary>

✗ A crash in the middle of a payment can end in a double charge, and it does for code that charges first. The server died before its only call to the provider, and the retry stops at the key, so the provider never hears of this payment.

</details>

<details><summary>D. An error back to the client, and no charge</summary>

✗ A store with separate states for started and done can spot a half-finished key and answer with a conflict. This store holds the key and nothing beside it, and a key it holds gets one answer from this handler: `paid`.

</details>

---

###### 11. Sending second copies during a sale's slowdown

A shop's catalogue client sends a second copy of a read to another replica once the first has waited a fixed 10 ms, the 95th percentile of last month's latency, and last month that added a few percent more reads. During a sale the backend is saturated, and almost every read takes about 50 ms.

What does the catalogue backend receive during the slowdown?

<details><summary>A. A few percent more reads, the same as last month</summary>

✗ A few percent is what you measured, and it is the figure the technique is known for. It was the share of reads past 10 ms on last month's latency. That share moves with the latency, and a wait fixed at 10 ms now catches almost all of them.

</details>

<details><summary>B. Nearly every read twice</summary>

✓ Right. Nearly every read reaches the backend twice, and a backend with no capacity left is handed about double the work at the height of the sale.

The cost of hedging moves with today's latency, since **a hedged request fires for each read still unanswered at the wait**. Last month 5 percent of reads passed it, by the definition of the 95th percentile, and the backend got 1,050 copies per 1,000 reads. At 50 ms nearly every read passes it, and the backend gets close to 2,000.

Cap hedges at a small share of reads, and take the wait from current latency: at the slowdown's own 95th percentile the extra stays at 5 percent. Reads are still slow then, since a hedge cannot add capacity.

</details>

<details><summary>C. Fewer reads, second copies replacing slow ones</summary>

✗ The first reply wins, and hedging is described as a rescue for a slow read. The client stops waiting on the loser and may cancel it, but both copies have gone out by then and the backend has received them both.

</details>

<details><summary>D. Copies of copies, many for each read</summary>

✗ A second copy waits past 10 ms as the first did, and a retry storm builds from repeats of repeats. A hedge here is one copy per read, and the client never hedges a hedge, so no read reaches the backend more than twice.

</details>

---

###### 12. Putting a breaker in front of a hung tax service

A checkout calls a tax service through a breaker. At noon the tax service stops answering, though it still accepts every connection.

| Setting | Value |
| --- | --- |
| Workers | 40 |
| Checkouts at noon | 10 a second |
| Timeout on the tax call | 30 seconds |
| Breaker opens after | 10 calls that ended in a failure |

What do checkout customers get in the first twenty seconds?

<details><summary>A. A fast refusal from the breaker, after the first ten calls</summary>

✗ Ten bad calls trip this breaker in about a second when a service returns errors at once, and errors at once are what a test usually sends. A call to a service that accepts and stays silent does not come back bad. None has ended yet, the count stands at zero, and the breaker lets the next one through.

</details>

<details><summary>B. Ordinary checkouts, with the tax line left off</summary>

✗ The pattern is usually drawn with a fallback beside the breaker, returning a default when the call fails. The checkout has none, and no call has failed so far, so the tax line and the rest of the page stay behind the wait.

</details>

<details><summary>C. A checkout page that hangs with no answer</summary>

✓ Right. Every customer's checkout hangs, and the first answer of any kind comes at 30 seconds. For those twenty seconds the breaker does nothing for them.

This breaker's count stays at zero because **a breaker that counts ended calls sees nothing of a call still hanging**. At 10 a second all 40 workers are held by the fourth second, every one waiting. The first calls end at 30 seconds, and the breaker opens at about 31. Breakers differ on this, and the setting decides whether a hang counts at all.

Shorten the timeout on any call a customer waits for and count slow calls as failures, then test the breaker against a service that goes quiet.

</details>

<details><summary>D. Ordinary checkouts, the breaker having taken the failing calls away</summary>

✗ The name comes from the switch in a fuse box, which sits between the fault and the house and cuts at once. This one acts on a count, and the count has nothing to add until a call ends, so every call still goes out to the tax service and takes a worker with it.

</details>

---

###### 13. Putting one breaker in front of four storage shards

A photo service wraps its storage client in one breaker that opens when 20 percent of recent calls fail. Photos live on four shards, each holding its own quarter of them, and calls spread evenly across the four. One shard goes down.

What happens to the photos on the other three shards?

<details><summary>A. They stop loading too</summary>

✓ Right. One dead shard takes nearly the whole photo service down with it: most of those photos stop loading as well.

**One breaker sees one stream of calls**: a quarter of them fail, which is over its 20 percent, and it opens and refuses every call, whichever shard the photo lives on. Its trial calls land on the dead shard one time in four, so it opens again soon after each pause, and the healthy shards' photos stay refused nearly all the time.

Give each independent backend its own breaker. With one per shard, the healthy shards see no refusals, and the dead one gets only trial calls to find out whether it is back.

</details>

<details><summary>B. They load as usual</summary>

✗ Shards exist so that a failure takes out one share of the data, and the storage itself behaves that way here. The breaker sits above them in the client and knows nothing of shards. Once it opens, its refusal covers every call it wraps, so healthy photos go too.

</details>

<details><summary>C. They load slower, taking the fourth shard's traffic</summary>

✗ In a balanced pool the survivors take a failed peer's load. A shard holds only its own photos, so the dead shard's calls have nowhere else to go, and the healthy three get fewer calls than before, since the breaker is stopping them in the client.

</details>

<details><summary>D. They load from a copy after a short wait</summary>

✗ Storage systems often keep copies, and a client that fails over to one is common enough. The breaker refuses inside the client, before any shard or copy is asked, and the four shards hold no copies to go to.

</details>

---

###### 14. Waiting for slow recommendations on a shop's product page

A shop's server has one pool of 200 workers for every page. Product pages arrive at 30 a second and call a recommendations service with a ten-second timeout, and checkout never calls it. One afternoon recommendations stops answering.

What happens to checkout that afternoon?

<details><summary>A. Checkout runs as fast as ever</summary>

✗ The dependency diagram shows no line from checkout to recommendations. The two share the pool, and product pages hold all 200 workers, so a checkout request finds none free when it arrives.

</details>

<details><summary>B. Checkout fails with recommendations timeouts</summary>

✗ Failures in a call chain travel upward, each caller reporting the error of the one below. Checkout makes no call that could time out on recommendations, so its customers get a wait and no recommendations error.

</details>

<details><summary>C. Checkout gets faster, with product pages gone quiet</summary>

✗ A broken feature usually means fewer requests for it and less work on the server. Those workers are held and not working, and product pages keep arriving at 30 a second to replace each one the timeout frees.

</details>

<details><summary>D. Customers at checkout queue for a free worker</summary>

✓ Right. Customers at checkout queue behind product pages for a worker, held up by a service the order never touches, and the wait keeps growing.

Thirty pages a second, each holding a worker for ten seconds, would need 300 workers, and the pool has 200, so it stays full and every new request of every route joins one line. Checkout is in that line because **a worker held on a slow dependency is lost to every route that shares its pool**. One route's waiting has reached another with no call between them, the first step of a cascading failure.

Put a bulkhead between them: with 150 workers for product pages and 50 for the rest, checkout no longer queues behind product pages while recommendations stays down. A shorter timeout on the recommendations call only raises the number of pages a second needed to fill the pool.

</details>

---

###### 15. Moving all messages to a backup provider at 5 percent errors

A login service sends one-time codes through a messaging provider, and automation moves all sending to a backup once errors pass 5 percent. The backup's contract sets a rate limit at a fifth of the volume and refuses what goes over it, and monthly drills that send 1 percent of traffic through it pass. One morning errors reach 6 percent.

How many codes reach customers after the switch?

<details><summary>A. Every code, on the backup's capacity</summary>

✗ The drills pass every month, and a green drill report is the only evidence anyone has on the backup. They run at 1 percent of traffic, far under a contract for a fifth, so they have never met the limit that the real switch runs into on its first second.

</details>

<details><summary>B. Nearly every code, a few minutes late</summary>

✗ Messaging systems are known for queueing, and a code that arrives late is a familiar complaint. A rate limit refuses anything above it outright and keeps no queue, so those codes never arrive at any time.

</details>

<details><summary>C. Fewer codes than before the switch</summary>

✓ Right. A fifth of the codes get through, down from 94 percent, which leaves most customers logging in that morning with no code.

A full switch hands the fallback the whole load at once, and **a path built for a fifth of the load delivers only that fifth**. If 1,000 codes a second go out, the main provider was delivering 940 and the backup delivers 200.

Move only as much as the backup's limit allows, and leave the rest on the main path, which was still delivering most of it. Drill at the volume a real switch sends.

</details>

<details><summary>D. The same share as before, from the other provider</summary>

✗ The automation's config treats the two providers as interchangeable, one name swapped for the other. Capacity sets the share here: before the switch 6 percent of codes failed, and after it everything above a fifth of the volume does.

</details>

---

###### 16. A thirty-second stall and an hour of errors

A stock tracker runs an API that takes its calls from one queue, in order, and its clients use a one-second timeout and three attempts per call. At nine its database stalls for thirty seconds, and then its queries run at their usual speed. At ten the API has the same number of customers as yesterday, its CPU is at 100 percent and most calls time out.

What keeps the API failing at ten?

<details><summary>A. Database connections the stall left broken</summary>

✗ A stall can leave dead connections in a pool, and a pool with no check goes on lending them out. A dead connection ends in an error or a wait, and neither fills a CPU. The CPU that is full here is the API's own, and the queries it does send run at their usual speed.

</details>

<details><summary>B. The backlog from those thirty seconds, still draining</summary>

✗ A stall leaves a backlog, and a backlog drains at the rate of spare capacity. Say 600 calls a second arrive and the CPU finishes 1,000: the stall's 18,000 calls would be gone in 45 seconds if nobody retried. An hour on, this queue is far longer than when the stall ended, so something is filling it faster than it empties.

</details>

<details><summary>C. A one-second client timeout too tight for the API's calls</summary>

✗ Timeout is the word on every failed call, and one second is not long. That second served the same customers yesterday without trouble, and in this state a longer timer does not help: with five seconds the queue still outruns it and calls keep failing.

</details>

<details><summary>D. Retried calls, and the API working on calls already abandoned</summary>

✓ Right. The API spends its whole CPU on calls whose clients have left, while those clients' retries take their place in the queue. That is how customers still get timeouts an hour after the database recovered, with nothing left to repair.

This is a metastable failure, in which **retries and work on abandoned calls keep each other going**. Say 600 fresh calls a second reach a CPU that finishes 1,000. The stall leaves a queue longer than a one-second timeout can wait, so each call times out while it waits and comes back as a retry, and the API still serves the abandoned original when its turn comes. Three attempts offer 1,800 calls a second, the API finishes 1,000 of them, and none of those has a client still waiting. With one attempt, or with a server that drops calls whose client has left, the same stall clears.

Stop looking for a fault and cut the load: drop calls whose client has left, or go to one attempt, or cap retries at a share of fresh calls. Waiting 2 seconds before each retry does not end it, since all 1,800 calls still arrive.

</details>

---

###### 17. Bringing a crashing fleet back

Twenty servers of a video API normally run at 80 percent. A spike crashed most of them, and it has passed, so traffic is ordinary again. Each server takes minutes to start, with start times differing by about four minutes across the fleet. A new one gets a full share from the few still up and runs out of memory within a minute.

Which step brings the fleet back?

<details><summary>A. Restart every server at once</summary>

✗ Off and on again works on most things, and if all twenty arrived in the same minute they could carry this traffic together. They arrive over about four minutes, so the first few take the whole load and die before the rest are up.

</details>

<details><summary>B. Block most traffic at the edge, then let it back in steps</summary>

✓ Right. With most traffic refused at the edge, the entry point in front of the servers, each server that starts survives its first minute and the fleet fills back up, and customers get refusals for several minutes and then a working service.

A cascading failure like this one **needs the load held to what the running servers can carry**, because the servers cannot catch up with it on their own. Say 3 survived, traffic is worth 16 servers and a start takes between 3 and 7 minutes. A server arriving now takes 4 servers' worth and dies inside a minute, so each is up for a minute in every six and only about 3 of the 20 are up at any moment, where 16 would have to be up together.

Have the edge admit a little less than each running server can carry, say 80 percent of it, and raise it as servers join. On those start times about two thirds of the traffic is refused until the last server is up, at about 410 seconds.

</details>

<details><summary>C. Add a third more servers and let them join as they start</summary>

✗ More servers is the usual answer to an overload, and a third more gives real room. New servers start at the same pace as the old ones, a few at a time, and each takes the same share and dies the same way, so a fleet of 27 still has only a few up at a time, where 16 are needed.

</details>

<details><summary>D. Hold everything as it is, now the spike has gone</summary>

✗ This traffic ran at 80 percent yesterday, and a capacity chart says the fleet can carry it. That 80 percent was spread over twenty servers. Today it is aimed at whichever few are up, and each returning server goes past its memory, so the fleet holds only a few of its 20 at a time.

</details>

---

###### 18. Running a failover monitor beside the service it watches

A bank's accounts service runs in zone A, with automatic failover that moves it to another zone when a monitor's pings fail. The monitor sits in zone A as well. A network fault between the zones drops many packets, and front ends in zone B start timing out.

What does the failover automation do about zone B's timeouts?

<details><summary>A. Nothing, and zone B keeps timing out</summary>

✓ Right. The automation does nothing, because as far as it can tell nothing has happened. Customers served from zone B keep getting timeouts from a service the monitor calls healthy.

This is a gray failure: **a monitor sees only the path its own pings travel**, and the callers travel another. These pings stay inside zone A and never cross the faulty link, so failed pings stay at zero.

Watch from where the callers are: with the monitor in zone B the same fault fails its pings, and failover acts once enough of them fail. Better still, feed the rule the error rate the front ends see.

</details>

<details><summary>B. It moves the service to another zone within a minute</summary>

✗ Failover exists for outages, and to customers in zone B this is one. The rule acts on failed pings and has none to act on, since the monitor's traffic never leaves zone A. An hour into the fault it still has not moved anything.

</details>

<details><summary>C. It moves the service away and back, over and over</summary>

✗ Partial faults make automation flap where a monitor catches them now and then. Flapping needs a first move, and this monitor's pings all succeed, so the service stays where it is.

</details>

<details><summary>D. It cuts zone B off from the service</summary>

✗ Automation built against split brain, two sides each acting as the leader, fences off the sick part of a system. The failover rule here is about pings alone, and it stays quiet, so zone B's front ends go on calling and losing many of those calls.

</details>

---

###### 19. Timing out an account read on a pooled cache connection

A food delivery app keeps each customer's account, with their name and home address, in a cache server, and your code reads it over a pool of connections that stay open. `pool` is your thin wrapper around them. The cache's protocol is plain text over TCP: a request is one line, and a reply is the stored value. At lunchtime the cache stalls for a second. One customer's read passes its 200 ms timeout and ends in an error, and that customer gets an error page. The stall ends, and a moment later a second customer opens their account page, and the pool hands their read the same connection.

```python
def load_account(customer_id):
    conn = pool.acquire()
    try:
        conn.send(f"GET account:{customer_id}")
        return conn.read_reply(timeout=0.2)
    finally:
        pool.release(conn)
```

What does the second customer's account page show?

<details><summary>A. Their own name and address</summary>

✗ The timeout error has reached your code and the connection has gone back to the pool, and in your process that is the end of the first read. The cache was never told: it answered when the stall ended, and that answer is next on the connection, ahead of whatever the second read asks for. The second customer's own account comes back only to the read after theirs.

</details>

<details><summary>B. An error from the client, for a reply that does not fit the request</summary>

✗ Your request named the customer's id, and your code still has that id in hand while it reads. The reply names nothing: it is a stored value, the first customer's account has the same fields as the second's, and the client has nothing to compare it with, so it returns it as a success.

</details>

<details><summary>C. An error from the pool, for a connection the timeout closed</summary>

✗ Socket errors usually end a connection: after a reset or a broken pipe it is gone for good. A read timer is a different kind of error, since it fired in your process and the connection stayed up through the stall. Your code releases it to the pool as it stands, and the second read finds it open with a reply waiting.

</details>

<details><summary>D. The first customer's name and address</summary>

✓ Right. The second customer's account page shows the first customer's name and home address, and your code counts the read as a success. One customer's address has gone to another.

A late reply still comes because **a read timeout ends the wait in your process and tells the cache nothing**. The first request is still accepted work, and its reply comes down that connection once the stall ends. With no id on a reply, only order pairs it with a request: each reply belongs to the oldest request not yet answered. The late reply sits at the front of the socket's buffer, and the next read takes what is at the front, whoever asked.

Close a connection whose read gave up instead of releasing it: the late reply goes with it, and the second customer's read opens a new connection and gets their own account. A protocol that ties each reply to its request, as HTTP/2 does with its streams, lets a client drop the late reply and keep the connection.

</details>

---

## Further reading

- [Metastable Failures in the Wild](https://www.usenix.org/conference/osdi22/presentation/huang-lexiang), 2022: 22 metastable failures from 11 organizations, with retry policy the most common effect that keeps a system stuck.
- [Metastable Failures in Distributed Systems](https://doi.org/10.1145/3458336.3465286), 2021: the state itself, and why its root cause is the sustaining feedback loop and not the trigger.
- [Gray Failure](https://doi.org/10.1145/3102980.3103005), 2017: failures that a detector misses and applications feel, and differential observability as their key feature.
- [The Tail at Scale](https://doi.org/10.1145/2408776.2408794), 2013: hedged requests and other ways to keep the slowest calls of a large service from setting its latency.

Prev: [Rate limiting and load shedding](../rate-limiting-and-load-shedding/README.md) · Next: [Consistency, consensus and clocks](../consistency-consensus-and-clocks/README.md)
