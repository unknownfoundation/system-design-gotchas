# Load testing

Prev: [Observability](../observability/README.md) · Next: [Reliability and disaster recovery](../reliability-and-disaster-recovery/README.md) · Reviewed October 2026

A load test sends a system traffic made by a program and records how it answers, to learn what a given load does before real traffic brings it. What comes out is a report, and the report describes one run: the traffic the program sent, on the path it took, against the data and the fleet it found.

- **Generator**: the program that sends the test traffic and times the replies, on one machine or several. It has CPU, local ports and open files of its own, and it can run out of them. Its timer stops only once its own code has read a reply, so a generator that falls behind adds that wait to the latency it reports.
- **Virtual user**: one simulated client inside a generator, running its script in a loop.
- **Think time**: the pause a virtual user takes between requests, standing in for a person reading.
- **Closed model**: a fixed number of virtual users, each sending its next request only after the reply to its last one.
- **Open model**: requests sent at a rate the test sets, whatever became of the earlier ones.
- **Target rate and achieved rate**: the rate a test was set to send, and the rate it sent.
- **Coordinated omission**: a test that sends less while the system stalls, so its figures leave out the wait the held-back requests would have met.
- **Ramp**: raising the load step by step over minutes, in place of starting at the full rate.
- **Throughput**: requests finished per second.
- **p99**: the 99th percentile of latency, the time that one request in a hundred exceeds.
- **Hit rate**: the share of reads a cache answers itself; the rest are misses, which go to the store behind it.
- **TTL**: how long a cached copy may be served before it counts as expired.
- **Zipf-like**: a skew in which a few items draw most of the reads and the rest trail off in a long tail.
- **Edge location**: one of a CDN's sites, with a cache of its own. A shield is a second tier of cache between the edge locations and the origin, the server that holds the real content.

Tests go by what they do with the load:

| Kind | What it does |
| --- | --- |
| Peak test | holds the expected peak for a while |
| Stress test | climbs past the expected peak until the system gives way, to see how it fails and whether it comes back unaided |
| Spike test | starts from little and goes to the full rate at once |
| Soak test | holds a steady load for hours or days |

A virtual user is not a customer. It renders nothing, fetches only what its script names and reads each reply as soon as its machine can, where a phone on a slow network holds its connection and the server's buffers for longer. It gives up only at the timeout you set and does not try again as an app does, so a run holds neither the retries that add load in production nor the customers who leave.

The first minutes of a run belong in the report under their own heading. Thrown away as warm-up, they leave the cold start after a restart or a flush untested.

A database much smaller than production's fits in memory, and a data set of a few rows makes its own lock waits. A script that reuses one login or one record gets refusals in place of the work it meant to run. A test of one service with its dependencies stubbed loads nothing they share, while a test of the whole stack names only the first part to give way. A smaller copy of production shrinks the parts that scale out and none of the shared limits.

One run is one sample: the same test repeated gives other numbers, more so on shared machines, and a p99 from a few hundred requests rests on a handful of them. A gate on a build turns thresholds into pass or fail, and it needs a stored baseline to see a slow drift across releases. Some teams test with live traffic, shifting more of it onto part of a fleet and stopping when errors or tail latency cross a line, which puts real customers at risk.

A request a run never sent, a limit it never reached, a fault it never met and a node it never lost all stay unknown after a pass.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- choose how a test sends its load, and say what a slowdown or a four-second stall does to the traffic and to the figures in its report
- work out whether a figure on the report belongs to your service, to the generator or to the path between them
- build traffic and data that a real day would recognise, from reused connections and one cloud region to a log's top endpoints and a recorded replay
- judge what a green run covers, compare two builds and plan the next run before a sale, a long weekend or a test in production

###### 1. Holding 200 virtual users while search gets slower

A travel site runs a load test every night: 200 virtual users, each sending a search, waiting for the reply and sending the next. Searches take about 100 ms. Tonight's build adds a call that takes each search to a full second, and the test starts on time with the same 200 virtual users.

What does tonight's run send the servers, next to last night's?

<details><summary>A. More searches a second, with virtual users piling up</summary>

✗ Slow searches pile up on a real site, and each virtual user stands for one more visitor. A virtual user holds one request at most and sends nothing while it waits, so the 200 can never have more than 200 searches in flight, and a slower reply means fewer sends.

</details>

<details><summary>B. The same searches a second</summary>

✗ The script and the number of virtual users on the report have not moved, and that number is what the plan calls the load. It fixes how many senders there are. How fast each one sends follows its reply time, and tonight that time went up tenfold.

</details>

<details><summary>C. The same searches a second, until the servers run out of threads</summary>

✗ A test is run to find where a site falls over, and a tenfold slowdown is a likely place. These virtual users cannot push it there: at most 200 searches are ever in flight, and the rate drops at the moment the slowdown begins.

</details>

<details><summary>D. Far fewer searches a second</summary>

✓ Right. The servers get about 200 searches a second tonight against about 2,000 last night. The build that slowed search also took nine tenths of the load off it.

Each virtual user sends again only when its reply arrives, so **on the closed model slower replies mean fewer requests sent**. Two hundred virtual users at 100 ms make 2,000 a second, and at a second they make 200. The two nights' latency figures now describe two different loads.

Read the achieved rate beside the latency on every run, and treat a fall in it as a finding.

</details>

---

###### 2. Writing the load test for a warehouse's handheld scanners

A warehouse runs 300 handheld scanners, one per operator, and each operator spends the shift walking the aisles, scanning parcels and reading each answer off the screen. You are writing the load test for the week before Christmas, when every operator works a full shift. A colleague has read that tests with a fixed set of users hide overload, and wants a generator that sends a fixed rate of scans whatever the servers answer.

Which load reproduces that week?

<details><summary>A. 300 virtual users who wait for each reply and pause between scans</summary>

✓ Right. The test needs 300 virtual users who wait and pause. An operator cannot start the next scan until the screen has answered, and no more than 300 scans are ever waiting.

**A load model copies how the traffic behaves**, and this traffic is closed by nature: a fixed crew, each person waiting for an answer. Say an operator takes ten seconds between scans and the servers slow to 20 scans a second. The 300 settle at a reply of about five seconds, while a generator that keeps sending the usual 30 a second builds a backlog of nearly 6,000 scans in ten minutes, more than 300 people can produce.

To choose a model, ask who sends the next request and when. Take the think time from real shift data, and keep the open model for traffic that does not wait, such as a public sale or a partner's webhooks.

</details>

<details><summary>B. A fixed rate of scans, sent whatever the servers answer</summary>

✗ Your colleague has the right worry for a public site, where strangers keep arriving whatever became of the earlier ones. Each of these 300 operators stands waiting for an answer on a screen, so a fixed rate sends scans nobody would send and reports an overload this warehouse cannot have.

</details>

<details><summary>C. 300 virtual users who wait for each reply and never pause</summary>

✗ A script with no pause is what a tool gives you until you add one, and it gets treated as a safe worst case. If a scan takes 50 ms, 300 virtual users with no pause send about 6,000 scans a second, far more than a crew walking the aisles, and the failure it finds belongs to that far larger load.

</details>

<details><summary>D. 3,000 virtual users who wait for each reply and pause between scans</summary>

✗ Ten times the expected load is a margin some plans ask of a peak test. That week brings a fixed crew, and a warehouse with 300 scanners has no way to put 3,000 people behind them.

</details>

---

###### 3. Reading a flat p99 at 5,000 virtual users before a sale email

The stress test at a ticket site climbs from 500 to 5,000 virtual users, each in a loop with no pause. Throughput flattens at 3,000 requests a second. At 5,000 virtual users the p99 sits at about 1.7 s with no errors, and the report says the site degrades gently. Next week a sale email brings 4,000 requests a second for ten minutes. The service queues what it cannot take at once.

What happens to the wait during those ten minutes?

<details><summary>A. It rises for a minute or so and then falls back</summary>

✗ On a good day a queue fills in a brief spike and clears within moments. A queue clears only while arrivals run below what the site finishes, and here they stay 1,000 a second above it for all ten minutes.

</details>

<details><summary>B. It stays below 1.7 s</summary>

✗ 5,000 is a larger figure than 4,000, and the stress test got 1.7 s at 5,000. The two figures count different things: 5,000 virtual users are senders who wait, and in their loop they send only 3,000 a second, which is less than the email brings.

</details>

<details><summary>C. It keeps growing until the burst ends</summary>

✓ Right. The wait keeps growing until the burst ends: 4,000 requests arrive each second, the site finishes 3,000, and 1,000 more join the queue. The people the email brings wait about 20 s after the first minute and about 200 s at the end.

The stress test could not show this, because it ran on the closed model. With 5,000 virtual users no more than 5,000 requests are ever in the system, and **a fixed count of senders puts a ceiling on the wait**: 5,000 over 3,000 a second, about 1.7 s. That flat p99 is a property of the generator. Arrivals on the open model have no such ceiling, and Open Versus Closed, a 2006 study, found mean response times on the open model that can exceed the closed model's by an order of magnitude or more at equal load.

Test public traffic on the open model, past capacity, and read how long the queue takes to drain. Then decide what the service should refuse once its queue is full.

</details>

<details><summary>D. It settles near the 1.7 s of the stress test</summary>

✗ The stress test went past 4,000 virtual users on its way to that flat line, and its report calls the result gentle. The line holds only for senders who wait for their replies. The email's readers do not wait for one another, so the queue passes it within the first minute.

</details>

---

###### 4. Sending one request every 12 ms from the generator

A thumbnail service runs one worker, and each request takes about 10 ms. You test it with a generator that sends one request every 12 ms, on the dot, and the report gives a p99 of 10 ms. The real callers are phones of customers who know nothing of one another, at the same average rate.

What p99 do the phones get?

<details><summary>A. A little above it, by a few milliseconds</summary>

✗ Real traffic is messier than a test, and a little jitter on each gap is the usual allowance. Independent phones do more than shift each gap by a little: several of them can land inside one 10 ms job, and each one then waits for all those ahead of it.

</details>

<details><summary>B. Many times the 10 ms</summary>

✓ Right. The phones get a p99 many times the 10 ms, with the worker no busier than in the test. Say arrivals are fully random at that average rate: the p99 comes to about 140 ms.

A request waits only if it arrives while the worker is busy. Evenly spaced requests each find it free, with 2 ms to spare. The worker is busy 83 percent of the time in both cases, but **independent arrivals bunch up, and a bunch has to queue**. The idle time between bunches cannot serve a request that has not arrived yet.

Set the generator's gaps to random, and add the real bursts you know of, such as a batch client that sends its work in one go.

</details>

<details><summary>C. About the same 10 ms</summary>

✗ The worker does the same work at the same average rate, busy 83 percent of the time either way. An average leaves out what each request meets: on the dot, every request finds the worker free, and at random a request often finds several ahead of it.

</details>

<details><summary>D. One request's wait on top of the 10 ms</summary>

✗ One worker has one job in progress at a time, and it has time to spare between jobs. A queue holds as many requests as arrived while the worker was busy, and at 83 percent busy it often holds several. A request behind ten of them waits about 100 ms.

</details>

---

###### 5. Stopping the service for four seconds in a hundred-second run

A bank's nightly test runs for 100 s with 10 virtual users, each sending a balance query, waiting for the reply and sending the next. Replies take about 1 ms. Halfway through tonight's run, the service stops answering for 4 s while its database flushes to disk, and then carries on as before.

What does the run's p99 say about those four seconds?

<details><summary>A. It reads about 1 ms</summary>

✓ Right. The p99 reads about 1 ms, as on any other night, though this stall would have held up every customer arriving in those four seconds. Only the maximum, 4 s, gives it away.

This is coordinated omission: **while the system stalls, a test that waits for replies sends nothing new**. Each of the 10 virtual users has one request waiting, and four seconds of trouble add 10 slow samples to 960,000. On a schedule of a request per millisecond for each virtual user, 4 percent of the requests would have fallen due inside the stall, and the p99 would read about 3 s.

Read the maximum beside the p99, and time each request from when it was due rather than when it left.

</details>

<details><summary>B. It reads about 4 s</summary>

✗ The stall fills 4 percent of the run, and a p99 leaves out only the slowest 1 percent. A p99 is ranked over requests, not over seconds of the run. These virtual users sent ten requests in those four seconds and 960,000 in all.

</details>

<details><summary>C. It reads tens of milliseconds</summary>

✗ A bad few seconds usually drag a figure up, and a mean would move a little. A percentile sits at one rank in the sorted samples, and 10 samples of 4 s stand far past that rank, leaving the p99 on the fast replies around it.

</details>

<details><summary>D. It is missing, with the run marked as failed</summary>

✗ Tools do fail a run when calls hang, and 4 s is a long time for a 1 ms call. Nothing gave up here: each of the ten waiting requests got its reply when the service came back, and the run counted them as ordinary successes.

</details>

---

###### 6. Moving the nightly test onto a fixed schedule

After a stall went unseen, you move a price API's nightly test to a fixed schedule: 1,000 requests a second, shared among 20 worker threads, each running the function below. `client.get` is a thin wrapper around the HTTP client, and replies take about 1 ms. Tonight the API stops answering for 4 s in the middle of the run.

```python
def worker(schedule):
    for due_at in schedule:
        clock.wait_until(due_at)
        started = clock.now()
        client.get("/price")
        histogram.record(clock.now() - started)
```

What does the histogram hold for the requests due during the stall?

<details><summary>A. An entry of up to 4 s for every request due in the stall</summary>

✗ The schedule fixes when each request is due, and a fixed schedule is the published cure for hidden stalls. This histogram does not use the schedule. Its subtraction starts from `started`, taken after `wait_until` returns, so a request due early in the stall and sent after it gets an entry of about 1 ms.

</details>

<details><summary>B. No entries for them, with each one skipped</summary>

✗ Some generators drop what they cannot send on time and report a count of the dropped requests. This loop has no such branch: for a time already past, `wait_until` returns at once, and the late request goes out and gets its entry.

</details>

<details><summary>C. An error for every request that went out late</summary>

✗ A request that misses its slot has failed the plan, and the plan is what this test now checks. No line here compares the time with `due_at` after the wait. `client.get` returns an ordinary reply, and the histogram records how long that one call took.

</details>

<details><summary>D. A long entry from each of the 20 threads, and ordinary ones for the rest</summary>

✓ Right. The histogram holds one entry of about 4 s from each of the 20 threads and entries of about 1 ms for the rest, though 4,000 requests were due in the stall. The p99 still reads about 1 ms, and the stall stays out of view.

A thread blocks on its call, so 20 calls hang and the requests behind them wait in their loops. When the API comes back, `wait_until` returns at once for each late request, `started` is taken, and a fast reply is timed from there: **timing from the send leaves out the wait before it**. That is coordinated omission, inside the timer. Timing from `due_at` gives over 3,000 entries above a second and, if the run lasts 100 s, a p99 of about 3 s.

Record the time since the request was due, `clock.now() - due_at`, and keep the achieved rate on the report beside it.

</details>

---

###### 7. Signing off a run that sent 4,100 a second against a target of 5,000

The peak test of a food-delivery API is set to 5,000 requests a second, and the lead engineer wants to sign off that figure against a p99 target of 100 ms. Server CPU stayed near 50 percent on every node. This is the summary of the run.

```text
target rate     5,000/s
achieved rate   4,100/s
p99             40 ms
errors          0
```

What does this run show about the API at 5,000 requests a second?

<details><summary>A. It holds 5,000 with a p99 of 40 ms</summary>

✗ 5,000 is what the test was set to, and every line the sign-off reads is green. The target is a request to the generator, and the p99 of 40 ms belongs to 4,100 a second. It carries over to no higher rate, least of all near a limit, where latency climbs fastest.

</details>

<details><summary>B. No result at 5,000, only one at 4,100</summary>

✓ Right. The run measured the API at 4,100 a second, the rate it got, and a sign-off at 5,000 would rest on a load that was never sent.

**An achieved rate under the target means something held the test back**, and the summary cannot say what. It can be the generator's CPU, its ports or its open files, a link or a quota on the way, or the servers themselves. Say the generator can send only 4,100 a second and the servers could take 8,200: they run at 50 percent with no errors, any wait the generator adds is already inside the 40 ms, and the summary looks like this one.

Treat an achieved rate under the target as a failed run, whatever the p99. Find what stopped it, watching the generator's own machine first, and run again.

</details>

<details><summary>C. It tops out at 4,100</summary>

✗ A flat figure under a target usually marks a limit, and the API is what is under test. The summary points away from the API: CPU is near 50 percent on every node, no request failed and the p99 is well inside its target, and a generator short of CPU or ports would give the same picture.

</details>

<details><summary>D. It lost the difference without logging it</summary>

✗ Overloaded systems do drop work, and 900 requests a second are unaccounted for here. A dropped request shows up as a timeout or an error, and this summary has none. The gap sits between the rate the test was set to and the rate it sent, so these requests were never sent at all.

</details>

---

###### 8. Reading 1,100 a second with the fleet at 5 percent CPU

At a travel agency, the launch test runs on eight generator machines in a private subnet, and they reach the site through the subnet's NAT gateway. The site's 20 backends sit behind a balancer set to IP hash, which picks a backend from the client's address. Throughput flattens at 1,100 requests a second with no errors, and CPU averaged over the fleet reads 5 percent. The generators, the gateway and the balancer all have room to spare.

What does the 1,100 a second measure?

<details><summary>A. The limit of a lock all 20 backends share</summary>

✗ A lock or a shared row gives a flat rate at low CPU, and 5 percent is very low. With a lock, requests would reach all 20 backends and wait there. Here 19 of them get no requests at all, and the one that gets them all runs out of CPU.

</details>

<details><summary>B. A limit the balancer puts on each address</summary>

✗ Balancers can cap what one address may send, and these eight machines share one. A cap turns the excess away, and this run has no errors. IP hash refuses nothing: it chooses a backend, and the requests wait on the one it chose.

</details>

<details><summary>C. What one backend can do</summary>

✓ Right. The 1,100 is what one backend can do. A launch plan built on this figure would size the site at a twentieth of what the fleet can carry.

A NAT gateway gives all eight generators one public address, and **IP hash sends every request from one address to one backend**. The other 19 sit idle, and the mean of 5 percent is one backend at full CPU, shared out over 20. Over many addresses the same hash evens out: say 10,000 addresses, and each backend takes about 5 percent of them.

Give the generators many source addresses on different networks, or turn the hash off for the test, and read CPU per backend, never only the mean.

</details>

<details><summary>D. What the 20 backends can do, with CPU to spare</summary>

✗ The whole rate went through the balancer to the fleet, and the fleet is what is under test. A fleet at its limit runs out of something on every backend, and a mean of 5 percent is nowhere near that. Nineteen of these backends never saw the test.

</details>

---

###### 9. Sending launch traffic down 200 reused connections

The launch test for a banking app sends 20,000 requests a second from generators that open 200 TLS connections at the start and reuse them for the whole run. The balancer that ends TLS holds at 15 percent CPU. On launch day the same 20,000 requests a second come from phones that have never reached this service, each opening its own connection for a visit of a few requests.

What does the balancer's CPU do on launch day?

<details><summary>A. It climbs above 15 percent</summary>

✓ Right. The balancer's CPU climbs above 15 percent, though the request rate has not moved. A launch plan that counts on 15 percent is short of what it will need.

A full TLS handshake does **public-key work that a request on an open connection skips**, and the test paid for 200 of them, all at the start. Say a visit is five requests, which makes 4,000 handshakes a second, and a 16-core balancer spends 0.12 ms on a request and 1 ms on a handshake. It goes from 15 percent to 40 percent, and the size of the jump depends on the key type.

Have the test open new connections at the rate real visits do, and set a target for new connections a second beside the one for requests.

</details>

<details><summary>B. It falls below 15 percent</summary>

✗ Spreading work out is how most things get lighter, and 200 connections carried all of it in the test. A request costs the same on whichever connection carries it, and each new connection adds a handshake on top.

</details>

<details><summary>C. It holds near 15 percent</summary>

✗ The request rate matches the test, and the request rate is what a capacity plan counts. It leaves out how those requests arrive. In the test the handshakes number 200 in the whole run, and on launch day they come by the thousand every second.

</details>

<details><summary>D. It holds near 15 percent while memory runs out</summary>

✗ A connection costs memory while it stays open, and that is what a plan counts per client. A handshake's cost falls on the CPU, in the seconds when it happens, so the balancer's CPU does not hold near 15 percent whatever happens to memory.

</details>

---

###### 10. Driving a launch test from one cloud region through a CDN

A video site has 200,000 title pages behind a CDN with edge locations on every continent and no shield, each page with a one-minute TTL. The launch test sends the forecast worldwide rate, a few thousand requests a second, with pages in last month's proportions, from generators that all run in one cloud region. At launch, the same rate comes from viewers all over the world.

What does the origin get at launch, next to the test?

<details><summary>A. Less than in the test, with more caches sharing the load</summary>

✗ More edge locations at work means more cache in front of the origin, and that is what a CDN sells. Those caches do not pool what they hold: with no shield, each edge location asks the origin for its own copy of a page, and each copy expires on its own.

</details>

<details><summary>B. About what it got in the test</summary>

✗ The total rate and the pages match the forecast, and launch checklists record a hit rate as the CDN's own. It belongs to the traffic each cache sees. In the test a few caches near the generators see the whole rate, and at launch each edge location sees only its share.

</details>

<details><summary>C. A burst in the first minute, then what it got in the test</summary>

✗ Empty caches fill once at the start of anything, and then they serve. A one-minute TTL means the filling never ends: every copy expires each minute, and a page read less than once a minute at an edge location misses every time it is read.

</details>

<details><summary>D. Far more than in the test</summary>

✓ Right. The origin gets far more at launch than in the test, so one sized from the test falls short.

A CDN sends each client to a nearby edge location, so generators in one region load only the few caches near them. There every page is read often, and a copy rarely expires between two reads. Over the world each edge location gets a sliver of the traffic, and **hit rate depends on the traffic each cache sees by itself**: a less popular page has expired by its next read. Say 5,000 requests a second over 60 edge locations, with popularity falling off by a Zipf-like curve of exponent 0.8: the hit rate falls from about 72 percent at one edge location to about 24 percent, and the origin goes from about 1,400 requests a second to about 3,800.

Run generators in the regions the viewers are in, in proportion, or add the gap to the origin's load when you size it. A shield would turn many of those misses into one request to the origin.

</details>

---

###### 11. Building the test script from the top endpoints in the log

A school-management service builds its load test from last month's access log: the five endpoints that make up 97 percent of requests, in their real proportions, at last month's peak rate. Each costs a few milliseconds of database work. The script leaves out the rest, among them a term report export, one request in a thousand, that keeps the database working for many seconds.

How much of production's database work does the test produce at that rate?

<details><summary>A. Nearly all of it</summary>

✗ The script holds 97 percent of the log's requests in their real proportions, and coverage gets reported that way. A database does not count requests: its load is rate times cost, and an export can cost thousands of times what a usual request does. The one request in a thousand brings most of the work.

</details>

<details><summary>B. A small part of it</summary>

✓ Right. At the same rate the test produces a small part of production's database work, and the database looks comfortable.

Counting requests leaves out what each one costs, and **load on a database is rate times cost**. Say every request but the export costs 5 ms and the export 20 s: averaged over all requests the test costs 5 ms of database work and production about 25 ms, of which the export brings 20 ms, so the test does about a fifth.

Rank the log by total time rather than by count, and run the exports at their real rates beside the rest, along with the batch jobs and other background work that an access log never shows.

</details>

<details><summary>C. More of it than production's</summary>

✗ The same total rate now falls on the five endpoints alone, so each one gets a little more than its real share. That extra is 3 percent more requests at a few milliseconds each, tiny beside the exports the script left behind.

</details>

---

###### 12. Drawing listing ids evenly from a catalog of ten million

A marketplace has ten million listings, and a shared cache that holds 200,000 of them. Last year's load test read one listing over and over and came out far too fast, so this year each virtual user draws a listing at random from the whole catalog, every listing equally likely. At the forecast peak the test's hit rate is 2 percent and the database runs at full CPU.

What hit rate does production show at the same peak?

<details><summary>A. Below 2 percent</summary>

✗ Real shoppers wander through odd corners of a catalog, and a long tail is what marketplaces are known for. A long tail does not make reads even. An even draw is the flattest traffic there can be, and any crowding on popular listings lifts the hit rate above it.

</details>

<details><summary>B. Close to 100 percent</summary>

✗ Last year's test showed what repeated reads do, and shoppers do crowd onto bestsellers. Last year's script read one listing, and shoppers also read a long tail. Under the skews studied for web traffic, a cache of 2 percent of the items still misses a good share of reads.

</details>

<details><summary>C. About the same 2 percent</summary>

✗ The cache and the catalog are the same, and random ids were this year's fix for an unrealistic test. The 2 percent is the cache's share of the catalog, and it is the hit rate only when every listing is equally likely. Real reads are not spread that way, so the figure belongs to the draw.

</details>

<details><summary>D. Well above 2 percent</summary>

✓ Right. Production's hit rate is well above the test's 2 percent, because real reads crowd onto popular listings and the cache holds those. The database at full CPU comes from this test's even draw.

Every listing has the same chance, so **an even draw makes the hit rate the cache's share of the catalog**. Real requests are skewed: Web Caching and Zipf-like Distributions, a 1999 study of web traces, found a Zipf-like curve whose exponent varies from trace to trace. A marketplace's own curve is an assumption here. Say the exponent runs from 0.64 to 0.83 and the cache keeps the 200,000 most read of the ten million: the hit rate comes to about 24 to 48 percent. A cache that evicts by recency gets less, though a simulation at a tenth of these sizes still came well above 2 percent.

Draw ids with the popularity your own log shows, tail included, so the cache gets the same chance it gets in production.

</details>

---

###### 13. Replaying yesterday's checkouts against this morning's copy

A shop wants its sale test to match real life, so it records yesterday's checkouts from the balancer, bodies and headers included, and replays them at the recorded pace against a copy of the production database taken this morning. Each checkout carries the idempotency key the app generated for it, a value the checkout service keeps for a week in that database. A small tool swaps in a fresh sign-in token for each recorded customer.

What does each replayed checkout get from the service?

<details><summary>A. Yesterday's saved answer, with no new order</summary>

✓ Right. Each replayed checkout gets back the answer saved yesterday under its key, with no new order and no stock or payment work, in about the time a key lookup takes. The report shows fast, successful checkouts while checkout's real work never runs.

**A replay repeats the requests but not the state they met**, and a recorded write meets a database where it has already happened. A service answers a key it holds with the saved reply, and this copy was taken after yesterday's orders, so it holds every key in the replay. Say a checkout took 120 ms yesterday and a lookup takes 3 ms: the replay reports 3 ms.

Restore a copy from before the recorded day, or give each replayed checkout a new key and new ids, and after the run count the new orders against the checkouts sent.

</details>

<details><summary>B. A new order, on top of yesterday's</summary>

✗ A replay sends real POSTs, and a POST to checkout creates an order. These carry the keys they carried yesterday, and the service looks up the key before it creates anything. The copy already holds each one, so no order is added.

</details>

<details><summary>C. A 409 conflict</summary>

✗ These keys were spent yesterday, and 409 is the status for a clash. A conflict goes to a repeat that arrives while the first is still running, and a key reused with a different body gets an error too. These match requests that finished yesterday, and a finished request has a saved answer to give.

</details>

<details><summary>D. Yesterday's order written again over the same rows</summary>

✗ Idempotent is usually taught as doing a thing again with the same result, as a PUT overwrites its row. A key works by not doing it again: the service finds the key and returns the saved reply before it touches an order or a stock row, so nothing is written.

</details>

---

###### 14. Watching search get ten times faster at minute 15

A booking site runs a one-hour test. Each of 1,000 virtual users signs in once at the start and then loops on a search, with a one-second pause. Sessions on the site last 15 minutes. The script counts any final status under 400 as a success. The lead engineer looks at this summary and says the caches took a while to warm.

```text
minutes   requests/s   median   errors
0-15      880          140 ms   0
15-30     990          12 ms    0
30-45     990          12 ms    0
45-60     990          12 ms    0
```

What are the virtual users getting from minute 15 on?

<details><summary>A. Search results, from warmed caches</summary>

✗ A cold system speeds up in its first minutes, and the lead engineer has put this down to that. Warm-up is gradual, and this is one step, at the exact age of a session, of more than ten times. From then on no search runs at all, cached or not.

</details>

<details><summary>B. Errors the report leaves out of its count</summary>

✗ A zero beside a sudden change is the first suspect, and reports do leave things out. The count here is right: every final status is under 400, and that is what the script was told to call a success.

</details>

<details><summary>C. A redirect to the login page, counted as a good search</summary>

✓ Right. Every search from minute 15 on ends in a redirect to the login page and goes into the report as a good search. Three quarters of the hour measures that page, and the 12 ms is the time it takes.

Each virtual user signed in once, its session ended at minute 15, and the site answered with a 302. A client that follows redirects ends on the login page with a 200, one that does not stops at the 302, and **a status under 400 says the server answered, never which page it sent**. The rate rises from 880 to 990 because each loop got shorter.

Check the body for something only a real search returns, count a failed check as an error, and sign in again as a real client would.

</details>

<details><summary>D. Search results from a cache in the generator</summary>

✗ Browsers keep what they have fetched, and a generator's HTTP client does much of what a browser does. A cache there would start working on the first repeat of a search, in the first minute. It would have no reason to wait for minute 15, the moment the sessions end.

</details>

---

###### 15. Testing the new build right after the old one

You compare two builds of an order API on one test environment. At nine, straight after a database restart, you run the script against the old build, and the p99 is 220 ms. You deploy the new build to the app servers and at half past nine run the same script: 140 ms. The release notes list a smaller log line and two renamed variables.

What does the fall to 140 ms say about the new build?

<details><summary>A. It is about a third faster</summary>

✗ Both runs used one script and one environment, and 220 ms to 140 ms is a measured gain. The environment had changed by half past nine: the first run had filled the database's memory with what the script reads, and the second run did not pay for those reads.

</details>

<details><summary>B. It is slightly faster, and the rest of the fall is noise</summary>

✗ One run is one sample, and a smaller log line does save a little. Noise moves a p99 either way, and this fall has a cause: the second run skipped a disk read for every page the first had read.

</details>

<details><summary>C. Nothing about it: the old build run again falls as far</summary>

✓ Right. The 140 ms cannot be credited to the new build, because the first run filled the database's memory. Run the old build again now and expect about the same fall.

A database keeps the pages it has read in memory, and **a deploy to the app servers leaves the database's memory as it was**. The first run pays for a disk read on each first read, and the second finds its pages waiting. Say the requests at the p99 are the ones that read pages not yet in memory, and the disk reads add 80 ms to them: the same build run twice gives about 220 ms and then about 140 ms.

Start each run from the same state, or warm the database first and run the builds in turn, old, new, old, new, so that any order effect shows.

</details>

---

###### 16. Taking two thirds of the servers out to find the fleet's limit

The team behind a music-streaming API takes servers out of the balancer on an ordinary afternoon, under live traffic, going from 30 down to 10 while they watch the latency. The p99 holds all the way down, and the report to the head of engineering says the fleet can carry three times today's traffic. All the servers share one primary database.

What load did the database carry with 10 servers left?

<details><summary>A. The same load as today</summary>

✓ Right. The database carried the same load as today, all the way down, since the live requests did not change and neither did their queries. The report's three times covers the servers alone.

**Removing servers from under a fixed load leaves the total where it was**, and the test measures one server and stops there. Say 9,000 requests a second make 3 queries each: each server goes from 300 requests a second to 900, and the database stays at 27,000 queries a second, where three times the traffic would be 81,000.

Report the result as the capacity of one server. To find the fleet's limit, run a test that raises the real load, so that it reaches the database and whatever else the servers share.

</details>

<details><summary>B. Three times today's load</summary>

✗ Each server left was working three times as hard, and each one sends the database its queries. Each has three times the requests because the others have gone, so the total is unchanged, and so is the total of queries behind it.

</details>

<details><summary>C. A third of today's load</summary>

✗ Each server keeps its own connections and sends its own queries, and two thirds of the servers have gone. Their traffic did not go with them: the balancer moved it onto the 10 that remain. The database may see fewer connections, and it sees the same queries.

</details>

---

###### 17. Choosing the last run before a ten o'clock sale

A shoe brand opens a big sale on its site at ten o'clock sharp, with a forecast peak of 50,000 requests a second. The fleet is sized for that peak ahead of time. Your test ramps from nothing to 50,000 over 30 minutes and then holds, and it has passed with room to spare. There is time for one more run before the sale.

Which run do you use it on?

<details><summary>A. A soak at 50,000 for four hours</summary>

✗ A long hold at peak is what finds leaks and slow failures, and it is the next test on most lists. It would suit a long event, but this sale's danger is its opening minute, and four hours at 50,000 on a fleet that warmed on the way up measures a warmed fleet.

</details>

<details><summary>B. A jump from idle to 50,000 inside a minute</summary>

✓ Right. Spend it on a spike test, on a fleet left quiet. A sale at ten o'clock sharp brings its crowd at ten o'clock, and the passed test has not tried that.

**A 30-minute ramp gives caches time to fill and connection pools time to open** before the peak arrives, and a sale gives them neither. In a simulation at a hundredth of the rate, with a cache of 50,000 keys and reads on a Zipf-like curve of exponent 0.8, the store behind the cache sees a peak about eight times higher on the jump than on the ramp.

Leave caches and pools as they will be at 9:59, and watch the store and the queues through the first minute. If the jump fails, warm the caches ahead of time and run it again.

</details>

<details><summary>C. A ramp to 100,000 over 30 minutes</summary>

✗ A margin of two times is the usual answer to a forecast nobody fully trusts, and forecasts do miss. The same ramp gives 100,000 the same warm-up it gave 50,000. It can pass as well and still leave the opening minute untried.

</details>

<details><summary>D. A repeat of the ramp, to rule out a lucky pass</summary>

✗ One run is one sample, and a pass nobody can repeat is worth little. Repeating this ramp repeats what it leaves out, since the fleet warms on its way up every time. A second pass makes the same gap look safer.

</details>

---

###### 18. Stopping a 30-minute test on a store with a 24-hour TTL

A theatre-booking site keeps sessions, one per visitor, in a shared in-memory store with a 24-hour TTL. Eviction, which removes entries when memory runs short, is turned off. The site's sale weekend runs at steady traffic. The load test at that traffic starts from an empty store, makes a new session for each new visitor, and lasts 30 minutes. Latency stays flat, and at the end the store's memory is at 20 percent.

What does the store's memory do over the sale weekend?

<details><summary>A. It holds near 20 percent until the 24-hour mark and then falls</summary>

✗ Nothing happens to a session until its TTL ends, so the batch from the test sits still until then. New visitors keep coming, though, and each brings a session that stays for 24 hours, so memory rises the whole time.

</details>

<details><summary>B. It climbs for the first hour and levels off as sessions expire</summary>

✗ Sessions expire on their own, and a store with a TTL does clean up after itself. It starts cleaning a full 24 hours after the first session went in, and the climb has no reason to level off before then. Levelling off at that point would take nearly ten times this store's memory.

</details>

<details><summary>C. It holds near 20 percent</summary>

✗ Traffic is steady, latency stayed flat, and 20 percent is what the run ended on. Flat latency says a lookup does not slow down as the store fills. The store started empty and reached 20 percent in half an hour, so that figure is a point on a slope.

</details>

<details><summary>D. It fills within hours</summary>

✓ Right. Memory fills in about two and a half hours, at a fifth of the store each half hour, with nothing leaving. From then on new sessions cannot be written, and visitors can no longer sign in on the first day of the sale.

**Under steady traffic, memory settles only one TTL after the start**, when sessions begin to leave as fast as they arrive, and 24 hours of sessions at this rate would need nearly ten times this store's memory. A 30-minute run sees only the start of that climb.

Size such a store from rate times TTL, run a soak test past the longest TTL where you can, and keep watching memory after the load ends.

</details>

---

###### 19. Marking test orders with a header in a production run

An online shop runs its peak test in production at three in the morning. Each test order carries a header, and checkout skips the payment when it sees one. A finance job reads the `orders` queue and adds up each day's sales for the morning report.

```python
def checkout(request):
    order = build_order(request)
    if request.headers.get("X-Load-Test") != "1":
        payments.charge(order)
    db.save(order)
    queue.publish("orders", {"id": order.id, "total": order.total})
```

What does the morning sales report show?

<details><summary>A. Real sales alone</summary>

✗ Test orders were marked, and the code skipped the payment for them. The mark is read once, in that `if`, and goes no further: `queue.publish` sends `id` and `total`, and the job adds up whatever comes down the queue.

</details>

<details><summary>B. Real sales and test orders added up together</summary>

✓ Right. The report adds real sales and test orders together, and finance reads a night's test as sales. Say three real orders of 10 and five test orders of 10: the report shows 80, against 30 that were paid for.

**A header lives for one request**: the handler reads it once, and neither the row nor the event carries any mark, so every later reader sees a test order exactly as it sees a sale.

Put the mark in the row and in the event, and make each consumer skip marked orders, or run test orders under accounts that every report leaves out. Keep spare capacity for the test load itself and a way to stop every generator at once, and after a run check the reports before anyone reads them.

</details>

<details><summary>C. Real sales, with test orders on a line of their own</summary>

✗ Each test order has no payment against it, and finance reports exist to show that kind of mismatch. This job reads the `orders` queue and adds up `total`, and payments are nowhere in what it reads, so it has nothing to set an order against.

</details>

<details><summary>D. No sales, with the job stopped on its first unpaid order</summary>

✗ Finance code is strict about money, and it stops on what does not add up. The event has no payment field to check: an unpaid test order and a paid real one reach the job in the same shape.

</details>

---

## Further reading

- [Open Versus Closed](https://www.usenix.org/legacy/event/nsdi06/tech/full_papers/schroeder/schroeder_html/), 2006: for a given load, mean response times come out significantly lower in closed systems than in open ones.
- [Treadmill](https://doi.org/10.1109/isca.2016.47), 2016: the flaws of earlier load testers, and a way to attribute tail latency that was applied to production workloads.
- [Web Caching and Zipf-like Distributions](https://doi.org/10.1109/infcom.1999.749260), 1999: web requests from a fixed community follow a Zipf-like curve, with an exponent that varies from trace to trace.
- [Kraken](https://www.usenix.org/conference/osdi16/technical-sessions/presentation/veeraraghavan), 2016: load tests run by continually shifting live user traffic, with latency and error rate as feedback between the shifts.

Prev: [Observability](../observability/README.md) · Next: [Reliability and disaster recovery](../reliability-and-disaster-recovery/README.md)
