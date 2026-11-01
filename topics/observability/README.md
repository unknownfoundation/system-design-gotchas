# Observability

Prev: [Releasing changes](../releasing-changes/README.md) · Next: [Load testing](../load-testing/README.md) · Reviewed October 2026

Observability is what a running system tells you about itself: the metrics, logs and traces its services send out, and the alerts built on them. Each signal keeps some things and leaves the rest behind when the request has gone.

- **Metric**: a named number tracked over time, such as requests served or connections in use, read by a metrics store.
- **Label**: a name and value attached to a metric, such as `route="/order"`, which lets a graph split it.
- **Series**: the stream of values for one combination of a metric's label values. A metric's cardinality is how many series it has.
- **Counter**: a metric that only goes up, from zero when its process starts. A gauge, such as connections in use, goes up and down.
- **Histogram**: a metric that counts requests per latency bucket, with the bucket edges set in advance.
- **Percentile**: the p99 is the value one request in a hundred exceeds.
- **Log**: a record a service writes for one event, often as fields a store can filter on, with the request's trace id so that one request can be followed through the services.
- **Trace**: the record of one request's path through the services, which shows the path it took and where its time went.
- **Span**: one timed step in a trace, such as a call to a service, with a start, an end and a parent.
- **Trace context**: the trace id and current span id that each service passes to the next, in the `traceparent` HTTP header or in a message's headers.
- **Baggage**: key-value pairs that an application adds to the trace context, so that later services can read them.
- **Head sampling**: deciding whether to keep a trace when its first service receives the request.
- **Tail sampling**: holding a trace's spans for a while, then deciding from what they show.
- **Probe**: a scripted request sent to check a service, along one path from one place.
- **Target and error budget**: the share of requests that must succeed in a period, such as 99.9 percent in a month, and the failures that share leaves room for. The burn rate says how fast errors are spending that budget.

The three signals answer different questions:

| Signal | What it answers |
| --- | --- |
| Metric | how much and how often, over months |
| Log | what happened at one event, by any field it recorded |
| Trace | which services one request crossed, and how long each step took |

A periodic scrape reads a value at that moment, so a gauge that rises and falls between two scrapes never shows it. A rate over a counter treats any fall as a restart from zero, so take the rate on each copy of a service and then add the rates. Under load a pipeline can drop what it cannot carry, and not every pipeline counts what it dropped.

A page on a cause, such as high CPU, can fire with no user affected and miss a fault with another cause, so pages usually go on what users see. One fault can raise alerts on every service that depends on it. A periodic job is watched by the age of its last success, since a job that never starts reports no failure.

The gaps show during an incident, when the graphs, traces and alerts you open were set up months earlier for other questions.

Unless a question says otherwise, traffic runs level through the day and the month.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- say what a metrics store kept after you add a label, a canary or a release, and read its percentiles knowing how they were made
- follow one request through a trace that crosses a proxy, a sampler, a queue's batch and an outside call, and say what the trace leaves out
- work out what a dead log store does to requests, and what a figure counts when a whole service crashes, a gateway retries, a callee slows down or a nightly rebuild goes wrong
- write alerts against a target, and say what each one does when errors stop, when a route is small, when copies vanish and when a region goes

###### 1. Adding an app version label to find last Tuesday's failures

A food delivery backend counts its requests in a metric labelled by route and status, and the metrics store keeps a year of it. Support suspects that version 4.2 of the phone app failed more often last Tuesday. Each request carries the app's version in a header, and the access logs record that header. This morning you add an `app_version` label to the metric and deploy it.

What does the metric show for last Tuesday, split by version?

<details><summary>A. Last Tuesday's failures, split by version</summary>

✗ A database table answers any new filter over the rows it already holds, and the metrics store holds a year of this metric. The store never held the requests: the service added 1 to a count per route and status as each one passed, and the version was not part of that count.

</details>

<details><summary>B. Last Tuesday's failures, all under the version each copy reports today</summary>

✗ Adding a column with a default to a table fills in the old rows. A metrics store has no rows to fill and no record of which version sent a count, so last Tuesday's series keep the labels they had.

</details>

<details><summary>C. Nothing split by version before this morning's deploy</summary>

✓ Right. The split by version starts at this morning's deploy. For last Tuesday, this metric gives support only the failures by route and status it always had.

A metric is added up as requests pass: the service adds 1 to a number for that combination of labels, and the request itself is gone. A label you add later **splits only what is counted from then on**.

The access logs hold one record per request with the version in it, so ask them, for as long as they are kept. For questions nobody has planned, keep one record per request with its fields, since you can work out a count from records.

</details>

<details><summary>D. An empty Tuesday, with the old series dropped by the change</summary>

✗ Resetting the schema of a test database throws out what it held. A new label only creates new series beside the old ones, so last Tuesday's series stay where they are and a sum by route still reads them.

</details>

---

###### 2. Adding a customer label to a latency histogram

A streaming service keeps a latency histogram labelled by route and status. A product manager wants latency per customer, so you add a `customer_id` label. In staging, with ten test customers, the metrics store barely notices. Production has two million customers, most of whom send a few requests a month, and the rollout does not change the request rate.

What does the label do to the metrics store in production?

<details><summary>A. Its load stays about the same</summary>

✗ The request rate is flat and staging barely noticed. A metric's cost goes by its series rather than its requests, and staging had ten customers where production has two million.

</details>

<details><summary>B. A set of series per customer, at the same request rate</summary>

✓ Right. The store takes on a set of series for each customer, with the request rate unchanged, and its memory and the bill go on climbing through the month as each customer sends a first request.

**Each distinct combination of label values is a series of its own**, per bucket, whatever its traffic, and each costs memory, disk and network for as long as it is kept. Say the histogram has 12 buckets across 20 routes and 5 statuses, 1,200 series today. With each customer on only one route and one status, the label makes it 24 million.

Keep `customer_id` out of metric labels. Put it in the request's log record or span, where it costs one field, and keep labels to values you could list by hand.

</details>

<details><summary>C. One more row per request, as in a log store</summary>

✗ A log store grows by one line per request, and a metric records what happened too. It keeps no rows: a request adds to a count, so the store grows with the combinations of labels, and a customer with one request a month costs a set of series, as one with a million does.

</details>

<details><summary>D. New series for active customers, with quiet ones costing nothing</summary>

✗ A quiet customer's series gets few writes, so it is cheap to update. It still sits in the store for the whole retention, and in the memory of each process that reported it, so a customer who sent one request last month goes on costing a set of series.

</details>

---

###### 3. Adding 170 ms to every request under fixed latency buckets

The checkout API of a ticket site keeps a latency histogram with bucket edges at 100, 200, 400 and 800 ms, and the dashboard shows its p95. Almost every request takes about 210 ms. Today's release adds about 170 ms to every request, and the on-call engineer watches the p95 line as it rolls out.

What does the p95 line do as the release rolls out?

<details><summary>A. It rises by about 170 ms</summary>

✗ The dashboard prints the p95 to the millisecond, as a stopwatch would. It is an estimate from bucket counts, and a request of 380 ms lands in the same bucket as a request of 210 ms, so the estimate has nothing new to work from.

</details>

<details><summary>B. It jumps up to the 400 ms edge</summary>

✗ The edge at 400 ms is the next mark above these requests, and a store could report a bucket's upper edge. Such a store would already have read 400 ms before the release, since the 210 ms requests were in that bucket, so the line would not jump.

</details>

<details><summary>C. It climbs slowly as old counts leave its window</summary>

✗ A graph over a window lags a real change while old values age out. Old and new counts sit in one bucket here, so nothing changes as they age, and the line has no reason to begin climbing.

</details>

<details><summary>D. It stays where it was</summary>

✓ Right. The p95 line stays flat while every customer waits 170 ms longer.

A histogram **keeps one count per bucket and no values**, and a request of 210 ms and a request of 380 ms each add 1 to the bucket from 200 to 400 ms. The counts do not change, and a percentile worked out from counts cannot change either. The line moves only once requests cross an edge, past 400 ms here.

Put edges close together around the latency you care about, since the error is as wide as the bucket. Even then, a release check that reads this line misses any change that stays inside one bucket.

</details>

---

###### 4. Averaging p99 across ten copies during a canary

A booking API runs on ten copies, and its dashboard shows the average of the ten copies' own p99 figures. Today one copy is a canary, a copy running a new release, and the balancer sends it one request in a thousand. Its p99 is 5 s. The other nine answer every request in about 50 ms. The dashboard reads about 545 ms, and the release manager asks whether customers are feeling it.

What is the p99 across all of the API's requests?

<details><summary>A. About 50 ms, as on the other nine copies</summary>

✓ Right. The p99 across all requests is about 50 ms, far from the dashboard's 545 ms, so customers hardly feel the canary.

A p99 belongs to one set of requests, and **an average of copies' p99 figures gives each copy the same weight, however little traffic it took**. The canary's 0.1 percent of requests is too small a share to fill the slowest 1 percent of all requests, and the rest of that 1 percent took about 50 ms.

Keep a histogram on each copy with the same bucket edges, add the bucket counts, and read the percentile from the total. Watch a canary on a panel of its own.

</details>

<details><summary>B. About 5 s, the canary's figure</summary>

✗ A p99 is about the worst requests, and the worst requests here are on the canary. There are too few of them: at most one request in a thousand comes from that copy, and the slowest 1 percent of all requests is mostly filled with 50 ms ones.

</details>

<details><summary>C. About 545 ms, as the dashboard reads</summary>

✗ The dashboard's line is the figure in front of you, and an average over copies is the usual roll-up. It gives the canary a tenth of the weight on a thousandth of the traffic, and it lands far above the 50 ms that 999 requests in 1,000 took.

</details>

<details><summary>D. No figure can be told from these numbers</summary>

✗ The warning that percentiles cannot be combined is right in general, since a copy's p99 leaves out how many requests it saw and how they were spread. Here the rest is known: 999 requests in 1,000 took about 50 ms, which settles the p99 whatever the canary did.

</details>

---

###### 5. Tracing a booking through an old fare proxy

A travel site traces every request. A booking goes from the gateway to the bookings service, through an old fare proxy, to the fare service. Each service's tracing library reads the incoming `traceparent` header, records a span, and sends the header onward. The fare proxy was written years ago: it creates no spans, and it forwards a fixed list of headers that does not include `traceparent`.

What does the trace viewer show for one booking?

<details><summary>A. Two traces, one ending at bookings and one starting at fares</summary>

✓ Right. The viewer shows two traces for one booking: gateway and bookings in one, and the fare service alone in another with a different id. The gateway's trace of a slow booking shows bookings waiting and nothing under it.

Trace context travels hop by hop, and **a service with no incoming `traceparent` starts a new trace with a new id**. Nothing joins the two later, because they share no id.

Add `traceparent` to the proxy's list, or have the proxy record a span of its own. When a trace stops short, look first for the hop that drops the header.

</details>

<details><summary>B. One trace, with a gap where the fare proxy sits</summary>

✗ A viewer draws one request as one timeline, and these spans come from one booking moments apart. The viewer joins spans by trace id alone, and the fare service's span carries a new one, so it never appears under bookings, gap or no gap.

</details>

<details><summary>C. One trace ending at bookings, with no spans from fares</summary>

✗ Tracing libraries follow the header, and here no header arrives. A service with no incoming context becomes the root of a new trace, and this site traces every request, so the fare service does record its spans, under an id the bookings trace does not hold.

</details>

<details><summary>D. No trace, and an error from fares about the missing header</summary>

✗ A missing required header earns a 400 from most APIs. Nothing about the booking needs `traceparent`: the tracing library takes its absence as the start of a trace, and the booking goes through.

</details>

---

###### 6. Halving the longest span on a slow product page

A furniture shop has a product page that takes about 900 ms. The handler calls reviews and stock at the same time, calls pricing once stock answers, waits for all three, and renders. This trace is typical, with times in ms from the start of the request. A colleague points at reviews, the longest span under the page, and the team halves it.

```text
page     0 ......................................... 900
reviews    20 ............. 420
stock      20 ...... 240
pricing               240 ...... 460
render                                    790 ...... 900
```

What does the next trace show the page taking?

<details><summary>A. The page shorter by all the time saved on reviews</summary>

✗ A trace is often taught as the sum of its steps, and reviews is the longest bar. That bar runs beside stock and pricing, and pricing finishes later, at 460 ms, so a shorter reviews leaves the last call to finish where it was.

</details>

<details><summary>B. The page shorter by part of that time, shared with the calls beside it</summary>

✗ Calls that hold one resource, such as a small thread pool, share a wait. Pricing starts at 240 ms, the moment stock answers, so it waited for nothing that reviews held, and the handler moves on when the last of them answers, which is pricing at 460 ms.

</details>

<details><summary>C. The page at about 900 ms, as before</summary>

✓ Right. The next trace shows about 900 ms, as before. The customer gains nothing from the team's work: reviews now ends at 220 ms instead of 420 ms, while pricing still finishes at 460 ms.

With calls in parallel, **only the path that sets the end time counts**. Here it runs through stock, pricing, a gap of 330 ms and render, and reviews is not on it. The gap is the largest piece of that path: time no span covers, such as a queue for a worker, a lock or code with no span.

Read a trace backwards from its end, following what each step waited for, and add spans across the gap before you change any call.

</details>

<details><summary>D. The page shorter, with the gap before render closing up too</summary>

✗ The gap comes right after the calls, where time would pass if the handler were still waiting on them. Reviews ended at 420 ms, before pricing did, and the gap begins at 460 ms, so whatever fills it is not waiting on reviews and does not shrink with it.

</details>

---

###### 7. Looking for the trace of one failed payment

A bank's payments API runs head sampling at its gateway and keeps 1 percent of its traces. About one payment in 100,000 fails with a timeout deep in the ledger service. Each log line carries the trace id, so when support reports a failed payment from this morning, you pull its id from the logs and open the trace store.

What do you expect to find in the trace store for that payment?

<details><summary>A. The whole trace, with the failure marked</summary>

✗ A tracer that sees an error and keeps the trace is how tail sampling works. At the gateway the request has not run, so there is no error to see, and the later services follow the gateway's decision.

</details>

<details><summary>B. No trace of that payment at all</summary>

✓ Right. You should expect nothing: with 1 percent of traces kept, 99 failed payments in 100 leave no trace at all, and you hold a log line with an id and no trace to open.

Head sampling **decides when the request arrives, before anything has gone wrong**, and a payment that will fail has the same one chance in a hundred as one that will not.

Decide after the request has run, with tail sampling, or log each failed payment in full.

</details>

<details><summary>C. Only the ledger service's failed span</summary>

✗ The ledger service knows it failed, and an error reporter sends exactly that one event. Under head sampling a span that was not sampled is not exported, whatever became of it, so the failed span goes with the rest.

</details>

<details><summary>D. A trace rebuilt from the services' logs</summary>

✗ Viewers show logs beside a trace, and these lines hold the same trace id. A viewer shows them only beside a trace that exists, and building one needs each step's parent, start and end, which a log line does not hold, so those lines stay your only record of the payment.

</details>

---

###### 8. Sending spans to five tail-sampling collectors

To keep every slow or failed trace, an insurer moves to tail sampling: collectors hold a trace's spans for 30 seconds after they first see it, then send it to the trace store if any span failed or took over two seconds. To share the work, services send their spans to five collectors behind a round robin balancer. A claim crosses eight services, none of them slow, and the last, an optional fraud check, fails at once while the claim goes through without it.

What does the trace store get for that claim?

<details><summary>A. All eight services' spans, as one trace</summary>

✗ Spans share a trace id and the store joins them by it, which is how a trace comes together when all its spans arrive. Only one collector sees a reason to send anything here, so the store never gets the other spans to join.

</details>

<details><summary>B. All eight services' spans, stored up to five times over</summary>

✗ Workers acting alone usually mean duplicates, and each collector does decide alone. A span goes to only one collector, though, so no two hold a copy, and the collectors with no failed span send nothing.

</details>

<details><summary>C. Nothing, with the trace dropped by the collector that saw it first</summary>

✗ Under head sampling the first service to see a request decides for all of it. A collector has no such role: it decides for the spans it holds, and the one holding the failed span sends it.

</details>

<details><summary>D. A fragment: the failed span and any others on its collector</summary>

✓ Right. The store gets a fragment: the failed span and whatever else landed on its collector. The failure shows with most of the claim's path missing, since the other collectors saw only spans that look fine and dropped them.

Tail sampling decides from the spans in hand, and it **works only if all of a trace's spans reach one collector**. A balancer that spreads spans by connection or by request, as round robin does, splits one trace across collectors. If each span lands on any of the five with equal chance, about 2.4 of the eight arrive at the collector holding the failed span.

Route by trace id: a first tier hashes each span's trace id and sends it to a second tier that does the sampling. Size that tier for 30 seconds of all your spans, since this is where the memory goes.

</details>

---

###### 9. Writing 200 orders' invoices in one transaction

Each order's event goes from the order service to a queue, with the trace context in the message's headers. The invoice consumer takes batches of 200 events, from 200 different traces, and writes all 200 invoices in one database transaction. You want to reach the slow write from any of those orders' traces, and your weekly report sums span time per service.

How do you record the write?

<details><summary>A. One span with all 200 traces as its parents</summary>

✗ A parent is whoever caused the work, and 200 orders caused this write. The `traceparent` header has room for one parent id and the tracing specifications give a span zero or one parent, so the 200 cannot all be parents.

</details>

<details><summary>B. One span under the first event's trace</summary>

✗ Whatever context is current becomes the parent, so a library left alone often does this, and the first order you open does show the write. The other 199 orders' traces end at the queue, with nothing that leads to the write.

</details>

<details><summary>C. One span in a trace of its own, with a link to each of the 200</summary>

✓ Right. Record one span in a trace of its own, with a link to each of the 200 events' trace contexts. A link ties the write to every order's trace, and the weekly report counts its time once.

A span has zero or one parent, so a tree cannot hold a write that belongs to 200 traces. A link **points from one span to a span in any trace and claims no parent**.

Start the batch span when the batch is read and add the links at creation, since a sampler sees only what exists then.

</details>

<details><summary>D. A copy of the span in each of the 200 traces</summary>

✗ Copying a post into each follower's feed is fan-out on write, and every order would see the write. Say it takes two seconds: the weekly report then sums 400 seconds for that service, for one write that took two.

</details>

---

###### 10. Putting the customer's email in trace baggage

A retail platform traces every service with automatic instrumentation from its tracing library, which adds tracing with no code of your own, and keeps the default settings. The edge service puts the customer's email in baggage, so that every service can add it to its spans. The order service calls an outside payment provider over HTTP.

What does the payment provider's server receive on that call?

<details><summary>A. The customer's email in a request header</summary>

✓ Right. The provider's server receives the customer's email in the `baggage` request header, where it can land in that company's own logs.

On default settings, automatic instrumentation **adds trace context and baggage to most outgoing requests**, and it does not ask where a request is going. To the tracing library, baggage is key-value text with no meaning, and the standard for it warns that values can travel on to other systems.

Keep personal values out of baggage, and carry an opaque id if a service needs one. Strip `baggage` and `traceparent` on calls that leave your trust boundary.

</details>

<details><summary>B. The trace context, with baggage held back at your edge</summary>

✗ Only the trace id shows in a viewer, so it is the part people think of as travelling. Baggage rides in a header of its own beside `traceparent`, the client writes both, and nothing at your edge is there to hold it back.

</details>

<details><summary>C. The customer's email, hashed by the tracing library</summary>

✗ Logging frameworks offer masking rules for emails and card numbers. A tracing library gives baggage no meaning, so it has no idea this value is an email, and it goes out as you set it.

</details>

<details><summary>D. No tracing headers on a call to an outside host</summary>

✗ To a person, a call to an outside provider and an internal call are different things. The client sees a URL and a request, and on default settings it adds the same headers to both.

</details>

---

###### 11. Writing every request's log line to standard output

A ticketing API writes one log line per request to standard output, from its request threads, with an ordinary write call. On each node a log agent reads that output from a pipe and sends the lines to a central log store. At 20:00 the central store goes down. The agent retries, fills its own fixed buffer, and stops reading.

What happens to ticket requests once the agent stops reading?

<details><summary>A. They carry on, with their log lines lost</summary>

✗ Logging counts as best effort. That needs a step that drops the line, and an ordinary write has no such step: at a full pipe, it waits.

</details>

<details><summary>B. They carry on, with lines piling up in the API's memory</summary>

✗ Language runtimes buffer standard output in a small fixed block. That block empties into the pipe, and once the pipe is full the flush waits too, so memory stays flat and the threads are what gets stuck.

</details>

<details><summary>C. They fail at once with a broken pipe error</summary>

✗ A write to a pipe with nobody at the far end fails at once, with a broken pipe error. That needs the reader to have closed its end, and this agent still has the pipe open and has stopped reading from it, so the writer waits.

</details>

<details><summary>D. They hang at the log write</summary>

✓ Right. With the central log store down, request threads stop at the log write one after another, and ticket sales hang with them.

A pipe holds a fixed amount, and **a blocking write to a full pipe waits until the reader takes something out**. The agent has stopped reading, so the pipe fills after a burst of lines, and every thread that reaches a log line waits there.

Set the path to drop when full, with a non-blocking write and a bounded buffer, and count what it drops. An unbounded buffer in the service's own memory grows until the process dies. Test by stopping the agent under load.

</details>

---

###### 12. Counting availability from the application's access logs

A logistics company reports availability each month from its application's access logs: requests with a status under 500, over all requests logged. Last month it came to 99.98 percent. On one afternoon that month, a bad deploy crashed every copy for 40 minutes, and the balancer in front answered customers with 502.

What did those 40 minutes take off the month's figure?

<details><summary>A. Their full share of the month</summary>

✗ Availability is taught as uptime over time, and 40 minutes is 0.09 percent of a month. This figure counts lines in a log, and none of those 40 minutes' requests reached a running application to write one.

</details>

<details><summary>B. More than their share, with customers' retries logged as failures</summary>

✗ Customers retry in an outage, and retries multiply what a live service sees. Each retry here meets the balancer's 502 as well and reaches no copy, so it writes no line of any kind.

</details>

<details><summary>C. Next to none of their share of the month</summary>

✓ Right. Those 40 minutes took next to nothing off the figure: the company reports 99.98 percent for a month with a 40-minute outage.

A figure **counts only what the place measuring it can see**: a copy that is down writes no line, so a failed request is missing from the top and the bottom of the ratio alike. If traffic runs level through the month, the same month counted at the balancer comes to about 99.89 percent.

Count at the balancer, and run a probe from outside for what never reaches it, such as a certificate chain only a real client checks.

</details>

<details><summary>D. A figure marked invalid for the gap in its logs</summary>

✗ A careful report flags missing hours, and some pipelines check for them. A ratio over the lines that exist cannot tell a quiet hour from a dead one unless someone built that check, and this one comes out as an ordinary percentage.

</details>

---

###### 13. Reading 9 percent errors behind a retrying gateway

On a food-ordering platform, the gateway retries a failed read once, on another copy. One of the menu service's eleven copies has gone bad and fails each call after a two-second wait. The menu service's own graph reads about 9 percent errors for the afternoon, its on-call engineer is paged, and the head of support asks how many customers are affected.

What are customers getting from the menu route this afternoon?

<details><summary>A. Errors on about 9 percent of menu loads</summary>

✗ The menu service's own figure is what paged someone, and it is the number nearest the fault. It counts attempts, and the gateway tries again on another copy after each failure, so none of those errors reach a customer.

</details>

<details><summary>B. Menus on every load, with some about two seconds slower</summary>

✓ Right. Customers get a menu on every load, and about one load in ten takes two seconds longer. The answer for support is that nobody sees an error and some customers wait.

**A graph below a retrying layer counts attempts**, and here the gateway turns each failed attempt into a success on another copy. With attempts spread evenly, one bad copy out of eleven takes one attempt in eleven, which is the 9 percent. The gateway's own figure shows no errors, and that is the layer customers meet.

Page on errors and latency as high up the stack as you can, here at the gateway, and keep the per-copy figures for finding the bad copy. Take the bad copy out of rotation, since each load that reaches it still waits those two seconds.

</details>

<details><summary>C. Errors on far more than 9 percent, with retries piling onto the service</summary>

✗ A retry storm piles onto a failing service until it falls over. One retry after each failure adds about a tenth to the load here, spread over ten healthy copies, which is far too little to start one.

</details>

<details><summary>D. Menus on every load, at their usual speed</summary>

✗ A retry that succeeds leaves the gateway's success rate all green. The customer still pays for it: that load first waits out the two seconds on the bad copy, and only then goes to a healthy one.

</details>

---

###### 14. Reading two error graphs when tax lookups slow down

In an online shop, the order service calls a tax service for each order. It gives each call one second, and then fails the order. Each team has a graph of its own service's 5xx responses, and a page on it. At 18:00 the tax service's disk slows down, and every tax lookup now takes two seconds.

What do the two graphs show at 18:05?

<details><summary>A. The order service's graph red and the tax service's flat</summary>

✓ Right. At 18:05 the order service's graph is red, since every order fails at its timeout, and the tax service's graph is flat, so the page goes to the order team and none goes to the tax team.

**A timeout is an error at the caller**: the order service gives up after a second and counts a failure, while the tax lookup carries on for its two seconds and answers with a success nobody reads. A 2017 paper, Gray Failure, calls this differential observability: one party is harmed while another sees no problem.

Give the caller a graph of time and timeouts per dependency, so the order team can name the tax service, and page the tax team on its own latency.

</details>

<details><summary>B. The tax service's graph red and the order service's flat</summary>

✗ The slow disk is in the tax service, and the tax team has a graph of that service's 5xx responses. A slow disk causes no error there: each lookup ends in a late success, and the service that stopped waiting counts the failure.

</details>

<details><summary>C. Both graphs red, with the tax service's first</summary>

✗ Cascade diagrams show a failure moving up from the bottom, with the tax service first. Nothing fails at the bottom here, so its graph has nothing to show at any point, and only the order service's goes red.

</details>

<details><summary>D. Both graphs flat, with only latency moving</summary>

✗ Slow and failed are usually filed apart, with latency on one graph and errors on another. The one-second limit joins them: past it, a slow call is a failed order, and a 5xx on the order service's graph.

</details>

---

###### 15. Rebuilding a product search index overnight

An outdoor-gear shop rebuilds its product search index each night. One morning, customers tell support that tents have vanished from the shop. Last night's rebuild left about a third of the catalogue out of the index, and the rest of it is fine. The team's pages on 5xx share and p99 latency stayed quiet all night. You add one alert for this kind of fault.

Which alert would have paged on last night's fault?

<details><summary>A. An outside probe that searches each minute and checks for a 200</summary>

✗ An outside probe walks the way a customer does, which is why people run them. Checking the status, it gets a 200 for a search with no results, and its one term tests one part of the catalogue, which may still be in the index.

</details>

<details><summary>B. The p99 of search latency falling below its usual level</summary>

✗ A sudden speed-up can mean work is being skipped, as an old on-call rule has it. The p99 sits among the slowest requests, the searches with full answers, and they take what they always took, so with a third of the searches coming back empty the line dips too little for a rule to tell from a normal day.

</details>

<details><summary>C. Errors written to the search service's log</summary>

✗ A bug that reaches the on-call engineer usually logs an error. The search service ran as written on the index it was given, so its code failed nowhere, and it logs no error for a search that finds no products.

</details>

<details><summary>D. The share of searches with no results, against its usual level</summary>

✓ Right. The share of searches with no results jumps from its usual level, to about a third if searches spread evenly over the catalogue. This alert pages once the new index takes searches, before support hears about tents.

A search for a missing product answers 200 with an empty list, quickly. Every layer that reads status codes counts it as a success and the slowest requests are untouched, so 5xx share and p99 barely move, and **a figure about the result itself can catch a wrong answer sent with a 200**.

Pick, for each service, a figure about what customers came for, and page on a sharp move from its usual level at that hour. Some searches find nothing on any day, so the rule needs that baseline.

</details>

---

###### 16. Paging on error share at the target's own limit

A video platform runs an API with a target of 99.9 percent of requests succeeding in a month, and the rule below pages the on-call engineer. This month's only errors come from the daily deploy, which runs at about 3 percent errors for three minutes.

```yaml
alert: api-errors
expr: error_share(window=5m) > 0.001
notify: page
```

How does the month go for the on-call engineer?

<details><summary>A. No pages, with three minutes shorter than the window</summary>

✗ Three minutes is shorter than the rule's five-minute window, and windows are there to smooth short blips. A window averages, so three minutes at 3 percent still come to 1.8 percent over five, far above the limit.

</details>

<details><summary>B. A page with every deploy, and the month's target met</summary>

✓ Right. The rule pages with every deploy, 30 pages this month, and the month's target is comfortably met, with about 6 percent of the budget spent, so a page at deploy time comes to mean nothing.

The target is a limit over a month, and **a five-minute share above 0.1 percent does not show the month's target at risk**, since a few minutes above it can spend almost none of the budget. A 99.9 percent target leaves room for about 43 minutes of total failure a month, and each deploy here spends the equivalent of 0.09 minutes of it.

Alert on how fast the budget is going, the burn rate.

</details>

<details><summary>C. One page, on the day the month's target breaks</summary>

✗ The rule's limit is the target's own 0.1 percent. It reads only the last five minutes, and the month, which never breaks here, is not in the rule at all.

</details>

<details><summary>D. A few pages, only on days with heavier traffic</summary>

✗ A busy day brings more errors, counted one by one. This rule reads a share, and the deploy's 3 percent is the same share at any traffic, so it fires every day.

</details>

---

###### 17. Waiting for a burn-rate alert to clear after a rollback

A payments API with a 99.9 percent target pages when its error share over the last 60 minutes passes 1.44 percent, a rate that would spend the month's error budget in about two days.

```text
14:00  every request starts failing
14:01  the page goes out
14:10  the rollback lands, and errors stop completely
```

When does the alert clear?

<details><summary>A. Within a minute or two of the rollback</summary>

✗ Status lights follow what they watch within a short delay, and the errors have stopped. This rule watches a share over the last 60 minutes, and the ten failed minutes are still inside it.

</details>

<details><summary>B. When the month's error budget has recovered</summary>

✗ The limit is worked out from the budget, as a rate that would spend it in about two days. The rule reads only the hour: a month's spent budget stays spent long after this alert has cleared.

</details>

<details><summary>C. About an hour after the rollback</summary>

✓ Right. The alert clears about an hour after the rollback, at about 15:09, so for those 59 minutes it stays red on a fixed service and says nothing about whether the rollback worked.

**A rolling window goes on counting failed minutes until they slide out of it**, even after the errors stop. Right after the rollback the share is about 17 percent, and it stays above 1.44 percent until under a minute of them is left.

Make the rule need a short window as well, so that it fires only while the last hour and the last five minutes are both over the limit. The five-minute share falls under the limit within five minutes of the rollback, and the alert clears then.

</details>

<details><summary>D. When the on-call engineer acknowledges it</summary>

✗ Paging tools keep notifying until someone takes the page. Taking it stops the notifications, and the rule goes on reading its window, true until the failed minutes have slid out.

</details>

---

###### 18. Finding a lunchtime checkout failure in the day's takings

A fashion retailer pages on its API when the error share over an hour passes 1.44 percent, the fast limit its 99.9 percent target sets. Browsing makes up 99 requests in every 100, and checkout the other one. At lunchtime a bad payment setting got every checkout refused immediately for 50 minutes, the share sat near 1 percent, nobody was paged, and finance found out from the day's takings. You want a page for this fault.

Which change gives you that page?

<details><summary>A. A rule of its own on checkout's requests</summary>

✓ Right. A rule of its own on checkout's requests reads 100 percent errors within minutes, long before finance reads the day's takings.

A share over all requests **lets a large route hide a small one**: 99 requests in 100 are browsing and healthy, so losing every checkout can reach only about 1 percent, whatever the window.

Give checkout its own rule. At quiet hours it needs a longer window or a minimum count of requests, since on a route with ten requests an hour, one failure reads as 10 percent.

</details>

<details><summary>B. Add a six-hour window with a limit of 0.6 percent</summary>

✗ Long windows catch what short ones miss, and the usual recipe pairs a fast rule with a slow one at a lower limit. Over six hours these 50 minutes come to about 0.14 percent, far under 0.6, so the longer window sees even less of this fault.

</details>

<details><summary>C. Shorten the rule's window to five minutes</summary>

✗ A long window delays an alert, so a shorter one is the first fix after a missed page. It changes how soon the share moves and not how high it gets: over five minutes the share is still about 1 percent, under the limit.

</details>

<details><summary>D. Page when checkout's p99 latency rises</summary>

✗ A broken route usually gets slow, and latency is the first graph people open. These checkouts are refused rather than slowed, so checkout's p99 has no reason to rise, and a rule watching for a rise stays quiet.

</details>

---

###### 19. An error alert over copies that stop reporting

The copies of a hotel booking API each keep a counter of 5xx responses, and a metrics store scrapes them every 15 seconds. The store evaluates this rule, the only one the team has for the service. At 03:00 a bad config stops every copy at once, and every booking fails. The metrics store stays up.

```yaml
alert: booking-errors
expr: rate(http_5xx, window=1m) > 5
for: 2m
notify: page
```

What does the rule do through the night?

<details><summary>A. It pages within about two minutes</summary>

✗ Every request is failing, and this rule pages when 5xx counts rise. That needs copies left to count their 5xx. Tonight none is left, so the rate has no value to compare with 5.

</details>

<details><summary>B. It never fires</summary>

✓ Right. The rule never fires, and nobody is paged all night while every booking fails.

The rule compares what its query returns with 5, and with no copy to scrape the query soon returns nothing. With nothing to compare, **the comparison never comes true**, and an empty result reads exactly like a quiet service. Tools differ on missing data: some alarms move to a state of their own and page only if you set them to, a tool that runs its own rules can raise a no-data alert by default, and a rule the metrics store evaluates itself, as this one is, stays quiet.

Add a rule that fires when the series is absent or a target cannot be scraped, and keep a probe outside that makes a booking.

</details>

<details><summary>C. It sends its own page saying the service stopped reporting</summary>

✗ The metrics store knows which targets it was scraping, and its own status pages list them as down. A page about a missing target comes from a rule someone wrote for it, and this team wrote only the one.

</details>

<details><summary>D. It pages on a spike, with the stopped counters read as restarts</summary>

✗ A rate treats a fall in a counter as a restart from zero, and these counters have gone. A fall needs a later, lower value to compare, and a stopped copy reports none, so the store sees no fall and no spike.

</details>

---

###### 20. Moving the alert rules to a second region

An e-commerce team once ran everything in one cloud region. So that a region's failure cannot take the alerts with it, they moved the metrics store and the alert rules to a second region, and a probe there now pages through a paging provider when the API stops answering. A game day, with the API stopped by hand, paged within two minutes. The API and the company's internal apps, sign-on among them, stay in the first region, the dashboards and logs sit behind that sign-on, and company sessions end each evening. Tonight that region's network fails, with the on-call engineer asleep.

What does the on-call engineer have to work with tonight?

<details><summary>A. A page, and the dashboards and logs as usual</summary>

✗ The metrics store and the rules are in the healthy region, which was the point of the move. The data is safe there, and the way in is company sign-on, which sits in the failed region.

</details>

<details><summary>B. No page, with the rules left holding no data</summary>

✗ A rule on the API's own counters has no data once they stop. This probe keeps working from the second region and records a failure on each try, and a failure is data, so the rule fires.

</details>

<details><summary>C. A page, and dashboards open with no sign-in while sign-on is down</summary>

✗ Some gateways let requests through on some paths when their auth check is down. These dashboards need a session that sign-on issued, and this engineer, signed out, has none.

</details>

<details><summary>D. A page, and a sign-in screen that will not load</summary>

✓ Right. The page arrives within minutes, since the probe in the second region sees the API stop, and the sign-in screen will not load. Sign-on sits in the failed region, and the dashboards and logs behind it are out of reach.

The metrics store and the rules moved, and sign-on did not: **whatever the path from alert to engineer still needs in the failed region fails with it**. The game day stopped the API by hand, so sign-on stayed up and the test never touched this path.

List everything that stands between a failure and a person acting on it, and test by cutting each one. Here that means a way into the tools with no shared sign-on.

</details>

---

## Further reading

- [Gray Failure](https://doi.org/10.1145/3102980.3103005), 2017: how a component can look healthy to its failure detectors while the applications that use it suffer.
- [Sifter](https://doi.org/10.1145/3357223.3362736), 2019: sampling traces by favouring those that a model of common behaviour fits poorly.
- [Moment-Based Quantile Sketches for Efficient High Cardinality Aggregation Queries](https://arxiv.org/abs/1803.01969), 2018: small summaries that merge quickly, for estimating quantiles over many sub-populations.
- [The Mystery Machine](https://www.usenix.org/conference/osdi14/technical-sessions/presentation/chow), 2014: building a model of how requests execute from the logs that components already write.

Prev: [Releasing changes](../releasing-changes/README.md) · Next: [Load testing](../load-testing/README.md)
