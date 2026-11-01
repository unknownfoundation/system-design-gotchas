# Replication

Prev: [Distributed locks and IDs](../distributed-locks-and-ids/README.md) · Next: [Sharding and partitioning](../sharding-and-partitioning/README.md) · Reviewed October 2026

Replication keeps copies of the same data on several machines: where one machine takes a write, the others receive it through a log of changes and apply it. It decides what a read can return, which saves survive the loss of a machine, and what a copy can give back after a mistake.

- **Primary**: the node that takes the writes, also called the leader. Its log records every change in order, each at a position.
- **Replica**: a node that keeps a copy by applying the primary's log. It can serve reads, wait to be promoted, or both.
- **Replay**: applying the log. A replica first receives the log and then replays it, and a read does not see what is only received. Replay can run with less parallelism than the primary's writes.
- **Replication lag**: how far a replica's replay trails the primary.
- **Failover**: promoting a replica to primary when the old one is lost, and moving clients to it. A wait for a replica protects a save only if the failover promotes a copy that holds it. Afterwards each side can hold writes the other lacks, so failing back loses one side's writes unless someone merges them.
- **Read your writes**: the promise that a person sees their own save on every later read, on any device, which on a lagging replica needs state kept with the account.
- **Monotonic reads**: the promise that a later read shows nothing older than an earlier one did.
- **Split brain**: two nodes acting as primary at once, each taking writes of its own.
- **Majority**: more than half of a group's nodes, such as 3 of 5. Any two majorities share a node.
- **Quorum**: the number of a key's replicas a write waits for, or a read asks. If reads plus writes exceed the replica count, the replicas a read asks and the replicas a write reached share at least one.
- **Last write wins**: a rule that keeps the value with the latest stamp. Of two concurrent writes, one is dropped and nobody is told.

Who takes writes sets what copies can disagree about:

- **One primary**: every write goes through one node, so copies lag but no two writes clash.
- **Multi-leader**: several nodes, often one per region, take writes and copy them to each other. A clash on one record is settled before the acknowledgement, by asking the other leaders, or after it, by a rule such as last write wins.
- **Leaderless**: a client writes to several replicas and reads from several. A store of this kind mends its copies by read repair, where a read that finds copies disagreeing writes the newest value back, and by anti-entropy, a background comparison run by the store or an operator, which checks copies whether or not anyone reads them.

A primary ships its log in one of three forms: statement-based replication sends the statements it ran, physical replication the storage-level changes, and logical replication the row values each change produced. A statement replayed can store something else: a random value is drawn again, and a function that reads the clock at replay gets a later time.

What a save can wait for before it is acknowledged:

| What a save waits for | Often called | What it adds to the save's time |
| --- | --- | --- |
| The primary's own disk | asynchronous | nothing beyond that disk write |
| Any one replica | semi-synchronous | one trip to whichever replica answers first |
| One named replica | synchronous | one trip to that replica, however far away it is |
| One named replica, with a limit on the wait | synchronous with a timeout | the same trip, never longer than the limit |
| A majority | majority acknowledgement | trips to the nearest copies that complete a majority |

Unless a question says otherwise, a store here has one primary and asynchronous replicas that replay whole transactions in the primary's commit order, and what a replica lacks when a failover starts has not reached it.

The trouble sits between what a primary has acknowledged and what one given copy holds at that moment.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- move reads, a long report or a page reload onto replicas, and say what each reader gets and what a lag graph or a catch-up plan tells you
- follow acknowledged saves and live traffic through a failover, whether the primary waits for a replica, for a limited time or not at all
- decide what a copy can give back or cost after a wrong delete, a replica left off or a skipped change
- trace a username, a delivery slot or a saved setting when several regions or no primary take the writes, and set how soon a cluster re-copies a silent node's data and where it places copies

###### 1. Adding four read replicas before a sale

At a ticket shop, about a fifth of the primary's work is reads. The machine runs out of CPU at every big sale and saves time out, and the last peak asked for about twice what it can do. Before the next sale you add four replicas and send every read to them.

What happens to the saves when the same peak comes back?

<details><summary>A. They spread across the five copies, and the timeouts end</summary>

✗ Comparison tables say a replicated store scales out, and five machines hold five times the CPU. Each save still runs on the primary, and every replica replays it as well, so the four new machines take none of the saves off the primary.

</details>

<details><summary>B. They stay on the primary, and most timeouts remain</summary>

✓ Right. Every save still runs on the primary, which has lost only its reads. Buyers at the next sale meet most of the timeouts they met at the last one.

Moving the reads frees the read fifth of the primary's work and nothing else, because **replicas share the reads while each of them replays every write**. Say the excess is what times out: at twice what the machine can do, three-quarters of the saves that timed out still time out.

Measure the split of reads and writes before you buy replicas. When writes are what runs out, reduce the write work or split the writes across several primaries by sharding.

</details>

<details><summary>C. They slow down further, the primary waiting for four copies</summary>

✗ A synchronous replica adds its round trip to every save, and here there would be four of them. These four are asynchronous: the primary acknowledges off its own disk and ships the log afterwards, so a save takes as long as it did, and the shortage stays one of CPU.

</details>

<details><summary>D. They stay on the primary, which now has room for them all</summary>

✗ Scaling checklists answer a struggling primary with read replicas, and this primary has handed over every read. Those reads were a fifth of its work, so at twice what it can do the saves alone still ask for more than the machine has, and whatever does not fit goes on timing out.

</details>

---

###### 2. Moving yesterday's sales report to a replica

Every night at 02:00 a finance report sums yesterday's closed orders. It has been slowing the primary, so you move it to a replica that usually runs thirty seconds behind. The next morning the query has finished without trouble, and the finance lead opens the report.

What totals does the finance lead see?

<details><summary>A. Yesterday's totals, the same as on the primary</summary>

✓ Right. The finance lead gets the same totals the primary would give, and the primary is spared a long scan.

The report sums closed orders up to midnight and runs two hours later. **Lag hides only what is newer than the lag**, and every change to those orders has long since been replayed, whole and in the primary's order. The report first misses an order when the lag passes those two hours.

Send reports on settled data to a replica, and have the job check against the primary that the replica has replayed past midnight before it starts, so that a bad night stops the run and finance gets no short total.

</details>

<details><summary>B. Totals short by the last thirty seconds of yesterday</summary>

✗ A report on the last hour comes up short by what the lag hides. Thirty seconds of lag at 02:00 reaches back nowhere near midnight, where this report's orders stop, so no order is missing.

</details>

<details><summary>C. Totals holding half-applied orders, read in the middle of replay</summary>

✗ Changes are being applied while the query runs, and a batch job interrupted halfway leaves rows half done. Replay applies each transaction whole, so a read gets an order with all its lines or none of them, and these orders finished replaying hours ago.

</details>

<details><summary>D. Totals that change from run to run with the lag</summary>

✗ A count of today's orders on a replica moves with the lag, up one run and down the next. Closed orders no longer change, so yesterday's sum comes out the same whether the replica is a second behind or an hour.

</details>

---

###### 3. Running a six-hour report on the replica behind the product pages

A shop's six-hour report reads the orders table, so you move it off the primary to the one read replica, which also serves the product pages. You set that replica never to cancel a running query for replay's sake. Soon after the report starts, a migration on the primary adds a column to the orders table, and price changes keep coming all night.

What do shoppers get from the replica while the report runs?

<details><summary>A. Fresh prices, as if no report were running</summary>

✗ Moving heavy reads off the primary is the usual advice, and one read does not block another. The replica has a writer as well, and that writer is replay: the report stops it at the column change, and the product pages go on reading a copy that has stopped moving.

</details>

<details><summary>B. Stale prices, and saves stuck on the primary as well</summary>

✗ A primary is held up by a synchronous replica, since it waits for that replica's confirmation. It waits for nothing from an asynchronous one, so saves take as long as they did, and the damage stops at the replica's readers.

</details>

<details><summary>C. Prices falling further behind as the hours go by</summary>

✓ Right. A shopper can see a price the shop changed hours ago, because prices on the replica fall further behind for as long as the report runs and are about six hours old by its end.

Adding a column needs the table to itself and the report holds the orders table open, so replay stops at that change until the report ends. **A replica replays its log in order, so everything behind a stuck change waits**, every price change included. A replica set the other way cancels the report after a delay and keeps the prices fresh, so this setting is a choice between the report and the pages.

Give a long report a replica of its own that no shopper reads. That copy still falls behind during a migration, and nobody's page depends on it.

</details>

<details><summary>D. Fresh prices, with only the new column held back until the report ends</summary>

✗ On the primary, a lock on the orders table leaves changes to other tables free to pass, and prices live elsewhere. Replay has one queue, the log in its order. A price change behind the column change cannot overtake it, so it waits for the report as the column does.

</details>

---

###### 4. Reading from the primary for a minute after a save

A delivery app reads from a replica, and after a save a cookie sends that customer's reads to the primary for one minute. Replica lag is usually under a second, and every test showed customers their change. Tonight a bulk import has the replica five minutes behind. A customer changes an address and reloads the page after thirty seconds and again after ninety.

What does the customer see at the two reloads?

<details><summary>A. The new address both times</summary>

✗ A minute is far longer than the usual lag, and every test passed inside it. Tonight's lag outlasts the minute, so the second reload comes after the cookie has run out and lands on a replica that has not got the save.

</details>

<details><summary>B. The old address both times</summary>

✗ Five minutes behind means lacking the new address, and any read on that replica gets the old one. The reload at thirty seconds never goes there: the cookie is still live and sends it to the primary, which has the save.

</details>

<details><summary>C. The new address, and then the old one again</summary>

✓ Right. The first reload shows the new address and the second shows the old one, so the customer sees a save undo itself.

The cookie is an attempt at read your writes, and **a fixed window is a guess at the lag**. At thirty seconds the read goes to the primary. At ninety the window has ended, and the read goes to a replica that will not have the save for another few minutes. Any lag up to ninety seconds shows the new address both times, which is why the tests never caught it.

Keep the save's log position and compare it with the position the replica has replayed, sending the read to the primary while the replica is behind. Keep that position with the account, since a cookie does not follow the customer to another device.

</details>

<details><summary>D. The new address, and then a spinner while the replica catches up</summary>

✗ A page that knows a save is on its way can wait for it, and some apps carry a log position to do that. The reload at ninety seconds carries nothing to wait on. Its cookie has expired, so the replica treats it as any other read and answers at once with what it holds.

</details>

---

###### 5. Refreshing a comment thread across three replicas

On a news site, a balancer sends each page view to any of three replicas, whose lag differs by a few seconds at the moment. A reader refreshes a thread and sees a comment posted a moment ago, then refreshes twice more within a few seconds.

What can the reader see on the next two refreshes?

<details><summary>A. The comment on both refreshes, whichever replica answers</summary>

✗ On a single database a comment you have read is still there on the next refresh. Here that refresh can go to any of the three, including one further behind than the replica that showed the comment. The comment can stay, and nothing promises that it will.

</details>

<details><summary>B. The comment gone, and then back again</summary>

✓ Right. The comment can vanish on one refresh and return on the next. For those few seconds the reader may take it for deleted.

Each refresh goes to any of the three, and **a read can land on a replica further behind than the one before it**, and the replica it lands on has not replayed the comment yet. The refresh after that can land on one that has. The promise this breaks is monotonic reads.

Pin a reader to one replica, or send the position last seen with each read so that a replica further behind does not answer. A pinned reader still moves when that replica leaves, and only the position survives the move.

</details>

<details><summary>C. The comment twice, once from each of two replicas</summary>

✗ A feed merged from several sources can show an item twice, and three copies of the thread exist. Each page view reads one replica, which holds the comment once or not yet, so no page carries it more than once.

</details>

---

###### 6. Changing a plan from a profile read on a replica

A billing app saves profile changes through the handler below, which reads from a replica as most of the app does. `primary.put` replaces the stored record with what it is given. Today the replica is five seconds behind. A customer changes a phone number, that save reaches the primary, and two seconds later the customer changes the plan.

```python
def change_plan(customer_id, plan):
    profile = replica.get(customer_id)
    profile["plan"] = plan
    primary.put(customer_id, profile)
```

What does the primary hold for this customer after both saves?

<details><summary>A. The new phone number, beside the new plan</summary>

✗ The handler's one assignment touches the plan, and no line of it mentions the phone. `primary.put` takes the whole profile, and that profile came from a replica without the new number, so the old number travels back with the plan.

</details>

<details><summary>B. The old phone number at first, and the new one once the replica catches up</summary>

✗ In the usual story of lag, a copy falls behind, catches up and things come right. Catching up only brings the primary's latest record to the replica, and that record now holds the old number, so the replica ends on it as well.

</details>

<details><summary>C. The new phone number, with the plan change refused</summary>

✗ A store that compares versions refuses a write built on an old read. The `primary.put` in this handler compares nothing and replaces the record, so nothing stops the write.

</details>

<details><summary>D. The old phone number, with the new plan</summary>

✓ Right. The primary ends up with the old phone number and the new plan, and the customer gets no warning that the account is back on a number replaced two seconds earlier.

`replica.get` returns the profile from before the phone change, the handler sets one field, and `primary.put` sends every field back, the old phone among them. The primary cannot tell an old value from a chosen one, so **a stale value written back becomes the primary's current value**, and replay carries it out to the replica as well.

Read from the primary when a read feeds a write, or write only the field you changed. A version check also protects the number, by refusing the plan change, so your code has to read again and retry.

</details>

---

###### 7. Reading a lag graph at zero all night

An order tool plots a lag graph: the last log position its replica has received minus the last it has replayed. At 19:00 a network change goes in and stays in place, orders keep coming, and the replica's network chart has been flat at zero ever since. At 03:00 the graph reads zero and no alert has fired.

What is the replica missing at 03:00?

<details><summary>A. Nothing: it holds every order placed</summary>

✗ A lag of zero means nothing is waiting, which is what the word promises. Nothing is waiting on this replica because nothing has reached it since 19:00, and the orders sit on the primary.

</details>

<details><summary>B. The last few seconds of orders</summary>

✗ A healthy link leaves a few seconds of orders in flight, and that is what lag usually means. A cut link carries nothing at all, so the gap holds every order since the cut, and it keeps growing.

</details>

<details><summary>C. Every order placed after 19:00</summary>

✓ Right. The replica holds no order placed after 19:00, so anyone reading it at 03:00 sees the orders stop eight hours ago while the graph reads zero.

The network change cut the link, so nothing more arrives, replay finishes what did arrive, and received minus replayed falls to zero and stays there. **Two positions on the replica compare it only with itself**, and lag is a distance from the primary, which they cannot see.

Compare the replica's position with the primary's, or have the primary write a heartbeat row on a schedule and alert on its age at the replica, which here would read eight hours.

</details>

<details><summary>D. Only the minutes after its last automatic reconnect</summary>

✗ Replication streams retry the connection on their own and pick up from where they stopped once it succeeds. The link has not returned: the network change is still in place, so no reconnect has happened and nothing after 19:00 has arrived.

</details>

---

###### 8. Falling an hour behind on a replica at the daily peak

After a rebuild, a replica joins an hour behind the primary, in the middle of the daily peak. Say replay on this machine runs a fifth faster than today's write rate, and writes hold at that rate. The replica serves no reads yet, and the on-call engineer plans to give it read traffic as soon as it has caught up.

When can the engineer expect the replica to catch up?

<details><summary>A. After several hours</summary>

✓ Right. The replica needs about five hours, and the read traffic the engineer planned to hand over stays on the other copies until then.

**A replica gains on the primary only at the spare rate, replay minus the writes still arriving**: at a fifth above the write rate, each hour of replay clears a fifth of an hour of the gap, so one hour takes five.

Keep reads off it until it has caught up, since reads take CPU from replay and shrink the spare rate, and estimate from the spare rate rather than from the size of the gap.

</details>

<details><summary>B. In a few minutes</summary>

✗ Replay skips parsing, checks and client round trips, so it is often cheaper than the first write. Clearing this gap in ten minutes would take about seven times the write rate, and this machine has a fifth to spare.

</details>

<details><summary>C. After about an hour</summary>

✗ The gap measures one hour, and replaying it takes nearly an hour on a machine that does nothing else. More arrives on this machine the whole time: while it replays the old hour, the primary adds nearly another hour of writes behind it.

</details>

<details><summary>D. Never, while saves keep arriving</summary>

✗ With replay at or below the write rate the gap never closes, since no copy can gain on that. Replay on this machine beats the write rate, so the gap shrinks every hour, only slowly.

</details>

---

###### 9. Losing the primary with two seconds unsent

A shop's primary acknowledges an order once it is on its own disk, and its one replica runs about two seconds behind. The primary's machine dies, the replica is promoted and the shop carries on. Confirmation emails went out for the last two seconds of orders. The next day the repaired machine, its disk intact, goes through the store's rejoin step and follows the new primary as a replica.

Where are those two seconds of orders once the machine has joined?

<details><summary>A. On every copy, sent across when the old machine returned</summary>

✗ A sync between two folders sends the newer files both ways, and the old disk holds those orders. A returning primary is not syncing: it follows the new primary's history and offers nothing back, so those orders never reach the others.

</details>

<details><summary>B. On no copy the shop reads from</summary>

✓ Right. Customers holding a confirmation email have orders the shop cannot find, since no copy it reads holds them.

An acknowledgement here waits for the primary's own disk and nothing else, and the promoted replica never had those orders. The failover made that replica's log the history every copy follows, and to follow it **a returning machine has to remove whatever it holds past the point where the two histories part**. Some stores write the removed entries to a file that nothing replays by itself, and others rewind and drop them.

Have the orders you cannot lose wait for a replica. After any failover, look for that file before you close the incident, since on a store that writes one it holds the only copy.

</details>

<details><summary>C. On the old machine alone, where anyone reading it sees them</summary>

✗ Two copies that have parted each keep what they have, and that holds for as long as the old machine stands apart. Joining ends that, since to replay the new primary's log the machine gives up its two seconds, and a read there no longer finds them.

</details>

<details><summary>D. In a list of conflicts waiting for someone to approve them</summary>

✗ Multi-leader stores can hold two versions of a record for someone to settle, and two histories exist here as well. The promotion has already chosen one, and no step compares them or waits for anyone's approval.

</details>

---

###### 10. Holding order numbers in a warehouse from before a failover

A shop's order numbers come from a counter on the primary, kept with the data it replicates. In the last few seconds before the primary dies, the warehouse system takes forty new orders by their numbers. The replica, those few seconds behind, is promoted and goes on taking orders. The next day the warehouse looks up one of those forty numbers to print a label.

What does the warehouse get for that number?

<details><summary>A. No order under that number</summary>

✗ A lost row is the harm failover guides warn about, and the first order did go with the old primary. Its number did not go with it: the new primary's counter reached that number again the same day, so the lookup finds whoever got it second.

</details>

<details><summary>B. The original order</summary>

✗ The warehouse holds a good number, and the shop came back from the failover and kept trading. The original order never reached the replica, so no copy the warehouse can ask holds it, and that number now belongs to another order.

</details>

<details><summary>C. A unique-key error on the repeated number</summary>

✗ An order number is a unique key, and a unique key refuses a repeat. It checks only the rows on its own copy, and the promoted copy never had the old row, so the second use of the number inserts cleanly.

</details>

<details><summary>D. Another customer's order under the same number</summary>

✓ Right. The warehouse gets another customer's order under the number it holds, so the label it prints carries the wrong customer's details.

**A counter replicated with the data stops at the last number the replica replayed**, and after promotion that replica hands out the next numbers again. After forty new orders, all forty numbers the warehouse holds mean a different order.

On promotion, move the counter past any possible loss, here by at least the forty. The lookup then finds nothing, which the warehouse can act on, and the first order stays on no copy.

</details>

---

###### 11. Acknowledging a save once the nearest replica has it

A booking platform runs a five-node store that acknowledges a save once one replica besides the primary has it on disk. The primary and one replica sit in the first site, and three replicas sit in the second, a second or so behind. Saves come back fast. A cut then isolates the first site, the three nodes in the second elect a new primary, and clients move there.

What does the new primary hold of the saves from the last moments before the cut?

<details><summary>A. No trace of those saves</summary>

✓ Right. The new primary holds none of those saves, so a customer whose booking came back fine finds it missing on the primary that now serves them.

Waiting for any one replica puts a save on two disks, here both in the first site. **Only a save on a majority is sure to share a node with every majority that can elect**, and two disks of five are no majority, so the three nodes in the second site can elect a primary while holding neither disk.

For bookings you cannot lose, wait for a majority of the five, which here means a trip to the second site on every save.

</details>

<details><summary>B. All of them, each kept on the new primary's disk</summary>

✗ Two copies protect a save from the loss of any one machine. The cut takes both machines away at once, since they share a site, and the second site elects without them.

</details>

<details><summary>C. All of them, fetched from the near replica during the election</summary>

✗ Voters compare how far each log has got, so an election can pick up the most advanced copy among them. The near replica is on the wrong side of the cut and takes no part, and the three voters can only compare what the three of them hold.

</details>

<details><summary>D. None yet, and all of them once the cut heals</summary>

✗ A healed cut lets replicas catch up, and the first site's two disks still hold those saves. Catching up copies from the primary, and the primary now sits in the second site, elected by a majority that never had them, so those saves have no way across.

</details>

---

###### 12. Losing a clinic's only synchronous replica at night

A clinic's records store runs one synchronous replica, with the option to fall back to asynchronous left off, and there is no other copy. The replica's machine dies at 23:00, the primary stays healthy, and nurses keep saving notes through the night.

What happens to the nurses' saves?

<details><summary>A. They go through, kept on the primary alone</summary>

✗ A healthy primary can work alone, and high-availability diagrams draw exactly that. A commit on this primary counts only after the replica confirms, no limit ends that wait, and no confirmation is coming.

</details>

<details><summary>B. They are refused at once, each with a message to retry</summary>

✗ A good API that cannot do what it promised says so at once. The primary can see that a connection has dropped, and that does not tell it the machine is gone for good, so each commit stays in its wait and no message goes to anyone.

</details>

<details><summary>C. They hang, each note waiting on the dead replica</summary>

✓ Right. Each save hangs, and the ward cannot record a new note until someone acts.

**A synchronous commit waits for one named replica to confirm**, and a dead machine sends neither a yes nor a no. So a pair run this way stops taking saves when either machine is down, and some stores go further and shut the primary down. Here the wait ends when the replica returns or when someone changes the store's setting.

Add a second replica, so a majority of three can confirm with one machine gone, or choose a fallback knowing what it gives up.

</details>

---

###### 13. Setting a ten-second limit on the wait for a replica

A payment ledger must not lose a record, and its commits used to hang when its one synchronous replica went away. So you set a ten-second limit on the wait, and past it the reply to the app has the same form as ever. The replica's machine stalls at 23:00 and stays stalled past midnight, and no alert is set on this.

What does an acknowledgement at midnight tell you about that save?

<details><summary>A. It is on both machines, the replica's disk included</summary>

✗ The setting is called synchronous, the reply has not changed since yesterday, and the limit was added to cure a timing problem. The limit reaches further than timing: once it passes, the primary commits without the replica, so the stalled machine has no copy of the midnight save.

</details>

<details><summary>B. At most ten seconds of saves are at risk</summary>

✗ A log shipped every ten seconds can lose at most ten seconds, and ten is the one figure in sight. The limit on the wait bounds how long a save waits. The saves at risk are all those since 23:00, and their number rises with each one.

</details>

<details><summary>C. It is in the replica's memory, short of its disk</summary>

✗ Some settings confirm when a replica has received a change, before its disk has it. Any such confirmation needs the replica to answer, and a stalled machine answers nothing.

</details>

<details><summary>D. It is on the primary's disk, and nothing more</summary>

✓ Right. At midnight an acknowledgement means the save is on the primary's disk and nowhere else, and a ledger that must not lose a record has been running on one machine since 23:00 with every reply as usual.

**A limit on the wait gives up the replica's copy once the limit passes**: from then on the primary commits without the replica for as long as it stays stalled. Every save since 23:00 sits on that one disk, and the count grows with each save. This store's reply carries no flag, while some others return an error at the limit.

Alert on the moment the primary stops waiting, and watch that state, not the setting's name. If neither a lost record nor a stopped ledger is acceptable, a majority of three goes on saving to two disks when one replica stalls.

</details>

---

###### 14. Promoting a replica while the old primary is still up

A travel site runs a failover tool that watches the primary from its own network. A cut separates the tool from the primary's rack, while the application servers can still reach that rack. The tool promotes the replica and changes the name the application servers use to find the primary, and it sends the old primary no command to stop. Their connection pools recycle over about twenty minutes.

Where do the bookings go during those twenty minutes?

<details><summary>A. All to the new primary, from the moment of promotion</summary>

✗ The tool changes the name, and a name change is what moves traffic. A connection that is already open does not look the name up again, so each server keeps writing to the old primary until its pool recycles.

</details>

<details><summary>B. Some to each primary, and neither holds the other's</summary>

✓ Right. Servers still on the old address write their bookings to the old primary, and servers that have moved write theirs to the new one. For twenty minutes one seat can be sold on both, and each primary lacks the other's bookings.

This is split brain. **Promoting a replica tells the old primary nothing**, and this one is healthy and still reachable by the application. Each server moves only when its pool recycles, so the servers reach the new address at different times.

Stop the old primary before you promote, or have a primary refuse writes once it loses contact with a majority.

</details>

<details><summary>C. All to the old primary, and none to the new one</summary>

✗ The primary never failed and the servers can still reach it. The promotion took effect all the same: the new primary accepts writes, and every server that recycles its pool sends its bookings there.

</details>

<details><summary>D. To the new primary, while the old one refuses the rest</summary>

✗ Refusing writes when the replica is gone is a setting some primaries have, and one with a synchronous replica and no fallback stops acknowledging. The old primary never needed its asynchronous replica to acknowledge a booking, so losing it changes nothing for the servers still there.

</details>

---

###### 15. Promoting a replica in the far region

A marketplace keeps its primary and its application in one region, with a replica in another region. A forty-second network cut sets off automatic promotion of the far replica, which the application's connections follow, and then the cut heals. The application stays where it was.

What do shoppers get after the cut has healed?

<details><summary>A. Normal pages from a healthy database</summary>

✗ The tool reports success, every copy is healthy, and tests inside one region never show a difference. The application still sits in the first region, and now every one of its queries goes to the second and back.

</details>

<details><summary>B. Normal pages, once the primary moves back by itself</summary>

✗ The network is back as it was, and the first region is where the primary lived. A promotion does not lapse when the cut that set it off heals, and no step here moves the primary back. The two copies no longer line up for that move either, since the first primary can hold writes from the cut that the promoted one lacks.

</details>

<details><summary>C. Old prices from a first-region copy that stopped replicating</summary>

✗ Stale data is the harm people watch for after a failover, and the first region's copy is the nearest one. Every query here goes to the promoted primary and none to that copy, so the prices are current.

</details>

<details><summary>D. Slow pages on a healthy database</summary>

✓ Right. Shoppers get slow pages: every query on a page now crosses between regions and back, while the promoted primary reports healthy.

**A failover can succeed for the database and still leave the primary far from its application**, and the page pays for that distance once per query. Say the regions are 70 ms apart there and back and a page runs thirty queries one after another: that adds about two seconds to every page.

Limit where a promotion may land, or move the application along with its primary.

</details>

---

###### 16. Running a wrong delete with four replicas

An operator on a logistics platform runs a delete on the wrong table at 10:00, and someone notices at 10:20. Two replicas run about a second behind the primary, and a third has a replay delay of one hour. The fourth was off for a memory swap from 09:50 to 10:05 and has been back to about a second behind since 10:15. The primary's log records a delete by each row's key. Backups are another team's matter, and you want the rows from a copy.

Which copy can give back the rows?

<details><summary>A. The replica with the one-hour delay</summary>

✓ Right. The replica with the one-hour delay still has the rows, so the operator's mistake can be repaired from it, provided someone stops its replay before 11:00.

Copies that keep up are no protection against a mistake, since **a replica replays a delete as faithfully as any good write**. Only the delay sets this copy apart: at 10:20 its replay has not reached 10:00 in the log.

Stop replay on that copy at once, then replay it forward to the moment before the delete, so it also picks up rows added since 09:20. A delay helps only with a mistake noticed inside it.

</details>

<details><summary>B. The replica that was off when the delete ran</summary>

✗ A replica is a second copy of the data, which is what you run one for, and this one was not running when the delete went through. Being off only postpones a change: back at 10:05 it asked for the log from where it had stopped, the 10:00 delete came with the rest, and by 10:15 it had lost the rows too.

</details>

<details><summary>C. The primary, by undoing the delete from its log</summary>

✗ The log records every change, and inside an open transaction a rollback does undo one. The delete has committed, and its log entries hold only keys, so the values are not there to put back. A store that logs a full image of each row can reverse a delete this way, and this log holds no such image.

</details>

<details><summary>D. No copy holds the rows any more</summary>

✗ The log reaches the delayed copy along with the others, so its disk has held the delete since about 10:00:01. Holding an entry is a different thing from replaying it: at 10:20 this copy is replaying 09:20, and a read there still finds the rows.

</details>

---

###### 17. Switching a replica off for a long weekend

A shop's operator switches a replica off on Friday for hardware work. The primary is set to keep every log entry this replica has not read, with no limit, so that it can always resume. The primary's disk has room for about a day of log at the usual write rate. The work runs into Monday.

What does the on-call engineer find on Sunday?

<details><summary>A. A primary running as usual, and a replica ready to catch up</summary>

✗ Replication guides show the replica coming back and picking up where it stopped. Resuming needs its log to be kept on the primary, and that log outgrew the day of room on Saturday.

</details>

<details><summary>B. A primary that has dropped the absent replica and carries on</summary>

✗ A silent client gets timed out on most connections, and this replica has been silent since Friday. A place in the log is not a connection, and it stays until someone removes it, whatever the disk has left.

</details>

<details><summary>C. A primary that has stopped taking saves</summary>

✓ Right. The engineer finds a full disk and a primary that has stopped taking saves, so the shop cannot take an order.

**A primary holding a place in the log for a replica must keep every entry the replica has not read**, and this replica reads nothing, so the day of room is gone by Saturday. Whether the store then shuts down on the full log disk or waits and retries until there is room, no save goes through.

Set a limit on the log kept for a replica, and alert as it fills. A limit shorter than the day of room would have dropped this replica's place before the disk filled and left it clear, and the price is a full copy on Monday.

</details>

<details><summary>D. A primary that has deleted old log, and a replica needing a full copy</summary>

✗ Retention limits clean up old log, and a returning replica then needs a full copy. The primary's setting keeps what the replica has not read, so it deletes none of that log, and the disk goes first.

</details>

---

###### 18. Skipping one failed change on a replica

At a lending service, a replica stops replay on a change it cannot apply to a row it holds, and lag climbs. The on-call engineer tells it to skip that one change, replay resumes, and within the hour lag reads zero and the alert clears. The change set one customer's credit limit, and that row is not changed again.

What does the replica hold for that customer a month later?

<details><summary>A. The new limit, sent across when the lag closed</summary>

✗ A replica that stops and resumes ends with every change applied once it has caught up, and the alert cleared within the hour. The skip took this one change out of the catch-up: the position moved past it, it was never applied, and the primary has no reason to send it twice.

</details>

<details><summary>B. The old limit, flagged by a background comparison</summary>

✗ Leaderless stores check their copies against each other with a background comparison, and a month gives one plenty of time. A replica of one primary trusts its log and compares nothing until someone starts a comparison, and the engineer only told it to skip, so nothing ever flags the old value.

</details>

<details><summary>C. The old credit limit, and no report of a difference</summary>

✓ Right. A month later the replica still holds the old credit limit and has reported nothing about it. A decision read from that replica, or a failover that promotes it, uses a limit the customer no longer has.

A skip moves the position past the change without applying it, so the gap closes while the row keeps its old value, and **lag measures a position in the log, not what the rows hold**.

Before you skip, ask whether the change has to be applied, and if so reconcile the row by hand. After any skip, rebuild the replica or compare it with the primary by hash, which would flag this one row.

</details>

<details><summary>D. A refusal to read the row it skipped</summary>

✗ A store that keeps checksums can refuse to read a damaged row. A skip leaves no such mark: the replica records a new position and nothing about the row, which goes on answering reads with no error.

</details>

---

###### 19. Picking one username in two regions

A social app takes sign-ups in two regions, each with a primary that has a unique index on usernames. The regions copy to each other asynchronously, about a second apart, with a rule that keeps one write of a conflicting pair. Nothing has failed. Two people pick the same username in different regions in the same second.

What do the two people get?

<details><summary>A. One refused on the spot by the unique index</summary>

✗ On a single primary a unique index is a guarantee, and this schema has one. Each region's index checks its own region, neither has seen the other's sign-up within that second, and both pass.

</details>

<details><summary>B. Both welcomed, and one loses the name a second later</summary>

✓ Right. Both people get a welcome, and about a second later one of them no longer owns the name they were told was theirs.

Each region's index finds the name free, since **a unique index checks only the copy it lives on**, and each primary acknowledges. The clash appears when the writes cross, on a healthy network. A rule such as last write wins then keeps one write, and the person whose write it dropped hears nothing.

Send each name's sign-up to one home region, so the check runs on one copy and the second person is refused at once. The price is a trip to that region for people far from it.

</details>

<details><summary>C. One refused after a short wait while the regions agree</summary>

✗ Checking with the other region before acknowledging would do this, at the cost of a wait on every sign-up. The regions copy after acknowledging, so nothing waits, and both replies go out before either write crosses.

</details>

<details><summary>D. Both welcomed into one shared account</summary>

✗ A merge rule on a document can fold two edits into one record, and these two rows share a key. The rule here keeps one write and drops the other, so two strangers' details never land in one account.

</details>

---

###### 20. Reporting a quorum write as failed

A leaderless store keeps three replicas, and a write waits for two. A customer changes a delivery slot, one replica stores the change, the other two do not answer in time, and the app shows "could not save". The customer gives up and closes the app. Each read asks two replicas picked at random, the newest stamp wins, and the store repairs a copy that disagrees.

What can later reads of that delivery slot return?

<details><summary>A. Either slot at first, then only the new one</summary>

✓ Right. A read that asks the two replicas without the change returns the old slot, and the first read that reaches the replica that kept it returns the new one and repairs the other replica it asked. From then on every read gets the new slot, so the customer told "could not save" gets a delivery at the slot they gave up on.

A failed quorum write means too few replicas replied in time, and **the one replica that stored the value keeps it, since nothing rolls it back**. Reads that only ever asked the other two would agree on the old slot and repair nothing, and then only anti-entropy would bring the new one.

Tell the customer the change is not confirmed, rather than not saved, and have the app read the slot back or repeat the write before it treats the change as dropped.

</details>

<details><summary>B. A failure on each read of that slot</summary>

✗ A careful API admits it when it does not know how a write ended. The store keeps no record of how the write ended, so a read sees two stamps, takes the newest and answers as usual.

</details>

<details><summary>C. The old slot every time</summary>

✗ A transaction that fails leaves nothing behind, and "could not save" is the message a failed transaction gives. Nothing here undoes the change on the one replica that stored it, and its newer stamp outranks the old slot on the first read that reaches it.

</details>

---

###### 21. Saving a setting while two replicas are cut off

A video service keeps the viewing settings of each profile in a leaderless store. Each key has three replicas, a write waits for two acknowledgements, and a read asks two of the key's replicas and keeps the newest stamp. When a replica cannot be reached, another node takes the write in its place, acknowledges it and holds it to pass on later. A network cut hides two replicas of one profile's key from the service for a minute. In that minute a parent switches the profile to children's titles only, and the app confirms the save. The cut then heals, and moments later the child turns on the television.

What can the television get for that setting?

<details><summary>A. The new setting on every read</summary>

✗ Two plus two is more than three, the sum quorum guides give for seeing your own write. That sum counts the key's replicas, and only one of them took this write, so a read of the other two finds the old setting.

</details>

<details><summary>B. A read error on each try while the write is still held</summary>

✗ When too few of a key's replicas can be reached, the read does fail, and this write has not finished reaching its replicas. The cut has healed, so all three answer, and none of them knows a write is held elsewhere: each replies at once with what it has.

</details>

<details><summary>C. The old setting from now on, with the parent's save thrown away</summary>

✗ A copy on a node that does not own the key is temporary, as a cache's is, and a cache gives way to the real data. This node deletes its copy only after passing the write on, and one replica took the write directly, so its newer stamp beats the old setting at the first read that compares the two.

</details>

<details><summary>D. The old setting on some reads, then the new one on all of them</summary>

✓ Right. A read that asks the two replicas the cut hid gets the old setting, so the television can still offer every title after the parent's save. Once the held write reaches those replicas, every read gets the children's setting, and nothing of the save is lost.

Reads plus writes come to four against three replicas, and that sum makes an overlap only among the key's own replicas. Here one replica took the write and nodes standing in for the other two filled the count, and **an acknowledgement from a stand-in puts the write on no replica a read will ask**. The field calls this a sloppy quorum, and calls passing the write on later hinted handoff. Other stores count only a key's own replicas, and they refuse this save during the cut.

For a setting that must hold on the very next read, require the acknowledgements from the key's own replicas. The parent then gets a refused save during the cut and can try again.

</details>

---

###### 22. Cutting the wait before re-copying a silent node's data

A backup service stores each block of data as three copies on nodes in three racks. Every node sends a heartbeat, a short "I am alive" message, over the links it uses for everything else, and at the nightly peak heartbeats already run up to three seconds late. They reached six when one dead node's blocks were copied last month. A node silent for ten minutes counts as dead, and the cluster then copies its blocks to other nodes from the surviving copies, one job per dead node at a steady pace, with no limit on how many jobs run at once. A copy that has started runs to the end, and any surplus copy is dropped afterwards. Worried about blocks sitting on two copies for ten minutes, you cut the wait to ten seconds. At tonight's peak a switch restart hides a rack of twenty nodes for thirty seconds.

What does the on-call engineer see through the rest of the night?

<details><summary>A. A minute of slow backups, then a normal night</summary>

✗ The rack is back in thirty seconds with every copy on it, and a fault usually ends when its cause does. The ten-second wait ended first: by then copies had begun for all twenty nodes' blocks, and each one runs to the end whether the rack has returned or not.

</details>

<details><summary>B. More nodes marked dead through the night, and ever more copying</summary>

✓ Right. The engineer sees working nodes marked dead one after another and copying that keeps growing, so tonight's backups crawl for hours over a fault that lasted thirty seconds.

The rack outlasted the ten-second wait, so copies began for all twenty nodes' blocks. Say those nodes hold 20 TB each and the cluster can move 10 GB a second: that is about eleven hours of copying, and it carries on after the rack returns. One node's copying took heartbeats to six seconds, and twenty nodes' copying, twenty times the load on the same disks and links, takes them past ten, so a working node counts as dead and has its blocks copied too. That closes a loop: **repair load can make working nodes look dead, and each one adds more repair**.

Put the wait back above what a switch or a machine takes to restart, and cap how fast repair may copy, so that the peak's traffic and the heartbeats keep their room. A node that has died for good then leaves its blocks on two copies for those minutes.

</details>

<details><summary>C. A night of heavy copying that ends with every block safer than before</summary>

✗ More copies sooner is why you cut the wait, and a copy is what protects a block. These blocks lost no copy in those thirty seconds, so the copying adds nothing and its surplus gets dropped, and all the while it makes nodes late enough for the ten-second wait to mark them dead.

</details>

<details><summary>D. The blocks on that rack gone for good</summary>

✗ The cluster's own record says twenty nodes are dead, and losing a rack is what disaster plans are written for. Each block has copies in two other racks, and marking a node dead erases nothing on its disks, so every block stays where it was.

</details>

---

###### 23. Growing a photo store from thirty nodes to a thousand

A photo-sharing service stores each photo as three copies on three nodes picked at random. On thirty nodes it has run for years without losing a photo. Growth takes it to a thousand nodes of the same kind and ten billion photos, with the same number of photos on each node and the same placement. A dead node's photos are re-copied from their other copies, as before. The capacity plan says durability is unchanged, since every photo still has three copies.

What should the team expect of lost photos at a thousand nodes?

<details><summary>A. Losses as rare as they were on thirty nodes</summary>

✗ Every photo still has three copies, and the plan counts copies. It counts them for one photo at a time. A loss needs only one photo of ten billion to have all its nodes dead, and a thousand nodes reach three dead at once far more often than thirty did.

</details>

<details><summary>B. Losses rarer than before, spread over more machines</summary>

✗ A bigger cluster re-copies a dead node from more places, so each photo spends less time short of a copy. Deaths grow with the node count as well: even if re-copying speeds up in step, three overlapping deaths come about 37 times as often on a thousand nodes as on thirty.

</details>

<details><summary>C. Losses more often, each a tiny share of the photos</summary>

✓ Right. Photos get lost more often than they did on thirty nodes, and each time only a tiny share goes, about sixty photos of ten billion, so a few customers at a time find a photo missing.

One photo is no worse off, since its own three nodes die together as rarely as ever. Ten billion photos share about 166 million possible sets of three nodes. **When copies land at random and items far outnumber the sets of nodes, almost every set holds all the copies of something**, so any three dead at once lose some photos. Three at once also comes far more often among a thousand nodes. If one node in a thousand is dead at any moment, its photos not yet re-copied, the chance of three at once is about 8 percent, against about four in a million on thirty nodes.

Decide by what one loss costs you. If each loss opens an incident whatever its size, keep three copies and place them in a limited number of fixed sets of three nodes, often called copysets. A photo is then lost only when one whole set dies, which is far rarer and costs about thirty million photos when it happens if each node sits in one set, and re-copying a dead node draws on fewer nodes. The Copysets study, 2013, reports that many operators prefer that trade. If a few lost photos cost little, random placement can stay.

</details>

<details><summary>D. Losses more often, each costing a whole node's photos</summary>

✗ A dead node's whole disk goes with it, and each node holds thirty million copies. Nearly all of those photos keep a copy on a live node, since a photo is gone only when all three of its nodes are dead, and three deaths leave about sixty such photos.

</details>

---

## Further reading

- [Session Guarantees for Weakly Consistent Replicated Data](https://doi.org/10.1109/PDIS.1994.331722), 1994: where read your writes and monotonic reads come from, as promises a session can ask of its replicas.
- [The dangers of replication and a solution](https://doi.org/10.1145/233269.233330), 1996: why update-anywhere replication becomes unstable as load grows, and how a master copy reduces it.
- [Probabilistically Bounded Staleness for Practical Partial Quorums](https://arxiv.org/abs/1204.6082), 2012: how stale a read from a partial quorum can be, and how to predict it.
- [Copysets](https://www.usenix.org/conference/atc13/technical-sessions/presentation/cidon), 2013: why random replication is almost guaranteed to lose data when many nodes fail at once, and how limiting where copies go makes such losses rarer.

Prev: [Distributed locks and IDs](../distributed-locks-and-ids/README.md) · Next: [Sharding and partitioning](../sharding-and-partitioning/README.md)
