# Caching

Prev: [Sharding and partitioning](../sharding-and-partitioning/README.md) · Next: [Choosing a data store](../choosing-a-data-store/README.md) · Reviewed October 2026

A cache keeps a copy of data from a slower store, so that a later read of the same key can skip the store. Four things decide how one behaves: how the copy gets in, how it leaves, how many copies exist, and how much of the store's load it has come to carry.

- **Store**: the database or service that holds the data of record, behind the cache.
- **Replica**: a copy of a cache node or a database that applies the original's changes, a little later.
- **Hit, miss and hit rate**: a hit finds the key in the cache; a miss does not, and pays for the lookup and then the store read; the hit rate is the share of reads that hit, and it counts a cheap read and a costly one alike.
- **Fill**: storing in the cache the value a miss has read from the store.
- **TTL** (time to live): how long an entry may stay before the cache drops it. Some caches can restart the clock on each read, and a key read steadily then never reloads.
- **Eviction**: the cache removing entries to make room when its memory is full; a common rule, least recently used, removes the entry read longest ago.
- **Invalidation**: your code removing an entry because the data behind it changed, most often by deleting its key.
- **Stampede**: many readers refilling one missing key at the same moment: every request arriving during the fill misses too, so with no coalescing the queries are the request rate times the fill time.
- **Request coalescing**: letting requests that miss on one key wait for a single store read and share its result, often called single-flight.
- **Serving stale**: a feature of some caches, which keep an entry past a first TTL and answer with it while a refresh is tried, up to a second, longer TTL.
- **Negative caching**: storing a marker that says a key has no row, so that the next lookup need not ask the store; a row added meanwhile stays hidden until the marker ends or a write deletes it.
- **Flush**: in write-behind, the cache writing its held changes to the store; on a cache, a flush empties every entry.
- **Warming**: filling a cache before it takes traffic; a standby region idle until a failover, and each node swapped in for a repair, start with none of it.

Unless a question says otherwise, an entry here has a TTL counted from its fill and leaves when it ends, and `cache.set` writes with no comparison.

A local cache lives inside one process of your service, so a read crosses no network and every process holds its own copy. A shared cache is a service on the network that all the processes use, so a read costs a network hop and there is one copy for all of them. A first read of a key gains nothing from either, and neither does a value that changes before it is read again, unless an old copy will do.

Four arrangements connect a cache to a store:

| Arrangement | On a read | On a write |
| --- | --- | --- |
| Cache-aside | your code checks the cache and, on a miss, reads the store and fills the cache | your code writes the store and deletes the key |
| Read-through | your code asks only the caching layer, which reads the store on a miss and fills itself | no write path of its own: the layer may pass writes on, or your code writes the store itself |
| Write-through | reads find what the last write left in the cache | the write goes to the cache and the store before the caller hears back, and pays for both |
| Write-behind | reads find what the last write left in the cache; a miss that evicts an unflushed entry writes it to the store first | the cache acknowledges the write and flushes it to the store later |

Filling on write has costs of its own: a new node holds only the keys written since it joined, and rows nobody reads push out rows people do.

The trouble seldom shows on one machine under light traffic. It comes with readers that overlap, several processes, several copies of the data and a store that has grown used to the cache.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- put a cache in front of a store and say what it does for page times, for how old a value can get and for data that lives nowhere else
- follow a changed row through deletes, transactions, replicas, write paths, processes and key names, and say what the next reader gets
- plan for the moments a cache slows, expires or goes empty, from one popular key at peak to a stalled partner, a flush, a nightly job and codes that do not exist

###### 1. Adding a cache in front of a legal document archive

A legal archive holds decades of rulings in a slow store. Readers open old rulings almost at random, and no ruling draws more reads than the others. You put a cache in front of the store. It holds a small share of the rulings, and a lookup takes a small fraction of a store read. The archive's owner asks what to expect.

Once the cache has filled, what do you tell the owner about page times?

<details><summary>A. Pages load far faster, with few reads reaching the store</summary>

✗ Every diagram of a cache draws one slow first read and a row of fast ones after it. Those fast reads need the same ruling to be asked for again while the cache still holds it, and reads here fall almost at random across a far larger archive, so nearly every read finds its ruling absent and goes on to the store.

</details>

<details><summary>B. Page time holds still while the store's reads drop by about half</summary>

✗ Capacity plans count a cache first as load taken off the store, before any gain in speed. The reads it takes off follow the share it holds: half the rulings would spare the store about half its reads, and this cache holds a small share.

</details>

<details><summary>C. Page time keeps falling week by week as the cache learns what readers want</summary>

✗ Where readers have favourites, a cache settles on the items they keep asking for, and its hit rate climbs as it does. No ruling here draws more reads than another, so there is nothing to settle on. If the cache holds 2% of the rulings, its hit rate is about 2% in the second tenth of a long run and about 2% in the last.

</details>

<details><summary>D. Page time barely moves and the store sees nearly the same reads</summary>

✓ Right. Pages load about as fast as before, and nearly every read still reaches the store. The owner gets little from this cache.

**A hit needs a repeat read of a key still in the cache**, and with reads spread evenly over far more rulings than the cache holds, the hit rate settles at the share it holds. If the cache holds 2% of the rulings, a store read takes 40 ms and a lookup 1 ms, a read averages about 40.2 ms against 40 ms with no cache, since every miss pays for the lookup on top of the store read.

Before you add a cache, count how often the same key is read twice within the time an entry would stay. In an archive read like this one, work on the store's own read path instead.

</details>

---

###### 2. Putting a one-minute local cache in front of a five-minute shared cache

On a travel site, the fare service keeps a local copy of each fare in every process, above a shared cache that all the processes fill from. A process that misses reads the shared cache and stores what it gets for one minute, and a shared miss reads the store and stores the fare for five. Nothing invalidates either cache. Support asks how long a customer can see a fare after it changes in the store.

What limit do you give support?

<details><summary>A. One minute, the local cache's TTL</summary>

✗ The local copy is what your code reads, and it never lives past a minute. The minute starts when the copy is stored, and the value it stores can already be nearly five minutes old by then, so the customer sees a fare far older than one minute.

</details>

<details><summary>B. Five minutes, the shared cache's TTL</summary>

✗ Five minutes is the longest TTL anywhere in the stack, and the shared cache holds what every process reads. The local copy starts its own clock on whatever the shared entry holds, even in that entry's last second, so the old fare outlasts the five minutes.

</details>

<details><summary>C. Longer than either of the two TTLs</summary>

✓ Right. You give support a limit of about six minutes, the two TTLs added together, because one process can show the old fare for that long after it changes.

**A TTL limits how long one entry stays, and the value inside may already be old when the entry begins**: a shared entry filled with the old fare right before the change serves it for up to five minutes, a process can copy it in the last moments of that time, and its local copy starts a fresh minute. The worst case comes to a little under six minutes, past both TTLs.

Store the shared entry's expiry with the value and end the local copy at that moment, which brings the worst case back to five minutes, or quote the sum.

</details>

---

###### 3. Login sessions and a promotion

A streaming app keeps its login sessions in a cache and nowhere else, with half the memory free and most of the rest held by sessions. The cache has a replica. When memory runs out, the cache evicts the entry read longest ago. A session is read only when its viewer comes back, while page entries are read constantly. A promotion then adds page entries, half as much again as the free memory.

After the promotion, what happens to viewers who were logged in?

<details><summary>A. They stay logged in through the replica's copy</summary>

✗ A replica exists so that data survives the loss of a node, and this one holds every session the first node holds. The replica applies every change as well, and an eviction is a change: the first node sends the replica a delete, so an evicted session leaves both nodes.

</details>

<details><summary>B. They get a server error on the next page</summary>

✗ A cache set never to evict can refuse new writes with an error once its memory is full. This one evicts, and a lookup for an evicted session returns a miss like any absent key, so the app sends that viewer to the login page.

</details>

<details><summary>C. Some are logged out, with no error in any log</summary>

✓ Right. Viewers whose sessions were read longest ago come back to a login page, the others carry on, and no log shows an error, since nothing failed.

Page entries are read constantly and sessions are not, so the sessions go first. **Eviction cannot tell an entry the store can reload from one that lives nowhere else**: a miss on a page entry reloads it, and a miss on a session has nothing to reload from. If the cache has room for 2,000 entries, 1,000 of them free, and holds 800 sessions, the promotion adds 1,500, which is 500 more than the free room. The 500 sessions read longest ago leave both nodes, and 300 remain.

Keep sessions in a store with a copy that survives, or in a cache kept apart from page entries, with eviction off and its memory watched, so that no promotion can push a session out.

</details>

<details><summary>D. All of them are logged out at once as the memory fills</summary>

✗ A process that runs out of memory can be killed by the operating system, and it takes everything with it. Eviction exists to prevent that: the cache removes only what it needs to make room, starting from the oldest read, so viewers who came back recently keep their sessions.

</details>

---

###### 4. Deleting the cache key after updating the row

A marketplace uses cache-aside: on a change, your code updates the row and then deletes the key. A seller changes a price from 10 to 12 while a buyer's read is under way, with the buyer's server paused in the middle of it.

```text
t0  buyer's read misses and loads the row: price 10
t1  seller's update commits: price 12
t2  seller's code deletes the key
t3  buyer's server resumes
```

What does the cache hold for that key once the buyer's read has finished?

<details><summary>A. Nothing, until a read loads the new price</summary>

✗ The recipe ends at the delete, and the key is empty right after it. The buyer's read has not finished at that point: its set comes after the delete at t3, and a set on an empty key is an ordinary set.

</details>

<details><summary>B. The old price, with a full TTL ahead of it</summary>

✓ Right. The key holds 10 and the row holds 12, with the entry's whole TTL still to run. Buyers keep seeing the old price, and the seller's change appears lost.

A delete removes only what is there when it runs, and in cache-aside **a reader that loaded the row before the update can store it after the delete**. A plain set cannot tell that what it carries is out of date, and the TTL's clock starts at the set.

Keep a TTL on every entry, since it is what ends this case. Where the cache offers it, hand out a token on each miss: the delete voids the token, the cache refuses the late set, and the next read loads 12.

</details>

<details><summary>C. The old price for good, on every later read</summary>

✗ The seller's delete has already run, and no later step in this recipe touches the key, so every read from here on hits it and never reaches the row. The entry still has its TTL: once that ends the key leaves, and the next read loads 12.

</details>

<details><summary>D. The new price, loaded right after the delete</summary>

✗ The miss path is the repair in this recipe: after a delete, the next read goes to the row. The only read in flight started before the update, so it carries 10, and its set at t3 puts 10 in the key, so the reads after it hit that entry.

</details>

---

###### 5. Deleting the key in an order update

In an order service, a handler updates a row and deletes the key. The store runs at read committed isolation, and its readers do not wait behind an open write. Right after the delete, before the handler has returned, another request reads the same order, misses and fills the key.

```python
@transactional
def update_order(order_id, total):
    db.execute("UPDATE orders SET total = %s WHERE id = %s", total, order_id)
    cache.delete(f"order:{order_id}")
```

Once the commit has landed, what does the cache hold for that order?

<details><summary>A. The old order, for the entry's whole TTL</summary>

✓ Right. The cache holds the order as it stood before the update, with a full TTL to run, while the row holds the new total. Anyone opening the order sees the old one.

`@transactional` runs the handler in one database transaction and commits after the handler returns, so the delete runs while the update is still invisible to everyone else. **A delete protects readers only once they can see the new row**, and here the reader misses, loads the old row, the only committed one, and stores it, and the commit lands behind it with no effect on the cache.

Delete in an after-commit hook, which runs once the commit has landed, and keep a TTL for the races that remain.

</details>

<details><summary>B. The new order, loaded by that read</summary>

✗ The `UPDATE` line has run by then, and the handler could read the new total back on the next line. The total belongs to the open transaction. At read committed, a read from another session sees only committed rows, so this one loads the old order and stores it.

</details>

<details><summary>C. Nothing, until a later read loads the new order</summary>

✗ The delete comes after the write in this code, line by line. The commit is what makes the write visible, and it comes after the handler. The other request's fill lands in the gap and survives, since no delete comes after it.

</details>

<details><summary>D. The new order, put there when the commit lands</summary>

✗ An object mapper's own cache is refreshed when a transaction commits, and a framework runs this commit. The cache here is a separate service, and the commit goes to the database alone: the only calls that touch the cache are the handler's delete and the reader's fill, and the reader's fill came last.

</details>

---

###### 6. Writing the new price into the cache on update

To avoid a miss after each change, a shop writes the new price straight into the cache instead of deleting the key. Your code updates the row, then calls `cache.set`. Two admin servers change one price at almost the same moment, on one cache node.

```text
t0  first admin's server writes 10 to the row, then pauses
t1  second admin's server writes 20 to the row
t2  second admin's server sets the cache to 20
t3  first admin's server resumes
```

Which price does the cache return from then on?

<details><summary>A. The first admin's price, while the row holds the second's</summary>

✓ Right. Reads return 10 and the row holds 20: customers see a price that the store has already replaced, and they go on seeing it until the entry's TTL ends.

Each server makes two separate writes to two systems, and **nothing makes the cache see two writers in the order the store did**. The first admin's set lands last, and no later miss repairs it, because the key is present. Had both servers deleted the key, no order of the two servers' steps could leave the old price, since a delete carries no value that could arrive late.

Delete the key on update and let the next read fill it. If a miss after each change costs too much, use a conditional write that carries a version and refuses an older value.

</details>

<details><summary>B. The second admin's price, the same as the row</summary>

✗ The second admin's write is the last one the store accepted, and a read of the row now returns 20. The cache has no idea of the store's order: it takes sets in the order they arrive, and the set of 10 arrives last.

</details>

<details><summary>C. The first admin's price on some reads and the second's on others</summary>

✗ Replicas can disagree for a while, after a pair of close writes for instance. One cache node holds this price, with one value per key. The later set replaced the earlier one, so every read gets 10.

</details>

<details><summary>D. No price, until a read fills the key</summary>

✗ A cache that hands out tokens on a miss refuses a write that has lost a race, and the key stays empty until the next read fills it. A plain set never refuses: each one replaces the value, so the key is never empty.

</details>

---

###### 7. Adding to a balance read from the cache

A loyalty programme gives a bonus by reading a customer's points through the cache, adding to them and writing the sum to the database. Since the entry was filled, a purchase has taken the row to 100, while the entry still says 80. A bonus of 10 then comes in, and no other write touches this row.

```python
def add_purchase(customer_id, points):
    db.execute("UPDATE accounts SET points = points + %s WHERE id = %s", points, customer_id)

def add_bonus(customer_id, bonus):
    points = cache.get(f"points:{customer_id}")
    if points is None:
        points = db.query("SELECT points FROM accounts WHERE id = %s", customer_id)
    db.execute("UPDATE accounts SET points = %s WHERE id = %s", points + bonus, customer_id)
```

Once the old entry has expired, how many points does the database hold for that customer?

<details><summary>A. The right total, with nothing lost</summary>

✗ The entry is stale, and a TTL is there to end a stale entry. Expiry does end this one, and the next read reloads the row, which now holds 90. The expiry removes the old copy and has no way to bring back what the write replaced.

</details>

<details><summary>B. A total with the bonus counted twice</summary>

✗ Double counting is a common fault of a retried write, and this code has both a cache and a store feeding one total. The function runs once and takes its base from one place only, the entry. The bonus arrives once, on a base that was already too low.

</details>

<details><summary>C. The total from before the bonus, and the write gets an error</summary>

✗ With optimistic locking, a write that started from an old version is refused, and the base here is old. The `UPDATE` here carries no version check, so to the database it is an ordinary update of one row, which succeeds and stores 90.

</details>

<details><summary>D. A total missing the purchase's points</summary>

✓ Right. The row ends at 90 where 110 is correct: the customer has lost the 20 points the purchase gave, and once the entry expires, the next read loads that 90 as if it were true.

A stale entry shown on a page ends with its TTL, since the row still holds the truth. Here **a sum computed from a stale value overwrites the row**, and neither expiry nor invalidation can restore what the row held.

Do the sum in the store, as `add_purchase` does: `UPDATE accounts SET points = points + 10` ends at 110 whatever the cache holds. Anywhere a value feeds a write, read it from the store.

</details>

---

###### 8. Filling the cache from a read replica

A booking site writes to the primary database, deletes the key right after the commit and fills misses from a replica. Today the replica is two seconds behind. Only your code writes to the cache.

```text
t0  guest's new dates commit on the primary, and the code deletes the key
t1  guest reloads the page: a miss, and the read goes to the replica
t2  the replica applies the change
```

Which dates does the guest see from that reload on?

<details><summary>A. The new dates from that reload on</summary>

✗ This is the cache-aside recipe, and on one database the next miss after the delete loads the committed row. The miss here reads the replica, which at t1 has not applied the commit, so the row it returns is the old one.

</details>

<details><summary>B. The old dates, for as long as the entry lasts</summary>

✓ Right. The guest who has only now changed the dates keeps seeing the old ones, because the reload stored the old booking and nothing replaces it before its TTL ends.

The delete ran on time for the primary, and the reload read the replica, which did not have the change yet. **A fill stores whatever its read source returns at that moment**, and the replica catching up at t2 changes nothing, because nothing tells the cache.

Send the delete once the replica has applied the write, in each region if every region has its own cache and replica, or send reads for a recently changed key to the primary.

</details>

<details><summary>C. The new dates, once the replica catches up and tells the cache</summary>

✗ The replica does learn of the change at t2, and it is where this cache fills from. Replication sends the change to the replica and stops there. Only your code writes to the cache, and it ran its delete at t0, so the replica catching up reaches no entry.

</details>

<details><summary>D. The old dates on that one reload, then the new ones</summary>

✗ Replica lag on its own costs one stale read, which the next read corrects. The stale read here was stored, so the next reload hits the entry and never reaches the replica, however far the replica has caught up.

</details>

---

###### 9. Writing to the cache and the store in one request

A clinic's appointment service does write-through in its own code: it writes the cache, then the database, then returns. Entries carry no TTL. A receptionist moves an appointment from 09:00 to 10:30, the database refuses the write, and the receptionist gets an error.

```python
def move_appointment(appointment_id, new_time):
    cache.set(f"appointment:{appointment_id}", new_time)
    db.execute("UPDATE appointments SET time = %s WHERE id = %s", new_time, appointment_id)
```

What do later readers see for that appointment?

<details><summary>A. 10:30, served from the cache</summary>

✓ Right. Readers get 10:30, a time the database rejected, while the row still says 09:00. The receptionist was told the move failed, and the schedule shows the new time anyway.

Write-through here means two writes to two systems, and **when the second write fails, the first one stays**. With no TTL nothing ends it: the cache still says 10:30 a day later.

Write the store first and touch the cache only on success, with a delete or a conditional write where two writers can cross, and keep a TTL on the entry for the crash between the two.

</details>

<details><summary>B. 09:00, with the cache write undone</summary>

✗ A database transaction that fails leaves nothing behind, and "write-through" names one path through two systems. The function makes two separate calls: the failed `db.execute` ends it, and no line in it touches the cache again, so 10:30 stays there.

</details>

<details><summary>C. 09:00, loaded after a miss</summary>

✗ The database refused the write and the caller got an error, so the failure is known somewhere. The cache is not told: no line clears it when `db.execute` fails. The key is present, so every read hits it and none reaches the database.

</details>

<details><summary>D. 10:30 at first, then 09:00</summary>

✗ A wrong value in a cache usually ends at its TTL. No TTL is set on these entries and nothing reconciles the two systems, so nothing replaces the 10:30.

</details>

---

###### 10. Running a ticket counter on write-behind

A ticket counter runs write-behind: each sale increments a counter in the cache, the buyer gets a confirmation at once, and the cache flushes to the database, which the sales report reads. In testing, every read after a write returned the right total. During a busy sale the cache process crashes and restarts a second later.

```text
flush_interval: 10 seconds
persistence: none
```

What does the sales report show once the cache is back?

<details><summary>A. Every sale up to the last flush, with that flush counted twice</summary>

✗ A delivery that breaks off is usually sent again after a restart, and the crash can cut a flush short. With `persistence: none` the restarted cache has no record of what it flushed or still owed, so it has nothing to send again, and each flush that reached the database stands once.

</details>

<details><summary>B. Every sale up to the last flush, and none after it</summary>

✓ Right. The report stops at the last flush, and the buyers since then hold confirmations for tickets the database has no record of.

From the acknowledgement until the flush, **a write-behind cache with persistence off holds the only copy of the write**, and a read that returns it proves only that the cache has it. With `persistence: none` a crash takes everything since the last flush: say one sale a second and a flush at t20, and a crash at t27 leaves 20 sales where 27 were confirmed.

Keep write-behind for numbers you can afford to lose. For sales, write to a durable log before you confirm, or flush before you answer the buyer.

</details>

<details><summary>C. Every confirmed sale, the last one included</summary>

✗ The tests read back every total correctly, and on a database a read proves the write is stored. The test reads came from the cache's memory, and the database at that point held only the last flush, so the report has nothing later to show.

</details>

<details><summary>D. Most sales, with only the very last increment gone</summary>

✗ A database's write-ahead log keeps everything except a small unsynced end, and a crash there costs very little. With `persistence: none` the cache keeps no log, so the loss runs back to the last flush, up to ten seconds of sales.

</details>

---

###### 11. Holding a price in ten processes

A shop runs ten servers behind a balancer that spreads each request across them, and each server keeps a local cache with a one-minute TTL. A manager updates a price. The server that handles the update deletes the key in its own cache and does nothing else. On your laptop, with one process, the new price showed at once.

What does a customer refreshing the product page see in the next minute?

<details><summary>A. The new price from the first refresh on</summary>

✗ The laptop test passed, and in a shared cache one delete does clear the only copy. Each of the ten processes holds its own, and the delete reached one of them, so nine still answer with the old price.

</details>

<details><summary>B. The old price on every refresh until the minute is up</summary>

✗ The TTL is one number in the config, shared by all ten servers. Each counts from its own fill, so the copies end at different moments, and the one server that deleted its copy has the new price right away.

</details>

<details><summary>C. The old price for as long as the refreshing goes on</summary>

✗ This customer keeps reading the page, and the reads keep landing on old copies. A read does not extend a copy: each counts from its fill, so all the old copies are gone within the minute, however many refreshes land on them.

</details>

<details><summary>D. The new price and the old price, by turns</summary>

✓ Right. The customer sees the new price, then the old one, then the new one, flipping as the balancer picks servers, and more refreshes turn new as the minute passes.

**A delete in a local cache clears one process's copy**, and the balancer sends each refresh to any of the ten. The other nine keep the old price until their copies end, each at its own moment within the minute.

Use a shared cache for values that must change together, or send each delete to every process and have each process clear its copies when it reconnects, since one that was disconnected never gets the message.

</details>

---

###### 12. Caching a report lookup behind a decorator

A software firm runs one reporting service for many tenants, its customer companies, on one shared cache, and the route checks that the caller is an admin in their own tenant. The first tenant's admin has opened a month's report, and a second tenant's admin now opens the same report for that month.

```python
@cached(key="report:{report_name}:{month}", ttl=LIFETIME)
def monthly_report(report_name, month):
    tenant = request_context.tenant
    return db.query(REPORT_SQL, report_name, month, tenant)
```

What does the second admin see?

<details><summary>A. Their own figures, from a fresh query</summary>

✗ The query filters by tenant, so each run of it returns one tenant's rows, and this time it never runs: `@cached` finds the first admin's entry under the report name and the month, and returns it before the function body starts.

</details>

<details><summary>B. The first tenant's figures</summary>

✓ Right. The second admin reads another company's figures on their own screen, with the access check passed and the query never run: one customer's numbers have leaked to another.

**A key has to name every input that changes the answer**, and this one names the report and the month, while the tenant reaches the function through the request context. Two tenants' answers share one key, and the first one stored goes to both.

Put the tenant in the key, and do the same for any other input read from context, such as locale or currency.

</details>

<details><summary>C. A permission error from the service</summary>

✗ An access check guards this route, and it would turn away a stranger. The check asks only whether the caller is an admin in their own tenant, which this caller is. The check looks at no data, and the data arrives from the cache afterwards.

</details>

<details><summary>D. An empty report for that month</summary>

✗ Row-level security in a database filters rows by tenant whatever the query asks for, and an empty result is its usual sign. A filter of that kind runs inside the database, and this read stops at the cache. The cached report arrives whole, with nothing between it and the screen to remove rows.

</details>

---

###### 13. Caching whole query results

A real-estate portal caches the result set of each search for an hour, under a hash of the query. When an agent changes a price, your code updates the row and deletes the listing's key, and in testing the listing's page showed the new price on its next load. This morning an agent lowers a price, and minutes later a buyer runs the same search another buyer ran shortly before the change, with that listing among the results.

What price does that buyer see beside the listing?

<details><summary>A. The new price, inside the same cached result</summary>

✗ Some client-side caches keep each object once, and a stored query result points at that object, so an update shows in every result that holds it. A result set stored whole has no such pointer. The price sits in the listing's row inside the entry, and a hit returns those rows without a look at the listing's key, so the buyer gets the price from before the change.

</details>

<details><summary>B. The new price, from a search that runs again</summary>

✗ A database with a query cache of its own clears the stored results whenever a table behind them changes, so the next run reads the rows again. This cache sits outside the database and hears only your code's calls. The row's update never reaches it, and the one delete it gets names the listing's key, so the search's entry stays in place and answers the buyer.

</details>

<details><summary>C. The old price until that result expires</summary>

✓ Right. The buyer sees the old price in the results while the listing's page shows the new one. The portal shows two prices for one listing until that search's entry expires, within the hour.

**Invalidation can reach only entries whose key your code can name**, and a search's key is a hash of its query, which does not name the rows inside it. After your delete, every search that holds this listing still serves the old price. The test looked only at the listing's page, which reads the key your code deleted.

Cache a search as a list of listing ids and fetch each listing through its own key, so one delete reaches every search that shows it. Which listings a search holds can still be up to an hour old.

</details>

<details><summary>D. The old price this once, then the new one on a repeat</summary>

✗ Serving stale hands out an old entry once while a refresh runs, and the reader after that gets the new value. That feature starts when an entry passes its first TTL, and this one has most of its hour left, so the read is a plain hit that starts no refresh. The next repeat of the search gets the old price too.

</details>

---

###### 14. Changing the shape of a cached value during a rolling deploy

A bank's account service uses cache-aside and renames a field in the object it caches under one key, and the code reads that object with a strict parser. Servers move to the new code a few at a time over twenty minutes. The database needs no migration.

During the rollout, what do requests read on the account key?

<details><summary>A. Errors on every server, for the whole rollout</summary>

✗ Two shapes meet on one key, and a strict parser rejects the one it does not know. The key holds one shape at a time, though, and the servers of that version read it cleanly: at any moment one set of servers is fine, and the failures fall on the others until a miss changes the shape.

</details>

<details><summary>B. Whichever shape was last written, with errors on the other servers</summary>

✓ Right. The key holds whichever shape the last miss wrote, and the servers on the other version fail to parse it. Customers of the bank get errors on account pages for as long as the two versions run together.

Old and new code share the key during a rollout, and **a key whose name carries no mark of the shape hands each version whatever the other wrote**. Which side fails can change each time the entry expires, since the next miss may land on a server of either version.

Write new code that reads both shapes and goes on writing the old one, and switch what it writes in a second deploy. Or put the shape's version in the key, so that each version reads and writes only its own, and change it only when the shape changes, since a key with a new name is an absent key.

</details>

<details><summary>C. Whichever shape, converted for each server</summary>

✗ The two shapes differ by one field name, which a line of mapping code would translate. The servers read with a strict parser, which rejects a shape it does not know and converts nothing, and the cache itself stores bytes.

</details>

<details><summary>D. The new shape only</summary>

✗ New servers are where the system is heading, and their shape does reach the key. Old servers are still running, and a miss on one of them writes the old shape. Until the first miss, the key holds what was there before the rollout began.

</details>

---

###### 15. Reaching a cache node that answers slowly

A hotel booking app falls back to the database on any cache error or timeout. One cache node starts replying in two seconds, under its three-second timeout, and it passes its health checks. The app serves pages from one pool of 40 threads, and at this busy hour dozens of calls a second reach that node.

What do guests get as the traffic goes on?

<details><summary>A. Every page slows, on healthy nodes too</summary>

✓ Right. Pages slow down across the whole app, including pages whose keys live on healthy nodes: a guest searching for rooms waits on a cache node that page never needed.

**A reply slower than normal but inside the timeout counts as a success**: the fallback never fires and the health checks stay green. Meanwhile each call holds a thread for two seconds. Say 100 pages a second, a quarter of them calling that node: 25 calls a second arrive, the pool can finish about 20, and the queue grows for everyone.

Set the timeout close to a healthy reply, so that slowness counts as a failure and the fallback fires, and cap the calls in flight to each node, so that one node can hold only its share of the pool.

</details>

<details><summary>B. The slow node's calls land on the database and overload it</summary>

✗ Your code wraps every cache call in a fallback, so trouble with the cache means a trip to the database. The fallback needs an error or a timeout, and a two-second reply is neither. Each call waits out the two seconds and returns its value normally, and the database sees nothing extra.

</details>

<details><summary>C. That node's keys move to the other nodes</summary>

✗ A node that fails its health checks is taken out, and its keys go to the others. The slow node passes every check, because it does answer. It stays in place and goes on receiving its share of calls.

</details>

<details><summary>D. Only that node's keys are slow, with other pages as before</summary>

✗ Keys are split across nodes, and on the cache side a fault on one node reaches only its share of them. The threads are not split: every page draws from one pool, and calls waiting on the slow node keep taking threads until other pages queue behind them.

</details>

---

###### 16. Loading the popular items at three in the morning

A grocery delivery app loads its popular items into the cache at three in the morning, each with the same one-hour TTL. Each item is read several times a second all day, and a miss refills it for another hour. The store takes the load at three in its stride, reading the whole set within seconds.

What happens to the store's reads over the next day?

<details><summary>A. A burst about every hour, all through the day</summary>

✓ Right. The store takes a burst of refills about once an hour all day, and the load it took in its stride at three comes back each hour, with quiet between.

**An expired key refills at its next read**, and with reads this frequent that read comes almost at once, so every key's new hour starts almost when the others' do. Say five reads a second: the middle 90% of refills fall within about 0.6 seconds in the first cycle and about 3 seconds after 24 cycles.

Add a random offset to each TTL. With up to 10% of the TTL either way and a read each second, the busiest minute holds about 9% of the keys at the first refill, against all of them with no offset.

</details>

<details><summary>B. Bursts that shrink each hour until the reads are even by midday</summary>

✗ Customers arrive at random, so each key's first read after expiry lands at a slightly different moment, and the refills do drift apart. At several reads a second the drift is small: say five reads a second, and the refills spread by about 3 seconds after a whole day, far too little to level an hourly burst.

</details>

<details><summary>C. The store falls behind at four and stays behind all day</summary>

✗ At four every item expires together while customers are reading them, on top of everything else the store does. The refill asks only for the items the store read within seconds at three, and once they are back each one holds for an hour, so the burst ends and quiet follows.

</details>

<details><summary>D. One burst at four, then evenly spread reads</summary>

✗ Mass expiry is taught as one event, and one load at three gives one expiry at four. The refills at four give each key a new one-hour TTL starting at about the same moment, so the same keys all expire together again at five.

</details>

---

###### 17. Using request coalescing on a fleet of fifty servers

A sports app runs fifty servers behind one shared cache. To stop stampedes you add request coalescing to the server code, with an in-memory map of the keys being filled, and nothing else in the read path changes. A hot key expires as a match starts: each server gets many requests a second for it, and the fill query takes about a second.

What burst do you size the store for when that key expires?

<details><summary>A. One query for the whole fleet</summary>

✗ The usual name for this is single-flight, and one query is what you added it for. A server knows about its own waiting requests and nothing of the other forty-nine, so each sends its own query.

</details>

<details><summary>B. One query from each request, as before</summary>

✗ Before you added coalescing this was the count, and each request still calls the same function. Coalescing does work inside a process: every request on one server shares that server's single query, so the count falls from requests to servers.

</details>

<details><summary>C. One query from each server, in turn behind a lock</summary>

✗ A lock in the cache would line the servers up, one fill at a time. This fleet has no lock or token on the key, so no server waits for another and the queries overlap.

</details>

<details><summary>D. One query from each server, all at once</summary>

✓ Right. You size for about fifty copies of one query in flight together. Coalescing removed the per-request queries, and the burst still grows with the number of servers.

The map of keys being filled lives in one server's memory, and **request coalescing reaches only the requests inside one process**. The shared entry leaves every server missing at the same moment, and each sends its own query during the fill.

To get one query per key, coordinate through the cache: a lock or a token on the key lets one server fill. Give the lock a TTL and readers a limit on waiting, since its holder can die. Or refresh the key before it expires, from one background job or from readers that each decide at random as expiry nears. Random refresh can still send a few queries, and the 2015 paper Optimal Probabilistic Cache Stampede Prevention works out the odds.

</details>

---

###### 18. Counting on a cache through a settings store outage

The configuration service of a mobile game reads a remote settings store through a cache. The team lists the cache as its protection against settings store outages. Then the settings store goes down for an hour.

```python
def get_settings():
    value = cache.get("settings")
    if value is None:
        value = settings_store.fetch()
        cache.set("settings", value, ttl=300)
    return value
```

What does the service return in the second half of that hour?

<details><summary>A. The last good settings on every call</summary>

✗ Resilience lists name cached data as what you serve in an outage, and the last good settings were in the cache when the store went down. The entry left five minutes later at most, with no regard for whether a replacement could be fetched, so the value is gone.

</details>

<details><summary>B. The last good settings, with one call every five minutes failing</summary>

✗ One reader trying the refresh while the rest get the old value is how serving stale works. An entry here leaves at its TTL, so after a failed fetch there is no old value to hand out.

</details>

<details><summary>C. Errors, each read trying the settings store</summary>

✓ Right. After the first five minutes every call fails, and each one also tries the settings store: game clients get errors until the store is back, with the cache doing nothing for them.

**A TTL ends an entry on schedule, whatever the state of the source**, and from then on every read here is a failing fetch.

Where the cache offers it, use serving stale with a second TTL longer than the outages you plan for, and space out the refresh attempts, or each read goes on trying the store that is down.

</details>

<details><summary>D. Empty settings on every call, with no error anywhere</summary>

✗ Empty settings would let the game start on its defaults, and a failed fetch has to end somewhere. `get_settings` catches nothing: when the fetch fails, the failure leaves the function before `cache.set`, so nothing is stored at all.

</details>

---

###### 19. Looking up coupon codes that do not exist

A shop checks coupon codes at checkout through a cache. One night the logs show a script trying code after code that does not exist, some of them many times over. Customers' real codes are all in the cache by then, and they keep hitting it.

```python
def check_coupon(code):
    coupon = cache.get(f"coupon:{code}")
    if coupon is None:
        coupon = db.query("SELECT * FROM coupons WHERE code = %s", code)
        if coupon is not None:
            cache.set(f"coupon:{code}", coupon)
    return coupon
```

As the script keeps going, what does the database see?

<details><summary>A. Each code once, with its repeats answered from the cache</summary>

✗ Every other key here goes to the database once and is served from the cache after that, and the script does repeat itself. The `cache.set` runs only when the query found a row, so a repeat finds the key as absent as the first try did.

</details>

<details><summary>B. A falling rate of queries as the cache fills</summary>

✗ Warm-up graphs show queries falling away as a cache fills, and the real codes did trace that curve. The script's codes put nothing in the cache, so their queries hold level for as long as it runs.

</details>

<details><summary>C. No queries, with each unknown code rejected at the cache</summary>

✗ The cache sits in front of the database on every read, and the shop's real codes never get past it. `cache.get` knows only the keys stored in it, and a key absent from the cache does not show whether the table has a row: the lookup comes back empty, and the code goes on to the query.

</details>

<details><summary>D. A query for every attempt, repeats included</summary>

✓ Right. Every attempt reaches the database, repeats included, and the cache gains nothing from any of them. The hit rate on real coupons stays high beside this.

**A cache that stores only the rows it found keeps nothing for a code with no row**, and each lookup of that code is a first lookup. If the script makes 10,000 attempts drawn from 500 codes, that is 10,000 queries and 0 entries.

Store a not-found marker with a short TTL, which is negative caching: if the markers outlast the script, those attempts cost 500 queries, one per code. A marker does nothing for codes that never repeat, and for those you check a set of the existing codes in front of the cache, so an unknown code never reaches the query.

</details>

---

###### 20. Serving recommendations during a partner's stall

On a shopping site, your client wraps a partner's recommendation call, and the item page caches what the client returns. The partner stalls for a minute, longer than the client's timeout, and during it a shopper opens an item that is not in the cache.

```python
def recommendations(item_id):
    try:
        return partner.fetch(item_id)
    except Timeout:
        return []

def item_page(item_id):
    recs = cache.get(f"recs:{item_id}")
    if recs is None:
        recs = recommendations(item_id)
        cache.set(f"recs:{item_id}", recs, ttl=3600)
    return recs
```

With the partner back after that minute, what do shoppers see for the rest of the hour?

<details><summary>A. Recommendations from the next call on</summary>

✗ An outage ends when the partner comes back, and the partner is healthy again within the minute. No call goes out to find that out: the empty list is an entry, so `cache.get` returns a value and `item_page` shows it.

</details>

<details><summary>B. An empty list for the shopper who hit the timeout only</summary>

✗ A timeout belongs to one call, and only one shopper waited through it. The result went into a cache keyed by item, not by shopper, so everyone who opens this item reads that same entry.

</details>

<details><summary>C. An empty list on every item page of the site</summary>

✗ The stall hit every call to the partner, and every item page goes through the same client. The key carries the item id, so an empty list lands only on an item opened during that minute with no entry in the cache. Other items go on showing what they hold or fill normally.

</details>

<details><summary>D. An empty list for every shopper on that item</summary>

✓ Right. Every shopper on that item sees no recommendations for the rest of the hour, long after the partner is back, and each of those page views counts as a success.

The wrapper turned a timeout into a normal return, so `item_page` cannot tell a partner with nothing to recommend from one that stalled. **To a cache, an empty list is a value like any other**, and this one stays for its whole TTL.

Let the wrapper say that it failed, so the caller can choose. Then store the failure briefly, under its own short TTL, since storing nothing on a failure sends every request to the partner. Say 20 requests a second on the item: 1,200 calls go to the struggling partner during the stall, against 12 with a 5-second entry.

</details>

---

###### 21. Flushing a cache with a one-week TTL

A catalogue service sets a one-week TTL on products that rarely change. Reads spread over the whole catalogue and the hit rate is near the top, so last year the team moved the store to a far smaller machine, and it has run comfortably since. A cache upgrade at midday flushes every entry. Traffic stays as it is.

What do you tell support to expect from customers in the minutes after the flush?

<details><summary>A. The same pages, a little slower, until the cache refills</summary>

✗ A miss is a slower read that still returns the page. That holds while the store can answer. The store now runs on a machine chosen for the misses alone, and full traffic is many times that, so most of the reads get no answer in time.

</details>

<details><summary>B. The same pages at the old speed</summary>

✗ Popular items come back after a few reads, and on many sites they carry the traffic. Reads on this catalogue spread over all of it, so no small set of items brings the hit rate back.

</details>

<details><summary>C. Timeouts and errors from a store past its limit</summary>

✓ Right. Customers get timeouts and errors, and support hears about it within minutes, because the store is taking traffic it was never sized for.

**A store sized for the misses cannot carry the reads the cache was taking**, and it can refill the cache only at its own capacity. Say 1,000 requests a second, 200,000 products and a warm hit rate of 99.5%, so the store sees 5 reads a second, and say it can do 7.5. In the first minute about 99% of requests fail, and within an hour the store has read back only a small part of the keys.

Do not flush a cache the store relies on. Replace nodes a few at a time and warm each one, and if you must flush, shed load at the door while the cache refills.

</details>

<details><summary>D. Errors for a few seconds, then normal pages once the popular items are back</summary>

✗ A flush empties the cache once, and each product needs only one read to come back. Refilling takes as many store reads as there are products, at the store's own limit: even a catalogue of 1,000 products, through a store that does 7.5 reads a second, would take minutes to get 90% back.

</details>

---

###### 22. Seeing the hit rate go from 99 percent to 98

At a ride-sharing service, a cache answers nearly all reads, and the store behind it does little else and has room for half as much load again. A neighbouring process takes memory from the cache's host, and the hit rate falls from 99% to 98%, one point on the panel the on-call engineer watches.

What happens at the store when that point goes?

<details><summary>A. Its load moves by about a percent</summary>

✗ The panel moved by one point, and 1% of anything is usually small. The point is 1% of reads, and the store sees the misses, which were only 1% of reads to begin with and are now 2%.

</details>

<details><summary>B. Its load rises a little, and it absorbs the rise</summary>

✗ Half as much load again is the kind of margin a capacity plan rounds to safe. Such a margin would cover a fall from 90% to 89%, which adds a tenth to the misses. This fall doubles them, and room for half again does not hold double.

</details>

<details><summary>C. Its load rises, then fades as the cache refills what it lost</summary>

✗ A hit rate dip after a restart does fade, because the cache has the memory to fill again. The cache here has lost the memory itself. The neighbour keeps it, so the entries that no longer fit keep missing.

</details>

<details><summary>D. Its load goes past what it can carry</summary>

✓ Right. The store's load doubles, past the half again it has room for: the riders' requests start to wait and fail over a change the panel shows as one point.

**The store's reads are the cache's misses**, so read the hit rate as a miss rate: 1% to 2% is double. At lower rates one point is small, 1.1 times from 90%.

Put the miss rate, or the store's reads, on the panel, and plan capacity against it. Take the memory back here, since the store cannot carry this for long.

</details>

---

###### 23. Reading the hit rate after the fix went out

At a social network, the cache lost its contents, and misses have saturated the store. Its queue now holds far more than two seconds of work, and it works through that queue in order, whether or not the client is still waiting. The original fault was fixed ten minutes ago, the cache nodes are healthy, and traffic has not changed.

```python
def read(key):
    value = cache.get(key)
    if value is None:
        value = db.query(key, timeout=2)
        cache.set(key, value)
    return value
```

What happens to the hit rate from here?

<details><summary>A. It stays near zero while the store stays saturated</summary>

✓ Right. The hit rate stays where it is, and users keep getting timeouts with the fault long fixed and every node healthy.

A fill follows a store read that succeeded, and with a queue this long no read returns inside its two seconds. The misses saturate the store, and **a store that cannot answer inside the timeout prevents the fills that would end the misses**. A 2021 paper, Metastable Failures in Distributed Systems, names such a loop as the root cause of these failures, rather than the trigger.

Break the loop by cutting load: throttle misses to the store's capacity so that reads finish in time and fills land. Lift the throttle in steps, since the keys the throttled traffic never asked for still miss when the rest of the traffic returns.

</details>

<details><summary>B. It climbs slowly back as each request fills an entry</summary>

✗ One request's miss adds one entry, which is how a cache warms on a quiet day. `cache.set` runs only after `db.query` returns inside its two seconds. With this queue no query does, so no request gets as far as its fill, and however many requests arrive, the entries stay as they are.

</details>

<details><summary>C. It climbs straight back to its old level</summary>

✗ The nodes are healthy, and the fix that went out ten minutes ago did cure the fault. A cache is only as full as its fills have made it, and the fills depend on a store that is still overloaded by the same misses.

</details>

<details><summary>D. It swings up and down as timeouts come and go</summary>

✗ Control loops with a delay oscillate, and an overload has that shape when clients back off together. The clients here do not ease off, and the store's backlog of abandoned queries does not shrink, so there is no good spell in which fills could land and raise the rate.

</details>

---

###### 24. Running a nightly report that scans every product

An e-commerce site runs a report job at 02:00 that reads every product, several times what the cache holds, through the shared cache. The cache evicts the entry read least recently. Almost no customers are online until 07:00.

What happens to the hit rate when customers arrive at 07:00?

<details><summary>A. It holds at the usual rate</summary>

✗ The popular products are read all day, every day, so they are always recent when customers are about. Least recently used looks only at the last read. By 07:00 those products were last read before the job started, so the job's reads now outrank them.

</details>

<details><summary>B. It dips deep as customers' reads refill the cache</summary>

✓ Right. The morning's first customers mostly miss: product pages run slow and the store is busy at the start of the shopping day, until their own reads push the job's entries out.

The job's reads are the latest the cache has seen, and **least recently used looks at recency and nothing else**, so one pass of reads that will never repeat outranks products read all day yesterday. If there are 5,000 products, room for 1,000 and 200 popular items that draw nine reads in ten, the hit rate falls from 92% to 28% over the first 50 reads and comes to 65% over the first 500.

Run the job outside the shared cache, against the store, or use an eviction rule that also counts how often an entry is read, with counts that fade so that yesterday's favourites do not stay ahead of today's.

</details>

<details><summary>C. The usual rate on popular items, with a dip on the rest</summary>

✗ On an ordinary day, eviction falls mostly on products few people read. The job reads several times what the cache holds, so every earlier entry goes, popular or not, and a popular product survives only if the scan reached it near the end.

</details>

<details><summary>D. It stays near zero all morning</summary>

✗ The job filled the whole cache with products few customers read, and a full cache has to evict before it can store anything. The job's entries go first, since none has been read since the job: say 5,000 products, room for 1,000 and 200 popular items that draw nine reads in ten, and the rate is already 65% over the first 500 reads.

</details>

---

## Further reading

- [Metastable Failures in Distributed Systems](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf), 2021: how a system stays in a bad state after its trigger has gone, with a cache's hit rate among the loops it describes.
- [Optimal Probabilistic Cache Stampede Prevention](https://www.vldb.org/pvldb/vol8/p886-vattani.pdf), 2015: readers that refresh a key early at random, each deciding alone, and how to set the odds.
- [Metastable Failures in the Wild](https://www.usenix.org/conference/osdi22/presentation/huang-lexiang), 2022: such failures looked for in real systems.

Prev: [Sharding and partitioning](../sharding-and-partitioning/README.md) · Next: [Choosing a data store](../choosing-a-data-store/README.md)
