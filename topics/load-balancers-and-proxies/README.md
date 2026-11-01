# Load balancers and proxies

Prev: [HTTP caching and CDNs](../http-caching-and-cdns/README.md) · Next: [Rate limiting and load shedding](../rate-limiting-and-load-shedding/README.md) · Reviewed October 2026

A load balancer spreads clients' requests over several copies of a service, its backends, from one address the clients know. A reverse proxy stands in the same place and works on the traffic as it passes: clients cannot reach the backends directly, and the proxy can end TLS, buffer replies, retry requests and answer on its own. A balancer that reads requests is a reverse proxy too, so this page says balancer where the choice of backend matters and proxy where the two connections do.

- **L4 balancer**: picks a backend once, when a connection opens, then copies bytes both ways. Under TLS it sees addresses, ports, the server name in the handshake and sealed bytes, and no path.
- **L7 balancer**: ends the client's connection, reads each HTTP request and picks a backend for each one. To route by path it needs the TLS key, since the path travels inside the encrypted bytes.
- **Multiplexed connection**: one long-lived connection that carries many requests at once, as HTTP/2 does.
- **DNS load balancing**: DNS hands clients the addresses of several balancer nodes and the client picks one.
- **Health check**: a small request the balancer sends each backend at a fixed interval. A set number of failures takes the backend out of the pool, and a set number of passes returns it.
- **Passive health check**: watching live traffic instead, by counting each backend's failed requests, in a time window or in a row, and removing it for a while.
- **Draining**: taking a backend out so that it gets no new requests while the ones it holds keep running.
- **Keep-alive**: reusing one connection for request after request. A proxy keeps a pool of these open to its backends, and many clients' requests can share one.
- **Idle timeout**: how long a side keeps a silent kept-alive connection before closing it, set separately on each side.
- **Gateway error**: a status the proxy writes itself. A 502 says it got an invalid reply from the backend, and a 504 that no reply came in time.
- **Framing**: how a reader of HTTP finds where a message ends. `Content-Length` gives its size, chunked coding marks each piece and the last, and a reply with neither ends when the connection closes, so a cut reply cannot be told from a whole one.
- **X-Forwarded-For**: a request header that lists client addresses, with each proxy adding at the end the address that connected.
- **TLS termination**: ending TLS at the proxy, so backends do no handshakes and the traffic behind the proxy is plain unless a question says it is encrypted again.

The usual ways a balancer chooses a backend are four balancing algorithms and a sticky session:

| Method | What decides |
| --- | --- |
| Round robin | each backend's turn, in order |
| Least connections | the count of requests in flight on each backend, all compared |
| Power of two choices | the same count, compared on two backends sampled at random |
| IP hash | the client's address alone |
| Sticky session | a cookie the balancer set earlier, naming a backend; what that backend holds in memory is lost with it |

When too few backends pass their checks, some balancers fail open and spread traffic over all of them, and others keep to those that pass. A balancer or proxy on this page has only the settings a question names, and a balancer keeps to those that pass.

Trouble starts with what a balancer or a proxy cannot see from where it stands: what a request costs, what a backend holds in memory, and whether the two sides of a connection agree on its limits and its messages.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- say where traffic lands when a fleet runs hot, when one backend breaks or restarts, when you add backends, and when two zones differ in size
- plan active and passive health checks and a drain so that a pool stays standing through an outage or a rollout
- trace a slow phone's download, a 502 no backend logged, a 504 on a long report and a restart during a charge through the two connections a proxy holds
- write a backend that stands behind a proxy: how an upload is framed, which forwarded address to believe and where a mailed link gets its scheme

###### 1. Running forty equal backends under round robin

Forty identical backends serve a search API behind a round-robin balancer. Most searches finish in milliseconds, and at any moment a few heavy ones are running, each holding half of a backend's CPU for about 30 seconds. Ahead of a launch, the capacity plan shows a mean CPU of 45 percent across the fleet.

What do the forty backends do as traffic climbs for the launch?

<details><summary>A. All forty reach their limit together, at about double today's traffic</summary>

✗ At a mean of 45 percent the plan has room to about double. No backend runs at the mean here. If three heavy searches run at once, those three backends stand near 90 percent and the other thirty-seven near 40, so the three reach their limit first and the rest are still far from theirs.

</details>

<details><summary>B. The busiest few run out early, and most of the fleet has room</summary>

✓ Right. Whichever backends hold the heavy searches run out long before the rest, and searches landing on them slow down while most of the fleet has room. The launch meets its limit well ahead of what the mean of 45 percent says.

Every backend gets the same number of requests, since **round robin counts requests and ignores what each one costs**, and each heavy search adds half of a backend's CPU to whichever backend its turn falls on. Say three run at once over an ordinary load of 40 percent: those three stand near 90 and the rest near 40. If the heavy searches stay at three and only the 40 grows, the three run out at about 1.25 times today's traffic. Which three changes as heavy searches end, and the spread stays.

Plan from the busiest backends and leave the mean out of it. Giving the heavy searches a pool of their own leaves round robin with requests of like cost, where equal turns do come out as equal load.

</details>

<details><summary>C. Round robin gives busy backends fewer turns, and all forty fill evenly</summary>

✗ The word balancer promises balanced load, but round robin ignores CPU and forgets what each turn cost: every backend here gets the same number of requests, down to the last one, including those holding heavy searches.

</details>

<details><summary>D. Every search slows by the same small amount on all forty</summary>

✗ A fleet's latency chart draws one line for every search on every backend. A backend near 40 percent queues nothing and runs each search as fast as before. Only searches sent to the backends out of CPU wait, so most stay fast and a growing few turn slow.

</details>

---

###### 2. Sending traffic to a backend that answers errors in one millisecond

A booking site keeps twelve backends behind a balancer using least connections. Say the evening peak brings 4,000 requests a second and a healthy backend takes 50 ms to answer. A bad configuration push reaches one backend, which now answers every request with a 500 in 1 ms. The health check only opens a connection to a port, and that port stays open.

How many of the bookings does that one backend receive?

<details><summary>A. About one in twelve, as it did before the push</summary>

✗ Twelve backends share the work, and the check passes as it did yesterday. One in twelve is what this backend gets at a quiet hour, when the healthy ones are often idle and ties go in turn. At this peak each healthy backend holds several requests for 50 ms each, and the broken one usually has the lowest count.

</details>

<details><summary>B. None of them, after its first few errors</summary>

✗ A run of 500s is the plainest signal a backend can give, and a passive health check would act on it. This balancer only passes each status along, and its check sees an open port, so the backend stays in the pool and its errors keep going out to customers.

</details>

<details><summary>C. All of them, with the other eleven idle</summary>

✗ A backend that answers in 1 ms clears its requests long before a healthy one does, and least connections sends each request to the lowest count. Even at 1 ms each, 4,000 requests a second keep a few in flight on it, and a healthy backend that has drained below that count takes the next request. The eleven healthy backends still get some of the bookings.

</details>

<details><summary>D. Most of them, far beyond one in twelve</summary>

✓ Right. About 73 of every 100 requests go to the broken backend, and most customers trying to book get an error while eleven healthy backends have room.

A healthy backend holds each request for 50 ms and this one finishes each almost at once, so its count stays lower and it wins most choices. **Least connections compares requests in flight and nothing else**, and here the low count means the backend has failed.

Add a signal the broken backend cannot pass by being quick. With a passive health check that removes a backend after five errors in a row, it takes five errors and then nothing until it returns. A check that sends a real request would also catch this push, since it gets the same 500.

</details>

---

###### 3. Putting a restarted backend back at the evening peak

At the evening peak each of a hotel-search site's ten backends holds about 40 requests, behind a balancer using least connections. A deploy restarts one backend with its in-memory cache of rates empty, and its health check passes within seconds. The same restart on a quiet staging pool looked normal.

What does the restarted backend receive as it comes back?

<details><summary>A. One search in ten, the same as each of the other nine</summary>

✗ A passed check makes it one of ten, and on the quiet staging pool the restart changed nothing. Staging had no gap for an empty backend to fill, since its counts all sat near zero. At this peak the backend comes back to a gap of about 40 requests, and the lowest count wins.

</details>

<details><summary>B. None until its cache has warmed, with the other nine carrying on</summary>

✗ An empty cache makes searches slow, and a check can be written to fail until a warm-up is done. This check passes within seconds, while the cache is still empty, and a check that passes puts the backend back in the pool. From there the balancer knows its count and nothing of its cache, so searches reach it at once.

</details>

<details><summary>C. Every new search at first, with the other nine getting none</summary>

✓ Right. New searches all go to the restarted backend until its count meets the others, and the customers behind them meet an empty cache together.

A health check says the backend answers, and nothing of what it holds in memory. Under least connections **a restarted backend starts at the lowest count however cold its cache is**, and the run lasts until its count nears the others' 40, whatever a cold search costs. Power of two choices softens this: the restarted backend gets about 20 of the first 100 searches, because it is only compared when it happens to be sampled.

Turn on slow start, a setting that raises a new backend's share over time. If its weight rises from 10 percent over 60 seconds, the run shrinks to a few searches in a row. Check first that your balancer allows it with your algorithm: some refuse to combine it with least connections.

</details>

<details><summary>D. A few searches, then more as its replies speed up</summary>

✗ Replies speed up as a cache fills, and some balancers can ease a new backend in with a setting of their own. Least connections alone looks at requests in flight and never at reply time, so the empty backend gets the longest run of searches it will ever get.

</details>

---

###### 4. Doubling the backends behind long multiplexed connections

An internal pricing service has thirty client services, each holding one multiplexed connection through an L4 balancer to one of four backends. The client config, the same for all thirty:

```text
protocol             HTTP/2
connections          1, kept open
max connection age   none
reconnect            on connection error
```

At the morning peak you double the backends to eight, and the four new ones pass their health checks within a minute.

What do the four new backends serve during the next hour?

<details><summary>A. About half the pricing calls, from the first minute</summary>

✗ With short connections the load on each backend halves when the backends double, because every new connection is a fresh choice. These thirty connections opened long ago, and a call on one of them is more bytes on it, with no choice for the balancer to make. The four old backends keep every call.

</details>

<details><summary>B. None of the pricing calls</summary>

✓ Right. All the hour's calls stay on the four old backends. The peak you scaled for still lands on the machines you meant to relieve, while the new four sit idle.

A multiplexed connection carries all of a client's calls for as long as it lives, and **an L4 balancer chooses only when a connection opens**. These connections live until a connection error, and until one comes the balancer has no choice to make.

To reach the new backends, change either side. An L7 balancer picks for each call and gives the new four half of them. Clients that reopen every 300 s move across as their timers fire, so the new four reach half of the calls only about five minutes later.

</details>

<details><summary>C. A growing share of the pricing calls, as connections turn over through the hour</summary>

✗ Clients with a maximum connection age do move across like this, one reopen at a time. These clients have none and reconnect only on a connection error, and a healthy connection gives them no cause to, so the new four's share stays at zero.

</details>

<details><summary>D. About half the pricing calls, after a burst of failures as the balancer resets connections</summary>

✗ Eight backends is a new pool size, and a pool that hashes clients over its size moves them when that size changes. This balancer chose once for each connection, and adding backends leaves the choices already made as they were. All thirty carry on, with no reset and no failed call.

</details>

---

###### 5. Adding three backends during a sale with carts in memory

An online shop holds each cart in its backend's memory and runs on three backends behind an L7 balancer with round robin and sticky sessions. In the middle of a sale the three backends run hot, so you add three more, and all six stay healthy.

Which of the next requests from shoppers who already have carts reach the new backends?

<details><summary>A. None of them, with each shopper staying on the backend they were on</summary>

✓ Right. No request from these shoppers moves, since each cookie still names a healthy backend. The shoppers keep their carts and their backends, and the three hot backends stay hot for as long as those cookies last.

**A sticky session's cookie overrides the balancing algorithm while it names a healthy backend**, and only visitors with no cookie go by round robin, over all six. The new backends' share grows only as new visitors arrive and old cookies run out.

Hold carts in a store all backends can read, and the cookie can go: new backends then help from the first request. A shorter cookie lifetime helps only once a cart can survive the move to another backend.

</details>

<details><summary>B. Any of them, with each request dealt in turn over all six</summary>

✗ Adding backends does change round robin, which now goes over all six, and a visitor with no cookie meets it. The balancer reads a shopper's cookie first and falls back to round robin only when there is none.

</details>

<details><summary>C. Their page views, with only the cart requests staying on the old backends</summary>

✗ The cart is the only thing these backends hold in memory, and any of the six can render a product page. The balancer reads the cookie on every request, whatever its path, so a shopper's product pages go to the backend the cookie names, along with the cart.

</details>

<details><summary>D. Those of some shoppers, moved to even out the six and finding their carts empty</summary>

✗ Sending work where there is room is what people buy a balancer for, and IP hash does send people elsewhere when a pool changes size. Room and the pool's size play no part here: each shopper's cookie names a backend that is still up, and the balancer does as it says. A cart is lost only with its backend, and all three old backends are healthy.

</details>

---

###### 6. Splitting backends eight and two across two zones

A ticketing service runs in two zones, separate parts of a region that fail independently, with one balancer node in each. The DNS answer lists both nodes. To save on traffic between zones, you keep each node sending only to the backends in its own zone. A capacity shortage then costs one zone six of its backends, which leaves this:

```text
Zone    Balancer node    Backends
A       1                8
B       1                2
```

How does the load on each backend in zone B compare with each in zone A?

<details><summary>A. The same load as each of the eight</summary>

✗ A single balancer spreading over ten backends gives each an even share, and these ten sit behind one DNS name. This balancer is two nodes, each of which takes its share of the clients and passes them only to its own zone, so two backends split about what eight split next door.

</details>

<details><summary>B. Several times the load of each of the eight</summary>

✓ Right. Each of the two carries far more than each of the eight, and zone B's two backends can run out of room while zone A's eight have plenty.

**DNS load balancing gives each node its share of the clients whatever stands behind it**, and each node then spreads that share over its own zone. If the clients split evenly, each of the two carries 25 percent of all traffic against 6.25, four times as much.

Turn on cross-zone load balancing: every backend then gets 10 percent, in either zone, and requests travel between zones to get there. Or keep the zones the same size, and raise an alarm when one shrinks.

</details>

<details><summary>C. No load, with zone B's node taken out of the DNS answer</summary>

✗ DNS failover can take a node out of the answer when its backends all fail, and zone B has lost most of its backends. Two healthy backends still make a working zone, so its node stays in the answer, clients go on reaching it, and those two take everything it gets.

</details>

---

###### 7. Running a database query in a health check

A news site holds its articles in memory and reads comments from the database. To make the health check thorough, you have each backend's check run a database query. The balancer checks every 5 seconds, takes a backend out after 2 failures and returns it after 3 passes. One evening the database restarts and stays down for 40 seconds.

What do readers get for most of those 40 seconds?

<details><summary>A. The balancer's own error on every page, articles included</summary>

✓ Right. By second 10 every backend is out of the pool, and for about 30 of the 40 seconds readers get the balancer's error on every page, articles the backends could have served included.

When the database goes down, **a check that queries a shared dependency fails on every backend at once**, and the balancer reads each failure as a broken machine. It has no way to tell that the backends fail for one reason, or that they could still serve articles.

Make the check ask only what this backend needs to serve its main pages, and let comments fail on their own. A balancer that fails open keeps articles served through this, though it also sends traffic to a backend that is broken.

</details>

<details><summary>B. Articles as usual, and comments failing</summary>

✗ The article code never touches the database, and only the comment code does. The check does touch it, on every backend, so all of them fail it together and the balancer stops sending articles too.

</details>

<details><summary>C. A 500 from the backends on every page</summary>

✗ A backend that cannot reach its database answers with its own 500, and comment requests do exactly that at first. Articles never ask the database, so they would not fail that way, and within 10 seconds the failed checks have taken every backend out. After that no request reaches a backend, and the balancer writes the error a reader sees.

</details>

<details><summary>D. Every page, slowly, as the backends wait for the database</summary>

✗ A database that stops answering usually means slow pages, since requests pile up behind its timeouts. Articles here wait on nothing, and comments never get the chance: within 10 seconds the backends are out of the pool, so no reader's request reaches one to wait.

</details>

---

###### 8. Taking out any backend with five failed requests in a minute

A service turns office documents into PDFs on six backends. The platform team turns on passive health checks at the balancer: a backend is removed for ten minutes once five of its requests fail within a minute, with 500s counted. A failed request is also retried on the next backend, up to three times. A customer's nightly script uploads twelve times a minute, always the same damaged file, and the parser answers it with a 500.

What does the pool hold a few minutes into the script's run?

<details><summary>A. All six backends, with only that customer's uploads failing</summary>

✗ Error reports grouped by customer show exactly this picture: one account red and the rest clean. The balancer counts by backend and has no idea of a customer. Each upload fails on up to four backends, and those failures stand against the machines, so the backends go and everyone's uploads go with them.

</details>

<details><summary>B. Five backends, with the one that met the file first taken out</summary>

✗ A passive health check is built to remove the one bad machine among healthy ones, and here the first backend to reach five failures goes. The removal changes nothing about the file. The next uploads fail on the backends still in the pool, each collects its five failures faster than the last, and the removals carry on down to none.

</details>

<details><summary>C. No backends, with every customer's upload refused</summary>

✓ Right. The pool is empty within about a minute and stays empty for the ten-minute removal. A damaged file from one customer has shut off PDFs for all of them.

**A passive health check treats a failure as a fact about the machine**, and here the failure belongs to the request. Each retry carries the file to another backend, where it fails and counts again, so healthy backends collect the failures that remove them.

Count broken connections and timeouts instead of 500s, and do not retry a request that got a 500. A cap on removals limits the damage: with at most one backend removed, five stay and no other upload is refused. Failing open limits it too, although the file still fails.

</details>

<details><summary>D. All six backends, with that customer's script blocked</summary>

✗ All the trouble here comes from one customer's script, and acting on where trouble comes from is the work of a limiter or a firewall rule, both often found beside a balancer. A passive health check is keyed by backend and does not look at the client. It cannot single out the script, so the script keeps uploading while the backends are taken away.

</details>

---

###### 9. Taking a backend out with a 30-second drain during exports

An accounting app builds each export on a backend, which can take up to three minutes, and then replies with the file and its `Content-Length`. A rollout runs these steps for one backend at a time:

```text
1. balancer: drain the backend (drain period 30 s)
2. when the balancer reports the drain complete: kill the old process
3. start the new version and wait for its health check
4. balancer: put the backend back
```

A user's export is running on a backend when its turn comes, with a minute of work left.

What happens to that export?

<details><summary>A. It finishes normally, with the drain waiting for it</summary>

✗ Draining exists to let running requests finish, and this export is one. A drain period caps that wait: at 30 seconds this balancer reports the drain complete whatever is still running, and the export needs 60.

</details>

<details><summary>B. It moves to another backend and finishes there</summary>

✗ The balancer has other backends, and new requests are already going to them. A running export lives in one process's memory, with its progress on that machine, and no balancer can carry that across. Starting again elsewhere would be a new export that the client has to ask for.

</details>

<details><summary>C. It reaches the user as a file that looks complete, with its last rows missing</summary>

✗ A reply with no framing that stops early cannot be told from a whole one, and an accounting file short of its last rows could go unnoticed. This reply declares its `Content-Length`, which the app knows only once the file is built. With a minute of work left no byte of it has gone out, so there is no part of a file to deliver.

</details>

<details><summary>D. It is cut off when the 30 seconds end</summary>

✓ Right. At second 30 the connection closes with the export unfinished. The user gets an error instead of a file, and everything the export has done so far is thrown away.

Draining stops new requests at once, and **a drain's timer decides how long running requests may go on**. This drain's 30 seconds end with the export still running, the balancer reports it complete, and the rollout kills the process. Some balancers have a drain with no timer, which leaves the backend up until someone stops it.

Set the drain to the longest request the backend can hold, 180 seconds here, and every export finishes. If that makes rollouts too slow, move exports off the request path into background jobs that can resume on another backend.

</details>

---

###### 10. Serving phones on slow networks behind a reverse proxy

A recipe site runs a small pool of eight workers, and each page takes a phone on a slow network 20 seconds to download. You put a reverse proxy in front, with response buffering on and a buffer larger than any page. Its page cache is off.

How long does one phone's request hold a worker now?

<details><summary>A. The whole 20 seconds of the download</summary>

✗ On a server with no proxy that is what happens: the worker writes to a socket until the phone has taken the last byte. With the proxy in front, the worker's socket leads to the proxy, which reads as fast as the worker can write and leaves the phone to its own connection.

</details>

<details><summary>B. No time at all, with the proxy answering from memory</summary>

✗ A cache also answers from memory, and reverse proxies often offer both. A buffer holds one reply on its way to one client and serves nobody else, so with the page cache off the proxy has no page until a worker builds one, and every request still needs a worker for as long as the build takes.

</details>

<details><summary>C. As long as the worker takes to build the page</summary>

✓ Right. The worker is free once the proxy has the page, and a phone's slow network stops tying up the site's small pool. If a page takes 0.2 s to build, the worker is held for about a fifth of a second, against about 20 seconds with no proxy.

With response buffering on, **the proxy reads the backend's reply as fast as it comes and sends it on at the client's own speed**, over a connection of its own. The phone still waits 20 seconds, now on the proxy's connection, and the proxy holds the page until the phone has it.

Leave response buffering on for ordinary pages and size the proxy's buffers for your largest replies. Switch it off where a reply has to reach the client as it is written, and the worker then waits for the phone again.

</details>

---

###### 11. Chasing 502s at quiet hours that the backends never logged

You are on call for an API. About one request in 5,000 gets a 502 from the balancer, mostly at quiet hours, and the backends' logs hold no trace of those requests. These are the timeouts in force:

```text
                      Balancer   Backends
connect timeout       2 s        -
reply timeout         30 s       -
idle timeout          60 s       5 s
request read timeout  -          30 s
```

Which change stops these 502s?

<details><summary>A. Raising the balancer's idle timeout to 120 seconds</summary>

✗ When a client's connection keeps dropping, a longer idle setting on that client is often the cure. Here the balancer would hold connections still further past the backends' 5 seconds, so backends go on closing them first and the failures stay at the same rate.

</details>

<details><summary>B. Raising the backends' idle timeout to 120 seconds</summary>

✓ Right. With the backends holding idle connections longer than the balancer does, the balancer is always the one to close first, and API clients stop getting 502s from a request that never reached a handler.

**The side that reuses a kept-alive connection has to be the one that closes it**, so the backend's idle timeout must be the longer. At 5 seconds the backend closes a silent connection at the moment the balancer may be sending on it, and the bytes meet a closed connection. The balancer reports the failure as a gateway error, and no handler saw the request, which is why the logs are empty.

Set the backends' idle timeout above the balancer's. If the backends must hold few idle sockets, bring the balancer's own idle timeout under theirs instead.

</details>

<details><summary>C. Raising the balancer's reply timeout to 120 seconds</summary>

✗ A 502 and a 504 are both errors the balancer writes itself, and a 504 means a timeout. A reply timeout governs how long the balancer waits for an answer. These requests never produce one to wait for, since they fail on the connection as they are sent.

</details>

<details><summary>D. Lowering the backends' idle timeout to 1 second</summary>

✗ Shorter idle limits free sockets, and a backend that drops idle connections sooner holds fewer that could go wrong. At 1 second the backends still close first, while the balancer counts each connection as open for 60, so it goes on sending on connections that are closing and the 502s go on.

</details>

---

###### 12. Running a 70-second report behind a 60-second proxy timeout

An insurer's analyst opens a report of a few totals that takes 70 seconds to build. The handler runs one blocking query and looks at its connection only when it writes those totals, in one write. The proxy in front talks HTTP/1.1 to the backend. After 60 seconds of silence it closes that connection in the ordinary way, and at second 60 the analyst sees a 504.

What does the backend do after the analyst sees the 504?

<details><summary>A. It runs the report to the end and logs a 200</summary>

✓ Right. The handler finishes at second 70, ten seconds after the analyst saw the error, and logs a 200, so the database does all 70 seconds of work for a report nobody receives.

The proxy wrote that 504 itself and closed its connection to the backend, and **a handler that is not looking at its connection does not learn of the close**. Its one write goes to a connection the proxy has already closed, and the backend's machine accepts a first write to a closed connection without an error, so the handler sees no failure.

Make the handler watch its connection or a deadline shorter than the proxy's timeout, and the work stops by second 60. For a report this long, run it as a job the analyst can collect later. Raising the timeout alone means a 70-second wait, and the next longer report meets it again.

</details>

<details><summary>B. It stops at second 60, when the proxy closes the connection</summary>

✗ Some runtimes do cancel a request when its connection closes, and the handler stops there. That cancel still has to reach the handler, and this one sits inside a blocking query that nothing interrupts. It next looks at its connection when it writes, with nothing to write until the query ends at second 70.

</details>

<details><summary>C. It runs the report to the end, fails on its write and logs an error</summary>

✗ A failed write is what a server logs when it writes to a connection that has gone. That failure appears only after the far side answers the first write with a reset, so it lands on a later write. This handler makes one write of a few totals, the write is accepted, and the log holds a 200.

</details>

---

###### 13. Restarting a backend in the middle of a charge

At a subscription service, the renewal handler is reached by a POST. It calls the payment provider once and holds no retry of its own. The request to the provider carries an amount and a card token. To hide restarts, the platform team gave the balancer these settings:

```text
retry next backend    on connection error
retry on timeout      off
retry methods         GET, HEAD, PUT, PATCH, DELETE, POST
```

One night a backend restarts right after its call to the provider returns, before it has replied.

What does the customer's statement show after the restart?

<details><summary>A. One charge for the renewal</summary>

✗ The handler holds one call and no loop, so reading the code gives one charge. A retry sits above that code: the break comes after the provider has charged, the balancer sends the same POST to the next backend, and that backend's handler makes its one call too.

</details>

<details><summary>B. No charge, with the first one rolled back by the break</summary>

✗ An aborted database transaction does undo its work when the connection breaks. A call to a payment provider has finished by then and belongs to another system, which never hears that a backend restarted. The first charge stands.

</details>

<details><summary>C. One charge and an automatic refund of a second</summary>

✗ Payment providers catch repeats by an idempotency key, a value that marks a request as a repeat. This request holds an amount and a card token, so the provider sees two ordinary payments, and it has no reason to refund either.

</details>

<details><summary>D. Two charges for one renewal</summary>

✓ Right. The customer's statement holds two charges for one renewal, and your handler's code has no retry that would explain them.

**A break after the request was sent does not show what the backend did**, and the balancer knows only that no reply came. It gives the request to another backend, whose handler runs from the top. The HTTP standard forbids this for a proxy: it must not automatically retry a request that is not idempotent, meaning one whose effect changes when it runs twice. POST is not, and a balancer can still be set to do it, as these settings do.

Limit the balancer's retries to idempotent methods, so a sent POST is not passed on. Then build an idempotency key from the renewal itself, so a second backend's call carries the same key and the provider keeps it to one charge.

</details>

---

###### 14. Sending an upload with both a length and chunked coding

A warehouse's old scanners upload readings with both headers below, and the length counts only the data bytes. The proxy reads the chunks and passes the message on unchanged over a backend connection it shares with other customers. The backend goes by `Content-Length` when it sees one, and the upload handler accepts any body. Another customer's request follows right behind it on that connection, sent before the upload's reply has come back.

```http
POST /scans HTTP/1.1
Host: warehouse.example
Content-Length: 5
Transfer-Encoding: chunked

5
hello
0

```

Whose request fails?

<details><summary>A. The next customer's, on that backend connection</summary>

✓ Right. The scanner gets a 200 and the next customer, who sent a good request, gets a 400. The scanner's upload is stored wrong as well: the handler has the size line and the first two letters instead of `hello`.

The proxy goes by the chunks and the backend by the length, and **two hops with different framing stop reading the same bytes at different places**. The proxy finds `hello`. The backend counts five bytes from the start of the body, which are the `5`, the line ending and `he`. What is left, starting with `llo`, stays on the shared connection, and the backend reads it as the start of the next message. It writes a 400 for that text, and replies on a connection are matched to requests in order, so the proxy hands the 400 to the next customer.

Have the proxy remove `Content-Length` before forwarding a chunked message, as the HTTP standard requires, and both requests get a 200. Have the backend treat a request with both headers as an error, which the standard also allows.

</details>

<details><summary>B. The scanner's own upload</summary>

✗ The scanner sent the odd message, and a 400 is how a server refuses one. This handler accepts any body, and the backend found the five bytes the length promised, so the upload gets a 200 and the bytes the backend did not read stay on the connection.

</details>

<details><summary>C. Nobody's, and both requests get a 200</summary>

✗ The length and the chunks both say five bytes of data. They count from different places: the chunk size line sits between the headers and `hello`, the length counts from straight after the headers, and part of the body is left behind on the connection.

</details>

<details><summary>D. Everyone's, on a backend shut down by the message</summary>

✗ Bad input crashing a parser is a common worry with any odd message. These bytes are ordinary text that does not parse as a request. The backend answers with one 400 and closes that connection, and it keeps serving on all the others.

</details>

---

###### 15. Reading three addresses in X-Forwarded-For

A bank's fraud team asks which address to record for each login. Your one balancer stands in front of the backends. Clients may send `X-Forwarded-For` themselves, and some come through proxies of their own. One login reaches a backend like this:

```text
connection from 10.0.8.1

POST /login HTTP/1.1
Host: bank.example
X-Forwarded-For: 10.1.2.3, 203.0.113.50, 198.51.100.7
```

Which address can you vouch for in the fraud log as where this login came from?

<details><summary>A. 10.1.2.3, the first entry</summary>

✗ Code that reads this header usually splits on the comma and takes the first value, and for a client that sends no header of its own the first entry is the one your balancer wrote. This client's request arrived with entries in it already. Whatever comes first was written before your balancer saw the request, and 10.1.2.3 is a private address, with no meaning outside the sender's own network.

</details>

<details><summary>B. 203.0.113.50, the first public entry</summary>

✗ Skipping private addresses cures the plainest fault of the first-entry habit, since 10.1.2.3 can no longer win. A public address costs a sender nothing to write. This one arrived inside the header from outside your network, so your balancer has no knowledge of it.

</details>

<details><summary>C. 198.51.100.7, the last entry</summary>

✓ Right. Write 198.51.100.7, the only entry your own balancer wrote. The fraud log then holds an address you saw connect, where the entries before it are only what the sender said.

Each proxy adds to the end of `X-Forwarded-For`, so **you can trust only the entries your own proxies wrote, counting from the right**. The balancer records the address on the connection it accepted. That address can still be the client's own proxy or office gateway, so it says where the login reached your network and may not name the person's machine.

Count back from the end one place per proxy of yours, so a second proxy moves you to the second-last. Keep the backends unreachable except through the balancer, or anyone can send a header that looks like its entry.

</details>

<details><summary>D. 10.0.8.1, the source of the backend's own connection</summary>

✗ A header can be written by anyone, and a connection's source address cannot be faked so cheaply. 10.0.8.1 is the balancer itself, which opens the backend's connections from inside your network. The fraud log would hold that one address on every login and tell the fraud team nothing about any of them.

</details>

---

###### 16. Building a password reset link behind a TLS-ending proxy

A clinic's patient portal sits behind a proxy that ends TLS, and the proxy answers any plain `http` request with a 301 to the same address over `https`. You add the password reset below, and on your laptop, with the browser talking to the backend directly, a reset asked for over `https` mails an `https` link. `mail.send` hands the message to the clinic's mail service.

```python
def request_reset(request, conn):
    patient = find_patient(request.form["email"])
    scheme = "https" if conn.tls else "http"
    link = f"{scheme}://portal.example/reset?token={new_token(patient)}"
    mail.send(patient.email, link)
    return reply(200, "check your mail")
```

A patient opens the portal at its `https` address, asks for a reset and follows the link in the mail.

What does that patient get from the reset?

<details><summary>A. An <code>https</code> link in the mail, which opens the reset form</summary>

✗ The patient's address bar says `https`, and on your laptop an `https` request got an `https` link. On the laptop `conn` was the browser's own connection. Here TLS ends at the proxy, which opens a plain connection of its own to the backend, so `conn.tls` is false and the mail says `http`.

</details>

<details><summary>B. An <code>http</code> link in the mail, which opens the reset form over plain <code>http</code></summary>

✗ A page opened over `http` stays on `http` as long as its links and forms are relative, and this link starts with `http`. The proxy's 301 comes before any page: the browser fetches the form over `https`, and the new password goes back over `https` too.

</details>

<details><summary>C. An <code>http</code> link in the mail, which ends on a browser error after a chain of redirects</summary>

✗ Redirecting on `conn.tls` behind a TLS-ending proxy is a known cause of a redirect loop, since every request reaches the rule plain. This handler only uses `conn.tls` to write a word into a mail, and the redirect here is the proxy's, which fires on plain requests alone. The browser's next request is `https`, so the proxy forwards it and the form comes back.

</details>

<details><summary>D. An <code>http</code> link in the mail, which opens the reset form over <code>https</code></summary>

✓ Right. The mail holds an `http` link, and the patient still lands on the reset form over `https` and sets a new password. The reset works, so nobody has a reason to look at the link, though the token in it can cross the network unencrypted on its way to the form.

The line that picks the scheme reads the connection the backend accepted, and **a backend behind TLS termination sees the proxy's connection, never the visitor's**. That connection is plain HTTP for every patient. A browser that sends the link as written puts the token in a plain request, and the proxy's 301 then moves it to `https`. A browser set to try `https` first skips that plain request, though the link in the mail stays as it was.

Build the link from a configured `https` address for the portal, which leaves the connection out of it. Where code has to know the visitor's scheme, have the proxy pass it in a header such as `X-Forwarded-Proto` and read that.

</details>

---

## Further reading

- [The Power of Two Choices in Randomized Load Balancing](https://doi.org/10.1109/71.963420), 2001: in its model of random arrivals, a customer who picks two servers at random and joins the one with fewer customers sees an exponentially shorter expected time in the system than one who picks a single server, and a third choice adds only a constant factor.
- [The Tail at Scale](https://doi.org/10.1145/2408776.2408794), 2013: how rare slow responses come to dominate a large service, and ways to limit their effect.
- [Metastable Failures in Distributed Systems](https://doi.org/10.1145/3458336.3465286), 2021: failures that outlast their cause, kept going by the system's own response.

Prev: [HTTP caching and CDNs](../http-caching-and-cdns/README.md) · Next: [Rate limiting and load shedding](../rate-limiting-and-load-shedding/README.md)
