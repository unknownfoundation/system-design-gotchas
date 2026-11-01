# Choosing a data store

Prev: [Caching](../caching/README.md) · Next: [Message queues and event streams](../message-queues-and-event-streams/README.md) · Reviewed October 2026

A data store is where a system keeps what it has to find again, and its kinds differ in how they find a record, in what one write can change and in what a failure can take with it. The kind you pick decides which questions stay cheap to ask later and which promises you get without writing them yourself.

- **System of record**: the store whose copy of a fact counts when two stores disagree.
- **Derived data**: anything built from a system of record, such as a cache or a report table, which can be thrown away and built again from it.
- **Primary**: the one node of a group that accepts writes.
- **Replica**: a copy that replays the primary's changes and can serve reads. Unless a question says otherwise, it replays them after the primary has acknowledged the write, which is called asynchronous replication.
- **Majority**: more than half of a group's copies. A store that waits for a majority acknowledges a write once that many hold it.
- **Log-structured store**: a store that puts each change in new files and leaves its older files as they are.
- **Tombstone**: the small record such a store writes for a delete, kept to hide the older values of that key.
- **Compaction**: the background merge of such a store's files, which drops overwritten values. It drops a tombstone too, once nothing older can need it and any grace period the store sets has run out.

The kinds, by how each finds a record and what one write does:

| Kind | Finds a record by | What one write does |
| --- | --- | --- |
| Relational store | any columns, through SQL and its joins | commits a transaction across many rows |
| Document store | the document's id or its fields | changes one document as one step; where a transaction across documents is offered, it costs more |
| Key-value store | the exact key | changes one key as one step |
| Wide-column store | a row key, often made of a partition key, which places the row on a node, and a sort key, which orders the rows inside that partition | adds cells to a log-structured store |
| Columnar warehouse | a scan of the columns a query names, each kept in blocks of its own, read whole | loads in bulk, with a single-row update a heavy operation |
| Search index | the words of a text, ranked by how well they match and forgiving typos | adds a document to the index's own log |
| Object store | the full key, or a list of the keys that share a prefix | replaces one whole object as one step |

Unless a question says otherwise, an object store on this page has a flat namespace: its keys sit in one list, with no directory tree.

One machine has a ceiling, and the load that reaches it is a fact you can measure. Each store you add is one more place that a single change has to reach, and the trouble with a choice tends to show up long after it, when a new query, a growing record or a power cut arrives.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- weigh a new product's orders and reports against the kinds of store, and say what moving a stale profile or a ledger to another kind will change
- keep a document or key-value store working when an author renames, an article draws a crowd, a field changes shape, product asks for a new page, a voucher meets an order or one key takes the traffic
- send a monthly total, a customer's billing page, a job queue and a ticket search to a store that can carry them
- keep activity files, a day's sales and contract PDFs in an object store, and test what a shortlisted store does with the orders it has acknowledged

###### 1. Storing forty orders a minute for a finance team's questions

A regional furniture shop is moving its ordering off spreadsheets. Each order takes stock down, creates the order and adds a ledger entry, and the accountant will not close a day on which those three disagree. About forty orders come in a minute. The API already speaks JSON, and the owners plan to open in other countries next year. Finance brings a new question every week: last week, which orders took stock that never reached the ledger; this week, which ledger entries belong to orders for products now out of stock.

Which store do you build on?

<details><summary>A. A document store holding each order whole</summary>

✗ The API already speaks JSON, an order saved whole needs no mapping, and some document stores offer transactions across documents, at a higher cost. Finance's weekly question cuts across orders, stock and ledger, and this layout was chosen for reading one order at a time. A relational store answers each with one new statement and no change of layout.

</details>

<details><summary>B. A key-value store keyed by order id</summary>

✗ Fetching a value by its key is the quickest read a store offers, and each order has an id. An order also changes stock and the ledger, and a store that updates one key at a time can lose a worker between those writes, with stock taken and no ledger entry. Finance's question has no key to start from either.

</details>

<details><summary>C. One relational store and a replica</summary>

✓ Right. One relational store with a replica to fail over to takes the stock change, the order and the ledger entry in one transaction, and each of finance's new questions is one new SQL statement.

Forty orders a minute at three records each is two row writes a second, a load where **the choice rests on what has to change together and what you will ask later**. If finance's question covers a week's orders, about 400,000, the store joins them in one statement, while code that joins by hand makes twice that many reads, one per order and one per ledger entry.

Build on it, and add a second writer when the measured load nears what one machine carries.

</details>

<details><summary>D. A wide-column store across six nodes</summary>

✗ Comparison tables set relational against NoSQL on scale, and six nodes leave room for the owners' plans. Today the shop writes two rows a second, which one machine carries. A wide-column store finds a row by its row key, so each new question from finance that does not start from that key scans the table.

</details>

---

###### 2. Moving off a document store to cure stale profiles

A social network sends each profile save to a primary and each profile read to one of two replicas, all on a document store. Users complain that after saving a profile they sometimes see the old one. The team puts it down to eventual consistency and moves to a relational store for its ACID transactions. The new store runs a primary and two replicas too, and the app routes to them as before. The save now runs in a transaction, and the new test suite passes.

What do users see after saving a profile on the new store?

<details><summary>A. Their new profile on every read, from either replica</summary>

✗ The usual comparison sets ACID against eventual consistency as a property of the kind of store, and the new test suite passes. ACID's promises concern transactions on the primary, and a read on a replica that has not replayed the save stands outside them, so the new store returns the old profile there as the old one did.

</details>

<details><summary>B. Their old profile now and then, as on the old store</summary>

✓ Right. Now and then users see the old profile after saving, as they did before, and the migration has left the complaint where it was.

**A read served by a replica returns what that replica has replayed so far**, whatever the data model. Both stores acknowledge the save on the primary and send it to the replicas afterwards, and with the same lag the same share of reads returns the old profile.

Before you change stores over a stale read, find out which copy the read went to. The fixes belong to replication and do not depend on the data model.

</details>

<details><summary>C. Their new profile, with only other users still getting the old one</summary>

✗ A transaction sees its own writes, and that is the first thing an isolation example shows. That transaction ended when the save committed on the primary. The profile page that loads next is a new read on a replica, which replays changes without knowing who made them, so the person who saved is as likely as anyone to get the old profile.

</details>

<details><summary>D. A long wait on each save, and then the new profile</summary>

✗ Stores sold as consistent often wait for their copies before they answer. A save waits like that only where replicas are synchronous, and these replay after the primary has acknowledged. A replica can still hand back the old profile, and the save is as quick as it was.

</details>

---

###### 3. Moving a ledger to a distributed relational store

A payments firm runs its ledger on one primary in one zone and is moving it to a distributed relational store that keeps SQL. The new store splits rows into ranges and keeps each range on three copies across three zones. A range's reads go to its leader, the one copy that takes them, and the leaders sit in a different zone from the app. A transfer reads eight rows, one statement at a time, then commits.

What does the transfer wait for on the new store?

<details><summary>A. One disk flush on one machine, as on today's store</summary>

✗ On one machine a commit is a flush to the log, and the code you are moving has never paid for anything else. The log is now spread across zones: today's flush becomes a majority of copies agreeing, and each of the eight reads has crossed to a leader before the commit starts.

</details>

<details><summary>B. Every copy in all three zones, for every row it touched</summary>

✗ Strong consistency gets taught as every copy agreeing. A commit on this store needs a majority of a range's copies, two of three, so the transfer never waits for the slowest zone. Its larger wait comes earlier, in the eight reads.

</details>

<details><summary>C. One wait between zones at commit, and nothing per statement</summary>

✗ On a store whose nearby copies serve reads, the cost of replication lands at commit and, with zones far apart, makes up most of the wait. These reads go to the range's leader in another zone, so code written one statement at a time pays a trip per row before the commit's wait begins.

</details>

<details><summary>D. A leader's reply per read, then a majority at commit</summary>

✓ Right. The transfer waits for eight trips to leaders in another zone, one per read, and then for a majority between zones, and the same SQL now spends most of its time on reads that used to cost next to nothing.

Here **a read that goes to a leader in another zone pays a trip per statement**, and the wait grows with the rows the code reads one by one. If a trip between zones takes 10 ms and a read inside a zone 0.5 ms, the eight reads cost 80 ms and the commit 20 ms, one trip to the leader and one for the majority.

Count the statements in each transaction before you move, read together what can share one statement, and place range leaders near the app where the store lets you.

</details>

---

###### 4. Putting the author's name in every comment

A forum keeps comments in a document store split into shards by thread, a shard being one of the parts the data is divided into, each on machines of its own. Each comment holds a copy of its author's name, so a thread page needs one shard. An author changes their name, and the job below walks their two million comments. A million in, the machine running the job dies, and nobody starts the job again.

```python
def rename_author(author_id, new_name):
    for comment_id in store.comment_ids_by_author(author_id):
        store.update(comment_id, {"author_name": new_name})
```

What do readers of the author's old threads see from then on?

<details><summary>A. The new name on every comment, once the store finishes the job</summary>

✗ Background work inside a store, such as building an index, carries on after the client that started it has gone. This job is a loop in your code, and the store sees only single updates. It holds no list of what comes next, so the other million comments keep the old name.

</details>

<details><summary>B. The new name on some comments and the old on the rest, for good</summary>

✓ Right. A million comments carry the new name and a million the old, and they stay that way. A reader of the author's old threads finds one person under two names.

The store keeps its promise per document: **each update is atomic on its own, and the store knows nothing of the job around them**. No transaction wraps the job to undo the million done, and the store holds no list of what comes next to finish the rest. Copying the name into every comment is called denormalization, and each copy is derived data that has to be written again when the name changes.

Have the job record its progress, since a rerun without a record repeats the million updates already done, or keep the author's id in each comment and look the name up when the page is built.

</details>

<details><summary>C. The old name on every comment, once the store rolls the job back</summary>

✗ A relational update inside one transaction rolls back when its client dies, and this job is one long update. Each comment here was its own step with nothing around it. The million updates already made are ordinary finished writes, and nothing ties them together for the store to undo.

</details>

<details><summary>D. Threads that stay unreadable, locked by the job that died</summary>

✗ A migration that alters a large table can hold a lock and stop every query behind it, and this job had got through a million documents when it died. Each of those updates held its document only while it ran, and a finished write holds nothing. A reader opens the thread and gets whichever name each comment has.

</details>

---

###### 5. Keeping every comment inside the article's document

A news site keeps each article as one document with its comments in an array inside it, on a store that limits a document to 16 MB, and comments run about 300 bytes. One article is on course for 200,000 comments, each sent through the code below, where `if_version` fails the write if another comment got in first and the caller tries again.

```python
def add_comment(article_id, comment):
    article = store.get(article_id)
    article["comments"].append(comment)
    store.put(article_id, article, if_version=article["version"])
```

What happens to the writes as that article fills up?

<details><summary>A. Larger writes with every comment, and then refused writes</summary>

✓ Right. Each comment costs more to write than the one before, and the store refuses them at the 16 MB limit, well short of 200,000. From there people commenting on that article get errors.

`store.put` sends the whole document back, the comments already there included, and **a write that carries the whole document costs more as the document grows**. Comment 50,000 rewrites about 15 MB, and the limit comes a little further on.

Keep each comment as a document of its own holding the article's id, and embed only what has a known limit, such as a few pinned comments, never an unbounded array.

</details>

<details><summary>B. The last comment written as cheaply as the first</summary>

✗ An append is the cheapest write there is, on a log or as a new row in a comments table. This array lives inside the article's document, which `store.put` sends back whole for each comment, so a later comment carries every earlier one with it.

</details>

<details><summary>C. Older comments moved out to a linked document by the store</summary>

✗ A large value in a relational table can move out of its row on its own, with the application never hearing of it. A document store's unit is the document, and a linked document exists only if your code writes one. `add_comment` appends to the array and nowhere else, so each new comment goes there until the store refuses the write.

</details>

<details><summary>D. One document split across nodes by the store</summary>

✗ Such stores promise to spread data across nodes. That spreading moves whole documents from node to node, and one document stays in one piece, so a 16 MB article sits on one node with its limit and its growing writes.

</details>

---

###### 6. Changing a phone field's shape with no schema

A clinic's booking app keeps patients in a document store. The release below is the whole change, and its tests, which create their patients through `save_patient`, pass. A patient last saved two years ago opens the app to book again.

```diff
 def save_patient(patient, country, number):
-    patient["phone"] = country + " " + number
+    patient["phone"] = {"country": country, "number": number}
     store.put(patient["id"], patient)

 def booking_screen(patient_id):
     patient = store.get(patient_id)
-    return render(patient["phone"])
+    return render(patient["phone"]["country"], patient["phone"]["number"])
```

What comes back from `store.get` for that patient's `phone`?

<details><summary>A. An object holding the old digits, upgraded by this first read</summary>

✗ Lazy migration moves each document to the new shape the next time it is touched, and opening the booking screen touches this one. That upgrade is code someone writes: a read that spots the old shape and saves the new one. `booking_screen` only calls `store.get`, so no write happens and the string stays as it was.

</details>

<details><summary>B. An empty phone field in an otherwise whole document</summary>

✗ An object mapper in your code can drop a value that does not fit its model, and the field then comes back blank. This code has no mapper, and the store itself does nothing of the kind: the old string comes back whole.

</details>

<details><summary>C. The old plain string, exactly as saved</summary>

✓ Right. The store returns the string saved two years ago, and `booking_screen` fails when it asks that string for `"country"`.

**A store with no schema keeps what was written**, and the release changed only what gets written from now on. Every patient nobody has saved since still holds the old shape, and the tests read only patients they created in the new one. This is often called schema on read: the migration moves into every piece of code that reads the field.

Make reading code accept both shapes, then run a job that rewrites the old ones, and take the old branch out only when that job has finished.

</details>

<details><summary>D. An error from the store in place of the old document</summary>

✗ The new code expects an object and this document holds a string, a mismatch that a relational column's type check or a validation rule would turn away. Those checks guard writes, and this store has none set. It knows nothing of what your code expects, so it hands the patient's document back as it was stored.

</details>

---

###### 7. Listing a customer's orders in a store keyed by order id

A meal-kit company keeps its orders in a store that offers get and put by exact key and a full scan, keyed by order id, with a year of orders in it. Product now wants a page of a customer's orders on every customer's account screen.

What does the new page need from you?

<details><summary>A. A second table by customer, plus the year's orders copied into it</summary>

✓ Right. The page needs a second table keyed by customer, new orders written to both tables, and the year's orders copied across. With all three in place, a customer with a long history opens a full list on the first day.

This store finds a record by its exact key and in no other way, so a secondary index here is a second table that you keep in step, and **a table you build yourself holds only what is written to it after it exists**.

Start writing new orders to both tables, backfill the year's orders behind them, and open the page when the backfill is done. Those two writes per order are still two steps, so keep a job that compares the tables.

</details>

<details><summary>B. One command that adds an index on the customer field</summary>

✗ Years of relational work have made adding an index a one-line habit, and the field is right there in each order. This store's whole interface is get, put and a scan: it has no such command, so the only index you get is one you build as a table of your own.

</details>

<details><summary>C. A quick scan of the store on each page view</summary>

✗ Key-value stores top the speed tables, and a scan is one of the three things this one offers. Those tables measure a get, and a scan reads every order in the store for each account screen. If the year holds about ten million orders, that is ten million items where a table by customer would read one page's worth.

</details>

<details><summary>D. A second table by customer, written with every order from today on</summary>

✗ Modelling guides show a table per query filling up as orders come in, and their examples start from an empty store. This one holds a year of orders, none of them in the new table. If customers order at random moments, about every 14 days on average, about nine in ten open an empty list on the first day, and about six in ten after a week.

</details>

---

###### 8. Keeping a voucher and its order in a key-value store

A coffee chain keeps the vouchers and orders of its app in a key-value store, whose client offers `get(key)`, `put(key, value)` and `put_if(key, expected, value)`, which writes only if the key still holds `expected`. A voucher can be spent once, on one order. Workers place the orders, and a worker can die at any point in a request.

Which design never shows the voucher and its order out of step?

<details><summary>A. The voucher marked pending, then the order, then the voucher marked spent</summary>

✗ A status field is how a workflow shows a step in progress, and a sweep can finish what a dead worker left. A worker that dies after the first write leaves a voucher pending with no order, and one that dies after the second leaves an order beside a voucher not yet spent. The app reads either until the sweep comes round.

</details>

<details><summary>B. A lock held around both writes</summary>

✗ A lock makes other workers wait while its holder does both writes. It does nothing for a holder that dies halfway: the first write stays, the second never happens, and the next worker through finds the pair out of step.

</details>

<details><summary>C. The order write, then the voucher, with retries on failure</summary>

✗ Writing the order first means a crash takes nothing from the customer. A dead worker retries nothing, and the voucher still reads unspent after that crash, so the customer can use it on a second order, and two orders stand on one voucher.

</details>

<details><summary>D. One record holding the voucher and its order</summary>

✓ Right. One record holding the voucher and its order changes in a single conditional update, which leaves the app only two things to find: a voucher unspent with no order, or spent with its order.

On this store only a single record changes in one step, and **a worker can die between any two writes that are separate steps**. `put_if`, a compare-and-set, is what spends the voucher once: the update applies only while the record says unspent, so a second order finds the voucher already spent.

Key the record by voucher and put in it what has to change together. The design works while a voucher belongs to one order; if one order could take several vouchers, the record no longer fits, and that need calls for a store with transactions across keys.

</details>

---

###### 9. Provisioning 40,000 writes a second for a table at 6,000

A flash sale runs on a managed key-value store that sells write capacity per table. Mid-sale, the table's settings and its dashboard read as below.

```text
provisioned writes   40,000/s
partitions           20
capacity split       even
capacity sharing     off

table writes         6,000/s
top key              product/kettle   5,000/s
```

What happens to the writes for `product/kettle`?

<details><summary>A. All of them accepted, with the table far under its provisioned capacity</summary>

✗ The bill and the dashboard both show capacity per table, and 6,000 against 40,000 leaves plenty. The store applies it per partition, and that one key's writes all land in one partition, where they meet a share of 2,000 a second while the dashboard still reads far under.

</details>

<details><summary>B. All of them throttled, along with every other product's writes</summary>

✗ A limit on a single server holds everything behind it once it is reached, and the table is what you bought capacity for. The other nineteen partitions stay inside their own shares here, so writes for products on those partitions keep going through, while this product's writes beyond its partition's share are sent back.

</details>

<details><summary>C. Most of them throttled, with other products' writes going through</summary>

✓ Right. That product's partition accepts about 2,000 writes a second and sends back about 3,000 as throttled: six in ten of its writes fail at the height of the sale, with the table far under what you paid for.

Capacity provisioned per table is spent per partition: twenty partitions of 40,000 give each 2,000. This is a hot key, since **all the writes of one key land in one partition**. Only a product sharing the kettle's partition competes for that 2,000. The other nineteen stay well inside their shares, which is why the table's figure gives no sign of it. A store that lends spare capacity to a busy partition would let these writes through up to what one partition can carry, and here capacity sharing is off.

Before a sale, compare your hottest key with one partition's share, rather than the table with its provisioned total, and spread that product's writes over several keys or buy enough that one partition's share covers them.

</details>

<details><summary>D. All of them accepted, with that product's key split across partitions</summary>

✗ A managed store is sold on scaling itself, and such stores do split a busy range of keys between partitions. A split divides keys, and one key stays one key: all 5,000 of its writes a second still land together, in one partition, at one share.

</details>

---

###### 10. Using a wide-column store for monthly revenue totals

A ride-hailing firm keeps a billion rides in a wide-column store. The store is on the firm's list of analytics stores. An analyst now runs a month's revenue total on the table below, and it comes back with the right figure.

```text
table rides
  partition key   customer_id
  sort key        ride_time
  column families 1, all fields

query
  SELECT SUM(amount) FROM rides WHERE ride_time BETWEEN :start AND :end
```

What does that query read?

<details><summary>A. Only the amount column on each node</summary>

✗ Column is in the store's name, and a columnar warehouse does read only the columns a sum needs. This store keeps each customer's rides together under the row key, the amount beside the other fields, so a node has no amount column to read alone.

</details>

<details><summary>B. Every partition on every node, row by row</summary>

✓ Right. The sum visits every partition on every node. The analyst's one number costs a read of that month's rides across the whole cluster.

**A wide-column store groups data by what is in its row key**, and this one groups by customer. A month's sum over all customers has no key to start from, so it has to go to each customer's partition in turn. A columnar warehouse keeps each column in blocks of its own, and a sum reads only the blocks it needs.

Keep the store for what it finds by key, and feed the rides to a columnar warehouse as derived data for the monthly totals.

</details>

<details><summary>C. One running total the store keeps per column</summary>

✗ Running totals are what a materialized view gives you, and a report that runs every month is the usual reason to build one. Nobody built one here, and the store keeps no total per column on its own, so the sum has to come from the rides themselves.

</details>

<details><summary>D. One stretch of rows, the month's, kept in time order</summary>

✗ `ride_time` is in the key, and a sorted store reads a range of its sort key as one cheap stretch. That holds inside one partition. The key starts with the customer, so this month's rides lie in a stretch of their own within every customer's partition, and no single stretch holds them.

</details>

---

###### 11. Serving a customer's page from the warehouse

Analysts at a subscription software firm get two-second sums over a billion rows from a columnar warehouse, whose settings are below. The billing page now reads one customer's row from it, and at nine in the morning two thousand customers open the page.

```text
max concurrent queries   8
row indexes              none
max queue length         unlimited
queue timeout            none
```

What do customers get at nine?

<details><summary>A. Their billing page in milliseconds, all two thousand of them</summary>

✗ One row is a tiny share of what those two-second sums read, and the warehouse is already known to be quick. It has nothing to jump to: finding one customer reads whole blocks of every column the query names, and at nine most of the pages have not even started.

</details>

<details><summary>B. Their billing page in about two seconds, all of them</summary>

✗ Two seconds is what the warehouse has answered in each time you have seen it. An analyst's sum starts at once, with few neighbours. At nine, eight lookups run and the other pages wait for a turn, and that wait comes on top of the lookup's own time.

</details>

<details><summary>C. Eight billing pages, and an overload error for all the others</summary>

✗ An API past its limit answers the extra calls with an error, and two thousand pages is far past eight. The eight here counts queries running, and this warehouse sets no limit on how many wait, so the ninth page's query joins the line and the last one joins behind it.

</details>

<details><summary>D. Eight billing pages at first, and a long wait for all the others</summary>

✓ Right. Eight lookups run and the rest queue, and most of the two thousand customers wait for their billing page: if a lookup takes a second, the median one waits minutes.

A lookup reads whole blocks of each column it names, and only eight lookups run at once. The two-second sums showed how fast the warehouse scans, and **a scan's speed and the number of queries running at once are separate limits**.

Serve the page from a store that finds a row by key, filled from the warehouse as derived data, and leave the warehouse to the analysts.

</details>

---

###### 12. Keeping a job queue in a wide-column table

A video platform keeps its encoding jobs in one partition of a wide-column table, on a log-structured store. Workers fetch the oldest row and delete it when the job is done, ten thousand jobs an hour, so about a hundred rows are live at any moment. The partition was created this morning, and no compaction has touched it since.

Six hours in, what does fetching the next job involve?

<details><summary>A. A read that steps over the morning's deleted jobs</summary>

✓ Right. To reach the oldest live job, the read steps over about 60,000 tombstones, so a queue holding a hundred jobs gets slower to fetch from with every job it finishes.

A delete here writes a tombstone, and **a read has to pass every tombstone in its path**, since a tombstone is what hides the older value. Jobs go oldest first, so the tombstones all pile up in front of the oldest live row, where the live count of a hundred does not show them. Compaction may not clear them soon: on one store the grace period is ten days by default, and a compaction at the end of the day clears none of them.

Use a message queue for jobs. If they stay in this table, start a new partition each hour, so that a fetch walks at most about an hour's tombstones.

</details>

<details><summary>B. The same hundred rows as at the start of the morning</summary>

✗ A table with a hundred live rows takes a hundred-row read on a store that removes rows in place, and a row count on this one still says a hundred. This store removes nothing at the delete. It adds a tombstone, and the read passes those too, about ten thousand more for each hour gone by.

</details>

<details><summary>C. One row, the oldest, found straight from the sort order</summary>

✗ Rows in a partition are sorted, and with a time in the sort key the oldest job sits at the front. The tombstones sort there too: every finished job left one, in the place its row held, so the front of the partition is all tombstones and the oldest live row lies behind them.

</details>

<details><summary>D. A read turned away while the deletes hold the table locked</summary>

✗ A run of heavy deletes can lock a table on a store built differently, and sixty thousand is a lot of deleting. A delete here changes no row already written: it is one more small write, added to new files, and it holds nothing. The workers' fetches and the next deletes have no lock to wait on.

</details>

---

###### 13. Storing support tickets that need fuzzy search and may never be lost

The support desk of a software vendor holds forty million tickets and is choosing where they live. Search has to rank by relevance and survive typos, and results may trail a save by a few seconds. No acknowledged ticket may ever be lost. The search index you priced writes its log to disk every five seconds, and the transactional store flushes before it acknowledges. That store's own text matching, tried on a copy, ranks by spelling alone.

Which design meets those needs?

<details><summary>A. The tickets in the search index alone, with replicas</summary>

✗ The index has replicas, accepts writes, and would leave one system less to run. The replicas cover one machine going, but the log reaches disk every five seconds, so a power cut across them all can take up to five seconds of acknowledged tickets. No other system holds those tickets, and the desk has nowhere to fill them back from.

</details>

<details><summary>B. The tickets in a transactional store, fed into the index</summary>

✓ Right. With every ticket in the transactional store and the index fed from it, the desk keeps every acknowledged ticket whatever the index does, and support gets its ranked search a few seconds behind each save.

The transactional store is the system of record and the index is derived data, and **derived data can be built again from the system of record**, so any ticket the index loses, or the whole index, can be refilled. The cost is the few seconds of lag, which the desk allows.

Feed the index from the store's committed changes, which is called change data capture, and keep a job that can rebuild it whole.

</details>

<details><summary>C. The tickets written to both from each request, retried on failure</summary>

✗ Dual writes are the simplest way to keep two copies, the code is short, and a retry covers a write that fails. The retry runs inside the request and dies with it, and a crash between the two writes leaves a ticket in the store and missing from search, or, with the index written first, a search result for a ticket the store never kept. No later step in that design puts it right.

</details>

<details><summary>D. The tickets in a transactional store, searched with its own text matching</summary>

✗ One store means nothing to keep in step, and a relational store can rank strings by similarity and forgive a typo. Your trial on the copy showed where that stops here: support needs a ranking that spelling alone does not give, and safe storage does nothing for it.

</details>

---

###### 14. Adding a line per event to one object per user

A fitness app keeps the activity of each user as one file in an object store, and each new event reads the file, adds a line and writes it back. Two events for one user reach two servers at the same moment, and the store's request log shows the calls below.

```text
server A   get   activity/user-1
server B   get   activity/user-1
server A   put   activity/user-1   condition=none
server B   put   activity/user-1   condition=none
```

Which lines does the object hold after those two events?

<details><summary>A. One event's line, and no sign of the other</summary>

✓ Right. The object holds the old lines and one new one. The other event is gone with no error on either server, which is called a lost update, and the user's activity is short by it.

**A write replaces the whole object**, so each server replaces it with its own read plus one line, and the later write stands. Only reads that overlap lose a line: where one pair finishes before the other reads, both lines stay, which is how this can pass for months and then fail on a busy day.

Write each event as an object of its own and merge later, or send each write with a condition, so that a write fails when the object has changed since it was read, and try again from the read.

</details>

<details><summary>B. Both lines, in arrival order</summary>

✗ A file opened for append takes lines from two writers and keeps both, and tools that show an object store as folders and files present this one as a file. This store has no append, so each server sends the whole file as it read it plus its own line, and neither read held the other's line.

</details>

<details><summary>C. The first line, with the second writer told the object is busy</summary>

✗ Two servers writing one object at one moment is the case a lock or a conditional write exists for, and either would give the second server an answer like that. These writes carry no condition and no server takes a lock, so each write goes through with nothing comparing it to what was read, and neither server hears about the other.

</details>

<details><summary>D. Both lines' bytes mixed together in one garbled file</summary>

✗ Two processes writing to one shared file can interleave their bytes. An object changes as one step: a reader gets the whole old object or the whole new one, never part of each, and the later `put` replaces the earlier one entire.

</details>

---

###### 15. Publishing a day's data by renaming its folder

A retailer's nightly job writes a day's sales as 400 objects in an object store under a temporary prefix, then renames that folder into its final place with the storage tool's move command. A dashboard job lists the final folder on a timer, and one of its reads falls in the middle of the rename.

What does that read find?

<details><summary>A. All 400 objects, or none of them</summary>

✗ Writing to a temporary name and renaming is the standard recipe for publishing a file, and it works on a file system, where a rename is one step. With no directory tree, the store has no folder to rename: the tool works through the objects one by one, and a read in the middle lists the ones that have arrived.

</details>

<details><summary>B. A busy error on the folder, and no list</summary>

✗ A file system can hold a directory busy while something works on it, and this folder is in the middle of a rename. The store holds no folder that could be busy, only keys that the tool copies and deletes. The dashboard job's list call is answered like any other, from the keys under that prefix at that moment.

</details>

<details><summary>C. Some of the day's objects and not the others</summary>

✓ Right. The read finds only the objects copied so far. The dashboard shows a day's sales short by whatever has not arrived, with no sign that the list is partial.

With a flat namespace, **a folder is only a shared prefix in the objects' names**, so renaming 400 objects means 400 copies and 400 deletes, and a read between them finds part of the set. Each object's copy is atomic, and nothing makes the set of 400 one step.

Write the objects under their final names, then write one manifest object listing them, and have readers trust only what a manifest names. One object's write is atomic, so a reader finds the whole set or the previous one.

</details>

<details><summary>D. Some objects cut off partway, half copied</summary>

✗ A large file being copied on a file system shows up long before it is whole, and this read lands in the middle of a copy. An object store shows a key only once its write is done: one object's copy is one step, so each object in the dashboard job's list is whole.

</details>

---

###### 16. Moving contract PDFs out of the database

The document system at a law firm has a 20 TB database, most of it contract PDFs held in a column, and backups and replica rebuilds now take days. You move the PDFs to an object store with the change below, and a partner asks what the move costs.

```sql
ALTER TABLE contracts ADD COLUMN pdf_key text;
ALTER TABLE contracts DROP COLUMN pdf_bytes;
```

What does the firm give up?

<details><summary>A. Durability: the PDFs are lost more often</summary>

✗ Cheap bulk storage has a long reputation as the weak place to keep things. If the PDFs run 2 MB each and fill nine-tenths of the 20 TB, there are about nine million, and one maker's design figure of 99.999999999% a year, a target for failing devices, works out at about one PDF lost in eleven thousand years. A wrong delete of the folder would take all nine million, and that is a matter for versioning, which keeps earlier copies of an object.

</details>

<details><summary>B. Partial downloads of a large PDF</summary>

✗ An object is written whole, and the plainest fetch returns it whole. A read can also ask for a byte range and get only those bytes, so a viewer can still bring in a long contract a page at a time.

</details>

<details><summary>C. Room for millions of PDFs in one folder</summary>

✗ A directory on a file system slows down as its files pile up, and 20 TB of PDFs makes a great many files. An object store has a flat namespace with no directory to fill, so the PDFs fit under one prefix.

</details>

<details><summary>D. A row and its PDF saved as one step</summary>

✓ Right. The firm gives up saving a row and its PDF as one step. A crash between the two saves can leave a contract row with no PDF behind it, or a PDF no row points to.

The database saved both in one transaction, and **a database and an object store share no transaction**. The order of the saves picks which mistake you can get: PDF first leaves a file with no row, and row first leaves a row with no file.

Save the PDF first and the row second, since an orphan file costs some storage and a row with no file points at a contract that cannot be opened. Run a job now and then that removes objects no row names, leaving recent ones, whose row may still be on its way.

</details>

---

###### 17. Keeping three copies of every order at a power cut

A takeaway ordering service runs on a store with the settings below. The zone it runs in loses power.

```text
nodes            3
zones            1
acknowledge at   2 of 3 nodes, in memory
flush to disk    every 10 s
```

Which orders are there when the zone comes back?

<details><summary>A. Every acknowledged order, filled back from the other copies</summary>

✗ Two copies confirmed each order, and a node that comes back fills its gaps from the others when machines fail one at a time. The power takes all three at once here, and memory goes with it, so an order that never reached a disk is gone from every node at the same moment.

</details>

<details><summary>B. Orders up to the last disk write, and none after</summary>

✓ Right. What comes back is what had reached a disk, so customers who ordered in the last seconds before the cut hold a confirmation for an order the service no longer has.

These nodes acknowledge from memory, and **a majority held in memory goes with the power when every node loses it together**. Take two orders a second and a cut 37 seconds in: 74 orders have been acknowledged, and with all three timers in step, 60 are on disk and 14 are gone.

Acknowledge only after a flush, or put the copies in zones that do not lose power together. Either one adds a wait to each order, and the second still leaves a window if every zone goes.

</details>

<details><summary>C. No orders at all, on a store that starts empty</summary>

✗ A store that keeps everything in memory starts empty after a restart, and these nodes do hold new orders in memory. They have disks as well, and each timer tick wrote out what the node held, so the orders up to that tick come back when the power does.

</details>

<details><summary>D. The later orders too, resent by the clients that placed them</summary>

✗ Clients resend a request when no answer came. These orders were answered, each customer saw a confirmation, and no client has a reason to send again, so the last seconds' orders stay gone.

</details>

---

###### 18. Comparing two stores that both promise no lost writes

A ticketing firm has a shortlist of two stores. The need is that no acknowledged order is lost when a node dies. Both passed the team's test, a clean shutdown of the primary between load runs, with no order lost. The table below shows the two at the settings you would ship.

```text
                  first store            second store
documentation     "no lost writes"       "no lost writes"
log file write    every commit           every commit
flush to disk     every 10 s             every commit
```

Which change to the test can tell the two stores apart?

<details><summary>A. Twice the load on two healthy clusters</summary>

✗ Holding up at twice the load shows something real, and a load test is the one you already have running. Nothing fails during this run, so every order stays on both stores and the two results match.

</details>

<details><summary>B. The same clean shutdown of the primary, with writes still running</summary>

✗ The team's test stopped the primary between load runs, so writes running through the shutdown are the gap you would close first. A clean shutdown stops taking writes and flushes what the store holds before the process exits, so the orders the first store acknowledged reach its disk too, and the two stores come out alike again.

</details>

<details><summary>C. A power cut to the primary, with writes still running</summary>

✓ Right. Cutting the primary's power while writes are running is the test that can separate them, since it is the one failure where their flush settings part ways.

Each commit writes the order to the log file, and that write lands in the operating system's memory on its way to the disk. A clean shutdown flushes it down and a killed process leaves it with the operating system, so neither of those loses an order on either store, while **a power cut takes whatever has not been flushed to disk**. If no replica yet holds the last seconds' orders, the first store comes back without the orders it acknowledged since its last flush, and the second has lost none of them.

Write the need as a failure, run that failure with writes going at the settings you would ship, and count the orders acknowledged against the orders still there. A pass shows that this failure at these settings kept them, and no more.

</details>

<details><summary>D. A hard kill of the primary's process, with writes still running</summary>

✗ A hard kill gives the process no warning and no chance to flush, and it is the crash a test script can stage with one command. The first store had already written each acknowledged order to its log file before it answered, and what a process has written stays with the operating system after the process is gone. Both stores read every order back on restart, so the run shows no difference.

</details>

---

## Further reading

- ["One size fits all"](https://doi.org/10.1109/ICDE.2005.1), 2005: argues that the single code line every major database vendor followed has failed, and shows why warehouse users were given a system of their own.
- [Life beyond Distributed Transactions](https://www.cidrdb.org/cidr2007/papers/cidr07p15.pdf), 2007: why application developers do not build large scalable applications on distributed transactions, and the weaker guarantees they build with instead.

Prev: [Caching](../caching/README.md) · Next: [Message queues and event streams](../message-queues-and-event-streams/README.md)
