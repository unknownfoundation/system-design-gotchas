# System design

Prev: [Reliability and disaster recovery](../reliability-and-disaster-recovery/README.md) · Reviewed October 2026

A system's design is its parts and what passes between them. Each part can be right alone and pass its own review, and the system still fails at the point where what one part sends out is what the next part takes in, or where a load nobody wrote down falls on a part everyone shares. This page is about those places: whole systems under stated conditions, and many customers on one set of machines.

- **Estimate**: a few rough numbers multiplied to judge a design before it is built, read for the term that decides the answer, with sizes held to the nearest power of ten.
- **Primary and replica**: the primary takes a store's writes and replicas keep copies of them, which is where reads often go.
- **Timeline**: the list of post ids a home feed reads from, kept per user.
- **Fan-out**: one action turning into many: one post into many timeline entries.
- **Fan-out on write and fan-out on read**: two ways to build a feed. The first writes a post into its followers' timelines when it is posted. The second gathers recent posts from the accounts a person follows when the feed opens.
- **Hold**: a seat kept for one customer for a fixed time while they pay, given back if they do not.
- **Webhook**: a request one company's server sends to another's to report an event, such as a payment going through.
- **Signed link**: a link to one object in an object store, made by your server and valid for a set time, so that a browser fetches the file from the store and not through your servers.
- **Log and offset**: a log keeps its messages in order for a set time after they are read, and a consumer's offset is its position in that log.
- **Idempotency key**: a value sent with a request so that the server can answer a repeat from its first result.
- **Tenant**: one customer of a service that many customers share.
- **Noisy neighbour**: a tenant or job whose load on a shared part slows the others.
- **Cell**: a small, separate set of workers that serves some of the customers, so that a failure stays inside it.
- **Pooling proxy**: a server between your app's copies and a database, which shares a few database connections among many clients.
- **Reservation**: the CPU or memory an orchestrator (the system that places a service's copies on machines) sets aside for one copy on a machine.

Customers can share a store in three usual ways: shared tables with a tenant column, a schema each, or a database each.

Several of these places have a page of their own, and this one only names them: a payment longer than its hold ([Distributed locks and IDs](../distributed-locks-and-ids/README.md)), a write to a store and a broker together ([Event-driven architecture](../event-driven-architecture/README.md)), reading your own write ([Replication](../replication/README.md)), a hot key ([Sharding and partitioning](../sharding-and-partitioning/README.md)), retries that multiply ([Timeouts, retries and circuit breakers](../timeouts-retries-and-circuit-breakers/README.md)), and a cold cache ([Caching](../caching/README.md)).

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- read an estimate for a link shortener, a feed or a video site, and find the term that sets its size or its bill
- follow a message, a seat, a payment and a mass notification through a chat, a booking site, a dispatch service and a push job when the load or a device changes
- check who can reach what, and what a repeat costs, around a redirect, a signed link, a key's lifetime, a deleted row and a retry loop under a scaler, when two sound parts meet
- put many customers on shared cells, a shared database, a pooling proxy and shared machines, and say who feels one customer's load

###### 1. Counting clicks on a link shortener

A link shortener serves 20,000 redirects a second, and its design document calls it read-heavy. A cache answers 99 percent of the redirects, and replicas take the misses. A hook on the `links` table deletes a link's cache key whenever its row is updated, so that an edited destination shows at once. This week marketing asks for a click count on each link, and you add a `clicks` column to the row and add one to it on each redirect.

What does the click count do to the cache's 99 percent?

<details><summary>A. It holds at 99 percent, with the primary taking the writes</summary>

✗ Your design document draws writes going to the primary and reads going to the cache, and the new column is a write. That write is also what the hook watches: each click updates the row, the hook deletes the key, and the next redirect finds nothing in the cache.

</details>

<details><summary>B. It collapses on every link, popular ones too</summary>

✓ Right. The cache can no longer hold the links people click. The replicas, which took 200 reads a second, now take about half of the 20,000 or more, and the primary takes 20,000 writes on top, in a design sized as read-heavy.

Each redirect updates the row and the hook deletes the key the next redirect needs, and **a cached value lasts only until the next write that invalidates it**. Say a read takes 1 ms and an update 2 ms, and redirects come at random times. A link clicked once a second almost never gets a hit, and on the busiest links, where a few redirects slip in between a fill and the next delete, the hit rate stays under one half.

Keep counts away from the row the hook watches: append clicks to a log or a table of their own, add them up in batches, and have the hook fire only when the destination changes.

</details>

<details><summary>C. It falls on rarely clicked links and holds on popular ones</summary>

✗ Popular keys are the ones a cache keeps, since they are read all the time. A popular link is also clicked all the time, and each click deletes its key, so the links with the most traffic lose their entries most often.

</details>

<details><summary>D. Only a dip of a point or two</summary>

✗ A deleted key has always cost one reload, and edits to a destination are rare enough that you never saw them on a graph. Clicks are far from rare: the hook now fires 20,000 times a second, on the very links being read, and each reload is undone by the click that follows it.

</details>

---

###### 2. Sizing timeline writers from the mean follower count

A social app builds each home feed ahead of time: every post immediately becomes one task per follower at the back of one shared queue, and workers write the post's id into each follower's timeline. The plan took 2,000 posts a second and a mean of 200 followers, and the workers were sized for 500,000 writes a second, with room to spare. The app's most followed account has 40 million followers. It posts at 9:00:00, and at 9:00:01 a nurse with 150 followers posts a photo.

When does the nurse's photo reach her followers' feeds?

<details><summary>A. Within a second, on the usual path</summary>

✗ The fleet's 500,000 writes a second is more than the plan's 400,000, and the nurse's post is only 150 of them. Her tasks still have to reach the front of the one queue, and 40 million tasks went in a second before hers.

</details>

<details><summary>B. Ahead of most of the big account's writes</summary>

✗ Her post is tiny beside the big account's, and many schedulers let tiny jobs go first. This queue is first in, first out and does not know whose post a task belongs to, so hers stand behind every task already there.

</details>

<details><summary>C. Never, with the post dropped from an overflowing queue</summary>

✗ A burst of 40 million tasks is the sort of thing that overflows a queue, and a full queue drops what comes next. Holding a backlog is this queue's job, and the workers drain it faster than new tasks arrive, so the photo goes out late with the rest.

</details>

<details><summary>D. After the big account's queued writes</summary>

✓ Right. The nurse's 150 followers get her photo about 80 seconds later, after the big account's 40 million tasks, though the plan had room to spare on every line.

The estimate multiplied posts by a mean, and **a mean follower count leaves out the largest account**, the term that decides this wait. At 500,000 writes a second that one post is 80 seconds of the whole fleet, and every later task in this queue stands behind it.

Take accounts above a set size off fan-out on write and read their recent posts when a follower's feed opens, which is fan-out on read for those accounts. Give what remains a queue in which a large post cannot stand ahead of small ones.

</details>

---

###### 3. Choosing a feed design for a network capped at 5,000 friends

A professional network is designing its home feed. Friendships are two-way and capped at 5,000 a person, and most people have a few hundred. The feed shows friends' posts newest first, with no ranking, and people open it about thirty times for each post they write. Three proposals reach the design review.

Which design do these conditions settle on?

<details><summary>A. Write each post to every friend's timeline</summary>

✓ Right. No account can pass 5,000 friends, so the worst post is 5,000 writes, and opening a feed stays one read of one list. The network gets a quick feed from one path to build and run.

**Fan-out on write breaks on a following with no upper bound**, and the cap removes it. If the writers manage 500,000 writes a second, the worst post takes about 10 ms of the fleet. With opens so far ahead of posts, doing the work once per post beats doing it on every open. A feed ranked at read time is another case, since a score that depends on the reader's moment cannot be written ahead.

Build the one path, writing ids into each timeline rather than whole posts. Check first that the cap holds everywhere, because a public page that can gather followers without limit would bring the problem back.

</details>

<details><summary>B. Read each friend's recent posts whenever a feed opens</summary>

✗ Reading at open needs no timeline store, and on a whiteboard it leaves nothing to keep in step. People here open the feed thirty times for each post they write, and each open reads every friend's list, so at 300 friends that is about 9,000 list reads for each post, against 300 writes.

</details>

<details><summary>C. Write small accounts' posts ahead and read large accounts' at open</summary>

✗ Write-ups of large networks' feeds end on this mix, because those networks have accounts with tens of millions of followers. The mix keeps such an account off the write path, and here no account passes 5,000, so the second path would be code and cost with nobody to serve.

</details>

---

###### 4. Planning the monthly bill of a video site

A start-up that hosts cooking videos is planning its first year on a public cloud, at list prices that charge for storage and for data sent. It expects a library of 100,000 videos of about 500 MB each once encoded, 2,000 new ones a month, and each video watched about 300 times a month through the cloud's CDN. The draft budget has four lines: storage, encoding, the servers behind the pages and catalog, and data sent to viewers.

Which line dominates the bill?

<details><summary>A. The storage holding the library</summary>

✗ A library of 50 TB is the largest figure in the plan, and storage is what people ask about when they hear video. Those bytes are kept once and sent 300 times a month, 15,000 TB in all, which leaves storage a small line even where a gigabyte kept costs more than a gigabyte sent.

</details>

<details><summary>B. The encoding of new uploads</summary>

✗ Encoding is heavy work for a CPU, and compute is what most cloud bills are made of. It runs once per video, on 2,000 new videos a month: if each takes 20 minutes at 3 cents a minute, the line is about $1,200.

</details>

<details><summary>C. The data sent to viewers</summary>

✓ Right. Data sent to viewers dominates the bill by far: the library is 50 TB, and viewers pull 15,000 TB a month if each view sends the whole file. A budget drawn around the library would be out by many times in the first month the videos are watched as planned.

The watch count multiplies one line and leaves the other alone, because **bytes kept are paid for once a month and bytes sent every time someone watches**. If a gigabyte kept for a month costs $0.007 and a gigabyte sent $0.01, the two lines are about $350 and $150,000. Even with a gigabyte kept at ten times the price of a gigabyte sent, data sent is 30 times larger.

Estimate by views before terabytes, and shop for the data price first. Encodings sized to the screen cut the bytes per view, while a cheaper storage tier hardly moves this bill.

</details>

<details><summary>D. The servers behind the pages and catalog</summary>

✗ Servers lead the bill on most sites you have costed, where a page is small and a request is the work. A video is bytes a viewer pulls from the CDN, and pages and catalog costing even $10,000 a month would be a small part of a line built on 15,000 TB.

</details>

---

###### 5. Opening a laptop after a weekend on the phone

A messaging app numbers the messages coming in for each user and keeps one `last_delivered` number per user: when a device acknowledges a message the server moves the number, and a device that connects is sent whatever comes after it. A customer has the phone app and the desktop app. She shuts her laptop on Friday evening and reads her messages on her phone all weekend. On Monday morning she opens the laptop.

What does the laptop get when it connects?

<details><summary>A. Everything sent from Friday evening on</summary>

✗ The server keeps every message and has seen the laptop stay away since Friday. It keeps one number per user, though, and the phone has moved that number past the weekend, leaving no record of what the laptop last received.

</details>

<details><summary>B. The messages after the phone's last acknowledgment</summary>

✓ Right. The laptop gets only what comes after the phone's last acknowledgment. The weekend's messages never reach it, and her desktop chat jumps from Friday to Monday with no gap marked.

**One pointer per user records the furthest any device has got**, and the server treats it as what the connecting device holds. With one device the two were the same. A second device breaks the match with no error on either side, since each connect and each acknowledgment follows the rule as written.

Keep a pointer per device, decide where a new device starts, and keep messages on the server long enough for a device that has been away to catch up.

</details>

<details><summary>C. The weekend's messages, copied over from the phone</summary>

✗ Some messaging apps do pair a desktop with a phone and sync through it. In this design each device talks to the server alone, and the server never asks the phone for anything, so the weekend's messages have no way across.

</details>

<details><summary>D. Her whole history, from the first message</summary>

✗ A fresh consumer on some logs starts from the beginning, and the laptop has been gone for days. Its connection starts from the user's number and has no position of its own, and nothing sends it the start of her history.

</details>

---

###### 6. Getting five messages to a phone with no signal

A messaging app sends a notification through the phone platform's push service whenever a message arrives for a phone with no open connection to the server, with the message's text inside. A hiker's phone has no signal for two hours in a valley. Five friends write to her in that time, and the push service answers 200 to all five notifications. You are writing the code that runs when her phone is back.

How should the app get all five messages onto the phone?

<details><summary>A. Treat a notification as a wake-up and fetch from the server</summary>

✓ Right. The notification only wakes the app, which then asks the server for everything after the last message number it holds, and all five arrive in order whatever the push service kept. The hiker's chat is whole as soon as the phone has signal and the app has run.

A push service is a third party's store with its own rules, and **a 200 from it means only that it accepted the notification**. Some services replace a pending notification with a newer one that shares a collapse key, a tag that marks them as replaceable, and the web push standard says a service may expire an accepted message before its time. Your server holds the full list and the app knows its last number, which makes the fetch exact however many notifications survive.

Keep the text in a notification as a preview, count on nothing inside it, and have the app fetch on wake-up and on every open.

</details>

<details><summary>B. Count on the service to hold all five until the phone returns</summary>

✗ A 200 from a queueing service usually means the thing is stored, and this service does hold notifications for a phone that is away. It holds them on its own terms: if the five share a collapse key, a tag that marks them as replaceable, they become one, and a service may drop an accepted message early, so the app cannot know how many will come.

</details>

<details><summary>C. Resend each notification until the service reports delivery</summary>

✗ Retrying until acknowledged is how deliveries get through in the rest of your system. A phone with no signal has no answer to give, so no report of delivery comes and the loop has nothing to stop on, and its repeats can reach the phone as duplicates.

</details>

---

###### 7. Selling 2,000 seats to a crowd of 100,000

A ticket site puts 2,000 seats for a concert on sale at ten o'clock, with about 100,000 people waiting. Choosing a seat holds it for five minutes while the customer pays. Last year about six holds in ten ended unpaid, and a seat whose hold ends is taken again within seconds. In its load test the site took 5,000 holds a second.

What decides when the last seat is sold?

<details><summary>A. The size of the waiting crowd</summary>

✗ More buyers empty a shop faster. This crowd runs out of seats within a second: all 2,000 are held and the rest wait, and a bigger crowd gets a seat back no sooner than a hold ends.

</details>

<details><summary>B. How fast paying customers get through the payment form</summary>

✗ Payment is the slowest step in a booking and the first thing a capacity plan points at. A customer who pays frees nothing for anyone else. The last seats to sell are those whose holders walked away, and each of those seats sits for the full five minutes however fast the form is.

</details>

<details><summary>C. How many holds a second the site can take</summary>

✗ 5,000 holds a second is what the load test reached, and it is plenty: every seat is held in under a second. From then on the site is waiting for seats to come back, and a site ten times as fast would get them no sooner.

</details>

<details><summary>D. How long an unpaid hold lasts</summary>

✓ Right. The hold's five minutes decide it. About six holds in ten lapse, and each one idles its seat for the whole hold before the next person gets it, which leaves the last seat unsettled until over an hour after ten o'clock.

The sale runs in rounds, because **once every seat is held, a seat comes back only when an unpaid hold ends**. All 2,000 go in the first second, about 1,200 come back five minutes later, and each round shrinks by the unpaid share. If each hold ends unpaid at random, six times in ten, the last seat's final hold ends about 83 minutes in on average, and with a two-minute hold about 33.

Decide the trade: a shorter hold ends the sale sooner and gives slow payers less time. Show people how many seats are only held, so a map full of holds does not read as sold out.

</details>

---

###### 8. Moving a seat map from a cache to the primary

A ticket site draws its seat map from a cache refreshed every 30 seconds. Taking a seat is one update on the primary, which changes nothing if the seat has gone. At ten o'clock 2,000 seats go on sale to 200,000 people looking at the map, and last Friday most taps ended in "That seat has gone". The product manager blames the stale map, so for this Friday you draw the map from the primary, which has the room for it.

What happens to the failed taps this Friday?

<details><summary>A. They stop, with every map current</summary>

✗ Reading the primary is the textbook cure for a stale read, and the primary holds each seat's own record. The map is current as it leaves the primary, and then a person looks at it for some seconds, in which the seat has gone.

</details>

<details><summary>B. Most of the taps still end in that message</summary>

✓ Right. Most taps still end in "That seat has gone", about as many as last Friday, and the primary now also draws the map for 200,000 people.

A person takes seconds to choose, and **a map is out of date from the moment it is drawn**. At this sale seats go faster than a person chooses. Say 2,000 taps arrive each second for 20 seconds, each on a seat picked at random from those the map shows free, five seconds after the map was drawn. The first five seconds alone bring 10,000 taps onto 2,000 seats from maps that show every seat free, whichever store drew them, and the taps far outnumber those 2,000 seats, so roughly nine taps in ten fail either way. A fresh map helps in a slow sale, where a seat outlasts the person choosing.

Leave the map on the cache, and treat a lost tap as the normal case at an on-sale: offer the nearest free seat, or take people's choices from a queue.

</details>

<details><summary>C. They fall to a few, in the moments a map takes to load</summary>

✗ The cache's 30 seconds were the largest gap you could see, and the page load is the only delay left once they go. The gap that counts runs from drawing to tap and includes the person choosing, and even a few seconds of that is longer than a seat lasts at this sale.

</details>

---

###### 9. Keeping a seats-left counter in the booking transaction

An events site on a relational store with row locks, at default settings, sells general-admission tickets, so each show has one counter and no seat numbers. `db` and `payments` are thin clients, and the card provider answers in about two seconds. One buyer's request has reached the charge when the next buyer's request reaches the same show.

```python
with db.transaction() as tx:  # commits when the block ends
    tx.execute("UPDATE shows SET seats_left = seats_left - 1 "
               "WHERE id = %s AND seats_left > 0", [show_id])
    charge = payments.charge(card, price)
    tx.execute("INSERT INTO bookings (show_id, charge_id) VALUES (%s, %s)",
               [show_id, charge.id])
```

What happens to the next buyer's update?

<details><summary>A. It waits a few milliseconds for the first buyer's update</summary>

✗ The statement has returned and your code has moved on to the charge, and a statement that committed alone would have let go of its lock by then. Inside a transaction the lock on the row stays until commit, and commit comes after the provider answers.

</details>

<details><summary>B. It runs at once, on its own snapshot</summary>

✗ Snapshots let readers go past writers, as every account of them says. This is an update, and two updates of one row cannot go ahead together: the second waits for the first transaction to end.

</details>

<details><summary>C. It waits for the first buyer's whole card charge</summary>

✓ Right. The next buyer's update waits for the first buyer's whole card charge, about two seconds, and its own charge then makes the buyer behind it wait the same. A big show sells about one ticket every two seconds, one after another.

**The lock an update takes on a row stays until the transaction ends**, and the call to the provider runs before that, inside the lock. The provider's speed has become your sale rate, and a provider that stalls stalls every buyer of that show.

Take the seat and commit, charge outside any transaction, then confirm in a second one, with a job that gives back seats whose charge never finished.

</details>

<details><summary>D. It is refused, with the counter changed under it</summary>

✗ Schemes that check a version refuse a write when the value has changed, and this counter is about to change. A store with row locks makes the second update wait and then runs it on the new value, so the next buyer gets a seat, later.

</details>

---

###### 10. Reading the booking when the customer returns from the payment page

A cinema's site sends the customer to a payment provider's hosted page. When the card goes through, the provider sends a webhook to your server, which marks the booking confirmed, and the provider redirects the browser to your done page. The traces from staging show the webhook going out a moment before the redirect, and every staging run has shown a ticket. The done page shows the ticket for a confirmed booking and a Pay button for a booking still unpaid.

What does a customer see right after the card goes through?

<details><summary>A. A Pay button, now and then</summary>

✓ Right. Some customers reach the done page before your server has handled the webhook, and they see a Pay button for a card that has been charged, so some of them pay twice.

The redirect and the webhook are separate messages, and **two messages sent in order to two parts of a system can finish in either order**. The browser's request goes straight to a page, while the webhook has to reach your handler and be committed, and a delivery that fails waits for the provider's retry. Staging hides this, with an idle handler and a short network.

Let the webhook decide, and give the done page a third state, "confirming your payment", which checks the status for a while and then asks the provider directly. Have the Pay button refuse a booking that already has a payment under way.

</details>

<details><summary>B. The ticket, every time</summary>

✗ The traces show the webhook leaving first, and every staging run ended on a ticket. The browser's request and the webhook take separate paths to your servers, and on a busy night a webhook handler can sit in a queue while the redirect goes straight to a page.

</details>

<details><summary>C. The ticket, unless the webhook is lost for good</summary>

✗ Providers retry failed deliveries for days, to keep a webhook from being lost for good. The gap is any moment before your handler finishes, and a delivery that fails once leaves the customer on the done page until its retry lands.

</details>

---

###### 11. Storing driver positions for a ride-hailing app

A ride-hailing app has 400,000 drivers online at its peak, and each phone sends its position every four seconds. Only dispatch reads positions, and it asks where a driver is now. Trip records and receipts are stored elsewhere, from trip-start and trip-end events. The design document says "no data loss" at the top, for the whole system.

Where should the position updates go?

<details><summary>A. The relational primary, with a synchronous replica</summary>

✗ The system's important data lives on that primary, and the top of the document says no data loss. It would spend 100,000 writes a second on rows overwritten four seconds later, each waiting on a replica, to protect values that nobody reads after a crash.

</details>

<details><summary>B. A durable log, read by consumers that fill a store</summary>

✗ A log is the standard answer to a high write rate, and it would lose nothing. Dispatch wants the latest position alone, and it would read through a consumer and a store to get what the phone sends again in four seconds.

</details>

<details><summary>C. A wide-column table that keeps every position</summary>

✗ Wide-column stores are a usual pick for high write rates of time-stamped rows, and this one would take them. It keeps every position when nothing here reads an old one, and the table grows by 100,000 rows a second to answer a question about now.

</details>

<details><summary>D. An in-memory store holding one latest position per driver</summary>

✓ Right. An in-memory store with one entry per driver fits: 400,000 phones sending every four seconds is 100,000 writes a second, each replacing the last, and a crash costs dispatch about four seconds. The "no data loss" line still holds for what it was written for, the trip records and receipts.

**Durability pays when a lost write cannot come back**, and this one comes back by itself: every phone sends again within four seconds, and a wiped store is whole in that time with no restore. If a position takes 100 bytes, all the drivers come to about 40 MB, a figure that does not grow with time.

Set requirements per kind of data rather than once for a whole system. If someone later wants routes or heat maps, add a log for them beside this store, and keep dispatch on the latest position.

</details>

---

###### 12. Sending a lunchtime offer to every phone

A food-delivery app is sending a lunchtime offer as a push notification to all 5 million of its phones. The send job has its own queue and its own workers, well apart from the API, which it never calls, and it sends at the rate it was tested at, 5 million in a minute. Last week's offer went to 100,000 phones with no blip on any graph. The offer goes out during the API's usual lunchtime peak of 3,000 requests a second.

What does the API see after the offer goes out?

<details><summary>A. Its usual lunchtime peak, unchanged</summary>

✗ The job runs apart from the API, and last week's offer left the graphs flat. The job never calls the API, but the people it reaches do, and this send is 50 times last week's, which turns a bump too small to see into a climb past the usual peak.

</details>

<details><summary>B. A climb within minutes, on top of the peak</summary>

✓ Right. Within minutes of the send the API sees a climb on top of its lunchtime peak, from people opening the app.

The load comes after the send, because **a mass notification loads whatever people reach when they open it**, here an API the job never calls. If opening the app costs 15 requests and 5 percent of the phones open within ten minutes, that adds about 6,250 requests a second to a peak of 3,000. Last week's 100,000 phones at the same 5 percent added about 125, which is why the graphs stayed flat.

Send in stages over half an hour, and stop if the API's latency climbs. Per-customer limits do not help here, since each phone stays within its own.

</details>

<details><summary>C. A slow rise over the afternoon</summary>

✗ You read many notifications the next time you pick up your phone, and an offer is no emergency. A phone buzzes when the notification lands, and the send took a minute, so the opens that do come straight away all land in the same few minutes.

</details>

<details><summary>D. A burst from the send job's own calls</summary>

✗ A job that calls an API as it builds each message loads that API directly. This one has its own queue and workers and one offer for everyone, and the load comes afterwards, from the phones.

</details>

---

###### 13. Putting a short link in front of a long secret link

A document site shares files by link, and each link carries a 128-bit random token, which a security review passed. Customers say those links are too long for slides, so you add a short-link service: `docs.example/aB3xY9`, six random letters and digits, which redirects to the long link. Two years on it has issued two billion codes.

What now stands between a stranger and a shared document?

<details><summary>A. The 128-bit token in the long link</summary>

✗ The security review passed that token, and it is still in every long link. The short link hands it over: a live code redirects to its long link, token and all, and a stranger never has to guess 128 bits.

</details>

<details><summary>B. The short code and the token together</summary>

✗ Two secrets in a row are stronger than one when the second stays hidden behind the first. Here the redirect is open to anyone who has a live code, so the 128-bit token is never tested against a guess.

</details>

<details><summary>C. The six characters of the short code</summary>

✓ Right. The six characters are all that is left, since whoever holds a live code gets the long link, token included, from the redirect. A customer's document now rests on a code a stranger can land on by chance.

**Whatever a redirect points at is only as hidden as the code in front of it**, and six letters and digits give about 56.8 billion codes, about 36 bits. With two billion issued, about one code in 28 is live. A 2016 preprint, Gone in Six Characters, found that five- and six-character short links from cloud services could be found by scanning, and that shared files could be reached that way.

Give a link to private content a code as long as its token, or keep short links for public pages and ask for a login behind them. A limit on guesses per address helps and does not bring back the 128 bits.

</details>

<details><summary>D. Nothing: the document has become public</summary>

✗ A redirect shows its target to anyone who asks, and people often call shortened links public. A stranger still needs a live code, and these documents appear on no list, so the protection has shrunk to about 36 bits without going altogether.

</details>

---

###### 14. Reloading a signed file link after leaving a folder

A team workspace keeps its files in an object store. When a member opens a file, your server checks the folder's permissions and hands the browser a signed link to the object, valid for one hour, and the link becomes the tab's address. At 10:00 a contractor opens a contract this way. At 10:20 an admin removes the contractor from the folder, and the change commits. At 10:25 the contractor reloads the tab.

What does the contractor get from the 10:25 reload?

<details><summary>A. An access-denied page, after a check with your server</summary>

✗ Your server issued the link and owns the folders, and every other request in the workspace goes through it. This request does not: the browser talks to the object store directly, which is the point of the link, and the 10:20 change stays on your server.

</details>

<details><summary>B. A login page, with the session no longer allowed in</summary>

✗ A browser sends its session with each request, and that is how every other page of the workspace knows who is asking. That session cookie belongs to your site's address and is never sent to the store's, which has no idea who the contractor is.

</details>

<details><summary>C. An error from the store, with the link on a revoked list</summary>

✗ Revocation lists are how a token gets cut off early, and the removal has committed on your side. The store has no such list from you, since your folder table is yours and the store sees only objects and signatures.

</details>

<details><summary>D. The contract, as it opened at 10:00</summary>

✓ Right. The store checks the link's signature and whether its hour has run out, both pass, and it serves the contract five minutes after the admin removed the contractor. The link keeps working for another 35 minutes.

The store has no call to make, because **a signed link carries its own permission**: your server did the check at 10:00, and the signature records the result for the store to verify. What ends a link early is the signer losing its rights or its key being rotated, and either one ends every other link made with it as well.

Keep signed links short, a minute or two, and make a new one on each open after the permission check. Files that must stop at once go through your server, which costs the bandwidth the link was meant to save.

</details>

---

###### 15. Rebuilding lost receipt rows from a payment log

A consumer reads charge requests from a log with seven days of retention, calls a card provider with the message id as its idempotency key, and saves a receipt row. The provider keeps a key for 24 hours. A bad release on Monday morning loses the receipt rows written since Thursday, so you rewind the consumer's offset to Thursday to rebuild them.

What does the rewind do to customers' cards?

<details><summary>A. A new charge for each message older than a day</summary>

✓ Right. Each message older than a day charges its customer a second time, and the last day's messages are answered from their first results. If the rewind covers 96 hours of even traffic, with a 24-hour key, three quarters of the replay become new charges, on customers who bought nothing on Monday.

**An idempotency key protects you only for as long as the other side keeps it**, and this design keeps the log for seven days and the key for one. Past that day the provider sees a request it has never met and charges the card.

Keep your own record of what has been charged, for as long as the log keeps messages, and check it before calling the provider. For this rebuild, leave the consumer alone and rebuild the receipts from the provider's list of charges.

</details>

<details><summary>B. Nothing new, with each key answered from its first result</summary>

✗ A key is made for this, and every retry you have tested, seconds after the first try, got back its first result. A Thursday message replayed on Monday carries a key the provider dropped days ago, and it arrives as a new charge.

</details>

<details><summary>C. Nothing new, with old keys refused by the provider</summary>

✗ An API that no longer knows a token usually refuses it, as with an expired session. An idempotency key only marks a repeat: a key the provider does not hold is a first use, and the provider runs the charge.

</details>

<details><summary>D. A new charge for every message back to Thursday</summary>

✗ A rewind repeats the work, and documents describe a key as protection for the retry of one call. The key does work inside its window: messages from the last 24 hours are answered from their stored results, and the older ones alone charge again.

</details>

---

###### 16. Rebuilding a search index a week after an account is closed

A community site publishes a `profile_updated` event holding the profile's fields each time a profile changes, and a consumer copies those fields from that log into the people search's index, and results show what that index holds. The log keeps 90 days. On Monday a member changes her bio and then closes her account. The closing job deletes her row from the primary and her cache entry, and support checks that the profile page now says not found. The next Monday, a change in the index's format means rebuilding it, so you start an empty index and rewind the consumer to the log's start.

What does a search for her name return once the rebuild ends?

<details><summary>A. No result for her name</summary>

✗ Support saw not found on her profile page, and her row has gone from the store everything else comes from. The index is built from the log and looks at no row, and the log still holds her updates, which the rebuild writes back.

</details>

<details><summary>B. A result with her id alone, name and bio blank</summary>

✗ Search results are often filled in from the primary as they are shown, and the primary has nothing left for her. This index stores the name and bio from each event, which is what lets a search match her name at all, and the result comes back whole.

</details>

<details><summary>C. Her name and bio, as in the last event about her</summary>

✓ Right. The search returns her name and bio as of her last update, from an index built a week after she left, and the link in that result opens a page saying not found.

Her row went from the primary and her entry from the cache, because **a delete acts on the store it runs on**. The log still holds her updates, and for 83 more days any rebuild would restore her.

List every place a record's data goes, and have closing an account publish an event that each of them handles, so that a replay ends with her gone. Keep personal fields out of long-lived logs where you can, so that a replay has less to restore.

</details>

<details><summary>D. A stopped rebuild, with the consumer failing on an unknown member</summary>

✗ Consumers often check each event against the database, and a missing member would trip such a check. This consumer reads events and writes the index, as it has from the start, and its rebuild runs to the end with her events among the rest.

</details>

---

###### 17. Giving each customer two of eight cells

A document-conversion platform runs 1,000 customers on eight cells, each a small set of workers. Today a hash sends each customer to one cell, and a customer's job that crashes workers takes down that cell and everyone on it. The proposal gives each customer two cells at random, and its client sends to the second when the first fails. A colleague objects that a bad customer would now reach two cells rather than one. Then one customer's bad job does kill both its cells.

Which other customers lose all service?

<details><summary>A. The same eighth of customers as before</summary>

✗ An eighth is what each outage has cost so far, and one bad customer still reaches its share of the platform. A neighbour now has a second cell to go to, so sharing a cell with the bad customer no longer costs it service.

</details>

<details><summary>B. Only those assigned the same two cells</summary>

✓ Right. Only customers on the same two cells lose all service: eight cells make 28 pairs, so that is about one customer in 28, about 36 of the 1,000, against about 125 today. More customers do feel the bad job, as the colleague says: they lose one cell of their two and keep working.

**Customers are out only when every cell they hold is dead**, and giving each customer a random few of the cells, often called shuffle sharding, makes that rare with no extra machines.

The client must fail over for this to work, and the surviving cells need room for those who arrive. Limit retries of a job that keeps killing workers, so that it stops at two cells.

</details>

<details><summary>C. Every customer using either cell</summary>

✗ Under today's hash everyone on a dead cell is out, and the colleague's objection counts everyone who uses a dead cell. About 46 percent of customers do use one of those two cells, and all but those on the exact pair have a client that moves them to their other cell.

</details>

<details><summary>D. None: only the bad customer's jobs fail</summary>

✗ Failing over to another cell is what the proposal adds, and the job is only one customer's. That same failover carried the bad job to its second cell, and a customer assigned exactly that pair has nowhere left to go.

</details>

---

###### 18. One customer's read-only report on a shared database

A bookkeeping service keeps all its customers in one relational database, where an update leaves the row's old version in the table and each transaction reads from a snapshot of the data as it stood when the transaction began. At 9:00 one customer's accountant starts a year-end report: one read-only transaction over that customer's own rows, which runs for three hours. All morning, other customers keep updating their invoices, some of them many times an hour. CPU on the machine stays around 30 percent.

What should other customers expect during those three hours?

<details><summary>A. No change to their pages</summary>

✗ Readers do not block writers on a store like this, and the machine's CPU has room. The report's snapshot makes the store keep old versions of rows changed since 9:00, whoever's rows they are, and other customers' reads have to get past them.

</details>

<details><summary>B. At times a wait on the report's locks</summary>

✗ A long report on a shared database is a usual cause of lock waits, on stores that lock in order to read. A reader working from a snapshot holds no row locks that a writer waits on, and nobody's update queues behind this report.

</details>

<details><summary>C. Pages showing their data as of 9:00</summary>

✗ The report does see the data frozen at 9:00, and a snapshot is often described as a frozen database. Each transaction has its own snapshot, which leaves other customers seeing their invoices as of now, and only the report as of 9:00.

</details>

<details><summary>D. Their pages getting slower as the hours pass</summary>

✓ Right. Other customers' pages slow down as the morning goes on, with no lock waits, no failed queries and CPU still around 30 percent.

In a store with snapshots, **a changed row's old versions stay while any open snapshot might need them**, whoever's rows that snapshot reads. Cleanup stops at 9:00 for everyone and can start again only when the report ends. If a busy invoice row is updated once a minute, it has about 180 old versions after three hours. One kind of store keeps them in the table, where reads pass over them, and another keeps them in a separate store of changes, which a read can have to walk back through. How much it hurts depends on the rate of updates.

Run long reports in short transactions that each cover a part, or on a copy kept for reports, and set a limit on the age of a transaction on the shared primary.

</details>

---

###### 19. Setting the tenant on a connection behind a pooling proxy

A billing service keeps all its customers in one table, and a rule in the database shows a connection only the rows of the customer named in a setting on that connection. Your code sends two statements per request, outside any transaction block: one sets the customer, and the next reads the invoices. Last month, when the copies outgrew the database's connection limit, you put a pooling proxy in front. It lends one of its database connections for one transaction, and it passes the setting straight through. Tests with one user at a time all pass.

What does a request for one customer get back under load?

<details><summary>A. Another customer's invoices on some requests</summary>

✓ Right. On some requests the read runs on a connection last set for someone else, and the customer gets another customer's invoices, in a normal reply with no error.

The two statements of one request can land on two connections, because the proxy lends connections one transaction at a time and **a setting stays on the database connection that received it**. Take two requests at once: the first sets customer A on connection 1, the second sets customer B on connection 2, and then each read is lent the other's connection. Some proxies stop sharing a connection once they see such a setting, which keeps the rule right and gives up the pooling for that client.

Send the setting and the read in one transaction, with a setting that lasts only for that transaction, and test through the proxy with two customers at once.

</details>

<details><summary>B. That customer's invoices, as in the tests</summary>

✗ The setting is sent right before the read, and with one user at a time the tests find it on the same connection every time. Under load the read is a new transaction, the proxy lends whichever connection is free, and that connection's setting came from whichever request used it last.

</details>

<details><summary>C. An error about an unset setting</summary>

✗ Software that needs a missing value usually fails loudly, and an unset value does that under some rules. Here most connections hold a value left by an earlier request: it is the wrong one, and the query runs normally.

</details>

<details><summary>D. Every customer's invoices in one reply</summary>

✗ A filter with nothing to compare passes everything in plenty of query builders. This rule sits in the database and compares each row with whatever the connection holds, so a request sees one customer's invoices, which may be the wrong customer's.

</details>

---

###### 20. A finance batch joining your API's machine at month end

A bank's API copy runs on a 16-core machine it shares with batch jobs. It reserves 2 cores and has no cap on the cores it may use, and at midday all month it uses about 3.5 cores, from cores nobody else wants, with fine latency. At month end a finance batch that reserves the other 14 cores starts on that machine and will use every core it gets. The API's traffic is the same as any other midday.

What happens to the API at midday on month end?

<details><summary>A. It keeps its 3.5 cores, and the batch takes the rest</summary>

✗ The API has been there all month on a flat graph, and it serves customers while the batch can wait. CPU sharing knows neither fact: it goes by the weights the reservations set, 2 against 14, and with both wanting every core the API gets an eighth of the machine.

</details>

<details><summary>B. It is held to about its 2 reserved cores, and requests queue</summary>

✓ Right. The API is held to about its 2 reserved cores while it needs 3.5. It gets through a little over half of each second's work, so requests queue and latency climbs for as long as the batch runs.

With idle cores around, a copy uses all it can, which is how the API reached 3.5 cores on a reservation of 2. **When cores run short, each reservation acts as a weight**, and with the batch wanting every core the split goes 2 against 14.

Reserve what the service needs at its peak, and read CPU use beside the reservation, since anything above it is borrowed. Keep batch work on machines of its own, or give it a lower weight where the two have to share.

</details>

<details><summary>C. It is evicted and restarted on another machine</summary>

✗ A machine short of memory evicts what is over its reservation, and this copy is over its reservation. CPU can be shared out moment by moment with nothing having to leave: one widely used orchestrator evicts on memory, disk and process counts, with CPU not among them, and the API stays put on fewer cores.

</details>

<details><summary>D. It gets half the machine, with the batch taking the other half</summary>

✗ Two busy programs on a laptop share it evenly, and both of these are busy. A laptop's programs have equal weights, and these two have weights of 2 and 14, so the split follows them: 2 cores and 14.

</details>

---

###### 21. Choosing an alarm for a bad release under a scaler

A site that sells second-hand clothes runs on a public cloud and takes 50 photo uploads a second. The upload request answers as soon as the photo is stored and a message is on the queue, and workers resize the photo from there. A message that fails comes back after 30 seconds, as many times as it takes, and each failure sends the whole message and its stack trace to the cloud's log service. A scaler adds and removes worker copies, with a target of 60 percent mean CPU and no maximum on copies. A design review takes one case: a release on a Friday evening makes the workers fail every time on one photo in 200, and the on-call engineer should hear of it that same evening.

Which of these should the on-call engineer's alarm watch?

<details><summary>A. Mean CPU across the workers</summary>

✗ CPU is the graph you open when workers are in trouble, and a retry loop burns CPU on work that never finishes. The scaler meets each step of that load with more copies until the mean is back at its target, while it can still add them. The line reads about 60 percent on Friday evening and again on Monday, and an alarm on it stays quiet while the copies climb.

</details>

<details><summary>B. Errors and latency on upload requests</summary>

✗ Errors and latency are the first lines on most on-call rotas, and they measure what customers get. The upload request answers before any worker touches the photo, so both stay flat however many messages keep failing. A service that stays up for every customer can still pay many times its usual bill to run the loop.

</details>

<details><summary>C. Worker-hours and log gigabytes per upload</summary>

✓ Right. Both ratios climb within the first hour of the bad release, while uploads stay quick and CPU holds at 60 percent, so the engineer hears of it that evening.

Clouds bill worker time by the second and logs by the gigabyte, whether or not a customer got anything for them. A scaler answers extra work with extra copies, and **with no maximum it holds CPU and latency flat while the bill grows**. The failing photos never leave the queue, so every hour adds more messages to retry. Say a photo takes 2 seconds of a one-core copy, a failure logs 20 KB, and list prices run about 4 cents a core-hour and 50 cents a gigabyte of logs. After one hour the copies have gone from 167 to 267, and after sixty hours to over 6,000, which adds about $9,000 to about $400 of usual workers. Dividing by uploads keeps a busy Saturday from paging anyone, since real traffic raises usage and uploads together.

Alarm on both ratios against last week's, taking them from the scaler's count of copies and the log service's own metric rather than from the bill. An alarm on failed deliveries also catches this bug, and the ratios also catch a runaway in which no delivery fails. Then give the surge a limit to meet: a maximum on deliveries, after which a failing message goes to a queue of its own, and a maximum on copies well above your busiest sale, so that a runaway reaches CPU and latency as well as the bill. Keep the budget alert as a backstop.

</details>

<details><summary>D. Month-to-date spend against the budget</summary>

✗ Clouds offer a budget alert for exactly this kind of runaway. The figures behind it arrive hours late, up to about a day on some clouds, and the alert stops nothing by itself. The loop also starts small: if a failure costs 2 seconds of a core and logs 20 KB, at about 4 cents a core-hour and 50 cents a gigabyte, six hours in it has added about $90, far below a threshold set for a month's spend.

</details>

---

## Further reading

- [Feeding Frenzy](https://doi.org/10.1145/1807167.1807257), 2010: choosing push or pull for each pair of producer and consumer, by how often one posts and the other reads.
- [Gone in Six Characters](https://arxiv.org/abs/1604.02734), 2016: a preprint showing that five- and six-character short links from cloud services can be found by scanning.
- [Performance Isolation and Fairness for Multi-Tenant Cloud Storage](https://www.usenix.org/conference/osdi12/technical-sessions/presentation/shue), 2012: giving each tenant its fair share of a shared store across many machines.
- [Retro](https://www.usenix.org/conference/nsdi15/technical-sessions/presentation/mace), 2015: tracking each tenant's use of resources across systems, and throttling heavy jobs at set points.

Prev: [Reliability and disaster recovery](../reliability-and-disaster-recovery/README.md)
