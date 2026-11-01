# Performance and capacity

Next: [DNS, TCP and TLS](../dns-tcp-and-tls/README.md) · Reviewed October 2026

Performance is how long a request takes and how many requests a system finishes; capacity is how much it can be asked for before those figures give way. Both come from a few parts: the workers, the queues in front of them, and the locks and counted things that requests hold.

- **Latency**: the time a request takes as its sender sees it, from sending to the answer.
- **Throughput**: requests finished per second. More workers let more requests run at once, and none of them faster: a 100 ms request takes 100 ms on 1, 4 or 16 workers, and flat out they finish 10, 40 or 160 a second.
- **Worker pool**: a fixed set of threads or processes that take requests from a queue, one request per worker at a time.
- **Utilization**: the share of its time a worker, a core or a disk is busy.
- **Saturation**: the state of a part asked for more than it can finish, so its queue grows or it refuses work.
- **Little's law**: the number of requests in flight equals the rate they arrive at times the time each one stays.
- **Mean, median and percentile**: the mean is the sum over the count, the median the middle value, and the p99 the 99th percentile, which one request in a hundred exceeds. Of 990 requests at 50 ms and 10 at 5 s, the mean is 99.5 ms and the median 50 ms.
- **Fan-out**: one request that asks many backends in parallel and waits for their replies.
- **Goodput**: answers that reach a caller still waiting for them, the part of throughput somebody uses.
- **Bottleneck**: the part that limits a system's throughput at the moment.
- **Connection tracking table**: a list of fixed size in which a firewall keeps one entry per open connection.
- **Headroom**: the capacity you keep spare for a peak or a failure.
- **Zone**: a group of machines that can fail together, on one power feed or one network.

A request's latency is working time plus waiting time, and a profile of CPU time shows the first and none of the second: time in a queue, on a lock, on a disk or on another service.

Unless a question says otherwise, a pool's workers take requests from one first-come queue with no limit on its length, a queued request waits until a worker takes it, requests arrive at random, and a caller that gets no answer does not retry.

A system runs into three kinds of limit:

| Kind of limit | What it counts | Where you meet it |
| --- | --- | --- |
| Busy time | how much of each second a part is working | workers, CPU cores, disks |
| A count | how many of a thing exist at once | threads, connection tracking table entries |
| One at a time | how long each request holds the only copy | a lock on one row |

Most of this stays out of sight until traffic, data or the fleet grows, and then the figure you trusted leaves it out.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- work out what a worker pool, its queue and a blocking call do to waits as traffic climbs, falls back or stays flat, and how many requests a crowd on the site keeps in flight
- read a latency or CPU figure on a bad evening, from a backend's p99 to a handler's timer, and say what it counts and what it leaves out
- find the limit that stands in the way when you add servers: a lock, a fixed table, a thread per peer, the next tier
- plan capacity for a lost zone, a year of data growth and next year's sale

###### 1. Pushing a worker pool from half busy to 90 percent

A tax-filing service runs each request on a pool of two workers, and a request takes about 20 ms of work on average. In March the pool is half busy. As the filing deadline nears, traffic rises until the pool is 90 percent busy.

What happens to the time a filing request takes?

<details><summary>A. It stays about what it was in March</summary>

✗ A pool at 90 percent still has a tenth of its time free, and nothing is being refused. That free time comes in short gaps between requests, and most arrivals find both workers occupied. Most of a filing's time now passes in the queue, with the work itself still about 20 ms.

</details>

<details><summary>B. It grows about as much as the traffic does</summary>

✗ Traffic grows 1.8 times, and the work in each filing has not changed at all. A queue does not grow on a straight line: each request that waits also holds up the ones behind it, so the wait climbs with how little idle time is left, and the filing takes several times as long.

</details>

<details><summary>C. It grows several times over, most of it spent waiting</summary>

✓ Right. If the work times are random, with a spread as wide as their 20 ms average, a filing near the deadline takes about 105 ms where March gave about 27. That is about four times as long on less than twice the traffic, and about four fifths of it is time in the queue.

The last stretch of load costs far more than the first, because **the wait in a queue climbs faster the less idle time is left**. More workers on one queue put off that climb: the same rise in traffic costs about 1.4 times with 16 workers.

Set a small pool's target from the latency you will accept, not from its distance to full, and add workers before the deadline rather than when the pool reaches 90 percent.

</details>

<details><summary>D. Filings start to fail with the pool nearly full</summary>

✗ Near the deadline the pool has little time left to give, and the next filing has to go somewhere. A queue with no limit turns nothing away at 90 percent, or at any load. Every filing gets its answer, later than in March.

</details>

---

###### 2. Running a sale with 5 percent more orders than the workers can take

The order workers at an online shop usually get orders at 90 percent of the rate they can finish them. A one-hour sale brings orders 5 percent faster than the workers can finish them. No customer abandons an order, and when the sale ends, arrivals fall back to the usual rate.

When do order waits return to what they usually are?

<details><summary>A. Long after the sale has ended</summary>

✓ Right. Waits come back about 30 minutes after the sale ends, and a customer who orders well after arrivals have fallen back still queues behind the sale's orders.

Saturation builds a backlog every second it lasts, and afterwards **the backlog drains only at the workers' rate minus arrivals**. The sale leaves 5 percent of an hour's work undone, 180 seconds of it at full speed. New orders still take 90 percent of the workers' rate once the sale is over, so only a tenth goes to the backlog, and those 180 seconds take half an hour.

Call the sale over from the queue's length or the wait, not from the arrival rate, and keep enough spare rate in the plan to clear a backlog as fast as you need it cleared.

</details>

<details><summary>B. Within moments of the sale ending</summary>

✗ Arrivals are under the workers' rate the moment the sale ends, and the traffic graph is back where it usually sits. Those arrivals join the back of a queue that holds 180 seconds of work, so the first order after the sale waits about 3 minutes, and the wait shrinks slowly from there.

</details>

<details><summary>C. A few minutes after the sale ends</summary>

✗ The backlog is 5 percent of an hour, 3 minutes of full-speed work. The workers never give it full speed: new orders keep coming at 90 percent of their rate, and the tenth left over needs half an hour to clear those 3 minutes.

</details>

<details><summary>D. They never do, and the queue stays as long as it got</summary>

✗ The sale's orders have piled up behind workers already near their limit on a usual day. A customer here who waits does not send the order again, so waiting adds nothing to the queue. With arrivals below the workers' rate, the queue is shorter every second, and it is empty in about half an hour.

</details>

---

###### 3. Sharing a queue between report exports and page views

An analytics app serves page views of about 10 ms each from a pool of two workers. You move the monthly report exports, about 10 s each, into the same queue, where they make up a tenth of the pool's work, and the pool sits at half busy with them.

What happens to page views after the move?

<details><summary>A. Page views take what they took before the exports</summary>

✗ Half busy is far from full, and the pool has room for both kinds of work. Utilization counts busy time and not how it is cut up: one export holds a worker for as long as a thousand page views, and while both workers are on exports, every view behind them waits.

</details>

<details><summary>B. Every page view slows by the same small amount</summary>

✗ A processor shared between tasks slows each of them a little, and a pool is one more thing its requests share. These workers run each request to its end, so the delay falls unevenly: the median view still waits nothing, and a view that arrives with both workers on exports waits seconds.

</details>

<details><summary>C. Page views go ahead, and the exports wait</summary>

✗ A 10 ms view asks almost nothing of a worker, next to an export of 10 s. This queue hands out work in order of arrival, so an export that came first takes the next free worker, and the views behind it wait.

</details>

<details><summary>D. Some page views wait seconds behind an export</summary>

✓ Right. Most page views are as quick as ever, and a small share keep the person who opened them waiting for seconds.

**On a first-come queue the wait depends on how much the work times vary as well as on load**: a long job holds one worker for its whole length, and views queue whenever exports hold both. Say views and exports arrive at random and each export takes 10 s: the median view waits nothing, about one in a hundred waits over a second, and the longest wait in six hours runs past 9 s. On views alone, the same pool at half busy keeps its waits far under a second.

Give the exports their own queue and workers, so no view ever waits on one. A queue that serves short work first helps far less if it never stops an export that has begun, since both workers can still be running exports when a view arrives.

</details>

---

###### 4. Calling a payment provider that slows down at steady traffic

A checkout service takes 200 checkouts a second, and each holds one thread from a pool of 300 for the whole of a blocking call to a payment provider. That call usually takes 100 ms. One morning the provider slows to 2 s a call, and traffic stays where it was.

What do the checkout servers run short of?

<details><summary>A. CPU, on every core</summary>

✗ Checkouts are slow and piling up on these servers, and CPU is the part that does their work. A thread blocked on the provider does no work. CPU per checkout has not changed, and only 150 of the 200 checkouts finish each second, so CPU falls to about three quarters of its usual level.

</details>

<details><summary>B. Free threads, while CPU drops</summary>

✓ Right. The pool runs out of threads and CPU falls: customers' checkouts queue for a thread on servers whose cores have less work than usual.

By Little's law, the calls in flight equal the rate times the time each one holds a thread, so **a slower call at the same traffic holds more threads at once**: 20 at 100 ms, and at 2 s it would be 400 against a pool of 300. Those 300 threads finish only 150 checkouts a second at 2 s each, and the rest pile up. A waiting thread costs no CPU, and the work on the cores drops with the checkouts that finish.

Watch threads in use alongside CPU, and give the call a timeout under 1.5 s, the longest hold that 300 threads carry at 200 a second. The timeout keeps the pool free for other work, though the checkouts that hit it still fail.

</details>

<details><summary>C. Nothing, at the traffic they always take</summary>

✗ Traffic has not moved from 200 checkouts a second, and these servers carry that every day. A thread stays taken for as long as its call lasts: the same 200 a second need 400 threads at 2 s, and with 300 the rest of the checkouts queue for one.

</details>

<details><summary>D. Bandwidth to the provider</summary>

✗ The network takes the blame for most waits on a remote call. A call waiting on its reply sends nothing, so bytes do not pile up on the wire while the provider works. With only 150 calls starting each second instead of 200, the traffic to the provider falls by a quarter.

</details>

---

###### 5. Ten thousand people on the site at once

Marketing promises ten thousand visitors at once on a booking site and asks whether the servers can take it. Each visitor reads for about 30 s between clicks, and each request takes about 100 ms.

How many requests are in flight at a typical moment?

<details><summary>A. About ten thousand, one for each visitor</summary>

✗ A visitor counts as online the whole time, and the promise says at once. The page they are reading is already in their browser, so the servers have nothing to do for them between clicks: each visitor has a request open for 100 ms out of every 30 s.

</details>

<details><summary>B. Tens of thousands, several for each visitor</summary>

✗ A click can bring a page and its parts, so one visitor can have several requests going at once. Those requests last 100 ms out of every 30 s: even ten to a click leave about 330 in flight, far short of tens of thousands.

</details>

<details><summary>C. A small share of the ten thousand</summary>

✓ Right. If each click makes one request, about 33 requests are in flight at a typical moment, a third of one percent of the visitors. A fleet sized for ten thousand requests at once is far larger than this launch needs.

These visitors spend nearly all their time reading, and by Little's law **requests in flight are the rate they arrive at times the time each one stays**. Ten thousand of them clicking every 30 s make about 330 requests a second, and each of those stays for a tenth of a second. The time per request moves this figure as well: at 1 s a request there are 333 in flight.

Turn the promise into a request rate, which needs the gap between clicks that the promise leaves out, and multiply by the time per request. Watch that time at least as closely as the crowd.

</details>

<details><summary>D. About a thousand, one visitor in every ten</summary>

✗ The promise leaves out how many visitors are active, and one in ten is a round figure to put there. Here it is a ratio of times you can read off the question: 100 ms of request in every 30 s of reading is a third of one percent. A thousand requests in flight would need each request to take 3 s.

</details>

---

###### 6. Waiting on a hundred backends for one page

The home page of a news site asks 100 backend servers in parallel and renders once all of them have answered. Each backend answers in about 10 ms, and in about a second on one call in a hundred, independently of the others.

How many page loads take a second or more?

<details><summary>A. One in a hundred, the same as each backend</summary>

✗ Every backend reports the same p99 of about a second, and each team reads that as a rare event. A page load makes 100 calls and is slow if any one is, and one in a hundred is what a page asking a single backend would see.

</details>

<details><summary>B. A few in a hundred, the rest absorbed by the parallel calls</summary>

✗ Calls go out in parallel so that their waits do not add up, and they do not. The page still renders only once all 100 have answered, so one slow call sets the time of the whole page, whatever the 99 quick ones do.

</details>

<details><summary>C. Every page load, with a hundred chances on each</summary>

✗ Adding a hundred chances of one percent gives a hundred percent. Chances of this kind do not add. Over a third of page loads draw no slow call at all and render in milliseconds.

</details>

<details><summary>D. Most page loads, but not all</summary>

✓ Right. About 63 percent of home page loads make a reader wait a second or more, though every backend is quick on nearly every call.

**A fan-out is as slow as its slowest call**, and each page load has 100 chances of drawing a slow one. The chance of escaping them all is 0.99 multiplied by itself 100 times, which leaves 0.366, and a wider fan-out leaves less. The idea and this example of 100 servers come from a 2013 paper, The tail at scale.

Set the budget on the page's own p99 and not on each backend's. Ask fewer backends, or give them a deadline and render what has arrived, which trades a slow page for one with a gap in it.

</details>

---

###### 7. Watching the median fall from 120 ms to 20 ms at the evening peak

The dashboard of a ride-hailing API shows the median over all responses, about 120 ms for most of the day. At the evening peak its servers run at full CPU with full queues, and they refuse most requests at once with a 503, the status for a server that will not take the work. The median falls to 20 ms.

What does a rider whose request is accepted get at the peak?

<details><summary>A. An answer in about 20 ms</summary>

✗ The median is the figure this dashboard leads with, and 20 ms is its best of the day. It is also how long a refusal takes. Once more than half the responses are refusals, the middle value is a refusal, and it no longer describes a request that went into the queue.

</details>

<details><summary>B. An answer slower than before the peak</summary>

✓ Right. A rider who gets through waits longer than before the peak, and the dashboard shows its best figure of the day.

**A median over all responses counts each refusal as a very fast response**, and once refusals pass half of them the median is their time. The accepted requests go to the back of a full queue, which holds each one for the places ahead of it divided by the rate the workers finish. Say 20 workers, about 120 ms of work a request, 60 queue places and 500 requests a second: about two thirds are refused, and the median of the accepted is about half a second.

Split latency by status, and show the refused share beside it. A shorter queue cuts the wait of those accepted and refuses about as many as before.

</details>

<details><summary>C. An answer in about 120 ms, as before</summary>

✗ The servers turn away most of what arrives, and 120 ms is what a request took all day. This queue is already full, so each accepted request first waits behind everything in it, and then gets the same 120 ms of work as it always did.

</details>

<details><summary>D. A 503 a little later, once the queue gives up on it</summary>

✗ A wall of 503s is what an outage shows the on-call engineer. The refusal happens at the door, before a request enters the queue. One that gets in stays until a worker takes it, and it comes back with a real answer.

</details>

---

###### 8. Reading CPU at 40 percent on a one-minute graph

A marketplace's image service resizes images, all of it CPU work, on one worker per core, and has a graph of CPU averaged per minute, steady at 40 percent. A partner's batch client makes up nearly all its traffic. Callers time out.

What is the CPU doing across each minute?

<details><summary>A. Holding near 40 percent throughout</summary>

✗ The graph is a flat line at 40 percent, minute after minute. This line is flat because each point averages a whole minute and every minute holds the same burst. Inside the minute the cores go from full to nearly nothing.

</details>

<details><summary>B. Holding near 100 percent, and the graph is wrong</summary>

✗ Callers are timing out on a service whose graph says it has room to spare. The graph's average is right here: full use for 24 s of a minute comes to 40 percent. For the rest of it the cores have almost nothing to run.

</details>

<details><summary>C. Flat out at the top of the minute, then nearly idle</summary>

✓ Right. The partner's client sends its work in a burst at the top of each minute. The cores run at full use through the first part and have almost nothing to do after it, and the callers caught in the burst are the ones timing out.

**An average over a minute cannot show seconds at full use**, and the queue forms in those seconds. If a burst keeps every core flat out for 24 s and leaves the rest of the minute idle, the average is the graph's 40 percent.

Graph CPU at 10 s or finer with the queue beside it, and ask the partner to spread its work across the minute, which leaves every core room at the same 40 percent.

</details>

---

###### 9. Timing a handler that stays at 40 ms during a surge

An API that sells tickets logs the time from a handler's first line to its last. In a surge that log stays at 40 ms, while partners calling the API report 3 s for each call. An empty status endpoint that the balancer answers itself stays at about 20 ms for the same partners.

Where do the partners' three seconds go?

<details><summary>A. Into a queue on the server, ahead of the handler</summary>

✓ Right. Nearly all of the three seconds pass in a queue on the API's server before the handler's first line runs, and both the partners' 3 s and the log's 40 ms are accurate.

The 40 ms is true of the handler alone: **a handler's timer starts when a worker takes the request**, and the time before that lies outside it, whether in the queue of new connections or in the worker pool's queue.

Measure latency from outside the server, at the balancer or the client, and log the time each request spent queued.

</details>

<details><summary>B. Into the network between the partners and the server</summary>

✗ When a server reports it was quick, the wire is the next suspect. The status endpoint crosses the same link as far as the balancer and stays at about 20 ms, so that stretch adds what it always added, and what grew sits behind the balancer, on the API's side.

</details>

<details><summary>C. Into the handler, and its log lines lag behind</summary>

✗ A surge puts the logging under load along with everything else, and a line can reach the log late. A late write does not change the figure in it: the duration is two clock readings taken inside the handler, and their difference is 40 ms whenever the line reaches the log.

</details>

<details><summary>D. Into the partners' own code, reading the replies</summary>

✗ A server team's first reply is often that the client measures itself badly. The same partner code reads the status endpoint's reply in about 20 ms, so it reads replies quickly, and the 3 s has passed before the API's own reply exists.

</details>

---

###### 10. Counting a record number of completed requests during an overload

The servers of a pricing API count completed requests, and during an overload the count reaches a record. Its callers give up after 1 s and retry once. This is one request's path at the height of it.

```text
t=0.0 s   caller sends the request, and the server's queue takes it
t=1.0 s   caller gives up and sends one retry
t=3.0 s   a worker reaches the first request and computes the price
t=3.1 s   the server writes the answer and adds 1 to its completed count
```

How many of those completed answers reach a caller still waiting?

<details><summary>A. Nearly all of the record count</summary>

✗ The completed count is what the servers report, and it has never stood higher. Completed means the server wrote an answer, and an answer to a request 3 s old has nobody at the other end, since its caller left at 1 s.

</details>

<details><summary>B. All of them, read late on the same connection</summary>

✗ TCP delivers what is sent, in order, and the answer goes out on the connection the request came in on. Delivery needs someone at the other end to read it. A caller that gives up closes that connection, so the answer written at 3.1 s has nowhere to go.

</details>

<details><summary>C. Only a handful of them</summary>

✓ Right. Only a handful of the answers reach a waiting caller, and the pricing API's callers get failures from servers running flat out.

Throughput counts the answers a server finishes and goodput only those a caller still wants, and **a queue that holds each request longer than its caller waits leaves no goodput**. Here the queue holds a request for 3 s against a caller's 1 s. The only useful answers go to the first requests of the overload, admitted before the wait passed 1 s, and the retries add load and no answers.

Keep the queue shorter than your callers' patience, or send the caller's deadline with the request and drop what has expired. Count answers returned inside the deadline.

</details>

<details><summary>D. About half, as each retry gets through the second time</summary>

✗ Each caller gets a second try, and the retry goes out as a fresh request. This retry joins the back of the same 3 s queue, and its caller gives up on it at 1 s too. It adds one more request that the server finishes for nobody.

</details>

---

###### 11. Doubling the servers when every order updates one row

During a flash sale each order holds the lock on one stock row for about 5 ms. Four application servers at 30 percent CPU top out near 200 orders a second, with more orders waiting behind them, so you go to eight servers.

How many orders a second can the site finish with eight servers?

<details><summary>A. About twice as many, on twice the servers</summary>

✗ Four servers are topping out, and eight have twice the room for the orders waiting behind them. Extra servers add room only for the part of an order that a server does on its own, and four already had CPU to spare for that. Every order on all eight servers still waits its turn for one stock row.

</details>

<details><summary>B. Close to the 200 orders of today</summary>

✓ Right. The site still finishes about 200 orders a second: the sale's customers queue as long as before, and eight servers run at half the CPU that four did.

Orders pass through the lock one at a time, and at 5 ms each only 200 can pass in a second, however many servers stand in line. Amdahl's law says the same in general: **the share of work done one at a time sets a ceiling that added machines cannot lift**. The 30 percent CPU was the sign that four servers with room to spare were already held back by something else.

Shorten the hold, since at 2.5 ms the ceiling is 400 with no new servers, or split the stock over several rows so orders stop sharing one lock.

</details>

<details><summary>C. About one and a half times as many, after overhead</summary>

✗ Scaling usually loses something to overhead, so one and a half times is the usual cautious estimate. The limit on this site is the lock, which allows one order per 5 ms whether four servers or eight are waiting for it.

</details>

<details><summary>D. None, as the servers deadlock on the row</summary>

✗ Lock trouble on a busy row often gets called a deadlock. A deadlock needs a cycle, two holders each waiting on the other. These orders line up for one lock and each releases it 5 ms later, so 200 a second keep finishing while the line behind them grows.

</details>

---

###### 12. Filling the connection tracking table during a tournament final

A sports-streaming API sits behind a firewall that keeps each open connection in a connection tracking table and refuses new connections once the table is full. Viewers hold one connection for a whole match. The firewall's CPU stays low, the API's servers have room, and the capacity plan counts on an alert on p99 latency for its warning. During the final the table gets down to its last few thousand free entries.

What does the p99 do as the last free entries go?

<details><summary>A. It climbs step by step as the table nears full</summary>

✗ Every capacity curve you have seen bends upward near the limit, the one for CPU and queues above all. That curve comes from waiting, and a table has nothing to wait in: an entry is free or it is not. A viewer connecting at the last free entry gets it as quickly as one at half full.

</details>

<details><summary>B. It jumps to the timeout as the firewall drops packets at random</summary>

✗ This firewall carries every viewer's packets, and its table is nearly full. The table holds connections and no packets, so nothing in it drops at random. This firewall refuses a new connection only once the table is full, and until then it refuses none and passes the packets on open connections as before.

</details>

<details><summary>C. It stays where it was all evening</summary>

✓ Right. The alert the plan counts on stays quiet right up to the moment viewers cannot connect, since the p99 does not move before the first refusal.

The connection tracking table is a count, and **a fixed count that refuses when it is full gives a latency graph no warning**. While one entry is free no viewer queues, and a packet's lookup in the table costs about what it did at half full, far below any p99. The viewer after that is refused outright. Once the table is full the refusals happen in the firewall, so the API's latency figure misses the failure even after it starts. Some firewalls push out an idle entry before they refuse a new one, which this one does not.

Alert on the table's own occupancy, early enough to act, and give any other limit of this kind, such as threads or a provider's quota, an alert on its own count.

</details>

---

###### 13. Adding servers to a fleet that keeps a thread per peer

Each of a data-ingest fleet's 1,000 servers keeps one thread for each other server, and about 100 more for its work, against a limit of 1,100 threads on a host. CPU and memory are fine. Ahead of a holiday peak, you add a fifth more servers.

Which servers reach the thread limit after the addition?

<details><summary>A. None, as the new servers take load off the rest</summary>

✗ More servers spread the request load, and that is why you are adding them. Peer threads do not follow the request load at all. Each host now keeps one for each of 1,199 others, and whatever its other threads shed as the work spreads comes nowhere near that growth, so the count passes 1,100.

</details>

<details><summary>B. Only the new ones, while they warm up</summary>

✗ New machines are the risky ones on a deploy day, cold and unproven. A thread count takes no notice of age. An old host needs a thread for each of the 200 new peers as well, and it began the day one thread short of its limit.

</details>

<details><summary>C. The busiest ones, holding the most requests</summary>

✗ The hottest server reaches a limit first wherever the limit follows traffic. This count follows the fleet: the quietest host and the busiest each hold 1,199 peer threads, and the requests they carry change only the small part left over.

</details>

<details><summary>D. Old and new alike</summary>

✓ Right. The addition meant to carry the holiday peak puts all 1,200 servers over the limit at once, old and new.

Adding servers raises the count on every old host too, because **a thread per peer ties each host's thread count to the size of the fleet, whatever its traffic**. A fleet whose peer threads sat well under half the limit, with the other threads shrinking as the load spreads, would see the count fall. These hosts hold nearly all their threads for peers, and one at 1,099 goes to about 1,282 even as its other threads shrink.

Before you scale out, count what grows with the fleet on each host, then raise that limit or cap how many peers each server keeps. A higher limit only buys room until the fleet grows again.

</details>

---

###### 14. Doubling the web tier in front of a busy database

At peak a media site's web tier runs at full CPU, and a third of its visitors time out in its queue and leave. The database behind it runs at 70 percent. Every request the web tier finishes makes the same queries, and you double the web tier.

What does the database see after the change?

<details><summary>A. More queries than it can answer</summary>

✓ Right. The database is asked for about 105 percent of what it can answer, and visitors who used to time out at the web tier now wait on a database queue that keeps growing.

The web tier was holding demand back: **a saturated tier caps what reaches the tiers behind it**, and removing that bottleneck sends them the rest. The database's 70 percent came from the two thirds of visitors who got through, and now all of them get through, half as many again.

Before you remove a bottleneck, add up the demand at the door and compare it with each tier behind, and scale the database first where it falls short.

</details>

<details><summary>B. The same query rate as before</summary>

✗ Database load is usually estimated from visitors, and the visitor count has not changed. The database never saw all of them. The third who timed out in the web queue sent it no queries, and now they do, so the query rate rises by half.

</details>

<details><summary>C. Fewer queries, from a web tier that wastes less work</summary>

✗ The web tier is at full CPU with visitors timing out, and that is work thrown away. A visitor who timed out in the web queue left before any query went out, so the database carried none of that waste. Serving those visitors adds queries.

</details>

<details><summary>D. A short jump as the queue drains, then the old rate</summary>

✗ A queue normally clears once and is gone, and the web tier's queue does shrink after the change. The visitors in it were never waiting for the database, since the ones who timed out left. What rises is the steady rate of queries, in every second of the peak.

</details>

---

###### 15. Losing one of three zones at 70 percent

A bank's API runs in three zones of equal size, each at 70 percent of what it can serve in business hours. Its customers reach it from outside the zones. One morning a power fault takes one zone down, and within seconds the balancer's health checks take it out of rotation.

What happens to the two zones that remain?

<details><summary>A. They absorb the traffic inside their 30 percent of room</summary>

✗ Each zone has 30 percent spare on every normal day. Each remaining zone now takes half of the lost zone's 70 on top of its own, and that half is more than its 30 percent of room.

</details>

<details><summary>B. They are asked for more than they can serve</summary>

✓ Right. The bank's customers wait longer every minute of the fault, though no request is refused, because each remaining zone is asked for about 105 percent of what it can serve.

The two remaining zones carry the lost zone's share on top of their own, 70 percent times three over two, which is 105 percent: **headroom has to be measured after the loss**. A zone asked for more than it can finish queues the surplus, so its waits grow every minute.

Set the target from the number of zones: under two thirds with three, under 75 percent with four. Then take a zone out on purpose at a busy hour and watch the waits.

</details>

<details><summary>C. They stay at 70 percent, as the lost zone's traffic goes with it</summary>

✗ A zone that holds its own users takes them with it when it fails. These customers are outside the zones, and their traffic does not stop with the power. Once the health checks drop the lost zone, all of it goes to the two zones still up.

</details>

<details><summary>D. A third of requests fail, and the rest run as usual</summary>

✗ A third of the capacity is gone, and a third of the requests were headed for it. The balancer stops sending to the dead zone within seconds and the zones still up queue what they cannot serve, so nothing fails at once. Requests keep being accepted, and each waits longer than the one before it, so none runs as usual.

</details>

---

###### 16. Testing a service at 2,000 requests a second, then running it at 1,500

The feed of a social app passed a load test a year ago, and traffic has held below the tested rate ever since. This morning's release added only one log line, buffered in memory, to the feed handler. This afternoon the feed's latency alert fires, and the on-call engineer puts the test report beside today's dashboard, then rolls the app servers back to last week's release.

```text
                             Load test, a year ago    Today
Requests a second            2,000                    1,500
App servers                  12                       12
Database memory              120 GB                   120 GB
Feed table                   40 GB                    180 GB
Database connections busy    4 of 8                   7 of 8
```

What does the rollback do to feed latency?

<details><summary>A. Feed latency falls back to where it was before the alert</summary>

✗ A release went out this morning, and the alert fired a few hours after it. One log line costs far less than a read from disk, and the extra time sits in the database's disk reads, which go on under last week's code as they do under this week's.

</details>

<details><summary>B. It falls for a few minutes after the restart, then climbs back</summary>

✗ On an overloaded service, a restart buys minutes by throwing away the queue, and the slowdown returns as the queue builds again. With 7 of 8 connections busy the database still keeps up, so it holds no backlog for a restart of the app servers to clear, and the feed shows no dip to climb back from.

</details>

<details><summary>C. The rollback changes nothing, and latency clears on its own within the hour</summary>

✗ The fleet passed 2,000 in a load test, and 1,500 sits well under it. That test ran with the whole table in memory, against a database that could serve far more than it can today. The latency comes from the reads that go to disk, and the table will be no smaller in an hour.

</details>

<details><summary>D. Feed latency stays where it is now</summary>

✓ Right. The feed stays as slow as it was: readers keep waiting, and the on-call engineer's time goes to the wrong tier.

The feed's 1,500 a second cost more now that the table has outgrown memory, because **what a request rate costs depends on the work behind each request**. At the load test all 40 GB sat in the database's 120 GB of memory. At 180 GB a read that misses memory goes to disk, whichever release asks for it: if reads fall evenly across the table, that is about one read in three, and it is how 7 of the 8 connections come to be busy at a lower rate than the test's. Last week's release sends the same reads to the same table, so the rollback changes none of that time.

Check what changed besides the code before you roll back, and rerun the load test against today's data. More memory or a smaller table puts the reads back in memory.

</details>

---

###### 17. Sizing next year's sale from last year's request log

A shop's request log, written by its servers, shows last year's sale as a flat top of 5,000 requests a second for 20 minutes. The balancer in front of those servers refused connections during those minutes. You plan next year's fleet from that peak.

What does the 5,000 say about last year's demand?

<details><summary>A. Only that it reached at least that much</summary>

✓ Right. The 5,000 is a floor and no more, and a fleet sized to it may refuse next year's buyers at the same place it refused last year's.

**A request refused at the door never reaches the servers that write the log**, so a flat top marks the limit and says nothing of how far demand went above it. Demand that peaked at 5,500, at 8,000 or at 60,000 writes the same log.

Find the demand at the door: count refused connections at the balancer for the next sale, and read how fast traffic was climbing before it hit the top. A refusal counter counts each retry again, so it overstates visitors, and it still gives you a ceiling to set beside this floor.

</details>

<details><summary>B. It counts all the demand, refused visitors included on their retries</summary>

✗ A visitor refused at the door can try again, and each retry is a real connection. A retry refused at the same door leaves no line either, and a visitor who gives up never appears. If demand peaked at 8,000, with 1,000 a second before and after the sale, and refused visitors retried for 60 s, about 3.3 million would leave without a trace in the log.

</details>

<details><summary>C. It is the demand, seen exactly</summary>

✗ A log is the record of what happened, and each line here is a real request. Its writers are the servers behind the balancer, and they cannot record what never reached them, so any demand above 5,000 gives this same flat top.

</details>

<details><summary>D. It is an average, and the real peak ran about a fifth above</summary>

✗ Twenty flat minutes hide whatever rose and fell inside them, and a fifth above this top would be 6,000. That is one possible case and no more: the log has no way to tell 6,000 from any other figure above the top, so no fixed share recovers the real one.

</details>

---

## Further reading

- [A Proof for the Queuing Formula](https://doi.org/10.1287/opre.9.3.383), 1961: the proof of the queueing formula L = λW, which the page calls Little's law.
- [The single server queue in heavy traffic](https://doi.org/10.1017/S0305004100036094), 1961: waiting in a queue with one server as it comes close to full.
- [The tail at scale](https://doi.org/10.1145/2408776.2408794), 2013: why a request that fans out to many servers is ruled by their slowest replies.
- [Validity of the single processor approach to achieving large scale computing capabilities](https://doi.org/10.1145/1465482.1465560), 1967: the paper behind Amdahl's law, on the single processor approach to large scale computing.
- [Eliminating receive livelock in an interrupt-driven kernel](https://doi.org/10.1145/263326.263335), 1997: receive livelock in an interrupt-driven kernel, and how to eliminate it.
- [Metastable Failures in the Wild](https://www.usenix.org/conference/osdi22/presentation/huang-lexiang), 2022: 22 metastable failures from 11 organizations, read from their public incident reports.

Next: [DNS, TCP and TLS](../dns-tcp-and-tls/README.md)
