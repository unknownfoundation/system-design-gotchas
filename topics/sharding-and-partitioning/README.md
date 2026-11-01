# Sharding and partitioning

Prev: [Replication](../replication/README.md) · Next: [Caching](../caching/README.md) · Reviewed October 2026

Sharding cuts one data set into slices and puts each slice on its own machine, so that writes and data too large for one machine have somewhere to go. Partitioning is the same cut, whether the slices stay on one machine or go to several. A partition key says which slice a row belongs to, a placement rule turns the key into a machine, and a router sends each request there. Unless a question says otherwise, each shard is one machine with no replicas, and shards are plain databases that know nothing of each other.

- **Shard**: one slice of the data and the machine that serves it.
- **Partition key**: the column whose value picks the shard. Rows with one value stay together. Joins, foreign keys and changes to several keys in one step work only within a shard unless a layer above the shards does that work, and changing the key after launch rewrites most rows.
- **Hash partitioning**: a hash of the key picks the shard, which spreads keys evenly and sends a read of neighbouring keys to every shard. Every machine must compute the same hash, and a language's built-in hash may not give it.
- **Hot key**: one key that draws more traffic than its shard can serve. A store's automatic split moves ranges and does nothing for one hot row, a cache in front of the shard takes reads and no writes, and even row counts can hide uneven load.
- **Rebalancing**: moving data between shards, while they serve traffic, to even them out.
- **Router**: the part that sends each request to the shard holding its key, and merges the replies when a request reaches several shards. A request touching every shard fails or comes back short when one shard cannot answer, depending on the router.
- **Local index and global index**: a local index is kept by each shard over its own rows. A global index covers all rows and is itself split across shards by the indexed value, so a write also goes to a second shard, and a column with few distinct values makes its own hot shard.
- **Replay**: applying a log of changes to another copy, here the changes made on the old shard while a move copies its rows.
- **Cutover**: the step in a move that switches a customer's traffic from the old shard to the new one.

Five ways to place a key on a machine:

| Way | What decides the machine | A read of neighbouring keys | A new machine |
| --- | --- | --- | --- |
| Range partitioning | each shard holds a span of the sorted keys | touches one shard | takes a range split off a full shard |
| Mod-N hashing | the key's hash modulo the node count (the remainder after dividing by it), computed on every request | touches every shard | changes the count in the formula |
| Consistent hashing with virtual nodes | each server holds positions on a circle called the ring, and a key goes to the next position clockwise | touches every shard | takes arcs from the servers beside its positions, more evenly the more positions each holds |
| Logical partitions | keys hash to a fixed set of partitions, and each partition sits on a machine | touches every shard | takes whole partitions, and each partition costs memory, metadata and recovery time on every copy |
| Shard map | a stored table of which key or partition sits where, cached by the routers | depends on how the map was cut | takes what the map is edited to give it, and a new router cannot route while the map's store is down |

The trouble sits around the key: one key too busy for its shard, a query that does not carry the key, a change that spans two shards, and the minutes while rows are moving.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- decide whether to shard at all, and choose a key for meter readings, a viral post, a customer larger than a machine or a parcel that changes city, and say what a busy key means for the quiet keys on its shard
- add or lose a machine on a busy day, on a cache or a store, and say which keys go where and who carries the extra work
- follow a sign-in lookup, a weekly report, an erasure of someone's data, a search, a sign-up and a money transfer across shards, and say what each one reaches and what it leaves behind
- move one customer to a new shard, through the copy, the map change and the week after

###### 1. Running a recipe database at 90 percent CPU

A food publisher runs its recipe site on one primary database (the copy that takes the writes) at 90 percent CPU through the evening, with no replica (a copy that replays the primary's writes). The query log shows 19 reads for each write, of about the same cost. Pages already go out through a cache that holds each one for a few seconds, the data fills a fifth of the disk, and editors search across all the recipes from a back-office screen. Last spring the publisher's shop database reached the same CPU, four shards split by order id brought it down, and the runbook has said to shard at that CPU ever since.

Which change do you make first?

<details><summary>A. Four shards split by recipe id</summary>

✗ Four shards split by id brought the shop's database down from the same CPU, the runbook says to do it again, and the CPU here would fall too. The split moves every row, adds a router and turns the editors' search into a query to all four shards, all to relieve reads that could leave the primary with no rows moved.

</details>

<details><summary>B. Two read replicas behind the recipe pages</summary>

✓ Right. The recipe pages' reads leave the primary for the two replicas and only the writes stay on it, so the evening peak stops being a problem with no row moved and the editors' search still running as one query.

**Shards make room for writes and for data**, and neither is the limit here: nineteen operations in twenty are reads, and a replica can serve those, at the price of some lag, on pages the cache already serves a few seconds old. Four shards would bring the CPU down too, as they did for the shop, but only after every row has moved.

Point the pages at the replicas and keep shards for the day the writes or the data outgrow one primary, since replicas take none of the writes off it.

</details>

<details><summary>C. A second primary, each taking half the traffic</summary>

✗ A second primary accepts writes as well as reads, where a replica takes only reads. Neither machine sheds a write, since every write is applied on both, and only half the pages' reads move where two replicas would take them all. Two editors changing one recipe on the two primaries now make a clash that something has to settle.

</details>

<details><summary>D. A move to a store that splits data on its own</summary>

✗ A store that partitions by itself takes the splitting and the balancing off your hands. It still needs a key chosen and every row copied across, and the load here is nearly all reads, which need no split.

</details>

---

###### 2. Splitting one database into four to avoid outages

A company that hosts online shops keeps them on one database machine, and when that machine fails every shop stops. To avoid those outages, you replace it with four machines of the same model, one shard each, with the shops divided evenly among them. Each page a shopper opens touches only the shard of that shop.

What do the shops see from machine failures after the split?

<details><summary>A. The other three shards carry on and take over the failed shard's shops at once</summary>

✗ Web servers behind a balancer cover for each other: one dies and the rest take its requests. Those servers hold no data of their own. A shard holds the only copy of its shops' rows, so the other three have nothing to serve for them until that machine is back.

</details>

<details><summary>B. Every shop goes down when any one shard fails</summary>

✗ A query that needs all four shards stops when one is gone. The pages here need one shard each, so a shopper on a healthy shard never touches the failed machine, and three quarters of the shops keep selling.

</details>

<details><summary>C. A quarter of the shops go down, and such outages come more often</summary>

✓ Right. Each failure now stops a quarter of the shops, and the company sees about four outages where it used to see one, so the outage the split was meant to remove has become smaller and more frequent.

**A shard holds the only copy of its slice**, so nothing covers for it, and four machines give four chances to fail. Say each machine fails twice a year and takes 6 hours to come back. One given shop is down about 12 hours a year either way, while the hours in which some shop is down go from about 12 to about 48.

The split buys room for data and writes. For a shop to survive a failed machine, each shard needs a replica of its own and a way to switch to it when the shard fails.

</details>

<details><summary>D. A quarter of the shops go down, as rarely as before</summary>

✗ A spec sheet gives a failure rate per machine, and each new machine matches the old one. That rate has not changed, and the number of machines has: between them, four fail about four times as often as one, so the company handles far more outages, each stopping whichever quarter sat on that machine.

</details>

---

###### 3. Splitting eight shards by the time a reading arrived

An energy company collects smart meter readings in a store that uses range partitioning over eight shards, with the time a reading arrived as the partition key. A range splits when it grows past a set size, and a balancer keeps the amount of data on the shards about equal. All eight hold about the same this morning.

Which shards take the writes arriving in any one second this afternoon?

<details><summary>A. Only the shard holding the newest range</summary>

✓ Right. In any one second every reading lands on one shard, the one holding the range with the latest times, and the other seven take no writes that second.

Range partitioning keeps neighbouring keys together, and **a key that only rises sends each new row to the top range**. A split cuts that range in two and the readings carry on into the upper half. A balancer that leaves the new top range where it is gives one shard the whole afternoon. One that hands each new top range to another shard makes the shards take turns, and over a long afternoon each ends up with about an eighth of the total, still all of the load on one shard at a time.

Place the readings by a hash of the meter id, and the writes spread over all eight. The price is that a read of one time span then reaches every shard.

</details>

<details><summary>B. All eight shards, about an eighth each</summary>

✗ The balancer's view shows eight shards of equal size, and across keys that arrive at random, equal data means about equal work. A balancer evens out what is already stored, some time after it arrives. A new reading has the highest key the moment it comes in, so each second's readings all go to one range on one shard, and the other seven take none of them.

</details>

<details><summary>C. The emptiest shard, chosen by the balancer</summary>

✗ A file system can put a new file on the disk with the most room, and the balancer here watches how full the shards are. The balancer moves ranges and has no say over a single reading. The key picks the range, the range sits on one shard, and that shard can be the fullest of the eight.

</details>

<details><summary>D. Both halves of the newest range, once it splits</summary>

✗ A split shares the work between its two halves when keys arrive all over the range. These keys only rise. The lower half is finished the moment it is cut, since no later reading can have an earlier time, and every reading after the split falls in the upper half.

</details>

---

###### 4. Doubling the nodes for one concert's ticket page

A ticket site places each event by a hash of its id on six nodes. A concert goes on sale, and its page alone draws 25,000 reads a second, when a node serves 10,000. You double the cluster to twelve nodes and let the store rebalance.

What happens to the concert's reads once the rebalancing ends?

<details><summary>A. Half of them move to a new node</summary>

✗ Doubling a cluster halves the work on each node in any capacity plan, a sum that counts on many small keys. This one key does not divide: about half of the time it moves to a new node and every read goes with it, so the reads travel all together or not at all.

</details>

<details><summary>B. They spread evenly across all twelve nodes</summary>

✗ A hash gives an even spread, and the events land evenly across the twelve. That evenness is a count of keys. The hash sends every read of this one key to the same place, so eleven nodes see none of the 25,000 reads a second.

</details>

<details><summary>C. The balancer moves them to an idle node, where they fit</summary>

✗ Six of the twelve nodes start out empty, and about half of the time the key lands on one of them. An empty node's limit is the same 10,000 reads a second, so the concert overloads it as it did the old one.

</details>

<details><summary>D. All of them still reach one node, over its limit</summary>

✓ Right. All 25,000 reads a second still reach one node, whichever node now holds the key, and that node still serves 10,000, so the concert's page stays overloaded on twice the machines.

**A hot key lives on one node whole**, and rebalancing moves keys whole, so the most a move can do is put the same overload on another node.

Put a short-lived cache of the page in front of the node, or serve it from copies of the key. A cache takes reads and no writes, which suits a page that changes far less often than it is read.

</details>

---

###### 5. Splitting a viral post's like counter over ten keys

A social app stores the like count of one post in one key. The post goes viral at about 2,000 likes and 50,000 views a second, which puts that key's shard at its limit, and each view shows the total. You split the counter: each like now adds to one of ten keys picked at random, on ten shards that held nothing of this post, and no other copy of the total exists. Count a read that reaches several shards as one operation on each.

How busy is each of the ten shards now?

<details><summary>A. A tenth of the likes and a tenth of the views</summary>

✗ Ten keys on ten shards is the cure on every list of hot key fixes, and the likes divide by ten. The views stay whole, because a view needs the total and the total now lies in ten pieces, so each shard gets all 50,000 views a second on top of its 200 likes.

</details>

<details><summary>B. About as many operations as the one hot shard took</summary>

✓ Right. Each of the ten shards takes about 200 likes and all 50,000 views, about 50,200 operations a second against the old shard's 52,000, so one shard at its limit has become ten of them and the post is no closer to fitting.

**Splitting a hot key over several keys spreads its writes**, since each like lands on one of them, and it does nothing for a read that needs them all.

Keep the ten keys for the likes, sum them into one key, and let something other than that key's shard serve the views, since a key read 50,000 times a second is the hot key again. A cache of the total on each app server, or several copies of the total, does that, and either way the count on screen runs a little behind.

</details>

<details><summary>C. Ten times the operations the one hot shard took</summary>

✗ Each view now makes ten reads where it made one, so the reads across the cluster rise tenfold, to about 502,000 operations a second with the likes. Those reads land one per shard, which leaves any one shard at about 50,200, close to what the old shard took and nowhere near ten times it.

</details>

<details><summary>D. A tenth of the likes each, with every view on the first shard</summary>

✗ The view asks for the post's counter as it always did, but the total now lies in ten pieces and no one key holds it, so a read of any single shard would be wrong. The view has to read all ten, and the first shard gets no more than the other nine.

</details>

---

###### 6. Outgrowing the largest machine with a team-chat workspace

A team-chat product uses workspace id as its partition key. One customer's workspace has grown to about five machines of work on its single shard, while most other workspaces sit idle. Reading a channel's newest messages is the main load, a search across a whole workspace is rare, and no channel comes near a machine's limit.

Which key lets this workspace's load spread?

<details><summary>A. Workspace id, on more shards</summary>

✗ Stores split what grows too big, and a range holding many key values can be cut in two. A split falls between key values, and this workspace is one value with all its rows together, so the cut has nowhere to fall and whichever shard holds it carries all five machines of work.

</details>

<details><summary>B. Workspace id plus a one-week time bucket</summary>

✗ A time bucket is the usual cure for a key that grows without end, and it caps how much one key holds. The load here follows the newest messages, and those all sit in this week's bucket, on one shard. The shards holding old weeks sit nearly idle, and this week's shard still takes nearly all of the work.

</details>

<details><summary>C. Channel id in place of workspace id</summary>

✓ Right. A channel's messages stay together, so a channel read still touches one shard, and the customer's five machines of work spread over many.

**A partition key keeps the rows of one value together, however many shards exist**, so this load can spread only under a key that differs inside the workspace. That works because no channel comes near a machine's limit, and a channel that did would hit the same limit on its own shard.

Rewrite the rows under the new key as a move you can check, and accept that the rare search across a whole workspace now asks every shard.

</details>

<details><summary>D. A hash of each message's id</summary>

✗ A message id has a different value on every message, and under that key writes fall evenly across the shards. A channel's messages scatter with them, so every channel read has to ask every shard, each shard takes all the reads and the main load has not been spread at all.

</details>

---

###### 7. Changing the city on a parcel sharded by city

A parcel delivery service places its rows by city, in your own code, over four databases. A parcel bound for Lyon, whose rows sit on shard 2, is rerouted to Paris, which hashes to shard 3. The courier in Paris opens the app and looks the parcel up.

```python
SHARDS = [db0, db1, db2, db3]

def shard_for(city):
    return SHARDS[stable_hash(city) % len(SHARDS)]

def reroute(parcel_id, old_city, new_city):
    shard_for(old_city).execute(
        "UPDATE parcels SET city = ? WHERE id = ?", new_city, parcel_id)

def lookup(parcel_id, city):
    return shard_for(city).query_one(
        "SELECT * FROM parcels WHERE id = ? AND city = ?", parcel_id, city)
```

What does the courier's lookup with the new city return?

<details><summary>A. No parcel, though the row still exists</summary>

✓ Right. The lookup asks shard 3, which has never held this parcel, and gets nothing back, so the courier in Paris finds no parcel while its row sits on shard 2 with its city set to Paris.

**In your code a row goes to the shard its key names only when it is inserted**, and an UPDATE changes a column where the row already is. Some sharded stores do move a row when its key value changes, at the cost of that move, and a plain database knows nothing of the other databases.

Place parcels by parcel id, which never changes, and keep the city as an ordinary column. Where a key has to change, your code must insert on the new shard and delete on the old one, two steps on two databases that can stop halfway.

</details>

<details><summary>B. The parcel with its new city</summary>

✗ On one database an UPDATE followed by a read of the new value always finds the row, and some sharded stores keep that promise by moving the row. In this code the UPDATE ran on shard 2, the row stayed there, and the lookup goes to shard 3.

</details>

<details><summary>C. The parcel with its old city</summary>

✗ Replicas teach you that a read can miss an update for a moment and return the row as it was. No replica sits in this path, and shard 3 has never held the parcel in any form, so the courier gets no row, old or new.

</details>

<details><summary>D. An error saying the row belongs on another shard</summary>

✗ A router that checks which shard owns a key answers a lookup sent to the wrong shard with a redirect. These four are plain databases that know nothing of each other: shard 2 ran the UPDATE because its WHERE clause matched, and shard 3 runs the query it was sent.

</details>

---

###### 8. Adding a fifth node to a cache under mod-N hashing

A photo site keeps thumbnails on four cache nodes, each key placed by its hash modulo the node count, in front of the image store they are filled from, the origin. The cache is warm and traffic is steady. At noon you add a fifth node and update the count in the code.

What does the origin get in the minutes after the change?

<details><summary>A. Misses only for the slice of keys the fifth node took</summary>

✗ The fifth node starts empty and a fifth of the keys now point at it, so those are sure to miss. The formula runs for every key alike, and the remainders change across the old nodes too, so about four thumbnails in five are read from a node that lacks them.

</details>

<details><summary>B. Fewer misses than before, with more cache memory</summary>

✗ More memory raises the hit rate, the share of requests the cache answers, and every sizing sum shows it. Memory was not the limit at noon: every thumbnail was already held. The new node starts empty, and the old copies sit on nodes the reads no longer ask.

</details>

<details><summary>C. A miss for most thumbnails, all at once</summary>

✓ Right. About four thumbnails in five now look on a node that does not hold them, so the origin takes those misses all at once, with traffic no higher than before.

**Mod-N hashing computes the node from the count**, so a new count changes the answer. A key keeps its node only if its hash leaves the same remainder for four and for five, and one key in five does. If 20,000 thumbnails are requested evenly at 1,000 requests a second, about 780 requests a second reach the origin in the first second, and by the fifth minute the moved keys have refilled.

Use consistent hashing with virtual nodes, where the fifth node takes about a fifth of the keys and the rest stay put. That fifth still misses once, while the new node fills.

</details>

<details><summary>D. No change: old thumbnails stay put and new ones go to the fifth node</summary>

✗ In a shard map that keeps its old entries, an old thumbnail stays put and a new one can go to the new node, since placement there is a record written once. Mod-N hashing keeps no record, and the client works the node out again on every read with the new count, so a read of an old thumbnail goes where the new count says while its copy waits on the old node.

</details>

---

###### 9. Losing one of ten cache nodes under consistent hashing

A news site runs ten cache nodes under consistent hashing, each node with one position on the ring and each at about 70 percent CPU. Each key sits on one node, and clients drop a dead node from the ring at once. During the morning peak, one node's power supply fails.

Where do the dead node's keys go?

<details><summary>A. All of them go to the next node around the ring</summary>

✓ Right. Every one of its keys goes to the next node around the ring, so that one node takes the dead node's traffic on top of its own, about 140 percent of its CPU, while the other eight stay near 70.

A key goes to the next position clockwise, and **with one position per node, all of a node's keys share one arc, so they share one next position**. With virtual nodes each server holds many short arcs: at 100 positions each, nine nodes take a slice and a receiver rises to about 78 percent.

Give each server many positions before the day you need them, and keep each node's CPU low enough to absorb its share of a lost neighbour's traffic.

</details>

<details><summary>B. An equal share goes to each of the other nine</summary>

✗ The ring is usually taught as the scheme that spreads load evenly, and with virtual nodes it does. With one position per node, the keys form one unbroken stretch of the circle with one next position after it, so eight of the other nine receive none of them.

</details>

<details><summary>C. They split between the two neighbours on the ring</summary>

✗ In the drawing, a gap on a circle closes from both sides, and the node before and the node after both touch it. The rule runs one way: each key walks clockwise to the next position, so the clockwise neighbour takes all of them and the anticlockwise neighbour takes none.

</details>

<details><summary>D. None are served from the cache while that node is gone</summary>

✗ A key belongs to its node, and a client that keeps a fixed node list would go on failing those reads for as long as the node is gone. These clients drop the node at once, so the next read of each key goes to its new owner, misses once and fills, and later reads of that key are served from the cache again.

</details>

---

###### 10. Adding a seventeenth machine for sixteen logical partitions

An order system runs on a store created with sixteen logical partitions, a count fixed at creation, and each of its sixteen machines holds one partition. The machines are getting full. You add a seventeenth machine and start the rebalancing.

What does the new machine hold?

<details><summary>A. A small share of every partition's orders</summary>

✗ Ring diagrams show a new node taking a sliver from each neighbour, and many stores advertise growth that way. In this store a move carries a whole partition, never part of one, and a key's partition stays what it was at creation, so no slice of orders can be cut off and sent over.

</details>

<details><summary>B. Half of the busiest partition</summary>

✗ Cutting a partition in two is what a store that splits by size or by load does, and the busiest one is where it starts. The count of sixteen here is fixed at creation, so no partition can be split, and the busiest one stays whole on its machine.

</details>

<details><summary>C. A new share of nearly every order, rehashed over seventeen machines</summary>

✗ Under mod-N hashing a new machine changes the count, and nearly every key moves: about 94 percent over seventeen machines. Logical partitions exist to stop that: a key hashes to one of sixteen partitions whatever the number of machines, so adding one changes no order's partition.

</details>

<details><summary>D. An empty disk and not one order</summary>

✓ Right. The new machine stays empty and the rebalancing ends at once with nothing moved, so the order system is as full as it was, with one more machine sitting idle.

**A move carries a whole logical partition**, and each machine already holds only one. Moving one would leave its old machine empty and the fullest machine as full as before, so a balancer has nothing to gain. Growth needs more partitions than machines: with 64 partitions on these sixteen machines, the seventeenth would take 3 or 4.

Create far more partitions than machines when the store is new, at the price each partition costs in memory, metadata and recovery time on every copy. With the count already fixed at sixteen, the way out is a new count, which places every key again.

</details>

---

###### 11. Adding a shard on the day one runs out of memory

A check-in app runs on two shards, in a store whose balancer moves chunks of records in the background while the shards keep serving. Records are far smaller than a storage page, the block a store reads from disk at one time, and were written in order of arrival, unrelated to the key that chunks are cut by. One shard's active data has outgrown its memory and its disk queue is growing. You add a third shard, and the balancer starts moving chunks, a small share of them in the first hours.

What does the full shard get during the first hours?

<details><summary>A. Less traffic and more free memory with each chunk that leaves</summary>

✗ Records leave, and on paper their traffic and their memory leave with them. Memory comes back a page at a time, and each page here holds records from many chunks. A page stays in memory while any record on it is still wanted, so the shard ends the first hours holding nearly every page it started with.

</details>

<details><summary>B. The copying on top of its traffic, and no memory back</summary>

✓ Right. The full shard keeps nearly all its traffic, reads each leaving chunk for the copy and gets no memory back, so its disk queue keeps growing and check-ins stay slow through the hours the third shard was meant to help.

**Rebalancing is work first for the shard giving up the chunks**, and here each chunk's records sit scattered on pages shared with other chunks, so those pages stay in memory after the records leave. Take 300-byte records, 13 to a page, on a shard of 100,000 pages: moving 5 percent of the records empties none of those pages and touches about half of them, and at worst the copy reads the pages it touches from a disk that already cannot keep up.

Add the shard while the full one still has memory to spare. Past that point, more memory or less traffic helps first, and the pages left part empty come back only when the store rewrites them.

</details>

<details><summary>C. The same load as before, with new rows going to the new shard</summary>

✗ The third shard arrives empty, with all its memory free, and the full shard's trouble is too much active data. New rows go where their key says, whichever shard has the memory, so the third shard gets only the new rows whose keys fall in the few chunks it holds so far, and the full shard still has every other row plus the reads that feed the copy.

</details>

<details><summary>D. A refusal of all writes while chunks are moving</summary>

✗ Some table operations inside one database lock what they change until they finish, and a move changes many rows. This balancer copies in the background while the shard serves. Writes pause at most briefly for a chunk as its ownership changes, and the app's other check-ins carry on.

</details>

---

###### 12. Looking up accounts by email on thirty-two shards

A video site shards accounts by user id, and at sign-in it finds the account by email, through a local index on each shard. The site grows from eight shards to thirty-two while sign-ins per second stay level, and the capacity plan divides the sign-in work by the shard count.

How many of the sign-in lookups reach each shard after the growth?

<details><summary>A. A smaller share of the lookups than before</summary>

✗ Dividing by the shard count is the capacity plan's whole method, and it works for lookups that carry the user id. An email does not name a shard, so the router cannot narrow the question down, and each shard still gets every lookup.

</details>

<details><summary>B. Only the lookups for emails it holds</summary>

✗ One shard holds each email, and a global index would send the lookup straight there. A local index covers one shard's rows and is read only after the lookup has arrived, so 31 of the 32 shards do the work of searching and answer that they have no such account.

</details>

<details><summary>C. Every lookup, as it had at eight shards</summary>

✓ Right. Every sign-in lookup still reaches every shard, so the drop to a quarter that the capacity plan expected never comes and each shard does as much sign-in work at thirty-two as it did at eight.

**A local index answers only for the rows on its own shard**, so a lookup by email has no shard to aim at and the router asks them all. At 1,000 sign-ins a second, for example, the work across all shards goes from 8,000 operations a second to 32,000 to find an email that one shard holds.

Keep a global index or a lookup table from email to user id. A sign-in then takes two hops, and each new account takes a write to a second shard.

</details>

<details><summary>D. None: the router answers them from its own map</summary>

✗ A router knows where every row lives as long as the question names the partition key, and that is the job a shard map does. The map here is cut by user id and holds no email, so it has nothing to answer from and passes the lookup on to every shard.

</details>

---

###### 13. Listing the top twenty products across eight shards

A marketplace run by a grocery chain shards its orders by seller over eight shards. Each shard's twenty best sellers are local favourites, a little ahead of staples that sell steadily on all eight. For a weekly report, the router asks each shard for its top twenty products by units, adds up the units per product and keeps the top twenty.

What does the weekly report's list hold?

<details><summary>A. A list of local favourites, and the staples missing</summary>

✓ Right. The report lists local favourites and leaves out staples that outsell each of them across the chain, so the list is missing the chain's best sellers.

**The router adds up only what the shards return**, so a product ranked a little below twentieth on every shard is dropped before any sum happens. Take a case built to show it: 25 staples at 100 units on every shard, 800 in all, under 20 favourites per shard at 110. The report shows no staple and each favourite at 110. Ordinary data drops fewer products than this case does.

Ask each shard for far more than twenty, which shrinks the loss without ruling it out. For an exact list, have each shard return its totals for every product.

</details>

<details><summary>B. The true top twenty of all the shards, a little more slowly</summary>

✗ Merging sorted lists gives a sorted list, and for rows it works: the top twenty rows of eight shards hold the true top twenty. A product's total is a different thing, with its units in eight places. A shard that leaves a staple out of its twenty leaves out that part of the sum, and here all eight do.

</details>

<details><summary>C. The staples at the top, each with too few units</summary>

✗ A total built from short lists can come up short, and a product that makes the cut on three shards and misses on five shows too few units. The staples in this report make the cut nowhere, since every shard's twenty are its favourites, so no staple reaches the sum and every figure in the report is a favourite's own.

</details>

<details><summary>D. No list: the router refuses to sort across shards</summary>

✗ A transaction across shards or a unique index over them is hard to come by once data is sharded. Sorting stays possible, since the router holds eight short lists in its own memory, adds and sorts them itself and returns a list with no complaint.

</details>

---

###### 14. Listing a person's data across shards for an erasure job

A job board keeps applications on 24 shards, behind a router that gives each query a time budget of 200 ms. When someone asks for their data to be erased, a job lists their applications through the router, deletes each one and closes the request. The code opens with the router guide's sample reply to a query that two shards did not answer within the budget. While the job runs for one person, a shard holding some of their applications takes five seconds to answer.

```python
SAMPLE_REPLY = {"status": 200, "rows": [...], "shards": {"total": 24, "failed": 2}}

def erase_person(person_id):
    reply = router.query(
        "SELECT id, shard FROM applications WHERE person_id = ?", person_id)
    if reply["status"] != 200:
        raise ErasureFailed(person_id)
    for row in reply["rows"]:
        delete_application(row["shard"], row["id"])
    erasure_requests.close(person_id)
```

What do the erasure record and the shards hold afterwards?

<details><summary>A. A failed job, stopped by the slow shard's timeout</summary>

✗ One call with a timeout fails with an error when its deadline passes, and this job has a branch ready to catch a bad status. The router's own status stays at 200 with the slow shard counted under `failed`, as in the sample, so the branch that stops the job never runs.

</details>

<details><summary>B. A job still waiting for the slow shard, its request open</summary>

✗ A query on one database is done when all of it is done, and a slow part holds up the whole. The router's budget ends this wait at 200 ms: the reply goes back to the job on time, long before the five-second shard has anything to add.

</details>

<details><summary>C. A closed request, and that shard's applications still stored</summary>

✓ Right. The record says closed and the applications on the slow shard are still stored, so the request reads as done while part of the person's data stays.

**This router answers with what arrived within its budget and reports the gap in a count**, and the job reads `status` and `rows` and never looks at `shards`. If the person has 40 applications, 7 of them on that shard, the job deletes 33 and closes. Routers differ here: some return partial rows by default, some only when asked, and others return an error.

Treat `failed` above zero as not done: delete nothing, keep the request open and run it again. Where the router offers it, ask for an error in place of a partial reply on any job that has to be complete.

</details>

<details><summary>D. A closed request, and every application gone, listed from another shard's copy</summary>

✗ Copies of a shard let a read fall back to one when the shard itself is slow. These shards have no replicas, so the slow shard's rows exist nowhere else, and they cannot be listed until it answers. The job never asks again once it has closed the request.

</details>

---

###### 15. Searching by colour for a seller's new listing

A second-hand clothes marketplace shards listings by id and keeps a global index by colour on other shards. Saves take no longer than they did before the index was added. A seller saves a red coat, and the next screen, loaded the moment the save returns, shows the coat's page beside the results of a red search.

Where does the seller find the coat on that screen?

<details><summary>A. On its page and in the red search</summary>

✗ On one database an index changes in the same transaction as its table, so whatever is saved is searchable. This index sits on other shards, and the save returned before its entry got there, so the red search has not heard of the coat yet.

</details>

<details><summary>B. In neither, as long as the index lacks it</summary>

✗ One save from the seller produces both the row and its index entry, and the next screen draws on both. The save waited only for the row: it came back when the listing's shard had the row, so the coat's page opens now, whatever state the index is in.

</details>

<details><summary>C. In the red search, not yet on its page</summary>

✗ A replica trails its primary, so one copy of the data can run ahead of another, and a search index is built to answer fast. The order runs the other way here. The row is written before the save returns, and the index entry follows it.

</details>

<details><summary>D. On its page, not yet in the red search</summary>

✓ Right. The coat's page opens, since its row is on its shard, and the red search does not list it yet, so the seller sees a live coat missing from the results beside it.

A global index keeps its entries on other shards, and in this store **the save returns before the entry is written there**, which is why saves take no longer. With an index that catches up after 300 ms, to pick a figure, the search finds the coat from then on, and nothing went wrong in between. Stores differ on this, and one that makes a write wait for its global index would show the coat in both places at the price of a slower save.

Open the seller's own listing from its row after a save, and treat the search as a view that catches up a moment later.

</details>

---

###### 16. Keeping a unique email column on accounts sharded by user id

A language-learning app shards accounts by user id, and its schema went to every shard unchanged, so each shard has its own unique index on email. A learner signed up with an email last week. Today someone signs up with that email again, and the new user id lands on a different shard from the first account.

What does the second sign-up with that email get?

<details><summary>A. An account of its own, with the same email</summary>

✓ Right. The second sign-up gets an account of its own, so the app now holds two accounts under one email, and a sign-in by email has two rows to choose from.

**Each shard checks its own rows**, and this row lands where the old one is not. A repeat is refused only when its new user id happens to land on the shard that holds the first account, so more repeats get through as the shards multiply: about half at 2 shards and about seven in eight at 8.

Claim the email first in a separate table whose partition key is the email, so every repeat lands on one shard and the unique index there can refuse it. Then the claim and the account are two writes on two shards, and your code has to handle one succeeding alone.

</details>

<details><summary>B. A duplicate-email refusal from the unique index</summary>

✗ A unique index on one database is the guarantee, and the migration copied it to every shard. A copy checks only the rows beside it. The new row's shard has never seen this email, so its index finds nothing to object to and the insert succeeds.

</details>

<details><summary>C. A short wait while the shards compare, then a refusal</summary>

✗ The machines of a cluster coordinate on plenty, from who leads to who holds which range. The shards here are plain databases, with no insert step that asks another one, so this insert commits on its own shard as fast as any other and there is no wait to see.

</details>

<details><summary>D. The first account overwritten with the new details</summary>

✗ An upsert replaces a row when its key is taken, and sign-up code is sometimes written that way. This insert goes to a different shard, and the first account's shard receives nothing at all, so last week's row stays as it was and a second row appears on the new shard.

</details>

---

###### 17. Nesting two transactions for a transfer across shards

A wallet app moves money between accounts on two shards. The sender holds 100, the receiver 0, and the transfer is 30. Your framework's `transaction()` opens a transaction on that shard's own connection and commits it when its block ends. The sender's shard restarts after the inner block has ended and before the outer block ends.

```python
def transfer(sender, receiver, amount):
    with shard_for(sender).transaction() as debit:
        debit.execute(
            "UPDATE wallets SET balance = balance - ? WHERE id = ?", amount, sender)
        with shard_for(receiver).transaction() as credit:
            credit.execute(
                "UPDATE wallets SET balance = balance + ? WHERE id = ?", amount, receiver)
```

What do the two accounts hold afterwards?

<details><summary>A. Both balances as they were</summary>

✗ Nested blocks in one function are written as one unit of work, and the indentation ties them together on the screen. The inner block's commit reached the receiver's shard before the restart, that shard stayed up, and nothing in this code can take a commit back, so the receiver keeps the 30.

</details>

<details><summary>B. The receiver has the money, and the sender still has it too</summary>

✓ Right. The receiver holds 30 and the sender still holds 100, so the wallets together hold 130 where they held 100.

Each block is its own transaction on its own database, and **two commits on two shards are two separate events**. The credit was final when the inner block ended, and the debit was lost uncommitted in the restart.

Keep balances that move together on one shard where the product allows it. Where it does not, write the transfer down first as a record, with steps that are safe to repeat and a job that finishes any left half done. Until that job runs, a reader can still see the half-done state.

</details>

<details><summary>C. The sender has lost the money, and the receiver has nothing</summary>

✗ The debit comes first in this code, and a vanished debit is the failure people fear first in a transfer. Its commit comes last, since the outer block ends after the inner one. The restart throws the debit away and leaves the sender at 100, and the credit has already committed.

</details>

<details><summary>D. The transfer applied on both, once the shard comes back</summary>

✗ A database replays its log at restart, and work that reached the log comes back whole. The debit never reached a commit, so the log has nothing to bring back, and no code runs the outer block a second time. The sender's shard returns with the balance still at 100.

</details>

---

###### 18. Copying rows to a new shard while edits keep arriving

A document editor moves one customer to a new shard. A bulk copy of the old rows and a replay of changes run at the same time, and both save to the new shard as plain writes. For one row, the log shows these steps, and nobody edits that row again.

```text
tick 10   copy reads the row at version 1 from the old shard
tick 11   a user's edit makes version 2 on the old shard
tick 12   replay writes version 2 to the new shard
tick 13   copy saves what it read to the new shard
```

What does the new shard hold for that row when both finish?

<details><summary>A. The edited row</summary>

✗ A log of changes carries the newest data there is, and migration write-ups list copy, replay and cutover as three clean steps. The replay wrote version 2 at tick 12, one tick before the copy's save replaced it, and by then the replay has passed this row.

</details>

<details><summary>B. Both versions, side by side</summary>

✗ A store with version history keeps both, and you can ask it for either one. A plain write holds one value per row, whichever arrived last, so the new shard has a single row here and it carries version 1.

</details>

<details><summary>C. The row as it was before the edit</summary>

✓ Right. The new shard ends on version 1 while the old shard holds version 2, so once the customer's traffic moves over, the user's edit has gone from the document, and both jobs report success.

**A plain write keeps whichever came last**, and the copy's write came last while carrying older data. It read the row at tick 10 and held that old copy through the edit and the replay. The replay moves forward through the log and does not return to a row it has written, so nothing corrects the row afterwards.

Keep a version on each row and have both jobs write only when their version is higher, so the copy's version 1 is dropped when it reaches a row already at 2. Then compare the rows on both shards before the cutover.

</details>

<details><summary>D. No row: the clash stops the copy</summary>

✗ A unique key stops a second insert of the same row, and a move writes the same row twice. These two jobs make plain writes, which do not clash: the later one overwrites the earlier one, the copy carries on and nothing is reported.

</details>

---

###### 19. Flipping the shard map when replay shows no lag

An invoicing service moves one customer to a new shard. Replay lag reads zero, so you change the customer's entry in the shard map and stop the replay, with no pause in writes. Forty application servers each keep a copy of the map, which they reload every thirty seconds at their own moments. The old shard stays up.

Where do the customer's invoices from the next half minute end up?

<details><summary>A. All of them on the new shard</summary>

✗ The map is the truth and one write changed it, so from where you stand the flip is instant. Each server works from its own copy, and that copy stays old until its next reload, so for up to thirty seconds some servers are still sending invoices the old way.

</details>

<details><summary>B. Two copies, one on each shard</summary>

✗ Writing to both sides is a standard safety step during a move, and this customer is in mid-move. Each server here sends an invoice to the one shard its map names, and the replay that could have carried it across has stopped.

</details>

<details><summary>C. None saved: the old shard refuses them and customers see failures</summary>

✗ The customer has left the old shard, the map says so, and in some stores a node refuses a write for a key it no longer serves. This old shard is a plain database with no idea the map changed. The insert succeeds, the server reports success and the customer sees nothing wrong.

</details>

<details><summary>D. Some on the old shard, the rest on the new</summary>

✓ Right. Invoices land wherever each server's copy of the map points, so those from servers yet to reload go to the old shard after the replay has stopped, and the customer's invoices on the new shard have gaps.

**A cutover is one write to the map's store and forty separate reloads on the servers**, and until its reload a server sends invoices the old way. Say each server creates one invoice a second for this customer and the reloads are spread evenly over the thirty seconds: about half of the 1,200 in that half minute land on the old shard. Replay lag goes on reading zero, since it measures what the replay still has to carry and the replay has stopped.

Pause this customer's writes through the flip and hold them until the last server has the new map. Or have the old shard check ownership and refuse the customer's writes, so servers retry on the new owner. Either way nothing lands on the old shard.

</details>

---

###### 20. Keeping a moved customer's rows on the old shard for a week

A loyalty-points firm moves one customer to another shard this morning. The replay stops at the cutover, the source rows stay on the old shard untouched for a week as a way back, and the customer goes on earning points on the new shard. The app finds each customer's shard through the shard map. Tonight the nightly report job runs as usual.

```python
def nightly_points_report():
    totals = {}
    for shard in SHARDS:
        rows = shard.query(
            "SELECT customer_id, SUM(points) AS points FROM points GROUP BY customer_id")
        for row in rows:
            totals[row.customer_id] = totals.get(row.customer_id, 0) + row.points
    return totals
```

What does tonight's report show for that customer?

<details><summary>A. The customer's right total, counted once</summary>

✗ The map says where the customer lives, and every read the app makes goes through it, so the old rows could as well not exist. The job is the one reader that skips the map. It reads the old shard like any other and adds what it finds there to the live figure.

</details>

<details><summary>B. The customer's moved points counted twice</summary>

✓ Right. The moved points count twice, so tonight's figure stands far above what the customer holds. Say 1,200 points were copied and 50 more have been earned on the new shard: the app shows 1,250, and the job adds up 2,450.

**The shard map steers only the reads that use it**, and this job asks every shard. The source rows are ordinary rows in an ordinary database, so the job adds them in.

Have the job read the map and count each customer only on the shard the map names, and delete the source rows when the week is out. Run that clean-up from the current map: after a rollback, a clean-up still holding the map that named the new shard deletes the rows on the old shard, which is live again, and the app finds nothing for the customer.

</details>

<details><summary>C. Every one of the customer's points counted twice, today's as well</summary>

✗ A way back has to hold today's points to be any use, and a move can keep it up to date by replaying new writes back to the old shard. This move sends nothing back: the replay stopped at the cutover and the old rows stand as they were then. Points earned since exist only on the new shard, and the report counts those once.

</details>

<details><summary>D. The customer's moved points missing from the total</summary>

✗ A report that visits shards one after another can miss rows that move while it runs: they leave a shard it has yet to read and arrive on one it has already passed. Nothing moves under this report: the copy finished in the morning and the source rows have stayed where they were, so the loop finds them on both shards and reads them twice, with none lost.

</details>

---

###### 21. Following a youth match on the cup final's shard

A sports app keeps the live score of each match under its id, placed by a hash over twelve shards. Each shard has three replicas (copies of its data), reads spread evenly across them, and a replica serves 10,000 reads a second. You never set a cap on how many reads a replica may hold waiting. On cup final night the final alone draws 40,000 reads a second, sent on a timer by the app. A youth-league match that draws a few reads a minute happens to hash to the same shard, and a parent keeps refreshing it.

What does the parent see as the final goes on?

<details><summary>A. The score as fast as on any other night</summary>

✗ The two matches are different rows, and no lock or transaction ties them together. They do share machines: all three replicas hold both matches, the final fills all three, and the youth match's read waits among the final's reads.

</details>

<details><summary>B. The score a little slower than usual, at one steady pace</summary>

✗ On a capacity chart the final asks for a third more than the 30,000 reads a second the replicas serve, a modest overrun. A replica past its limit serves no faster and leaves the rest waiting, so the next refresh joins a longer line than the last one did.

</details>

<details><summary>C. No score at all, while the final's score keeps loading</summary>

✗ The final's score sits in memory, and the youth match's score may have to come from disk. A disk read costs that one read some extra time. A worker takes the next waiting read without asking how popular its key is, so the youth match's read gets its turn like any other, and the final's reads wait the same time.

</details>

<details><summary>D. The score taking longer with each refresh</summary>

✓ Right. Each refresh takes longer than the one before, and the parent ends up waiting behind reads of the final for a score almost nobody else reads.

The final and the youth match sit on the same three replicas, and **reads of all the keys on a replica wait for the same workers**, whichever key they ask for. A replica serves at most 10,000 reads a second however many arrive. The final sends each one about 13,300 a second, so about 3,300 more are waiting with every second that passes, and ten minutes in a new read waits behind about two million others, over three minutes. With no cap, that line grows for as long as the final draws more than the replicas serve. Once the wait passes the app's own timeout, the parent gets an error in place of the score.

Cap the waiting reads on each replica, so it refuses a read rather than hold it for minutes, and set a limit per key at the router, so the refusals fall on the final's excess. A short-lived cache of the final's score then takes most of those reads off the shard.

</details>

---

## Further reading

- [Consistent hashing and random trees](https://doi.org/10.1145/258533.258660), 1997: where consistent hashing is defined, with the rule that adding or removing a machine moves the least needed to keep the load balanced.
- [Consistent Hashing with Bounded Loads](https://doi.org/10.1137/1.9781611975031.39), 2018: a cap on each server's load near a chosen multiple of the average, with few moves per change, for the case where plain consistent hashing leaves some servers heavily overloaded.

Prev: [Replication](../replication/README.md) · Next: [Caching](../caching/README.md)
