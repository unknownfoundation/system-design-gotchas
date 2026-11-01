# Microservices

Prev: [WebSockets and real-time delivery](../websockets-and-real-time-delivery/README.md) · Next: [Service-to-service security](../service-to-service-security/README.md) · Reviewed October 2026

Microservices split one application into services that each own a business capability and its data, deploy on their own and talk over the network. The split decides what a release needs, how far a failure reaches and what one request costs on its way through.

- **Monolith**: one deployable whose modules call each other inside one process. A modular monolith keeps it one deployment and enforces the boundaries between modules, with public interfaces, data each module owns and tooling that flags what goes around them.
- **Service boundary**: the line around what one service owns, usually a business capability and the data that must stay consistent with it. Moving behaviour across it takes a change in two services and in the API between them, and the same entity, such as a customer, can mean different things on each side.
- **Distributed monolith**: services that are deployed apart and still have to ship together.
- **Database per service**: each service keeps its data to itself, and others reach it through its API. Services may share a database server without sharing tables, which leaves only its capacity shared. Sharing tables also shares the schema and the row locks, and a rule against it holds only once credentials enforce it.
- **Chatty calls**: many small calls between two services for one piece of work.
- **Fan-out**: one request at the top leading to many calls below it. What it does to the slow tail of a request is on [Performance and capacity](../performance-and-capacity/README.md).
- **Service registry**: the list of each service's copies and their addresses, which a copy joins when it starts. Service discovery is how a caller finds a copy.
- **API gateway**: one entry point for outside clients that hides the split into services and can merge calls for them. Business rules that gather there turn it into something every team has to change.
- **Backend for frontend**: a gateway for one kind of client, shaped to what that client needs.
- **Service mesh**: a sidecar proxy beside each copy that carries its calls, with settings from a central controller. It can take on retries, breakers and mutual TLS.
- **Contract**: what a caller relies on in a service's API: its fields, their types and what they mean. In a consumer-driven contract test, each caller states what it needs and the provider's build checks that.
- **Tolerant reader**: a client that takes the fields it knows and skips the rest, so that a provider can add fields without breaking it.
- **Strangler fig**: moving off a monolith piece by piece, with a facade in front that routes each piece to its new service.

A caller finds a copy in one of three ways:

| Way | What it does |
| --- | --- |
| Client-side discovery | each caller reads the registry, keeps a list and picks a copy, so each language needs its own copy of that logic |
| Server-side discovery | the caller sends to a balancer, which reads the registry and picks a copy, at the price of one more hop |
| DNS names | a name resolves to the copies, with a TTL on the answer |

Each call between services is a place for what [Timeouts, retries and circuit breakers](../timeouts-retries-and-circuit-breakers/README.md) covers, and spreading calls over a callee's copies is on [Load balancers and proxies](../load-balancers-and-proxies/README.md). One operation across services cannot be one transaction, and [Event-driven architecture](../event-driven-architecture/README.md) takes up sagas, the outbox and read models. Rollouts in which old and new copies serve together, and a wrong setting pushed to every service at once, belong with releasing changes.

A cycle of required calls that survives rolling restarts can stop a cold start, since each side needs the other up. Two versions of an API can run side by side at a cost for as long as the old one lives. Team structure tends to shape service boundaries.

Most of the trouble comes from a call that used to be a line of code and now crosses a network, and from what two services still share after the split.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- say what moving a module out of a monolith clears up, what a chain of services does to a checkout, and where a rule that spans two services has to live
- work out what a page, a handler or a launch sends once its calls cross a network, and what follows when one of those calls fails or stalls
- explain where calls go during a registry outage, what a backend for a phone app changes, and what a mesh does to your calls
- change a shared library or a field in an API, or move a route off a monolith, and say what each caller then gets

###### 1. Moving PDF export out of a single deployable

An insurer's claims app runs as one deployable on twelve identical servers. Your team keeps three complaints on its list: the test suite takes 50 minutes, modules read each other's tables, and every few days a memory leak in the PDF export module takes a server down with the claims it was handling. Export is one module of about forty. You move it to a service of its own, on servers of its own.

Which complaint does the move clear up?

<details><summary>A. The 50-minute wait for the test suite</summary>

✗ A small service builds in minutes, and this move makes the app smaller. Export is one module of about forty, so the other modules' tests still run on every change, and the wait stays close to 50 minutes.

</details>

<details><summary>B. Modules reading each other's tables across the app</summary>

✗ Clean boundaries are what people split a monolith for, and a network is hard to reach across. The move draws that boundary around export alone, and the modules left behind share one process and one database, so they go on reading each other's tables.

</details>

<details><summary>C. Claims lost when the leak takes a server down</summary>

✓ Right. A leak now takes down an export server, and the twelve app servers keep the claims they hold. The suite takes about as long as it did, and the modules still read each other's tables.

A process shares its memory with everything in it, and **moving code to another process separates its crashes, memory and scaling from the rest**. The modules left in the deployable still share one test run and one set of tables.

Move a part out for what only a process boundary gives, such as a crash to contain or load that differs from the rest. Slow tests and reads across modules get fixed inside the deployable, as a modular monolith does.

</details>

<details><summary>D. All three complaints, now the deployable is split</summary>

✗ All three complaints grew up in one deployable, and this move breaks it apart. The other modules still ship together, with their test run and their shared tables, and only the leak left with export.

</details>

---

###### 2. Splitting a checkout into five services at 99.9 percent

A company that sells tickets used to run its checkout as one application, which answered 99.9 percent of requests over a quarter. Five teams now run it as five services, each on its own servers and with its own release schedule. A request goes through basket, pricing, seats, payment and confirmation in turn, each waiting on the next. This quarter, each service reports that it answered 99.9 percent of the requests it got, counting only failures of its own.

How does the share of checkouts that succeed compare with the old 99.9 percent?

<details><summary>A. It rises, with each failure now staying inside one service</summary>

✗ The split promises that a failure stays inside one service, and that holds where a request can do without the service that failed. Checkout needs all five: a failure in payment still ends the customer's request, and each of the five now brings failures of its own.

</details>

<details><summary>B. It falls below 99.9 percent</summary>

✓ Right. If the services fail at different moments, about 99.5 percent of checkouts succeed, so about five in a thousand now fail where one did, while every service's report still reads 99.9.

A request succeeds only if every service answers, and **every link in the chain can end the request on its own**: five links at 99.9 percent multiply to about 99.5. Each report counts that service's own requests, and a request that dies in payment does nothing to basket's figure. Only failures that always struck at the same moment would hold the old 99.9, and separate servers and release schedules work against that.

Set the target on the whole checkout, measured from the customer's side, and take out of the chain what need not answer now, such as sending the confirmation after the reply.

</details>

<details><summary>C. It holds at 99.9 percent, the figure each service reports</summary>

✗ Five reports say 99.9, and the old application's figure was 99.9 too. Each report counts only that service's requests, and the failures are spread over five places, so about one checkout in 200 fails somewhere along the chain.

</details>

---

###### 3. Shipping discount rules through catalog and pricing

A grocery delivery app runs a catalog service and a pricing service, both owned by one team. The catalog decides which products a member's discount covers, and pricing works out the amount, each from its own rules. All 30 changes to discount rules last year touched both services, and the two releases went out within an hour of each other. The team has the next rule ready and wants to stop the double release.

Which change lets a new discount rule go out in one release?

<details><summary>A. Put the rule in a library that both services import</summary>

✗ Repeating yourself is the thing to avoid, and with one copy of the code the edit happens in one place. A library is built into each service, so catalog and pricing each run the old rule until each is rebuilt and shipped, and that is two releases.

</details>

<details><summary>B. Put a queue between catalog and pricing</summary>

✗ A queue frees two services from having to run at the same moment, which is why it comes up whenever two services are tied together. The rule still sits half in each, so a new rule needs a new catalog and a new pricing, whatever carries the messages between them.

</details>

<details><summary>C. Version the API between catalog and pricing</summary>

✗ Versioning an API frees the order of two releases, since either side can go first. A new rule still changes the code on both sides, so the team makes two releases in whichever order it likes.

</details>

<details><summary>D. Move the whole rule into pricing</summary>

✓ Right. With the whole rule in pricing, and catalog asking pricing which products a discount covers, a new rule changes pricing alone, and the team ships it in one release.

Catalog and pricing are a distributed monolith, and the cause is where the logic sits: **when a service boundary cuts one decision in two, every change to that decision touches both sides**. All 30 changes last year were that one decision.

Since one team owns both services, moving the boundary costs one refactor. Catalog now has to ask pricing, so check what those calls add to its pages. If the two still change together after the move, merge them into one service.

</details>

---

###### 4. Putting a data service in front of a shared database

At a logistics firm, six services read and write one database, and each schema change has to be agreed among six teams. A platform team puts a data service in front of it: only the data service holds database credentials, and the others call fixed endpoints such as `GET /orders/{id}`. The architecture diagram now shows no shared database. A month later the orders team wants a lookup of orders by parcel number, a column the `orders` table already holds.

What does the orders team have to release to ship the lookup?

<details><summary>A. The data service first, then the orders service</summary>

✓ Right. The lookup needs a new endpoint on the data service and then an orders release that calls it, and the orders team waits on the platform team for a query over a column its own table already holds.

The data service is a cut by technical layer, holding every service's queries, and **each new question about data has to pass through that layer's release**. Database per service asks something else: each service keeps the data of its own capability along with the queries over it.

Split by capability: the orders service holds the `orders` table and its queries, and the others ask orders for what they need.

</details>

<details><summary>B. The orders service alone</summary>

✗ Orders now talks to an API instead of a database, and its code is its own to change. The query lives on the far side of that API: `GET /orders/{id}` finds an order by id, and a lookup by parcel number needs an endpoint only the data service can add.

</details>

<details><summary>C. All six services, in one coordinated release</summary>

✗ Every schema change at this firm has taken six teams, and this request is about the data. The column already exists, so the schema does not change: the lookup is one new endpoint and one caller of it, and the other services never hear of it.

</details>

<details><summary>D. A new database for orders, then the orders service</summary>

✗ Database per service is the rule of this style, and orders is the team asking. That rule would move the table, a far larger job than this lookup needs, and the data service can already reach the column.

</details>

---

###### 5. Checking stock before taking the last jacket

At a clothes shop, orders and stock used to share a database. Stock is now a service with its own database, one copy that handles its calls one at a time, and it offers `available`, which returns the count, and `take`, which lowers it by the quantity given. Orders asks `available`, and if enough are left it calls `take` and saves the order. The last jacket of a line is on sale when two customers press buy at the same moment, and each request lands on its own orders copy.

What do the two customers get?

<details><summary>A. One confirmed order and one sold-out page</summary>

✗ Stock is the one owner of the count and handles its calls one at a time, which is how a store with a single thread keeps order. One at a time applies to each call: both `available` calls return 1 before either `take` runs, and a `take` lowers the count by what it is given.

</details>

<details><summary>B. Two confirmed orders for the one jacket</summary>

✓ Right. Both customers get a confirmation and the count ends at minus one: the shop has sold a jacket it does not have.

`available` and `take` are two calls a round trip apart, and the answer from the first is true only when it is sent: two checks see 1, then two takes subtract. In the shared database, one transaction could lock the row from the check to the write, and no transaction spans the two services now. **A rule such as a count that stays at zero or above holds only inside one atomic step**, and that step has to live in the service that owns the data.

Give stock one call that lowers the count only if enough are left and says which happened, and have orders act on that answer. Then one customer gets the jacket and the other a sold-out page.

</details>

<details><summary>C. Two sold-out pages, with both orders rolled back</summary>

✗ Two writers on one row make a conflict, and a database answers a conflict by aborting. These calls do not conflict: two reads and two subtractions each succeed on their own, and no transaction spans them to abort, so orders saves both.

</details>

---

###### 6. Moving the customer lookup behind HTTP

`customers` used to be a module of an invoicing app, and `customers.get` read its table. It is now a service, and `customers.get` keeps its signature as a thin client that makes an HTTP call over a connection kept open. All tests pass against a stub of the client. A wholesaler's invoice page lists 50 invoices at a time, often several for one customer.

```python
def invoice_page(invoices):
    rows = []
    for invoice in invoices:
        customer = customers.get(invoice.customer_id)
        rows.append((invoice.number, customer.name, invoice.total))
    return rows
```

What does one page now send to the customers service?

<details><summary>A. One request carrying all 50 ids, batched by the HTTP client</summary>

✗ HTTP clients do a lot for you, from pooling to compression, and a kept connection does save work. It saves the connection setup, and each `customers.get` has to return its customer before the loop moves on, so no batch can form.

</details>

<details><summary>B. 50 requests at the same moment, over one connection</summary>

✗ HTTP/2 can carry many requests over one connection at once, and this client keeps a connection open. Multiplexing needs requests in flight together, and this loop starts the next `customers.get` only after the last has returned.

</details>

<details><summary>C. 50 requests, each waiting for the reply before it</summary>

✓ Right. The loop sends 50 requests, one at a time. Say a round trip takes 2 ms: the page now spends 100 ms on lookups it used to read from its own table.

Keeping the signature hid the change, from the tests too, since a stub answers at once. A call in a loop cost almost nothing inside a process, and **over a network the same call costs a round trip on every pass**, which makes these chatty calls.

Add a call that takes a list, such as `customers.get_many(ids)`, and send the page's ids together. If two services still make many small calls for one piece of work, ask whether the boundary sits in the right place.

</details>

<details><summary>D. One request per distinct customer, the repeats served from the client</summary>

✗ A page that names one customer several times is a case for a cache. A thin client holds nothing between calls, so a customer on ten invoices of the page is asked for ten times.

</details>

---

###### 7. Rewriting the stock call with await

On a furniture shop's site, the orders service calls stock before it confirms an order. When stock ran slow, every worker thread sat waiting, so you rewrite the handler on `stock`, a thin non-blocking client. The load test looks far better. One afternoon the stock service goes down for ten minutes.

```python
async def place_order(order):
    reservation = await stock.reserve(order.items)
    await db.insert(order, reservation)
    return confirmation(order)
```

What does a customer who places an order in those ten minutes get?

<details><summary>A. An error, the same as before the rewrite</summary>

✓ Right. The customer gets an error, as before the rewrite: the `await` on `stock.reserve` fails once the call fails, and the line that saves the order never runs.

**`await` frees the thread while the call is out and still needs the reply**, and a non-blocking client still sends a request and waits for its reply, so orders needs stock up exactly as it did. The rewrite changed the cost of waiting: a slow stock no longer holds one thread per order, which is why the load test improved.

To take an order while stock is down, put the reservation on a queue and confirm before it runs. That is a decision to sell stock you may not have, and [Event-driven architecture](../event-driven-architecture/README.md) takes it up.

</details>

<details><summary>B. A confirmation, with the stock reserved once the service is back</summary>

✗ Messaging is called asynchronous too, and with a queue the work does happen later. `await` sits in the handler and stores nothing: nobody keeps the reservation to run later, and the call fails while the customer waits.

</details>

<details><summary>C. A page that stays open for the ten minutes, then a confirmation</summary>

✗ A suspended call costs almost nothing, and waiting was the whole trouble with the old version. A call to a service that is down fails, on a refused connection or the client's timeout, and that failure ends the `await` with an exception.

</details>

<details><summary>D. A confirmation, with the order saved and no stock reserved</summary>

✗ A non-blocking call returns at once and the code carries on, which is true of a call fired without `await`. With `await` the next line runs only after the reply, so `db.insert` never runs and no order is saved.

</details>

---

###### 8. Filtering a page of orders by a city in another service

A support tool lists orders over 100 euros from customers in Lisbon. It used to be one SQL join with `LIMIT 20`. Orders and customers now live in two services: the tool calls `GET /orders?min_total=100&limit=20`, gets those orders' customers from the customers service and joins in memory. An agent opens the list, and the first page comes up empty.

What does that empty page tell the agent about orders over 100 euros from Lisbon?

<details><summary>A. That no such order exists</summary>

✗ An empty page has meant no results in every list you have used, and it meant that when this was one join with `LIMIT 20`. The 20 now cuts the orders before any city is known, so this page covers 20 orders, and the Lisbon ones can start on the next.

</details>

<details><summary>B. That the first 20 orders over 100 euros are from other cities</summary>

✓ Right. The page says only where its 20 orders came from. Lisbon's orders can start on the next page, so an agent who stops here concludes there are none.

SQL filters the rows and then cuts the page. Here the orders service cuts the page, since it knows nothing of cities, and the filter runs afterwards on what the customers service returned. **Filtering after the cut can empty a page with matches still behind it**, as on about half of all pages if one customer in thirty lives in Lisbon and the orders over 100 euros come from customers chosen at random.

Keep a separate table that holds each order with its customer's city, fed from both services, so the filter runs before the page is cut. Where a city has few customers, ask the customers service for their ids first and pass those to the orders service.

</details>

<details><summary>C. That the customers service failed on this page</summary>

✗ A failed call often shows up as a blank panel, and this page needs two services. The filter runs after the page is cut, so a page can come up empty with both services healthy.

</details>

---

###### 9. Keeping the account screen up when orders fails

A bookseller's backend for its phone app builds the account screen from the profile and orders services, through thin clients. After an outage took whole screens down, you wrapped the orders call in a fallback, and the backend's error rate has been flat since. This evening the orders service is down for ten minutes, and a customer who placed an order yesterday opens the app.

```python
def account_screen(customer_id):
    profile = profile_api.get(customer_id)
    try:
        orders = orders_api.list(customer_id)
    except ServiceError:
        orders = []
    return {"profile": profile, "orders": orders}
```

What does that customer see?

<details><summary>A. An error screen, with the orders service down</summary>

✗ A failed call fails the request, and that is how this screen behaved before your change. The `except` catches `ServiceError` and puts an empty list in its place, so the backend answers normally and the app draws the screen.

</details>

<details><summary>B. The screen with yesterday's order, from the backend's last good reply</summary>

✗ A cache can serve its last good answer when its origin fails. This code keeps nothing between calls: the fallback is a literal `[]`, and yesterday's order is nowhere in the reply.

</details>

<details><summary>C. The screen, saying they have no orders</summary>

✓ Right. The app draws the screen with no orders, while yesterday's order sits safe in the service that is down, and the customer cannot tell this from a true empty list.

An empty list says the customer has no orders. This reply has no field that can say the orders call failed, and **without such a field a failure reaches the client as a false answer**, here in a reply with status 200. The flat error rate comes from the same `except`: the backend counts no failure, and neither does the app.

Give each part its own status in the reply, so the app can show that orders are unavailable, and count caught failures in a metric of their own.

</details>

<details><summary>D. The screen, with a note that orders are unavailable</summary>

✗ A page that degrades well shows a greyed-out panel with a note, and that is what you meant the change to give. The reply is a profile and a list, and a list cannot tell a failed call from a customer with no orders, so the app has no reason to show a note.

</details>

---

###### 10. Launching a twenty-item widget on the home page

An online shop adds a "picked for you" widget to its home page, which gets 2,000 views a second at peak: twenty products, each shown with its price. The page calls the recommendations service, which has no prices of its own and gets them from the pricing service through the call the product pages use, which takes one product. The recommendations team doubles its copies and load-tests at 2,000 requests a second with pricing stubbed, and the test passes. Pricing takes about 3,000 calls a second at peak from the product and checkout pages, and its team sizes it for twice that. On Saturday morning the widget goes live.

What gives way first?

<details><summary>A. Recommendations, the new service behind the widget</summary>

✗ A launch is where new code meets traffic, and this is the new code. Recommendations doubled its copies and passed a test at the 2,000 requests a second it now gets, so it has room for the traffic that reaches it.

</details>

<details><summary>B. Nothing, with pricing's spare capacity covering the widget</summary>

✗ Pricing is sized for 6,000 a second and takes 3,000, and 2,000 more views would fit. A view is not one price: it shows twenty products, and the product pages' call takes one, so the widget adds 40,000 calls a second.

</details>

<details><summary>C. The home page, with one more call on each view</summary>

✗ The widget is the one thing that changed on the home page, and every view now waits for it. That is one call per view, 2,000 a second to a service tested at that rate, and the page's servers have no prices to fetch themselves.

</details>

<details><summary>D. Pricing, and the checkout pages that call it</summary>

✓ Right. Pricing struggles first and the checkout pages slow with it, though pricing's team changed nothing.

**Each level of a call tree multiplies the calls of the level above by its fan-out**, and each view now asks for twenty prices: 2,000 views a second send 40,000 calls to a service sized for 6,000, about seven times over once its own traffic is counted. This fan-out puts the largest load on the deepest shared service, the one furthest from the launch and from the team running it.

Follow a new feature's calls down the tree before launch and give each owner its numbers. A call that takes a list of products costs pricing one call per view, and a cache of prices in recommendations cuts it further.

</details>

---

###### 11. Calling back into orders from pricing

At a pharmacy chain, orders calls pricing for each order, and pricing reads its database and then calls orders back to read how many orders the customer has placed, for its loyalty discount. Each service has 200 worker threads across its copies, one per request, with further requests queued, and calls block with a 30-second timeout. They run at about 30 percent CPU on 150 orders a second. At lunchtime the database behind pricing stalls for two seconds and then recovers.

What do orders and pricing do once the database is back?

<details><summary>A. Both catch up within seconds of the stall's end</summary>

✗ A short stall leaves a short backlog, and 30 percent CPU leaves room to drain it. Draining takes free workers, and these all wait on each other, so the two services stay stuck with the database already healthy.

</details>

<details><summary>B. Both sit at idle CPU while calls wait out their timeouts</summary>

✓ Right. Both services hold every worker with almost no CPU in use, and no order finishes until calls start to time out 30 seconds in, though the database came back long before.

During the stall, pricing's 200 workers fill up waiting on the database, and orders' 200 fill up waiting on pricing, which at 150 orders a second takes under two seconds. When the database returns, each pricing worker calls orders back and needs a free orders worker, and every one of those is waiting for pricing: **once two pools that wait on each other are full, neither can free a worker until a call times out**.

Break the cycle: pass the customer's order count in the call to pricing, so that pricing has nothing to call back for. Where a call back must stay, give it a pool of its own, which other work cannot fill.

</details>

<details><summary>C. Both run at full CPU, working through the backlog</summary>

✗ A backlog is work to burn through, and burning through work takes CPU. These workers are blocked on calls and use almost no CPU, so the figure falls during the trouble and the backlog waits.

</details>

<details><summary>D. Pricing slows while orders carries on with its workers free</summary>

✗ The stalled database is pricing's, and orders never touches it. Each orders worker waits on a pricing reply, so at 150 orders a second all 200 are held in under two seconds, with none left to answer pricing's calls back.

</details>

---

###### 12. Scaling up during a registry outage

During a sale weekend, the service registry behind a retailer's services goes down for twenty minutes. Callers hold lists of copies and replace them from it every 30 seconds. The stock service has ten copies, and as traffic climbs the orchestrator, the system that runs the copies, starts ten more, which pass their health checks. Calls go directly from caller to copy.

What happens to calls to stock over those twenty minutes?

<details><summary>A. Calls to stock fail until the registry is back</summary>

✗ A registry is a single point of failure, and everything here finds everything else through it. Callers only read it every 30 seconds, and a caller whose read fails goes on using the list it holds, so calls to the ten copies carry on.

</details>

<details><summary>B. The twenty copies share the calls evenly</summary>

✗ Scaling up adds capacity once health checks pass, and these ten have passed. Each caller sends only to the copies on its own list, and the way onto that list is the registry, which is down.

</details>

<details><summary>C. The ten old copies take every call, and the new ten sit idle</summary>

✓ Right. The ten old copies take every call and the ten new ones get none. The sale's climb lands on half the fleet, and nothing fails until those ten run out of room.

Callers read the list they hold, so the outage does not touch calls to copies already on it. **Under client-side discovery a new copy reaches callers only through the registry**: it cannot register, and they cannot refresh. Its passing health check says the copy is fit to serve and tells no caller it exists, so scaling stops working for as long as the outage lasts.

A copy that stops during the outage stays on the lists, so have callers move to another copy when a call fails. Run the registry as a replicated group, and watch for copies that receive no calls.

</details>

<details><summary>D. Calls go to the old copies, then fail as each list entry's TTL runs out</summary>

✗ A cached answer expires, as a DNS answer does after its TTL. A caller keeps its list until a refresh replaces it, so with no refresh the old entries stay. On discovery through DNS names, a TTL can run out instead.

</details>

---

###### 13. Adding a phone backend in front of six calls

At a courier company, the phone app draws its delivery screen with six small API calls in a row, because each call needs an id from the reply before it, and drivers use it out on the road. You add a backend for the app: a service in the data centre that makes the six calls itself, and the phone makes one call to it. In review, someone points out that this adds a hop to every screen.

What happens to the phone's wait for the delivery screen?

<details><summary>A. It grows by one hop</summary>

✗ Every hop adds latency, as the review comment says. The new hop costs about what one call inside a data centre costs, and it removes five of the phone's own round trips, each far longer.

</details>

<details><summary>B. It stays about the same, with the same six calls still running</summary>

✗ The services do the same work as before, and six calls still run. That work was never most of this wait: the phone waits on its trips over the mobile network, and it now makes one of them.

</details>

<details><summary>C. It falls only on repeat visits, once the new backend has cached replies</summary>

✗ Layers in front of services often cache replies, and a repeat visit is where a cache pays. This backend saves time on the first visit and caches nothing, and the saving comes from the trips the phone no longer makes.

</details>

<details><summary>D. It falls to about one phone round trip</summary>

✓ Right. The wait falls to about one phone round trip and a little more, and the delivery screen opens several times faster. Say a round trip takes 200 ms on a mobile network and 1 ms inside the data centre, and each call takes 5 ms of work: six calls from the phone take about 1.2 seconds, and one call to the backend about a quarter of a second.

The six calls depend on each other, so they run in order, and each one made from the phone pays a full trip over the slowest link on the path. A backend for frontend **moves that sequence to the side of the link where a round trip is cheap**. The hop it adds costs about what one call costs in the data centre.

Keep it beside the services and keep it thin, so that it shapes replies for the app while business rules stay in the services.

</details>

---

###### 14. Turning on mesh retries for all routes

On a food-delivery platform, the orders service calls ledger through a client that logs each attempt it makes. The platform team turns on retries in the service mesh for all routes, and the release note reads "no code changes, more resilience". A week later a bad deploy makes ledger answer 503 to every call for ten minutes.

| Layer | Attempts per call | Retries on |
| --- | --- | --- |
| orders client | 3 | 503 |
| mesh, all routes | 3 | 5xx, connection reset |

During those ten minutes, how do the requests reaching ledger compare with the attempts in the orders log?

<details><summary>A. Several times as many requests as attempts in the log</summary>

✓ Right. Ledger gets several times what the log records: each of the client's 3 attempts goes to the proxy, which makes up to 3 of its own, so one call reaches ledger 9 times while the orders log shows 3.

The proxy sits below the client and answers it once per attempt, so the client sees one 503 after the proxy has used up its tries. A service mesh is transparent in both directions: the code needs no change, and **the code's log cannot see what the mesh does**. The policy also covers connection resets on every route, after which ledger may have done the work, and the proxy cannot tell which calls are safe to repeat. Why attempts multiply across layers is on [Timeouts, retries and circuit breakers](../timeouts-retries-and-circuit-breakers/README.md).

Count requests at the callee or from the proxies' own figures. Keep retries in one layer, and leave out of the mesh's policy any route whose calls are not safe to repeat.

</details>

<details><summary>B. The same number of requests as attempts in the log</summary>

✗ A mesh takes on retries so that code need not, and the release note says no code changed. The client's attempts are still on, and the log counts only those: each goes on to the proxy, which repeats it where the log cannot see.

</details>

<details><summary>C. Fewer requests than attempts, with the proxies absorbing the repeats</summary>

✗ Proxies can also hold calls back, with breakers and limits. This policy sets retries and nothing that holds calls back, so every attempt in the log leaves the proxy, and a failing one leaves it up to 3 times.

</details>

---

###### 15. Changing a default in a shared contracts library

A marketplace's services share a `contracts` library that defines the requests they pass as JSON, with a serialiser that leaves out any field holding its default value. Version 3 changes the default of `Shipment.signature_required` from true to false, and no method signature changes. Each service's tests run against stubs built from the library, and they all pass after the upgrade. Orders takes version 3 and ships on Monday. The courier service, on version 2, is due to release next week. On Monday afternoon a customer who chose "leave at the door" buys a lamp, and orders sends the courier service the shipment.

What does the courier service do with that lamp?

<details><summary>A. It asks for a signature at the door</summary>

✓ Right. The courier service asks for a signature, so the lamp is not left at the door as the customer chose.

Orders on version 3 holds false, which is now the default, so it leaves the field out. The courier service on version 2 finds no field and fills in its own default, true. Each service's stubs come from its own copy of the library, so the tests put one version on both sides. **At run time each service holds the library version it was built with**, and across a fleet that releases apart the versions can differ for weeks.

Treat what goes over the wire as the contract: test an old reader against a new writer, and add a new field where a meaning changes, with defaults always sent.

</details>

<details><summary>B. It leaves the lamp at the door</summary>

✗ The library exists to keep both sides in step, and its tests are green on every service. Those tests run one version at a time, while here version 3 leaves out the false that version 2 needs to see, and version 2 reads the gap as true.

</details>

<details><summary>C. It refuses the shipment for a missing field</summary>

✗ Strict parsers refuse a message with a field missing, and this message lacks one. The library drops such fields as normal practice, so version 2 expects gaps and fills this one with its default.

</details>

---

###### 16. Adding an optional field a client does not know

Customer records live in a customer service, behind `GET` and `PUT` on `/customers/{id}`. You add an optional `delivery_note` field to the JSON, and the support tool starts filling it in. Billing's client, a tolerant reader, was generated a year ago and has not changed. When a customer moves, billing fetches the record, sets the new address on its object and sends it back with `PUT`. A customer leaves a note on Monday, and on Wednesday billing records their new address.

What does that customer's record hold on Thursday?

<details><summary>A. The new address and the note, which billing's client carried through</summary>

✗ Adding an optional field is a safe change, and billing has never heard of this one. Its class holds only the fields it knows, so its `PUT` goes out without the note and replaces the record that had it.

</details>

<details><summary>B. The old address and the note, with billing's write refused</summary>

✗ Servers validate writes, and billing's body lacks a field the record now has. The field is optional, so a body without it is valid, and the service accepts the `PUT` and replaces the record with that body.

</details>

<details><summary>C. The new address and no note</summary>

✓ Right. The record holds the new address and no note. The customer's instruction has gone, with no error on either side.

A tolerant reader skips what it does not know, and **what a client skips on the way in it cannot send on the way out**. Billing's object has no `delivery_note`, the JSON built from it has none, and a `PUT` replaces the record. Adding the field was safe for every client that only reads. A client on a binary format that keeps unknown fields and writes them back out would carry the note through, and JSON parsed into a class does not.

Send only what changed, with `PATCH`, or make the service refuse a `PUT` based on an older version of the record. Before adding a field, look for clients that write back what they read.

</details>

---

###### 17. Removing a field nobody reads

Every reply from the customer service has carried a `fax` field since its first version, and the published schema has marked all its fields required from the start. Before removing it you search the code of the five calling services, and no handler reads it. The invoices service's use is typical, through a client generated from that schema. You take `fax` out of the replies on Tuesday.

```python
customers = CustomersClient(base_url, validate_replies=True)

def invoice_header(customer_id):
    customer = customers.get(customer_id)
    return f"{customer.name}, {customer.address}"
```

What does the invoices service get from Tuesday's lookups?

<details><summary>A. The same customers as before, with nothing changed</summary>

✗ The search of five services found no reader, and a field with no reader does nothing. The generated client is a reader: `validate_replies=True` checks every reply against a schema that requires `fax`, and the reply no longer has it.

</details>

<details><summary>B. Customers as before, with the fax number read as empty</summary>

✗ A missing JSON field comes back as null in most code, as `customer.fax` would if a handler asked for it. The check runs before any handler does, and a required field that is missing fails it, so no customer object comes back.

</details>

<details><summary>C. A failure only on customers who have a fax number</summary>

✗ Only records with a fax number have a value to lose. The reply lacks the field on every customer, so the required check fails on every customer.

</details>

<details><summary>D. A failed parse on every customer lookup</summary>

✓ Right. Every lookup fails, though no handler reads `fax`: the generated client checks the reply, finds a required field missing and fails the call before `invoice_header` sees a customer, and invoices stop on Tuesday.

A search of the code finds the handlers, and the check reads the whole reply: **checking a reply against a schema makes the client depend on every field the schema requires, used or not**. That is the coupling a tolerant reader avoids. A consumer-driven contract test shows the provider only the fields each caller states it needs, so it would pass here while this client still checks `fax`.

Mark the field optional in the schema first, wait until every caller's client has been regenerated and released, and only then remove it from the replies.

</details>

---

###### 18. Moving invoices behind the facade

A utility is moving off its monolith by routing: a facade in front sends `/invoices` calls to a new invoice service and everything else to the monolith. All `/invoices` traffic has gone to the new service for a month, and its logs look right. Deleting the monolith's invoice module is on next quarter's plan, and every night the monolith runs its billing as it always has, which makes the day's invoices. Today a new VAT rate takes effect, and the team ships it in the new service.

What VAT rate does tonight's billing run put on its invoices?

<details><summary>A. The new rate, now all invoice traffic goes to the new service</summary>

✗ A month of logs shows all invoice traffic at the new service, and the facade is there to send it. Those logs cover the traffic that passes the facade, and the billing run stays inside the monolith, where it calls the old module with the old rate in it.

</details>

<details><summary>B. The old rate</summary>

✓ Right. Tonight's invoices carry the old rate, and customers are billed at a VAT that no longer applies.

The billing run calls the monolith's invoice module inside its own process, and the new rate is in another service. **A facade sees only what crosses it**, and a call between modules never reaches it, so the month of clean logs covers the traffic from outside and the old code stays live on every internal path.

Before you count a route as moved, list the module's callers inside the monolith and point them at the new service. Then delete the module, so that a caller you missed fails at build time.

</details>

<details><summary>C. No rate, with the run failing on a module cut off behind the facade</summary>

✗ A module with no route to it is gone as far as outside callers can tell, and the plan is to delete it. It is still built into the monolith, and a facade cannot reach inside a process, so the run works as it always has.

</details>

<details><summary>D. Both rates, with two invoices per customer</summary>

✗ Migrations do run old and new side by side, as a shadow run does, to compare results. A shadow run needs someone to send one customer down both paths, and the run's calls stay in the monolith, so each customer gets one invoice, from the old module.

</details>

---

###### 19. Flipping the address route back after two days

A travel firm is moving off its monolith by routing. Two days ago its facade began sending every `/addresses` call to a new address service, which follows database per service and took a one-off copy of the monolith's table that morning. Now a bug shows up in the new service's validation, and the on-call engineer follows the rollback plan: flip the route, one line in the facade, live in seconds.

What address do customers who moved during those two days now see?

<details><summary>A. Their new address, which both services hold</summary>

✗ The new service started as a copy of the monolith's table, and replication keeps copies of a table in step. These two have no such link: since the cut-over, edits have gone to the service's database alone, and the monolith's table stopped that morning.

</details>

<details><summary>B. An error until the two databases are merged</summary>

✗ Migration plans have a step for reconciling two sources of truth, and here are two stores that disagree. The monolith does not wait for that step: it reads its own table, finds an address and shows it, with no error to see.

</details>

<details><summary>C. The address they had before the cut-over</summary>

✓ Right. They see the address from before the cut-over, and a customer who moved finds the old home on the screen. The monolith reads its own table, and two days of changes sit in the new service's database.

The route flips in seconds, and **the data stays where it was written**. A strangler fig reverses cleanly for reads, and for writes only as long as they reach both stores.

Before a cut-over that moves writes, decide how they get back: copy them to the monolith as they happen, or keep a tested job that copies them back. Without either, treat the route as one-way and fix the validation in the new service.

</details>

---

## Further reading

- [Life beyond Distributed Transactions](https://www.cidrdb.org/cidr2007/papers/cidr07p15.pdf), 2007: why large applications are built without distributed transactions, and how the unit that can change atomically shapes them.
- [The tail at scale](https://doi.org/10.1145/2408776.2408794), 2013: why a request that fans out to many servers is ruled by their slowest replies.
- [Microservices](https://arxiv.org/abs/1606.04036), 2016: where the style came from, the state of the art at the time and its open problems.

Prev: [WebSockets and real-time delivery](../websockets-and-real-time-delivery/README.md) · Next: [Service-to-service security](../service-to-service-security/README.md)
