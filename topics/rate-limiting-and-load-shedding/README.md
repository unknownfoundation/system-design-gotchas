# Rate limiting and load shedding

Prev: [Load balancers and proxies](../load-balancers-and-proxies/README.md) · Next: [Timeouts, retries and circuit breakers](../timeouts-retries-and-circuit-breakers/README.md) · Reviewed October 2026

Rate limiting and load shedding both refuse work so that the rest gets done. A rate limit counts what one client sends and refuses past a set amount, whatever state the servers are in; load shedding looks at how a server is doing and refuses what it cannot serve, whoever sent it.

- **Rate limit**: a cap on what one client may send in a stretch of time, enforced by a limiter in a gateway or in the service.
- **Quota**: the amount a rate limit allows one client, such as 100 calls a minute.
- **Key**: what the limiter counts against: an account, an API key, an address. Everything that shares a key shares a quota.
- **Shared store**: a store outside the gateways that holds the counters, so all of them read and write one count. The other place for a counter is each gateway's own memory.
- **Failing open**: letting a call go ahead when the limiter cannot finish its check. Failing closed refuses the call instead.
- **Shedding**: a server refusing work because of its own state, such as CPU or calls in flight, in its plain form without regard to the path or the client.
- **Concurrency limit**: a cap on how many calls may run at once. A rate limit counts calls as they arrive; this one counts calls still running.
- **429 and 503**: a 429 tells a sender it has sent too many requests in a stretch of time, and the specification leaves the counting to the server. A 503 says the server cannot handle the request now, through overload or maintenance. So a 429 is about this client, and a 503 about the server.
- **Retry-After**: a response header giving how long to wait before the next request, as a number of seconds or a date. The server picks the value.
- **Backpressure**: a slow part making its senders slow down, as a full buffer does when it refuses writes or makes them wait.

A limiter can count in four ways:

| Way | What it keeps | When it refuses |
| --- | --- | --- |
| Fixed window | one counter for each clock minute or other fixed stretch | when the current stretch's count reaches the quota |
| Token bucket | tokens that refill at a steady rate, up to the bucket's size | when a call finds no token to take |
| Leaky bucket | a line of calls of fixed length, let out at a steady rate | when a call finds the line full |
| Sliding window | a log of the times of the calls in the past minute or other stretch, or the count of this clock minute and of the last one | with the log, when it already holds the quota; with the counts, when this count, plus the part of the last count that the past minute still covers, reaches the quota, which approximates the log |

If a log also records the calls it refuses, a client that retries faster than the limit stays refused for as long as it retries.

A shared count can still overshoot. Ten clients that read a counter at 95 together all pass a check against 100, and the counter ends at 96, unless the increment and the check are one step. Nodes that exchange their counts now and then let through more as the interval grows, a limit counted per region lets a customer who sends 100 to each of three regions get 300 through a limit of 100, and a store that evicts a counter hands its customer a fresh quota.

Unless a question says otherwise, a limiter answers a refused call with a 429 at once, and a client that gets one does not try again.

The trouble comes from what surrounds a limiter: the next gateway, the limiter's own store, what a client does with a refusal, and the work a server has done before it refuses.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- work out what a per-customer quota, a clock-minute counter, a token bucket or a sliding window lets through when a campaign, a batch or a morning sync arrives
- run one limit across many gateways and a shared store, and say what a customer gets when a server dies, the fleet grows, the store hangs or a CDN moves in front
- choose what a limit counts, and follow what a refusal tells a client and costs a server
- keep a service answering when too much arrives, from its health probes and queue length to the order it serves in, the calls it refuses, the parts of a page it leaves out and the signal its scaler reads

###### 1. Running a campaign evening when no customer passes its quota

A messaging API gives each of its 200 customers a quota of 1,000 calls a second, enforced by a limiter in the gateway, and nothing else stands in front of the fleet. The fleet handles 50,000 calls a second, and calls it cannot serve at once wait in its queue. On a holiday evening, 120 customers run campaigns at 600 calls a second each.

What do those 120 customers get from the service that evening?

<details><summary>A. No 429 at all, and calls that time out</summary>

✓ Right. Not one of the 120 gets a 429, since each sends 600 a second against a quota of 1,000. Their calls slow down in the fleet's queue and then time out.

**A per-customer quota counts each customer alone**, and the fleet's load never enters that count. The campaigns together send 72,000 calls a second to a fleet that does 50,000, even with the other 80 customers silent, and the queue grows by 22,000 calls each second. For a client that gives up after two seconds, every new call is timing out within about five seconds of the start.

Put load shedding on the fleet, since it reads the fleet's own state: with a bounded queue that answers 503 at once, about three calls in ten get the 503 and the other seven are answered in time. Shedding only chooses who is refused, so also set the quotas against what the fleet can do.

</details>

<details><summary>B. A 429 for each customer's fair share of the excess</summary>

✗ The limiter sits in front of the fleet on every diagram, and sharing out something scarce is a job you would want done there. This limiter holds one number per customer, 1,000, and compares it with that customer's 600. It has never heard of the 50,000, so it finds no excess to share, and the calls go on to the queue.

</details>

<details><summary>C. A 429 for the heaviest customers, and a healthy fleet</summary>

✗ A limit is usually added to stop a few heavy senders, and it stops them well. The heaviest sender this evening is at 600 against 1,000, so the limiter has nobody to catch, and a fleet asked for about 1.4 times what it can do is far from healthy.

</details>

<details><summary>D. Lower quotas for everyone, set by the limiter</summary>

✗ Some limiters lower their limits as a service slows, which takes a signal from the fleet, and the gateway reads none. Its 1,000 stays as written all evening while the queue behind it grows.

</details>

---

###### 2. Setting a limit of 100 a minute on a clock-minute counter

You count each partner's calls to an API in one counter per clock minute, with a limit of 100 a minute. A partner's batch job starts at 11:59:59 with the first calls it has sent this hour, and it sends as fast as the servers answer, about 1,000 calls a second.

What gets through in the two seconds around noon?

<details><summary>A. One quota at most, over those two seconds and the next minute</summary>

✗ The limit is written as 100 a minute, and a reader of the API document takes that to cover any minute. A counter of clock minutes knows only those, so at noon the count for 12:00 is new and at zero, and a second run of 100 calls goes through at once.

</details>

<details><summary>B. A quota spread evenly, a call or two each second</summary>

✗ A pacing client or a leaky bucket does turn 100 a minute into an even flow. A fixed window has no notion of pace: it accepts every call until the count reaches 100, and the first 100 calls of this batch arrive inside one second.

</details>

<details><summary>C. 100 calls before noon and 100 more right after it</summary>

✓ Right. The partner gets 100 calls through in the last second of 11:59 and 100 in the first second of noon: 200 calls in two seconds under a limit written as 100 a minute.

A fixed window keeps one number for each clock minute, and **each new window's number starts at zero, whatever came a moment before**. Any sender that reaches the limit on both sides of the edge gets double, and this batch reaches it within a second on each side.

If the double matters, count with a sliding window log, which holds those two seconds to 100, or a sliding window counter, an approximation that holds them near 100 for less memory.

</details>

<details><summary>D. 100 calls before noon, and a partner blocked for going over</summary>

✗ Going over a limit gets an account locked on plenty of login pages. A counter keeps no record of who went over: the refusals of 11:59 leave nothing behind at noon, and the partner starts 12:00 with a clean count.

</details>

---

###### 3. A quiet night before the morning sync

A fleet of delivery vans syncs with the depot's API at seven each morning. All the vans share one token bucket that refills at 100 tokens a second and holds up to 10,000, and the backend behind it handles about 150 calls a second. Overnight the vans sit parked and send nothing. At seven they all wake up.

What does the backend receive in the first second of the sync?

<details><summary>A. 100 calls, the refill rate, with the rest refused</summary>

✗ The refill rate is the number an API document quotes as the limit, and 100 a second fits under this backend's 150. The refill only says how fast tokens come back. After a night of no calls the bucket holds its full 10,000, and those go out before the refill matters.

</details>

<details><summary>B. Up to about ten thousand calls in that one second</summary>

✓ Right. The bucket lets through up to about ten thousand calls in one second, to a backend that handles about 150.

A token bucket that nobody draws from keeps filling until it is full, and every token it holds can go out at once: **the bucket's size sets the burst, and the refill rate only sets what follows it**. A hundred seconds of quiet fill this one, and a night is far longer.

Set the size to the burst the backend can take. A bucket that small refuses most of the rush at seven, so start the vans over several minutes as well.

</details>

<details><summary>C. 100 calls, with the rest held in the bucket for later seconds</summary>

✗ A leaky bucket does hold calls and let them out at a steady pace, and its name is nearly the same. A token bucket holds tokens, never calls. A call that finds no token gets a 429 on the spot, and with 10,000 saved tokens the first second is far from 100.

</details>

<details><summary>D. Nothing from most vans, with the bucket shut by the rush</summary>

✗ A fuse or a circuit breaker opens when too much flows through it. A bucket has no such state: it hands out tokens while it has them and refuses when it is empty. The first calls go straight through to the backend, and only the vans that come after them get a 429.

</details>

---

###### 4. Killing a server between two counter calls

At a mapping company, the limiter gives each customer 100 calls a minute. `store` is your thin client for a shared store: `incr` adds one to a counter, creating it if it is missing, and `expire` makes a counter vanish after a set number of seconds, its TTL. During a rollout, the orchestrator kills a server right after `store.incr` returns, on a customer's first call of a minute. The customer keeps calling as usual, within its limit.

```python
def allow(customer):
    key = f"limit:{customer}"
    count = store.incr(key)
    if count == 1:
        store.expire(key, 60)
    return count <= 100
```

What happens to that customer's calls from then on?

<details><summary>A. A fresh count at the next minute, as always</summary>

✗ Every other customer's counter empties each minute, and so did this one until today. In this code the minute exists only as the TTL on the key, and this key never got one. It does not start again at the next minute or any other, and the count climbs.

</details>

<details><summary>B. A TTL set by the next call, and normal limits</summary>

✗ The next call runs the same code, and `store.expire` is right there in it. That line sits behind `if count == 1`, and the next call's count is 2. Only the call that creates the key can set its TTL, and that call died with the server.

</details>

<details><summary>C. Normal replies at first, then a 429 on every call</summary>

✓ Right. The calls pass while the count climbs to 100, and from call 101 on every call gets a 429, minute after minute. A customer inside its limit is locked out of the maps.

The increment and the TTL are two calls, and **a kill between two calls keeps the first and loses the second**: the key stands at 1 with no TTL. The next call's count is 2, so the `if count == 1` line that sets the TTL never runs again, and the key stays until the store evicts it or restarts empty. If the customer sends five calls a minute, the first refusal comes after 20 minutes.

Make the two calls one step, through a script or a combined command the store runs whole, if yours offers one. A kill at the same point then leaves no key without a TTL.

</details>

<details><summary>D. No limit at all, from a counter left broken</summary>

✗ A limiter that fails open lets calls through when its store fails, and a damaged key is a kind of store failure. The key still counts, and `count <= 100` still compares, so it is not damaged. The fault runs the other way, against a customer who never went over.

</details>

---

###### 5. Going from three gateways to ten at a launch

You run a ticketing API on three gateways behind a balancer that rotates calls among them. Each gateway counts in its own memory, with a limit of 100 calls a minute per customer. For a launch you go to ten gateways and change nothing else, with no shared store and no customer tied to one gateway. A reseller's script sends far more than the limit.

What happens to the most a customer can get through in a minute?

<details><summary>A. It grows with every gateway you add</summary>

✓ Right. The launch loosens the limit: the most a customer can get through goes from 300 calls a minute on three gateways to 1,000 on ten.

Each gateway counts only the calls it sees, and **counters in each gateway's own memory never add up**. The real limit is 100 times the number of gateways, for a customer who sends enough to fill every count.

Keep the counters in a shared store, which gives 100 with any number of gateways at the price of a call on each check. Tying each customer to one gateway gives 100 too, and that count goes when the gateway does.

</details>

<details><summary>B. It stays at 100 a minute</summary>

✗ The configuration file says 100 and nobody touched it. That setting belongs to each counter, and there are now ten counters that never talk to each other, so the reseller's calls pass 100 long before any one gateway has seen 100 of them.

</details>

<details><summary>C. It shrinks, with each gateway taking a share of the 100</summary>

✗ A balancer divides the work among instances, and a careful limiter can divide a quota the same way. Here calls are divided among the gateways and the quota is not: each one still holds the whole 100 from its configuration.

</details>

<details><summary>D. It disappears, with no gateway refusing anyone</summary>

✗ Moving a part that keeps state onto more machines is a usual way to break it. These counters need no coordination to work, and the reseller still collects plenty of 429s: one for every call past the 100th that a gateway sees.

</details>

---

###### 6. A limiter store that stops answering

On an online shop's API, gateways with 200 workers in all carry 100 calls a second. Before each call a gateway asks a shared store for the customer's count, with a one-second timeout, and a failed check lets the call go ahead. The rest of the handling takes 50 ms. At midday the store hangs: it accepts connections and never answers.

What do customers get while the store hangs?

<details><summary>A. Replies as quick as before the hang, and no limits</summary>

✗ A store that is down and refuses connections fails the check at once, and then a call goes ahead with almost no delay. A hung store accepts the connection and says nothing, so the only failure a gateway can see is the timeout, a full second into each call.

</details>

<details><summary>B. Fast replies, after the first timeout teaches the limiter to skip the store</summary>

✗ A client with a circuit breaker does stop calling a part that keeps failing. The shop's gateways have none: they ask the store before every call and keep nothing from the last one, so the next call goes to the hung store like the one before it.

</details>

<details><summary>C. Normal limits, counted in each gateway's memory</summary>

✗ A limiter built with care keeps a local count as a fallback for this moment. This gateway's failed check lets the call go ahead and does nothing else: a customer far above its limit is refused nothing.

</details>

<details><summary>D. Every reply a second slower, and no limits</summary>

✓ Right. Each reply takes about 1.05 seconds where it took about 0.05, and no customer is refused, however far over its limit.

A gateway fails open only after the check has failed, and **a store that hangs fails the check only when the timeout ends**. Every call waits out that second, and its worker stays held for it plus the 50 ms. The 100 calls a second now keep about 105 workers busy where about five were enough, and the 200 workers have room for that, which is why the cost stops at one second.

Give the check a far shorter timeout, and a breaker that stops asking once the store stops answering, so calls skip the wait. Test it against a store that hangs, since a store that refuses connections fails fast and hides the wait. A count in each gateway's memory can hold a rough limit in the meantime.

</details>

---

###### 7. Moving a per-address login limit behind a CDN

A forum allows 10 calls a minute to its login endpoint for each source address, to slow password guessing. Traffic grows, and the forum moves behind a CDN: DNS now sends each visitor to an edge location, a group of the CDN's servers near them, and you let each location's few addresses through the forum's firewall. Nobody changes the limit, and a test login from your laptop works that afternoon.

Who hits the login limit on a busy evening?

<details><summary>A. Only people on the same edge location as someone mistyping a password</summary>

✗ This limit exists to catch bad passwords, and before the move a member with the right one never came near it. The count takes every call to the login endpoint, right password or wrong, and a few addresses now carry all the logins at that location, so on a busy evening first tries alone fill each count within the minute, with nobody mistyping at all.

</details>

<details><summary>B. Anyone logging in through a busy edge location, first try or not</summary>

✓ Right. Everyone logging in through a busy edge location shares a few counts, and a member who types the right password on the first try gets a 429 because of strangers.

An edge location ends the visitor's connection and opens its own to the forum, and **a limit keyed on the source address counts whoever opened the connection**. The key is now one of the edge's few addresses. If the edge connects from 4 addresses, at most 40 calls a minute get through, however many people are logging in. The afternoon's test passed because one call on a quiet edge location leaves every count far below 10.

Key the limit on the visitor's address, which a CDN passes to the origin in a request header. Read it only from the CDN's own entry, on connections that came from the CDN, since any sender can write that header too.

</details>

<details><summary>C. Nobody, not even someone mistyping a password over and over</summary>

✗ Behind a proxy one person's tries no longer pile up under an address of their own, and a limit that no longer sees who is calling is the usual worry. The count that person lands on is the crowd's as well: on a busy evening each of the location's addresses passes 10 within the minute, so the next wrong password gets a 429 like any other call.

</details>

<details><summary>D. The same people as before the move, mistypers and whole offices</summary>

✗ The CDN writes each visitor's address into a request header, the same address the limit counted before the move. A limit keyed on the source address never opens that header: it counts the connection, which now belongs to the edge, so someone at home who gets the password right is refused too.

</details>

---

###### 8. Putting exports and lookups under one calls-a-minute limit

You sell a records API at 600 calls a minute for each customer, and one pool of 20 workers serves all calls in order of arrival. A lookup takes 5 ms and an export takes 40 seconds. One customer starts an export every second, 60 a minute, while the other customers keep sending lookups within their limits.

What happens to the other customers' lookups while those exports run?

<details><summary>A. Lookups answered as quickly as ever</summary>

✗ The limiter reports no breach, and it is the part you put there to stop one customer hurting the others. It compares 60 with 600 and has nothing to say about the 40 seconds each of those calls holds a worker. The pool fills, and the next lookup finds no worker free.

</details>

<details><summary>B. Lookups slipping ahead of the exports in line</summary>

✗ Short jobs slip through on a pool with spare workers, so a 5 ms lookup seldom notices an export. The exports ask for twice the workers the pool has, so none are spare and a lookup joins the same line as them.

</details>

<details><summary>C. Long waits behind the exports</summary>

✓ Right. A lookup with 5 ms of work waits minutes for a worker, and every customer stays inside its limit the whole time.

An export is one call on the limiter's count like any other, because **a rate limit counts calls and never how long they run**. Exports at one a second would hold 40 workers, twice the pool, so the line behind it only grows, and a lookup waits about a second longer for every second the exports run.

Add a concurrency limit per customer, so that one customer's exports take their turn and fewer of them start. With at most 5 exports running, lookups stop waiting.

</details>

<details><summary>D. A 429 from the limiter</summary>

✗ A limit is there to protect the service, and this service is in trouble. The limiter counts each customer's calls per minute, every one of them is inside 600, and it refuses no lookup, however long that lookup ends up waiting.

</details>

---

###### 9. Telling refused clients how long until the window resets

During a sale, a shop's partner API takes orders from many partners' clients under one shared limit per clock minute. Each 429 carries a Retry-After with the seconds left in that minute, and the clients honour it.

When do the refused clients come back?

<details><summary>A. Spread over the next minute, as they were refused</summary>

✗ The refusals were spread over a minute, and a delay that is the same for everyone keeps the spacing. These delays differ: a client refused early is told a long wait and one refused late a short wait, and each wait ends at the same moment.

</details>

<details><summary>B. Together, in the first second of the next minute</summary>

✓ Right. Every client refused during a minute comes back when that minute ends, all of them in the first second of the next minute, and the sale's traffic turns into a spike on the minute.

A Retry-After is a value the server picks, and here that value is the time to the window's reset, so **a wait counted to a shared moment sends everyone back at that moment**. Take 1,000 clients that each call once at a random second of minute 0, with a limit of 100 a minute: the busiest second of that minute holds about 30 calls, and second 60 holds 900.

Add a random spread to the value. With the seconds left plus a random 0 to 60, second 60 holds about 7 calls and the busiest later second about two dozen. The limit still lets in the same 100 a minute, so the last client gets through no sooner.

</details>

<details><summary>C. One by one, released from a line kept by the limiter</summary>

✗ A sale site's waiting room does hold customers in a line and call them forward. A 429 ends the call: the limiter holds nothing and calls nobody back, and each client returns on its own timer.

</details>

<details><summary>D. Each one a full minute after its own refusal</summary>

✗ One minute is the number in the limit, and a server can send a fixed wait if it chooses. The Retry-After on these replies comes from the window's reset and is whatever is left of the current minute: a few seconds for a client refused late in it.

</details>

---

###### 10. Refusing calls after parsing and logging them

On the night of a final, a sports API gets 20 times what its fleet can serve, nearly all of it from clients over their quota. The limiter runs on the fleet's own servers, after authentication and body parsing, and it writes a log line for each call it refuses. Say those three steps come to a fifth of a served call's work.

How does the night go for customers inside their quota at the peak?

<details><summary>A. Answers as quick as on a quiet night</summary>

✗ What the limiter admits is well inside capacity, and its diagram has one small arrow going back to the client. That arrow costs CPU: each refusal has already gone through authentication, parsing and a log line, so the served calls stand in line behind all that work.

</details>

<details><summary>B. A 429, the same as everybody over quota</summary>

✗ A flood often ends in an outage page for everybody, and one 429 for all would be that page. These customers are under quota, so the limiter says yes to each of them. Their calls go on to a worker, in a fleet with no CPU left.

</details>

<details><summary>C. Prompt answers from backends the balancer has relieved</summary>

✗ A balancer moves traffic away from a backend that is merely busy. Every backend here carries its share of the refusals, so none has room to give, and a call moved along goes from one full backend to another.

</details>

<details><summary>D. Timeouts on the calls the limiter let through</summary>

✓ Right. The limiter admits their calls, and those calls wait behind the refusals until they time out. Staying inside the quota did not help, since the CPU went to the refusals.

**A refusal costs whatever work ran before it**, and that cost grows with the flood. With 20 times the fleet's capacity arriving and each refusal at a fifth of a served call, the refusals alone need about four fleets of CPU.

Check the limit before the expensive work, ahead of body parsing, and sample the log line. At a fiftieth of a served call the same flood needs less than the fleet, and every call inside quota is answered in time. A larger flood can still reach the fleet's limit, so the edge in front has to refuse most of it.

</details>

---

###### 11. Shedding health probes along with everything else

The backends of a streaming service refuse any request past a cap on requests in flight, and that check is the first line of every handler. The balancer shares requests evenly among the backends in rotation, sends each one a health probe every second, a small request on a path of its own, and removes a backend after three failed probes. The evening peak runs well over what the caps allow.

What happens to the backends during the peak?

<details><summary>A. The balancer removes them one after another, and the rest get more traffic</summary>

✓ Right. Backends drop out of rotation one after another while still serving at their cap, and each one that leaves pushes its viewers onto the rest, where refusals rise.

A probe is a request, and **a shedder that decides from its own state alone refuses a probe like any other call**. To the balancer, three refusals in a row mean a dead backend.

Exempt the probe path from shedding, so that a busy backend still answers it. The probe then says the process can answer and no longer says how busy it is, so the balancer needs another signal for that.

</details>

<details><summary>B. They stay in rotation, each refusing only its excess</summary>

✗ Shedding was added to keep backends up, and in the design each one refuses its excess and carries on. The probe is part of that excess. At the cap it gets the refusal a viewer's call gets, and three in a row take the backend out.

</details>

<details><summary>C. The balancer sends each less traffic, reading the refusals as a request to slow down</summary>

✗ A refusal is a signal to slow down, and backpressure is the word people use for it. The balancer reads one thing, whether probes pass, and has no dial for sending a backend less. A failed probe counts toward removal, so a backend gets its full share or nothing.

</details>

<details><summary>D. The busiest one leaves rotation, and the others hold steady</summary>

✗ When one backend fails, the fleet carries on without it, which is what a balancer is for. Removal gives this fleet no relief, since the backend's share moves to the others, which are at their caps too, and their probes begin to fail as well.

</details>

---

###### 12. Growing a queue from 100 to 10,000 before a sale

Last year's sale at a concert-ticket site ended in queue-full errors, so you raise the queue from 100 places to 10,000. The workers serve 200 requests a second in order of arrival, and each starts on whatever it takes from the queue. A client gives up after two seconds. This year, arrivals stay well above 200 a second for ten minutes.

What reaches customers at the height of the sale?

<details><summary>A. A short wait, and nearly all of them served</summary>

✗ A queue does absorb a spike, and half a second at 600 a second followed by 100 a second has every request answered in time on 10,000 places. With arrivals above 200 a second for ten minutes, the line never drains, and the waits pass two seconds a few seconds in.

</details>

<details><summary>B. The same as with a queue of 100</summary>

✗ A queue changes when work happens and not how much gets done, and the workers finish 200 requests a second at either size. What differs is whether anyone is still there to read them. With 100 places the answer reaches a client still waiting, and with 10,000 it comes long after the two seconds are up.

</details>

<details><summary>C. A reply in time for almost none of them, and workers busy throughout</summary>

✓ Right. Almost nobody gets a reply in time, while the workers run flat out on requests whose customers have left.

**When the wait in a queue outlasts the client, the workers answer nobody**, and two seconds at 200 a second puts that line at 400 places. Put arrivals at 300 a second: 100 places have about two requests in three answered in time and the third refused at once. With 10,000 places the same third is refused once the queue is full, the other two wait about 50 seconds, and the answers in time fall from 200 a second to none.

Keep the queue shorter than capacity times patience, here 400 places, and let a worker drop a request whose client has gone. Serving the third that is refused takes more capacity or less work.

</details>

<details><summary>D. A crashed server, its memory filled by the queue</summary>

✗ Memory is what runs out when a queue has no end, and 10,000 places are an end: if a request takes 2,000 bytes, 10,000 of them come to 20 MB, which leaves the server up and the queue full.

</details>

---

###### 13. Serving the newest request first during an overload

On a price-comparison site, the workers serve 200 searches a second, and the evening peak runs 25 percent above that for ten minutes. A client gives up after two seconds, and the queue holds 2,000 requests, served oldest first. You change only the order: workers now take the newest request, and a full queue pushes out its oldest.

What do customers see during the overload now?

<details><summary>A. Most searches answered in time, the rest abandoned</summary>

✓ Right. About four searches in five get an answer inside the two seconds, where the old order answered almost none, and the other fifth wait past that and give up.

With a long queue the oldest requests are the ones whose clients have left, and **serving the newest spends the workers on clients still waiting**. The share answered in time comes close to capacity over arrivals, which is four in five here.

Serve newest first where requests are independent and order between them does not matter. A first-come queue short enough to clear inside the two seconds, under 400 places here, does as well, since it answers what it admits in time and refuses the rest at once.

</details>

<details><summary>B. Nearly every search timing out, as before</summary>

✗ Capacity has not changed, and reordering a queue creates none. The workers still produce the same number of answers. In the old order those answers reached nobody, and taken from the newest end they reach a client still inside its two seconds.

</details>

<details><summary>C. Every search slowed by about the same amount</summary>

✗ A first-come queue with room to spare shares its delay out evenly. Newest-first shares nothing evenly: a fresh search is served almost at once, and one that has fallen to the bottom of the pile stays there past the two seconds.

</details>

<details><summary>D. Replies going to the wrong customers, with their order lost</summary>

✗ On a protocol that returns replies in order over one connection, a change of order would scramble them. Each search here has its own wait on its own connection. A worker's reply goes back to the connection that asked, and the queue's order does not touch that.

</details>

---

###### 14. Marking checkout as critical at the gateway

In a food-delivery app, the gateway gives each call a priority label, a field saying how much the call matters: critical for checkout, sheddable for browsing. It reads the label for its own decisions and forwards each call as the client sent it. Behind it, an inventory service that is overloaded in its own right refuses any call past a cap on calls in flight.

What do checkout calls get at the inventory service?

<details><summary>A. A place in front of the browsing calls</summary>

✗ The gateway marked them critical, and the priority rule sits in its configuration. The mark stays in the gateway, so the inventory service gets calls with no label, and it has no line to reorder: it admits or refuses each one as it arrives.

</details>

<details><summary>B. The same odds of refusal as browsing calls</summary>

✓ Right. At the inventory service a checkout call is refused as often as a browsing call, and the calls that end in a payment get no protection.

The gateway forwards each call as the client sent it, and **a priority label does nothing at a service that never receives it**. This shedder decides from the calls in flight at the moment a call arrives, so the path cannot change the odds.

Send the label on with the request at every hop, have each shedder read it, and keep part of the cap for critical calls, which moves the refusals onto browsing. Set the label at your own gateway, since a client that can set it for itself can mark everything critical.

</details>

<details><summary>C. Refusal ahead of the browsing calls</summary>

✗ A shedder can go for the expensive requests first, and a checkout does more work than a page of menus. The inventory service's cap counts calls in flight and nothing else. Cost does not enter its decision, so a checkout that arrives at a full cap meets the refusal any other call would.

</details>

<details><summary>D. No refusals anywhere along the path</summary>

✗ A gateway that sorts traffic at the front door leaves little to think about behind it. The sorting ended at that door. The inventory service refuses past its own cap whatever the gateway decided, and a checkout call it refuses fails there.

</details>

---

###### 15. Refusing one call in five across a three-step checkout

A flower shop has more customers than its servers can take on the eve of a holiday. An order takes three calls of similar cost: basket, address, payment. You turn on a shedder that refuses one call in five at random, whichever step it belongs to. A customer who meets a refusal leaves the site.

How many orders does the evening lose to the shedder?

<details><summary>A. One order in five, the same as the calls</summary>

✗ A shedder shows one percentage, the share of calls refused, and that is the number on the incident report. An order is not one call: it has three chances to meet a refusal, and a customer here leaves on the first.

</details>

<details><summary>B. Few orders, most refusals falling on people only trying a basket</summary>

✗ A sales funnel shows most shoppers at the top never buying, so a refusal there costs little. Refusals at random land on any of the three steps, and these customers came to order. Of the lost orders about 41 percent are refused at the basket, and the rest had already got further.

</details>

<details><summary>C. Only the orders refused at the basket, and none past it</summary>

✗ A check at the door runs once on a request, and what it admits goes on. The shedder sits on every call, so getting past the basket protects nothing: about 26 percent of the lost orders fall at payment, with two answered calls behind them.

</details>

<details><summary>D. More than one order in five, some as late as payment</summary>

✓ Right. The evening loses about half of its orders, 49 percent, though the shedder refuses one call in five, and some of those customers have a basket and an address behind them when payment is refused.

An order needs all three calls through, and **refusals at random add up along the steps of a task**. Each step lets four calls in five through, and three steps in a row leave about half. A late refusal also throws away work the servers already did: for every 100 customers, they answer about 42 calls for people who then leave.

Refuse at the first step, per customer, and let an order that got past it finish. The customers turned away at the basket are still lost, since the servers are no bigger, and the servers spend their work on orders that complete.

</details>

---

###### 16. Putting an in-memory list between the API and a slow writer

A fitness-tracker API runs this code on a server with 8 GB, for events of about 1,000 bytes. `db` is your thin client for the database, and the writer saves 500 events a second. Since eight this morning, 800 events a second have been arriving.

```python
from collections import deque

events = deque()

def post_event(event):
    events.append(event)
    return 202

def writer():
    while True:
        if events:
            db.insert(events.popleft())
```

What do the tracker's users get by late afternoon?

<details><summary>A. Replies slowed to the writer's pace</summary>

✗ Before the list, the call waited for the database, and a slow step did slow every request. `post_event` now returns right after the append, so the writer's pace never reaches the reply, and the API answers as quickly in hour six as in hour one.

</details>

<details><summary>B. Quick 202s, with the oldest events dropped from the list</summary>

✗ A bounded ring buffer drops its oldest entry to make room. Nothing here removes an event except the writer, which takes the oldest off the list to save it, and nothing caps the length of `events`, so every event stays in memory until its turn.

</details>

<details><summary>C. Quick 202s for hours, then a crash that loses every waiting event</summary>

✓ Right. The trackers get a quick 202 for about seven hours while the list grows. Then the process runs out of memory and takes the list with it, and events the API accepted are gone for good.

**A buffer with no bound pushes nothing back at its senders**, and the difference between the rates goes into memory at 300 events a second. At 1,000 bytes each, 8 GB fills at about 7.4 hours.

Bound the list and answer 503 when it is full. That is backpressure: the senders feel the writer's pace, and the server stays up. Events still in the list die with a crash, so a 202 that must mean "safe" needs a durable queue.

</details>

<details><summary>D. A 429, once the list grows long</summary>

✗ When work piles up something has to say no, and a 429 is what you have seen APIs answer. `post_event` has one return, 202, and never looks at the length of the list.

</details>

---

###### 17. Watching the same CPU with a shedder and a scaler

A photo service runs ten copies at 60 percent CPU. A shedder on each copy refuses work to hold CPU at 65 percent, refusals included, and readings wobble by a point or two. A scaler, the part that sets how many copies run, adds one when mean CPU stays above 70 percent for five minutes. Traffic doubles over an hour.

How many copies does the fleet run an hour in?

<details><summary>A. As many as the doubled traffic needs</summary>

✗ Following traffic is what autoscaling is sold on, and without the shedder this traffic would pass 70 percent in about ten minutes. The shedder stops each copy's CPU at 65, and the scaler reads that number and not the traffic behind it.

</details>

<details><summary>B. The same ten, with customers' calls refused</summary>

✓ Right. An hour in, the fleet still runs ten copies, and the scaler finds no reason to act. About 46 percent of calls are refused if refusals cost nothing, and more as they cost more.

The shedder holds CPU at 65, and **a scaler that reads CPU cannot see the traffic a shedder holds back**. A wobble of a point or two leaves the mean under 70, though the doubled traffic would need 18 copies at 70 percent.

The shedder carries the wait while new copies start. To get those copies, have the scaler read what the shedder cannot hold down, such as refused calls, or set its level under the shedder's, so that they arrive before shedding starts.

</details>

<details><summary>C. More with every five minutes, without end</summary>

✗ Refusals cost CPU, and CPU drives the scaler, so a runaway loop has the right parts. The level the shedder holds includes its refusals, and the reading stays at 65 however many it refuses.

</details>

<details><summary>D. Fewer than the ten it started with</summary>

✗ Scalers remove idle copies, and the refused traffic has gone away. A copy at 65 percent is far from idle, and its CPU has risen since the start of the hour, so this scaler has no reason to remove one.

</details>

---

###### 18. Limiting the shipments endpoint to 6,000 calls a minute

A logistics platform opens a tracking API to carriers and online shops. After a load test you add this limit at the gateway. In the test, one client sending 7,000 calls a minute got a 429 past 6,000, and the backend stayed quick.

```yaml
rate_limits:
  - route: /v1/shipments
    requests: 6000
    per: minute
    key: route
```

A month later, a bug puts one carrier's sync tool into a loop of 300,000 calls a minute. A small shop checks its parcels about ten times a minute, as it does every day.

What does the small shop get during the loop?

<details><summary>A. Quick replies on every call, with the 429s going to the carrier</summary>

✗ In the test the client was refused at 6,000, and ten calls a minute is nowhere near that. The client in the test was the only caller on the route. `key: route` gives the route one count and holds no number for the carrier or the shop, so the limiter cannot pick the carrier out, and once the count is full the shop's next call gets the 429 the loop's calls get.

</details>

<details><summary>B. 503s and timeouts from a swamped backend</summary>

✗ The loop sends fifty times what the backend was tested at, and a flood like that is how services fall over. This one stops at the gateway, which refuses everything past the 6,000, so the backend takes what it took in the test and answers the calls that reach it as quickly as before.

</details>

<details><summary>C. A reply on every call, after a wait behind the carrier's calls</summary>

✗ A leaky bucket holds calls in a line and lets them out at a steady pace, and in everyday talk throttled means slowed down. This limiter answers a call that finds the count full with a 429 at once and keeps no line, so nothing of the shop's waits behind the carrier's calls.

</details>

<details><summary>D. A 429 on nearly every call, and a quick reply on the rest</summary>

✓ Right. Nearly every one of the shop's calls gets a 429, though it sends ten a minute against a limit of 6,000, and the shop loses its parcel tracking for as long as the carrier's bug runs.

**A limit keyed on the route holds one counter for everyone on that route**, and the limiter never asks who is calling. The answers follow arrivals: about one call in fifty is answered, for the carrier and the shop alike. The test could not have shown this, since one client at 7,000 a minute gets 6,000 through whether the key is the route or the customer.

Give each customer a quota of its own, your own tools included, so that no one caller can fill the count, and keep the 6,000 as a ceiling above them if the backend needs one. With 1,000 a minute per customer, the loop gets 1,000 through and every one of the shop's calls is answered.

</details>

---

###### 19. Dropping a hotel page's reviews while the origin is busy

On a hotel-booking site, the page handler leaves out the guest reviews and the similar-hotels list whenever CPU on the origin, the server that builds the page, is above 80 percent, and still answers 200, so that a busy evening costs the origin less. A CDN sits in front. The `ETag`, the tag a cache sends back to ask whether its copy still holds, is the version number of the hotel's row, and when the tag matches the row the origin answers 304. Every page goes out with headers like these:

```http
HTTP/1.1 200 OK
Content-Type: text/html
Cache-Control: public, max-age=600
ETag: "41"
```

At eight one evening a national TV spot names a small hotel. The CDN holds no copy of its page, and for the next twenty minutes the origin runs above 80 percent. People keep opening the page all night, and the hotel's row does not change.

What does a traveller see on that page the next morning?

<details><summary>A. The page without its reviews</summary>

✓ Right. The traveller gets the page with no reviews and no similar hotels, long after the origin has calmed down, and the page stays that way until the hotel's row next changes.

The CDN's first request came during the rush, and it stored the 200 it got, the reduced page. When the ten minutes of `max-age` run out, the CDN keeps that copy and sends `"41"` to the origin to ask whether it still holds. The row is still at 41, so the origin answers 304, and **on a 304 a cache keeps the body it stored and restarts its TTL**. This repeats all night, and the origin never renders the full page for this copy. HTTP expects a tag written plainly, as here, to change whenever the body changes, and this one follows the row, so two different bodies share it. An edge location, one of the CDN's groups of servers, that was first asked after the rush holds the full page instead.

Give the reduced page a tag of its own, such as `"41-reduced"`. The first revalidation after the rush then brings back the full page, within ten minutes of the rush's end, and a short `max-age` on the reduced page makes that sooner. Any reduced answer that a cache can store needs the same check of its headers.

</details>

<details><summary>B. The full page, with its reviews</summary>

✗ The copy's ten minutes ran out long ago, and the origin has room to render the whole page again. Running out does not empty the cache: the CDN asks with its tag first, and `"41"` still matches the row, so the origin answers 304 without rendering anything, and the stored page goes on with no reviews.

</details>

<details><summary>C. The page without its reviews, then the full page on a reload</summary>

✗ A reload is the first thing anyone tries on a page with something missing, and it does get past the browser's own copy. The CDN comes next: it answers from its copy or asks the origin with the same tag and gets another 304, and either way the reload shows the same page with no reviews.

</details>

<details><summary>D. A blank page</summary>

✗ A 304 has no body, and while a traveller waits, the origin sends the CDN a 304 for this page. The CDN holds the stored body and sends it on to the traveller in an ordinary 200: a complete page, short of its reviews.

</details>

---

## Further reading

- [Eliminating receive livelock in an interrupt-driven kernel](https://doi.org/10.1145/263326.263335), 1997: how a machine under heavy input can spend all its time on interrupts, and how scheduling that work eliminates this.
- [SEDA](https://doi.org/10.1145/502034.502057), 2001: a service built from stages joined by explicit queues, with adaptive load shedding when demand exceeds capacity.
- [Metastable failures in distributed systems](https://doi.org/10.1145/3458336.3465286), 2021: a failure pattern in distributed systems, with a framework for it and examples from years of operating them at scale.

Prev: [Load balancers and proxies](../load-balancers-and-proxies/README.md) · Next: [Timeouts, retries and circuit breakers](../timeouts-retries-and-circuit-breakers/README.md)
