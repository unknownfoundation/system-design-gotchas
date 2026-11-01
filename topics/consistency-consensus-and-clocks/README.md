# Consistency, consensus and clocks

Prev: [Timeouts, retries and circuit breakers](../timeouts-retries-and-circuit-breakers/README.md) · Next: [Distributed locks and IDs](../distributed-locks-and-ids/README.md) · Reviewed October 2026

Consistency here is the promise a store makes about what a read may return when the data sits on several machines. The same word also names the rules a transaction must keep and plain copies agreeing, so ask which one a document means. Consensus is how a group of machines settles one order of writes, and clocks are what machines fall back on when nothing else tells them which event came first.

- **Partition**: a cut in communication between machines that are all up. Each side sees only timeouts, cannot tell a dead machine from a slow one, and has no upper limit on how late a message can arrive. The CAP theorem says that during a partition a store cannot both answer every request and act as one up-to-date copy; its "available" means every request to a working node gets an answer, which asks more than uptime.
- **Quorum**: in a store with no leader, how many copies a write must reach and a read must ask, usually sized so that any read quorum and any write quorum share a copy.
- **Tombstone**: a record written in place of a deleted value, so that a replica which missed the delete can learn of it. A store purges its markers after a set time.
- **Consensus group**: machines that keep one log of writes in one order. Unless a question says otherwise, a write is committed and acknowledged once a majority holds it on disk, so each write costs a round trip to a majority and a disk sync.
- **Majority**: more than half of the members on the group's list.
- **Leader**: the one member that orders the writes and sends them to the others, with regular heartbeats between them.
- **Election**: what a member starts when it hears no heartbeat within its timeout. It raises the term, a number every leader carries, and asks for votes. A member votes once per term, and only for a candidate whose log holds at least what its own does.
- **Local read**: a read a member answers from the copy it holds, with no trip through the log, no check with a majority and no lease (a time-limited promise that no other leader exists).
- **Two-phase commit**: one transaction across several databases, in which a coordinator asks each to prepare and vote, collects the votes and sends the decision.
- **Clock skew**: the gap between two machines' clocks, which syncing narrows and never closes.
- **Last write wins**: a rule for two writes to one key, under which the store keeps the write with the higher timestamp and drops the other.

Makers add their own safeguards around elections and a leader's hold on its role, so unless a question says otherwise, a group here runs only the rules in the lines above.

A store that stays open on both sides of a partition merges afterwards. A merge comes out the same in any order when its operations commute, as with additions to a counter or members of a set, and CRDTs, conflict-free replicated data types, are built on this. Which work still needs coordination depends on the rules the data must keep.

What stores promise:

| Promise | What a reader may rely on |
| --- | --- |
| Eventually consistent | once writes stop and messages get through, all copies hold the same value |
| Causal | a change made after another by the same person, or after seeing it, never appears before it, on any key |
| Serializable | transactions give a result that matches some order, one at a time |
| Strict serializable | serializable, and that order matches real time: a transaction that starts after another has finished comes after it |
| Linearizable | one key acts as a single copy: a read that starts after a write has finished sees it. The same promise across several keys is strict serializable |

These promises are tested by faults, so a clean test run on a healthy network proves little about them.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- read what a store promises and say what a guest, a shipping worker, a manager taken off an album or a bank customer can see next, and what a save pays for that promise on a quiet day
- follow a network cut through a consensus group, a seat count, a deleted account and a price read mid-write
- run a consensus group through a dead leader, a replaced disk, a local read, a move to two sites, a shorter timeout, a list grown from three members to five, one broken switch link and a slow disk under the leader, and a two-phase commit through a dead coordinator
- judge what a timestamp can settle when it orders two log lines, checks a token or picks between two writes

###### 1. Reading a guest's booking four hours behind after a network cut

A hotel chain keeps bookings in a store whose documents call it eventually consistent and give its replication lag as typically under a second. One afternoon the link between two regions is cut for four hours. Early in the cut a guest changes a booking on their phone, then flies to the other region, where the phone shows the old booking until twenty minutes after the link returns, and the new one from then on. A colleague files a bug against the store.

Which promise did the store break?

<details><summary>A. Its documented lag of under a second</summary>

✗ The documents print that second, and it is the lag you see on every ordinary day. "Typically" reports what was measured on a healthy link. The promise itself carries no time, so a guest four hours behind breaks no promise.

</details>

<details><summary>B. It broke none of its promises</summary>

✓ Right. The copies met once the link was back, and until then the phone was allowed to show the old booking, so the colleague's bug has no promise to point at.

Eventually consistent promises that **copies end up with the same value once writes stop and messages get through**. It names no time and promises nothing about what a read returns along the way, so a cut of four hours keeps it as well as a cut of a minute. The second in the documents is the lag on a healthy link, a measurement and no part of the contract.

Close the bug. If a guest must see a fresh booking, make that a requirement of the read, by sending it to a copy that holds the write or by choosing a store whose promise covers it, and decide what the phone shows while the link is down.

</details>

<details><summary>C. Its promise that a guest reads back their own change</summary>

✗ On one database server a read after a write returns that write, and most code you have written counts on it. Reading back your own change is a separate promise, and these documents do not list it: the change sits behind the cut in the first region, and the copy the phone reads holds only the old booking.

</details>

<details><summary>D. Its promise that copies agree once the link is back</summary>

✗ Messages were crossing again, and for twenty minutes the phone still showed the old booking. "Once" sets a condition and no deadline: four hours of writes have to cross the link while new ones keep arriving, and copies that meet twenty minutes later have kept the promise.

</details>

---

###### 2. Sending a message right after the commit

A shop's order service commits an order on the primary database, then publishes an event to a queue. The event carries the order's id. A shipping worker takes it a few milliseconds later and looks the order up by that id on a replica, a read-only copy the primary sends changes to after each commit.

What does the shipping worker get when it looks the order up?

<details><summary>A. The order, every time</summary>

✗ The publish line comes after the commit line, and inside one process that order is real. The worker's lookup goes to the replica, and the event and the copy to the replica leave separately, so the event can land first and the lookup finds nothing.

</details>

<details><summary>B. The order, after the replica holds the read back</summary>

✗ Some stores can hold a read until a replica has caught up to a position you name. The event here carries only the id, so this read names no position, and the replica answers from whatever it holds at that moment.

</details>

<details><summary>C. The order, after the queue holds the message back</summary>

✗ The event exists only because the commit happened, and the database and the queue serve one order service. The queue knows nothing of the replica: it delivers the event as soon as it can.

</details>

<details><summary>D. No such order, now and then</summary>

✓ Right. The worker gets no such order whenever the event outruns the copy, and shipping sees a not-found for an order committed moments ago.

**Replication and the queue are two separate paths with nothing tying them together**, so either can arrive first. If replication takes 20 ms and the event 3 ms, the worker reads before the replica has the order.

Put what the worker needs into the event, or have this one lookup read from the primary. A retry on not-found also works, at the price of treating an order that truly does not exist the same way.

</details>

---

###### 3. Removing a manager from an album, then posting

On a photo app, a user removes their manager from an album and then posts a photo to it. The album's member list and its photos live on different shards, separate parts of the store, and each shard copies to the manager's region on its own. The app shows an album's photos to anyone on its member list. A moment later the manager opens the album there.

What can the manager see on opening the album?

<details><summary>A. The new photo</summary>

✓ Right. The person the user took off the album can see the photo posted after the removal.

**Each key is copied on its own, with no order kept between keys**, so the photo can reach the manager's region before the member update does. If the manager reads in that gap, the region still lists the manager, the app's check passes and the photo shows.

Use a store that keeps causal order across keys, which holds the photo back until the member update has arrived. The manager can still read the old album until the update lands, because a causal promise covers order and sets no deadline.

</details>

<details><summary>B. A refusal, and never the photo</summary>

✗ The user removed the manager before posting, and the app checks the member list before it shows a single photo. The two updates travel on separate shards with nothing between them, so this region can hold the photo while its list still names the manager, and the check passes.

</details>

<details><summary>C. Neither photo nor album for a few seconds</summary>

✗ Apps on your phone show a spinner while they sync, and a region catching up is a kind of sync. A read here waits for no copy: it gets what the region holds at that moment, so the album opens at once.

</details>

<details><summary>D. A conflict notice in place of the album</summary>

✗ In version control, two changes close together can stop and ask a person. The removal and the photo touch different keys, so neither overwrites the other and the store has nothing to flag.

</details>

---

###### 4. Showing the balance right after a serializable transfer

A bank runs accounts on a distributed store that offers two levels, serializable and strict serializable, and the bank chose serializable. Read-only transactions there run on a snapshot at the nearest replica. A customer's transfer commits, the app shows it as done, and the customer opens the balance page, whose read is one of those read-only transactions.

What balance can that page show?

<details><summary>A. The new balance, every time</summary>

✗ Isolation tables in courses put Serializable on the last and highest row, with nothing above it. Those tables rank what concurrent transactions can do to each other, and leave out how fresh a read is. This read comes from a snapshot on a replica that may not hold the transfer yet.

</details>

<details><summary>B. No balance: the read aborts as a conflict</summary>

✗ Serializable databases abort transactions that conflict, and anyone who has run one has written the retry loop. A conflict needs a write to collide with, and the page's transaction writes nothing and reads a finished snapshot, so nothing aborts it.

</details>

<details><summary>C. The balance from before the transfer</summary>

✓ Right. The page can show the balance from before the transfer, and a customer who was told the transfer is done then finds the money where it was.

**Serializable asks only for a result that matches some one-at-a-time order**, and that order need not follow real time. An order with the balance page first and the transfer second explains the old balance, even though the page opened after the transfer was done.

Run this read at strict serializable, the level that adds real time, or send the page's read to a copy known to hold the transfer. The replica's snapshot still suits pages where a slightly older balance does no harm.

</details>

<details><summary>D. The debit on one account, with no credit on the other</summary>

✗ Replicas apply changes one after another, so at some moment a replica can hold the debit and not yet the credit. A snapshot at this level shows whole transactions: a debit without its credit fits no order at all, so the page shows both sides of the transfer or neither.

</details>

---

###### 5. Protecting saves against losing a region on a calm day

A game studio keeps player accounts in two far-apart regions and turns on the setting that keeps every confirmed save safe if a whole region is lost. All month the network stays healthy.

What does a save cost a player in that healthy month?

<details><summary>A. A trip to the other region on every save</summary>

✓ Right. Every save waits for a trip to the other region and back, which makes a player's saves slower all month with no failure to blame.

**A save survives the loss of its region only once the other region holds it**, so it waits for that far copy on every write, partition or not. If the round trip between the regions is 80 ms, a 2 ms local save takes 82 ms. The CAP theorem speaks only of partitions, so it leaves this price out.

Keep the wait for the saves that must survive losing a region, such as a purchase, and let progress saves stay local and copy afterwards, accepting that a region lost at the wrong moment takes the latest of them along.

</details>

<details><summary>B. Nothing extra, short of a link failure</summary>

✗ The CAP theorem as usually taught is about what you give up during a partition, and this month has none. The setting carries no such condition: a save waits for the other region's reply whether or not the network has ever had a bad day.

</details>

<details><summary>C. A trip to the other region on the first save of each session</summary>

✗ Connections set up once and reused pay on the first request only, as a TLS handshake does. A session has nothing to reuse here: each save is new data the other region does not hold yet, so a later save in a session costs what the first one did.

</details>

<details><summary>D. A trip to the other region only when two saves collide</summary>

✗ Optimistic locking pays only when two writers collide, and on most saves no other writer exists. A save cannot learn whether it collides without asking the other region, so the asking is itself the trip, and a save with no rival pays for it too.

</details>

---

###### 6. Cutting one node off a three-node group

A configuration store runs as a consensus group of three nodes, one per rack. A switch failure cuts off the rack that holds the leader, with clients left on both sides of the cut.

What happens to writes from the clients on the two-node side?

<details><summary>A. Refused by every node, on both sides of the cut</summary>

✗ Comparison tables file a store like this under choosing consistency over availability. That label describes requests to a side with no majority, and the two-node side has one: it elects a leader and commits, and only the cut-off node stops.

</details>

<details><summary>B. Accepted on both sides, then merged when the link returns</summary>

✗ Stores built to stay open on both sides take writes everywhere and merge afterwards, a real design used for carts and counters. A consensus group works another way: the cut-off node holds one vote of three, so it commits nothing and leaves nothing to merge.

</details>

<details><summary>C. Held until the old leader is back</summary>

✗ Every write in this group goes through the leader, and the leader is on the far side of the cut. The role is not tied to that machine: the two nodes hear nothing from it, one times out and asks for votes, and their writes resume long before the rack returns.

</details>

<details><summary>D. Accepted again after a short pause</summary>

✓ Right. Clients on the two-node side see writes stop for a moment, then carry on under a new leader.

**A side that holds a majority can elect a leader and commit**, and two of three is a majority. The two nodes stop hearing heartbeats, one times out and raises the term, the other votes for it, and the new leader commits with both of them, so the pause lasts about one timeout.

Give clients the address of every node and let writes retry through the pause. Clients stranded on the cut-off rack still cannot write until the switch is back, so keep what must always write away from a single rack.

</details>

---

###### 7. Selling the last seat on both sides of a cut

A cinema chain sells tickets from two regions that stay open during a link failure, and each showing's seat count is a CRDT counter. Each till checks that the count is above zero, subtracts one and prints a ticket. With one seat left in a showing, the link fails and each region gets a buyer.

What do the sales and the seat count come to once the link returns?

<details><summary>A. One sale undone by the merge, and a count of zero</summary>

✗ Conflict handling in most tools ends in one clean result, and the merge is the step meant to find trouble. This merge only adds numbers: it sees two subtractions, with no idea they were for one seat, and undoes neither ticket.

</details>

<details><summary>B. Two tickets sold, and a count of minus one</summary>

✓ Right. The count reads minus one and two buyers hold tickets for one seat, so one of them reaches the showing with nowhere to sit.

A CRDT counter keeps each region's subtractions and adds them up, so the merge takes off both. Each till read its own region's count, saw one seat and subtracted, and **a rule that spans both regions, such as "never below zero", cannot be checked on one side of a partition**.

Give each region a share of the seats before any cut, which is coordination done in advance. With the last seat assigned to one side, one ticket is sold, at the price that the other region can turn a buyer away while the seat is still unsold.

</details>

<details><summary>C. One ticket sold, the other till refusing, and a count of zero</summary>

✗ On one server the check above zero protects the last seat, and that is the code you tested. Each till here reads its own region's count, and with the link down both still read one, so neither has a reason to refuse.

</details>

<details><summary>D. Two tickets sold, and a count of zero</summary>

✗ Each region's count reads zero when the link returns, and copies that match give most merge tools nothing to do. The counter merges what each region subtracted and ignores the number each one shows, so both subtractions come off the one seat and the count goes below zero.

</details>

---

###### 8. Deleting an account while a replica is away

A leaderless store keeps each account on several replicas, and a background sync copies any value a peer lacks. A delete is written as a marker, kept ten days and then purged. A customer's account is deleted on the day one replica goes off for two weeks. That replica comes back with its disk as it was, and the sync takes it in like any other.

What happens to the customer's account once that replica has been back a while?

<details><summary>A. The account stays deleted everywhere</summary>

✗ The delete reached every replica that was up, and a background sync exists to bring the last one level with the others. The marker that would carry it is gone by the time this replica returns, so nothing tells it of the delete, and its row is still there for the next sync.

</details>

<details><summary>B. The account back on that one replica, deleted on the others</summary>

✗ A replica that was away is usually the odd one out, and the others outvote it on a read. The others hold neither a row nor a marker, so they have nothing to outvote it with. To the sync that gap is a missing value, and it fills the gap from the returned replica.

</details>

<details><summary>C. The account back on every replica</summary>

✓ Right. The account comes back on every replica with no error anywhere, and the customer whose account was deleted has a live one again.

**The tombstone is the only thing that tells a sync a row was deleted**, and after ten days it is purged. When the replica returns, the others hold nothing for the key, and its row looks like data they lack, which the sync copies to them.

Rebuild from scratch any replica that has been away longer than the marker's lifetime instead of letting it rejoin, and finish a full sync of every replica within that lifetime. A longer lifetime gives you more time for this, and keeps more markers on disk.

</details>

<details><summary>D. A conflict flagged for a person to settle</summary>

✗ After two weeks away, the returned replica and the others disagree about whether this account exists. A conflict needs two values to compare, and the others hold nothing for this key. The sync finds one value and a gap, and has no one to ask.

</details>

---

###### 9. Reading a price during a slow write

A shop's price store keeps three copies and has no leader. A write waits for a quorum of two copies, a read asks two and takes the newest value it gets, and a read writes nothing back. A price update is on its way:

```text
t1  the update reaches copy 1, and copies 2 and 3 still hold the old price
t2  a first reader asks copies 1 and 2, and has its answer
t3  a next reader asks copies 2 and 3, with the update still on its way to them
```

What do the two readers get, in order?

<details><summary>A. The old price twice</summary>

✗ The update has not reached its quorum, and the client that sent it has no acknowledgement yet. Copy 1 takes no account of that: it holds the new price with its stamp and hands them to anyone who asks, and the first reader takes the newest of what it gets.

</details>

<details><summary>B. The new price twice</summary>

✗ Two and two come to more than three, and R + W > N is taught as the rule that guarantees the newest value. The overlap counts copies holding the write, and so far that is copy 1 alone. The next reader asks the two copies without it and gets the old price.

</details>

<details><summary>C. The new price, then a refusal</summary>

✗ Some clients keep track of versions and turn down one that goes backwards. This read compares only the two answers it got, and the next reader has never seen the first reader's answer, so it has nothing to turn down.

</details>

<details><summary>D. The new price, then the old price</summary>

✓ Right. The first reader gets the new price and the reader after it gets the old one. For two people reading one after the other, the price moves forward and back again.

**Overlapping quorums protect only finished writes**: once two copies hold the new price, any two copies asked include one that has it. At t2 and t3 the update sits on copy 1 alone, so the first reader, asking copies 1 and 2, sees the new price and the next, asking 2 and 3, sees the old one. A linearizable store would never allow new followed by old, and this one makes no such promise.

Have a read write the newest value back to a quorum before it answers. A first reader that puts the new price on copy 2 leaves every later pair with a copy that holds it. Only a read that finds its copies differ pays for that extra write.

</details>

---

###### 10. Losing a leader right after it acknowledges

A trading firm keeps account limits in a five-node consensus group. The leader acknowledges a client's write and crashes a moment later, and the other four nodes elect a new leader.

What does the new leader hold of that write?

<details><summary>A. Nothing: the write is gone with the old leader</summary>

✗ Failover articles warn that a leader's death loses the last writes, and with asynchronous replicas it does. This leader waited for a majority before it acknowledged, so two other disks already held the write when it crashed.

</details>

<details><summary>B. The write, as acknowledged</summary>

✓ Right. The new leader holds the write, the client's acknowledgement stands and the new limit stays in force.

The acknowledgement came only once three of the five held the write on disk, and an election needs three votes as well. **Two majorities of five always share a node**, so among any three voters at least one holds the write, and that voter refuses a candidate whose log has less. A node missing the write therefore cannot win.

Treat an acknowledgement from this group as final while its disks keep their logs. The promise lasts only while an acknowledgement means a majority on disk, so a setting that acknowledges before the disk sync, or before the majority answers, gives it up.

</details>

<details><summary>C. The write or nothing, by luck of which node wins</summary>

✗ Timeouts run out in no set order, so which of the four nodes asks for votes first comes down to chance. A vote here goes only to a candidate whose log holds at least what the voter's does, and one of any three voters holds the write. Every node that can win has it, so the outcome is one and the same.

</details>

<details><summary>D. The write, marked pending for the client to confirm</summary>

✗ A crash in the middle of a commit can leave a state someone has to clear up, as a stuck two-phase commit does. This write is past that point: a majority held it when the client heard yes, so the new leader treats it like any committed entry and asks the client nothing.

</details>

---

###### 11. Replacing a failed disk on one member of the group

A three-node consensus group holds a logistics firm's routing settings. A write is on the leader and member 2, member 3 is behind, and the client has its acknowledgement. Member 2's disk fails and is replaced, it restarts empty under its old identity, and the group takes it back as it would any member after a crash. A moment later, before member 2 has copied anything and before the write reaches member 3, the leader goes down.

What state is the group in a minute after the leader goes down?

<details><summary>A. The write kept, with member 2 caught up and voting</summary>

✗ A restart after a crash ends that way: the member returns with its log and copies what it lacks. Here the one node that could supply the write is down, member 2 has nothing and member 3 never had it, so no live member has anything to catch up from.

</details>

<details><summary>B. The group waiting for member 2 to catch up before any vote</summary>

✗ Databases often keep a rebuilt copy out of service until it has caught up. The group took member 2 back as it would after a crash, and an empty log tells it only that it is behind, as any member can be, so it answers a vote request at once.

</details>

<details><summary>C. The group stopped, short of a majority</summary>

✗ With a dead disk you count a dead member, and one real member of three is no majority. The count goes by who answers, and nothing asks what they remember: members 2 and 3 both answer, they are two of three, and the election succeeds.

</details>

<details><summary>D. The write gone, and the group taking new writes</summary>

✓ Right. The write is gone and the group runs on, taking new writes, while a client holds an acknowledgement for a write no live member has.

The leader and member 2 were the majority that held the write. **A majority keeps a write safe only while its members remember it**, and member 2 forgot the write when its disk was replaced. With the leader down, members 2 and 3 form a new majority that never held it, so whichever of them wins lacks it. Once the new leader has committed a write of its own, the old leader follows the new log when it returns.

Treat a lost disk as a lost member: remove it from the member list and add its replacement as a new member, so that it never counts as remembering anything. That change needs a majority, so make it while the other two are up.

</details>

---

###### 12. Reading a flag from the nearest member of the group

A feature flag lives in a consensus group of five, and your servers read it with a local read from whichever member they are connected to. You flip the flag through the leader, and the write is acknowledged. A moment later a server connected to another member reads the flag.

What can that server read?

<details><summary>A. The old flag</summary>

✓ Right. The server can read the old flag though the group has acknowledged the new one. For a moment, servers on a member that lags act on the old setting and the rest act on the new.

**A local read takes no part in the order consensus decides**, and the acknowledgement took only three of the five, so up to two members can still lack the flag. Such a member answers from its copy and cannot tell the copy is behind. A later message brings them the flag, and from then on a read from any member returns the new one.

For a read that must see the flip, use the group's read through the log or its majority check, and pay the round trip. A flag your servers can afford to read a moment late can stay on the local read.

</details>

<details><summary>B. The new flag, every time</summary>

✗ The flip went through consensus and the group acknowledged it, which is the job a consensus group exists to do. That acknowledgement took three members of five, and this member may not be one of them, so its copy can still hold the old flag and a local read returns that copy.

</details>

<details><summary>C. The new flag, after that member waits for it</summary>

✗ Other kinds of read on the same group hold the answer back until the member has caught up. A local read sends nothing to anyone, so this member has no way to learn there is a newer flag to wait for, and it answers straight away.

</details>

<details><summary>D. A refusal from a member that knows it is behind</summary>

✗ A database replica can report its lag, and clients sometimes turn away one too far behind. This member hears of a newer write only when that write reaches it, so its copy has every sign of being complete and it has no grounds to refuse.

</details>

---

###### 13. Reading a paging rota from the old leader

A paging system keeps its on-call rota in a five-node consensus group. So that nobody reads from a member that lags, clients send all reads to the leader, each as a local read. A network cut leaves the leader alone with one client, and the other four elect a new leader and commit tonight's change to the rota. That client then reads the rota.

What does that client get back from the old leader?

<details><summary>A. A refusal, the moment the old leader lost its majority</summary>

✗ A leader with no majority behind it can commit nothing, and this one lost its majority at the moment of the cut. A local read commits nothing: it sends no message and answers from the copy the node holds, so the lost majority never comes into it and the client gets an answer. Some makers add a timer that steps a leader down when a majority stops answering its heartbeats, and this group has no such timer.

</details>

<details><summary>B. A not-leader error naming the new leader</summary>

✗ Consensus clients carry retry code for the reply "not the leader, try this member", and that reply is how reads stay with the current leader. A member can send it only after hearing of the new leader, and the four who elected one are across the cut, so this node names nobody and answers from its copy.

</details>

<details><summary>C. The on-call number from before the change</summary>

✓ Right. The client reads the number from before the change, with no sign of trouble, so a page sent on that answer goes to whoever was on call before tonight.

**A replaced leader does not know it has been replaced until it hears from the others**, and a local read asks nobody. The old leader goes on answering from its own copy for as long as the cut lasts. Sending reads to the leader protects against a member that lags, and this node is no longer the leader.

Have the leader confirm with a majority before each read, or run the read through the log. A cut-off leader cannot finish either, so this client waits or gets a refusal and never a stale number. A lease also closes the gap, but it holds only while clocks run at nearly the same rate and the leader never pauses past the lease.

</details>

<details><summary>D. No answer, the read hanging</summary>

✗ A cut-off node's writes hang, since they cannot reach a majority. A local read needs no one else: the node is up, it holds its copy, and the client is on its side of the cut, so the reply comes back at once.

</details>

---

###### 14. Splitting four nodes between two sites

A coordination store runs as a consensus group of three nodes in one site. For disaster safety it moves to four nodes, two in each of two data centres. Then the link between the sites fails, and every machine stays healthy.

Which site keeps taking writes?

<details><summary>A. The site holding the leader</summary>

✗ In a two-site database with a primary, the primary's site carries on when the link fails. This leader needs three of four to commit and can reach two, so its writes never commit and its site stops with the other.

</details>

<details><summary>B. Both sites, reconciled when the link returns</summary>

✗ Active-active designs let each healthy site serve its own side, and both sites here are healthy. In this group a write commits only on a majority, and a side of two has none, so neither site has any writes to reconcile.

</details>

<details><summary>C. The site with the most recent writes</summary>

✗ Elections compare logs, and a node with an older log loses. That comparison only picks among candidates that can gather enough votes, and here a candidate needs three and can reach two on either side.

</details>

<details><summary>D. Neither site, while the link is down</summary>

✓ Right. Neither site takes writes while the link is down, with all four machines up. The move made for disaster safety lets one link failure stop every write.

A majority of four is three, so **an even split leaves neither side with a majority**. Four nodes survive one failure, the same as three, and five survive two.

Run an odd number of members and place them so that no single link cuts the group in half. Three nodes split two and one leave the two-node site a majority, and five, two in each main site and one in a third, let the group outlive the loss of either main site.

</details>

---

###### 15. Cutting the leader timeout from ten seconds to one

A platform keeps its service registry in a consensus group that takes ten seconds to replace a dead leader, so a team cuts the timeout to one second. Each night, at the peak write hour, the leader's disk stalls for two seconds, and its heartbeats stop with it for those two seconds.

What does each night's stall now do to the group?

<details><summary>A. A new leader, and a pause in writes</summary>

✓ Right. Each night the group elects a new leader and writes pause for a moment. The registry keeps working, and its log shows a change of leader every night.

**A timeout cannot tell a dead leader from a slow one**, so two seconds of silence now counts as a death. If the first follower's timer, drawn a little above the one-second timeout, runs out 1.3 s into the stall, writes resume about 1.3 s after the stall began, and no acknowledged write is lost. At the ten-second timeout the same leader would stay and writes would pause for the full 2 s, so the shorter timeout has not lengthened this pause.

Set the timeout with the longest stall you expect in mind, and watch the term number, since a term that rises every night points at the disk rather than at a dead machine.

</details>

<details><summary>B. The same leader, with writes resuming after the stall</summary>

✗ A stalled leader is still alive, as a process check would say, and at ten seconds it would have kept its place. The followers see only silence: one times out inside the stall and raises the term, so the old leader wakes to find it has been replaced.

</details>

<details><summary>C. Two leaders taking writes at once</summary>

✗ When the disk returns, the old leader has not yet heard of the new one, and for a moment two nodes act as leader. The old leader's writes carry the old term, the followers reject them because they hold a higher one, and it commits nothing more.

</details>

<details><summary>D. Committed writes lost at the change of leader</summary>

✗ The stall lands at the peak write hour, so each change of leader comes with writes on their way. The winner's log must hold at least what its voters' logs do, so it holds every committed write, and each night loses none.

</details>

---

###### 16. Losing the coordinator after the yes votes

A shop's order commits across a stock database and a payments database by classic two-phase commit, run by one coordinator process with no standby. Both databases have voted yes when the coordinator crashes, before it sends its decision. It stays down, and the two databases are left to themselves.

What happens to the order's stock and payment rows while the coordinator is down?

<details><summary>A. Both databases roll them back on a timeout</summary>

✗ A timeout is the usual cleanup for a stuck transaction, and before it votes a database may roll back. The yes vote ended that freedom: stock cannot know that payments has not been told to commit, so it cannot roll back alone without risking a split order.

</details>

<details><summary>B. They stay locked in both databases</summary>

✓ Right. The rows stay locked in both databases for as long as the coordinator is down, and other orders that need that stock wait behind them.

**A database that voted yes has handed the decision over**: it promised to commit if told, and it cannot tell which decision was made or whether the other database has heard it. Rolling back alone splits the order if the decision was commit and the other was told, and committing alone splits it if the decision was abort and the other was told. So it waits, and a prepared transaction holds its locks while it waits. Some databases let an operator settle it by hand or can be set to presume an outcome, which can split the result.

Make the decision survive the coordinator: run it on a consensus group, as the 2006 paper Consensus on Transaction Commit describes, so that another node can finish the round. Alert on transactions that stay prepared for long.

</details>

<details><summary>C. Both databases commit the rows, and the order goes through</summary>

✗ Every vote was yes, and yes from everyone is the rule for a commit. Each database saw only its own vote, and the coordinator could still have decided to abort, so neither has what it needs to act.

</details>

<details><summary>D. One database commits and the other rolls back</summary>

✗ A crash in the middle of anything leaves it half done, and a split order is the failure you fear. A split takes a database acting alone, and these two wait: neither commits nor rolls back, so nothing comes apart while they hold their rows.

</details>

---

###### 17. Stamping a cancellation and a label 3 ms apart

A customer says they cancelled before the parcel shipped. The order server sits in a data centre and the warehouse server on the warehouse's own site, and both set their clocks over the internet from one time service, which reports no fault. No message passed between the two servers in that time. Their logs show:

```text
12:00:00.120  order server      cancellation saved
12:00:00.123  warehouse server  label printed
```

What do those two stamps tell you about the cancellation and the label?

<details><summary>A. The cancellation came first, by 3 ms</summary>

✗ Both logs print three decimals, and log viewers merge lines by those digits. The digits show the format's precision: each clock's error goes unprinted and can be larger than 3 ms, so the real order may run either way.

</details>

<details><summary>B. A clock fault on one of the two machines</summary>

✗ When logs disagree with a customer, a broken clock is the first suspect. The time service reports no fault and none is needed, because two healthy clocks synced over the internet can differ by more than this gap.

</details>

<details><summary>C. Nothing about which came first</summary>

✓ Right. The two stamps do not show which came first, so they cannot settle the customer's claim either way.

Each stamp carries its own clock's error, so **a gap between two stamps means something only if the errors differ by less than the gap**. Clock skew between machines synced over ordinary wide networks can exceed 3 ms with no fault reported, and with no message between the servers, nothing else orders the two events.

Put the order into the messages: have the warehouse ask the order server before it prints, or carry a logical clock, a counter every message pushes forward. Where a store must take order from time, it waits out the clock's error before confirming a commit.

</details>

<details><summary>D. The two happened at one moment</summary>

✗ Each clock's error can exceed 3 ms, and stamps this close get called a tie. A tie is as much a claim as an order: the save and the printing were two events, one of them came first, and these stamps cannot say which.

</details>

---

###### 18. Checking a five-minute token across a fleet

A login service signs each token with a not-before time of that moment and an expiry five minutes later, both from its own clock. Ten API servers check both times against their own clocks, with no leeway and no call back to the login service. One server's clock runs 30 seconds behind, and a balancer spreads each customer's calls over the fleet.

What do a customer's calls get right after signing in?

<details><summary>A. Accepted on most servers, refused as not yet valid on that one</summary>

✓ Right. Calls pass on nine servers and that one refuses them as not yet valid. A customer who has only now signed in sees a call fail here and there, and a retry that lands elsewhere passes.

**Each server judges a token's times by its own clock**, and the JSON Web Token standard says a token must be refused before its not-before time. To a server 30 seconds behind, that time lies in the future for the token's first 30 seconds, and from then on it accepts the token.

Allow a small leeway for clock skew, which the standard permits: with 60 s, none of these calls is refused, though each token then lasts that much longer past its expiry. Watch the clocks as well, since the leeway only covers skew smaller than itself.

</details>

<details><summary>B. Refusals from every server, for every login</summary>

✗ Login outages get put down to clocks, and a badly wrong signing clock can stop every login. Only one API server is off here, the login service's clock is not in question, and the other nine judge the token correctly.

</details>

<details><summary>C. Every call passing, on every server</summary>

✗ A token's expiry is where you usually think about clocks, and at that end a slow clock only makes a token last longer. The refusal sits at the other end: for the first 30 seconds, that server reads the not-before time as still to come.

</details>

<details><summary>D. Refusals from that one server on every call, until its clock is corrected</summary>

✗ A machine whose clock is hours off refuses everything it checks. Thirty seconds is small beside five minutes: once the slow clock passes the not-before time, that server accepts the token like the others, and the refusals stop 30 seconds in.

</details>

---

###### 19. Correcting an address twice after a save in another region

A delivery app keeps profiles in a store that takes writes in two regions, stamps each with its own server's clock, and settles writes to one key by last write wins. The first region's clock runs five minutes ahead. A customer saves an address there, and over the next minute sends two corrections to the second region. All three writes are acknowledged.

Which address does the profile hold once the regions have synced?

<details><summary>A. The second correction, the last write the store saw</summary>

✗ The rule's name says last, and the second correction was the last thing the customer did. The store compares stamps and ignores when anything arrived, and the first address carries a stamp five minutes ahead of real time, so it outranks both corrections.

</details>

<details><summary>B. The first address</summary>

✓ Right. The profile holds the first address and both corrections are dropped after being acknowledged, so the next parcel heads for the address the customer corrected twice.

**Last write wins keeps the highest stamp, and a fast clock's stamp lies in the future**, so the first address outranks both corrections sent within the minute.

Stop deciding by wall clock. Give each key a version that a write reads and raises, so a correction that has seen the save sorts after it, or keep version vectors, which show two writes as concurrent and leave your code to settle them.

</details>

<details><summary>C. All three addresses, flagged as a conflict</summary>

✗ Stores with version vectors do notice writes made side by side and keep them all. This store holds one stamp per value, so it compares two numbers, drops the lower and flags nothing.

</details>

<details><summary>D. The last correction, once the fast clock is put right</summary>

✗ Clock fixes are routine, and a corrected clock makes the next stamps right. The first address keeps the stamp it was stored with, five minutes ahead, and the corrections were already dropped, so putting the clock right brings neither back.

</details>

---

###### 20. Growing a three-member group to five, one host at a time

A video platform keeps its job schedule in a consensus group of three, and each member takes the member list from a settings file when it starts. The team wants five members. The runbook says a member list changes one member at a time, so your script handles one host per step and waits for each to come up: new members 4 and 5 start with a file listing all five, then members 3, 2 and 1 restart on that file in turn. Member 3 is the leader when its turn comes, and its restart sets off an election.

How many leaders can that election give the group?

<details><summary>A. One, from any of the five members, as at any restart</summary>

✗ A rolling restart is how a fleet stays up through a change, and your script never touches two hosts in one step. The hosts moved one by one, while the list on them jumped from three names to five: members 1 and 2 still need two votes to elect a leader, and members 3, 4 and 5 need three.

</details>

<details><summary>B. One, from members 1, 2 or 3, with 4 and 5 waiting to catch up</summary>

✗ New hosts start with an empty log, and groups that add a member through their own log can keep it out of votes until it catches up. A member that takes its list from a file votes as soon as it is up. The rule on logs limits who can win, so members 4 and 5 hand their votes to member 3, which came back with its log.

</details>

<details><summary>C. None, until members 1 and 2 are on the new file</summary>

✗ Members 1 and 2 no longer agree with the others on who belongs, and in a consensus group a disagreement usually means a stop. The members never compare files. Members 1 and 2 count votes on their own list, where two of three is enough, and they have two.

</details>

<details><summary>D. Two in one term, each acknowledging writes</summary>

✓ Right. The election can give the group two leaders in one term, and each can acknowledge schedule changes the other never sees.

Each member counts votes on the list in its own file. **A majority of the old list and a majority of the new list need not share a member**: two of three on the old file, three of five on the new one. So members 1 and 2 can elect one leader while members 3, 4 and 5 elect another. Once the script finishes, the next election goes to the longer log, and writes acknowledged only on the shorter log are gone.

The runbook's "one at a time" means one member added per change to the list, and this script took the list from three names to five.

Change the list through the group's own membership change: add one member, wait until a majority holds the change and the new member has caught up, then add the next. Where the list lives only in files, stop the whole group for the switch.

</details>

---

###### 21. Losing one switch link between two members of the group

A payments platform keeps its routing rules in a consensus group of three, one member per rack. A faulty switch starts dropping all traffic between the rack of member 1, the leader, and the rack of member 3. Every other path works, so member 2 reaches both, and clients reach all three. Writes keep arriving.

Who leads the group while the link stays broken?

<details><summary>A. Member 1 stays leader, and member 3 falls behind</summary>

✗ Members 1 and 2 are two of three, and two of three is all a commit needs. Member 3 does not sit quietly behind that majority: it hears no heartbeat, so it raises the term and asks member 2 for a vote, and that term reaches member 1 through member 2.

</details>

<details><summary>B. Member 3 becomes leader once, and writes stay steady under it</summary>

✗ Failover hands the work to whoever noticed the trouble, and member 3 notices first. It has missed the writes members 1 and 2 committed, so member 2 refuses it a vote. If it won at a quiet moment, member 1 would be the one hearing no heartbeats, and it would raise the term in turn.

</details>

<details><summary>C. Member 1 stops being leader, and writes pause for an election</summary>

✓ Right. Member 1 stops being leader although it is up and member 2 still hears it, and writes pause until an election finishes. One bad link between two healthy machines has interrupted the routing rules.

Member 3 hears no heartbeat, so its timeout runs out, it raises the term and asks member 2, the one peer it can reach, for a vote. Member 2 takes the higher term whether or not it gives the vote, and from then on it rejects member 1's heartbeats as out of date. **A leader that learns of a higher term steps down at once**, and the rejection tells member 1 the new term.

This can repeat while the link is broken: if member 1 wins the next election, member 3 still hears nothing from it and raises the term again. Only a leader every member can hear lasts, here member 2, and no rule in a vote favours it.

Turn on pre-vote, a check in which a candidate first asks whether it could win and keeps its term until it could, or have a voter ignore a vote request that arrives soon after it heard from a leader. With either, member 3 falls behind and member 1 stays, and the group has no margin for another failure until the switch is replaced.

</details>

<details><summary>D. Members 1 and 3 both lead, one on each side of the link</summary>

✗ Each end of the broken link hears nothing from the other, and that is how a split with two leaders starts. Member 2 sits between them with one vote per term, and it passes on the higher term: heartbeats from a leader on the older term get rejected, so member 1 cannot commit alongside a member 3 that won.

</details>

---

###### 22. Leaving a slow disk under the leader overnight

An online grocer keeps its delivery slots in a consensus group of three, one member per rack, with an election timeout of five seconds. At 02:10 the disk under member 1, the leader, starts to wear out: its syncs go from 2 ms to 400 ms and stay there, with no errors. Write latency jumps from 5 ms to about 600 ms, and the alert for it lands in a queue the team opens at nine.

What does the team find at nine?

<details><summary>A. A follower leading, with writes back at 5 ms</summary>

✗ Failover exists for a sick leader, and this one has been sick since 02:10. The followers cannot see its disk: they start an election when heartbeats stop for five seconds, and member 1's heartbeats keep coming, so the role stays where it is.

</details>

<details><summary>B. Member 1 still leading, with writes still at about 600 ms</summary>

✓ Right. Member 1 still leads at nine, and every write since 02:10 has taken about 600 ms. The delivery slots kept saving all night, each one waiting on one bad disk, and no failover ever began.

An election starts only when heartbeats go missing, since **followers judge a leader by its heartbeats, never by how long its writes take**. A heartbeat carries no write, so it has no data to sync. Where a maker runs heartbeats and syncs in one loop, a heartbeat can slip by up to 400 ms, far inside five seconds. Every write in this group enters through the leader and waits behind its syncs, the one running and then its own, so one disk sets the pace for all three members. A leader that commits once its followers' disks hold the write, without waiting for its own sync, would escape that wait, and the 600 ms shows this one does not. The same disk under a follower would have cost nothing, because the leader and the other follower make a majority without it.

Alert on each member's sync time, and page someone when the leader's climbs. Then stop member 1, or hand its role to a follower where the group offers that, and replace the disk once writes are back at 5 ms.

</details>

<details><summary>C. A new leader every few seconds, with writes pausing at each change</summary>

✗ Some groups' logs do warn of late heartbeats on slow disks, and a leader that keeps losing its role is the failure those warnings point at. A late heartbeat starts an election only once it is later than the timeout. Here a sync can hold one back by 400 ms against five seconds, so no follower's timer runs out.

</details>

<details><summary>D. The group read-only, with every write refused</summary>

✗ A file system that hits disk errors can remount itself read-only, and this disk is wearing out. These syncs finish and report success, only late, so member 1 and a follower keep making a majority, and each slot still saves in about 600 ms.

</details>

---

## Further reading

- [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf), 2014: how a group elects a leader, why a committed write survives it, why a local read can go stale and how members change.
- [Consensus on Transaction Commit](https://doi.org/10.1145/1132863.1132867), 2006: why classic two-phase commit blocks when its coordinator fails, and a commit protocol that does not.
- [Don't Settle for Eventual](https://www.cs.cmu.edu/~dga/papers/cops-sosp2011.pdf), 2011: a wide-area store that keeps causal order across keys, built around cases like the photo album.
- [Coordination Avoidance in Database Systems](https://arxiv.org/abs/1402.2237), 2014: a test for which work needs coordination and which can run on both sides of a cut.
- [Time, Clocks, and the Ordering of Events in a Distributed System](https://doi.org/10.1145/359545.359563), 1978: where happens-before and logical clocks come from.
- [Tolerating Slowdowns in Replicated State Machines using Copilots](https://www.usenix.org/conference/osdi20/presentation/ngo), 2020: why one slow leader slows a whole consensus group, and a protocol that keeps its normal latency with one slow replica.

Prev: [Timeouts, retries and circuit breakers](../timeouts-retries-and-circuit-breakers/README.md) · Next: [Distributed locks and IDs](../distributed-locks-and-ids/README.md)
