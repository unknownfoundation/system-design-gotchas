# Releasing changes

Prev: [Containers and orchestration](../containers-and-orchestration/README.md) · Next: [Observability](../observability/README.md) · Reviewed October 2026

Releasing a change moves a running system from one version of its code, schema, data or settings to the next while customers keep using it. Deploying puts code on machines and releasing exposes it to customers, and the two can happen apart.

- **Copy**: one running instance of a service; a service runs many behind a balancer.
- **L7 balancer**: a balancer that reads each HTTP request before it passes the request to a copy.
- **Orchestrator**: the system that starts, restarts and replaces copies from a template, its description of what each copy runs.
- **Health check**: a small request sent to a copy to ask whether it can take traffic. A balancer sends traffic only to copies that pass.
- **Control**: the copies left on the old version during a release, watched beside the new ones so that the same hour and the same day's traffic reach both.
- **Bake time**: the wait between two stages of a release, in which the new version runs before more copies get it.
- **Feature flag**: a value read while the code runs that picks a code path.
- **Kill switch**: a flag kept so that a person can turn a path off with no new build.
- **Migration**: a change to the schema or the stored data, run as a step of a release.
- **Expand and contract**: add the new shape beside the old, move the code and the data across, and remove the old shape once nothing uses it.
- **Backfill**: a job that fills a new column or a new store with the rows that already exist.
- **Dual write**: code that writes each change to two stores; [Event-driven architecture](../event-driven-architecture/README.md) covers what happens when one write fails.
- **Shadow read**: a read also sent to the new store and compared in the background, while the customer gets the old store's answer.
- **Mirroring**: a balancer copying live requests to a new version and throwing its replies away.
- **Config push**: a change to settings, rules or data that the code reads, sent out on a path of its own.

| Kind of release | What it does | What it costs |
| --- | --- | --- |
| Rolling | replaces copies a batch at a time, so old and new serve together | a batch's capacity for the length of the release |
| Blue-green | brings up a second full environment, then flips the balancer to it | two environments, and all customers move in one step |
| Canary | gives the new version to a small share first, and compares it with the control | a slower release, and a control to keep running |

A system that can take downtime can stop everything for a change instead.

A release tool can be set to undo a release when an alarm fires. On a rolling release a rollback is a release too: it runs at a release's pace, needs the old build kept, and also takes out any fix bundled with the fault. It leaves config and secrets held outside the template where they are, and the old version can crash against them. Rolling forward ships a fix instead. A down migration builds the old shape again, without the old values. An app installed on a phone cannot be recalled, so old versions keep calling while people keep them.

Moving data between stores usually goes in this order: dual write, backfill, shadow reads, read switch, write switch, removal of the old store. Once writes go only to the new store, the old one falls behind, and a way back needs writes sent the other way.

It bites while two versions run side by side, on the paths a release does not cover, and on the way back.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- say what a customer gets while two versions serve, and where a release stands when nobody is watching it
- read what a flip to green, a canary, a set of stages and mirrored traffic have and have not tested
- change a flag, a column or a whole store under running code, and pick what to check before each step
- push settings and roll back knowing what travels where and what stays behind

###### 1. Opening a seat map during a rolling release

A coach company runs its site on twelve copies behind an L7 balancer. Today's release adds a seat map: the booking page now holds a script that calls a new route, `GET /seats`. The rolling release replaces two copies at a time, and at lunchtime six are on the new version and six on the old. On staging, where the whole environment is replaced at once, the map never failed. A passenger gets the new booking page and reloads the map a few times.

What does the script get from `GET /seats` over those few calls?

<details><summary>A. A seat map, with each call going down one connection to one copy</summary>

✗ A browser reuses its connection, and that is why follow-up calls are fast. Behind an L4 balancer, which passes a connection along without reading the requests in it, that connection would stay on one copy. An L7 balancer reads each request and chooses a copy for it, so the next call can land on an old copy and get a 404.

</details>

<details><summary>B. A seat map, with the balancer retrying any 404 on another copy</summary>

✗ Retries are among a balancer's settings, and they exist to work around a backend in trouble. A 404 is a complete reply from a copy that is working, so by default the balancer hands it to the script, and only a failed connection or a timeout sends a request elsewhere.

</details>

<details><summary>C. A seat map on some calls and a 404 on others</summary>

✓ Right. About half the calls return the seats and the others return a 404, and the passenger watches the map come and go on a page that never changed.

The page and its script's call need not reach the same copy, because **an L7 balancer chooses a copy for each request**. An old copy has never heard of `GET /seats`, and staging, replaced all at once, never held this mix.

Ship the route in one release and the page that calls it in the next, so that every copy can answer before any page asks. Run your release tests against a pool that holds both versions.

</details>

<details><summary>D. A 503 from the balancer for as long as the versions are mixed</summary>

✗ A balancer can route by URL path, and a 503 is a reply a balancer makes for itself. It makes that reply when no healthy copy is left. Here all twelve pass their checks, and the call goes to a copy, which returns the seats or a 404.

</details>

---

###### 2. A new version that exits at startup

A payroll service runs twenty copies, and at 17:00 on Friday an engineer starts a release, which the orchestrator runs two copies at a time. The new version opens a larger connection pool. The first six new copies start and serve, and from the seventh on a new copy cannot get a connection, because the database has reached its limit, and it exits a second after it starts. The team's alarms watch errors and latency, and the engineer goes home for the weekend.

What does the engineer find on Monday morning?

<details><summary>A. A mix of old and new copies with two failing, and no alert sent</summary>

✓ Right. The fleet has sat at twelve old copies, six new and two failing since Friday evening, and nobody has heard about it. Payroll has run all weekend on two versions, with two copies fewer.

An orchestrator starts the next batch only when the last one reports healthy, so the failing pair holds the release where it stopped and the old copies go on serving. **With no automatic rollback turned on, a stalled release shows up only as a status field**, and the orchestrator takes no other action. An automatic rollback undoes the release when its alarm fires.

Give every release a deadline, and page someone when it passes. Treat a stalled release as unfinished work: finish it or undo it.

</details>

<details><summary>B. The same mix, with the on-call paged on Friday evening</summary>

✗ Release screens mark a stuck release, and a failure is what a page is for. The mark is a status field that no alarm reads, because the team's alarms watch errors and latency, which stay normal because the old copies answer.

</details>

<details><summary>C. Twenty new copies, the last two through after retries</summary>

✗ A restart loop clears a passing fault, and the orchestrator keeps restarting these two. The database has no connections to give all weekend, so each attempt exits like the one before and the count of new copies stays at six.

</details>

<details><summary>D. One payroll request in ten failing at the two copies that exit</summary>

✗ Two copies of twenty hold a tenth of the traffic when all twenty serve. The balancer sends nothing to a copy that has not passed its health check, and these two never live long enough to pass, so requests go to the eighteen that answer.

</details>

---

###### 3. Mistyping the payment address in a rolling release

A shoe shop runs its checkout service on twenty copies, and the orchestrator releases two at a time with its default settings, which usually takes ten minutes. Its health check is `GET /healthz`. Today's build holds a wrong `PAYMENT_URL`, and half an hour after the release starts, someone in support looks at the release tool.

```python
@route("/healthz")
def healthz():
    return "ok"

@route("/checkout", methods=["POST"])
def checkout(order):
    return payments.charge(PAYMENT_URL, order)
```

Where does the release stand at that moment?

<details><summary>A. Halted after two copies, with eighteen old ones serving</summary>

✗ Halting on failure counts is a common feature, and the screen shows the counts beside the progress bar. The next batch started on time, because the release's only input is the health check and `healthz` answered on both new copies.

</details>

<details><summary>B. Undone by the orchestrator, with twenty old copies serving</summary>

✗ Some pipelines undo a bad release on a burst of 500s. That takes an alarm wired to the release or a setting turned on. This orchestrator runs on its defaults with copies that pass their check, and it has no reason to undo anything.

</details>

<details><summary>C. Paused, with the two new copies out of the balancer</summary>

✗ A balancer that watches live traffic can drop a backend that returns errors, where that is turned on. Dropping a copy from the balancer tells the orchestrator nothing: it reads `healthz` and starts the next batch.

</details>

<details><summary>D. Finished, with twenty new copies serving</summary>

✓ Right. The release finished twenty minutes ago with all twenty copies on the new version. Every checkout fails, and the shop has taken no payment since the last old copy left.

**At default settings a rolling release moves on what its health check says**, and this check asks only whether the process is up. Each new copy answers `healthz`, so the next batch starts, and ten batches later the old version is gone. The orchestrator never sees the replies customers get.

Wire the release to an alarm on the new copies' errors, and run a real checkout against the first batch before the second starts.

</details>

---

###### 4. Bringing up green two hours before the flip

A ticket seller releases blue-green: two full environments, each with web copies and with worker copies that turn a paid order into a PDF ticket. They differ only in version, with the same settings and the same addresses for the database and the queue. Green, with a redesigned PDF ticket, comes up at 09:00 so the team can check it, and the balancer is due to flip to it at 11:00. A fan buys a ticket at 10:00.

Which version makes that fan's PDF?

<details><summary>A. Blue's worker, with green idle until the flip</summary>

✗ Every drawing of blue-green shows one busy environment and one idle one, and at 10:00 the balancer still points at blue. A worker never passes through the balancer: it reads its queue from the address in its settings, and green's workers have been doing so since 09:00.

</details>

<details><summary>B. A worker of either version, whichever takes the job first</summary>

✓ Right. Either version can make it. Some fans get the redesigned ticket before the team means to show it, from a build still being checked.

In a blue-green release **the balancer moves only what comes through it**, and workers pull from the queue at the address in their settings. Green's workers hold the same address as blue's, so they take production jobs from the moment they start, and a queue gives each job to whichever worker asks first.

Give green's workers a queue of their own while you check them, and move the workers across as a step of the flip, on purpose.

</details>

<details><summary>C. Blue's worker, with green's making a second PDF</summary>

✗ On a topic each subscriber gets its own copy of a message, and here two pools read from one address. A queue hands each job to one worker, so the fan gets one PDF, from whichever pool took it.

</details>

<details><summary>D. Green's worker, with blue's workers stopped at 09:00</summary>

✗ On one machine a new deployment replaces the old one's background jobs. Green is a second environment, and bringing it up touches nothing in blue: at 10:00 blue's workers are still taking jobs beside green's.

</details>

---

###### 5. Starting a table scan on a one-copy canary

A food delivery app runs twenty copies of its order service on one database. The new version starts a background job that scans the `orders` table once a minute, and at lunchtime, with each copy handling hundreds of requests a second, one copy runs it as a canary and the other nineteen are the control. Ten minutes in, the database is at 95% CPU. The release tool compares the canary's latency and errors with the control's, then says whether the release goes on.

What does the comparison report at that point?

<details><summary>A. The two groups alike, and a go</summary>

✓ Right. The comparison finds nothing between the two groups and lets the release go on, though every customer in the app is now waiting on a database at 95% CPU.

The canary and the control both wait on one database, and **a comparison can find only what differs between them**. If a reply takes 33 ms at 40% CPU and 400 ms at 95%, both groups show 400, so their ratio is 1.0 while each is 12 times slower than it was. A canary limits what runs on one copy, and what that copy does to a backend it shares reaches the control as well.

Check absolute limits on latency and on the load of each shared backend beside the comparison, and stop on either one.

</details>

<details><summary>B. A canary slower than the control, and a stop</summary>

✗ New code runs on the canary, so harm belongs on the canary, and that is how the method is taught. The job's harm sits in the database. A control request and a canary request wait in one line there, so the two groups slow down together and neither looks worse.

</details>

<details><summary>C. The control slower than the canary, and a go</summary>

✗ The control carries nineteen times the canary's traffic, so it sends the database nineteen times the queries. The database serves them all from one queue, and a query's wait does not depend on which copy sent it, so each group's replies slow by the same amount.

</details>

<details><summary>D. Too few canary requests to judge, and a hold</summary>

✗ Analysis tools want a minimum sample, and one copy in twenty is a small part of the fleet. At hundreds of requests a second, ten minutes on that copy gives the tool far more than a comparison needs.

</details>

---

###### 6. Widening a release every fifteen minutes

A pharmacy chain has a pipeline that takes a new version from one copy to six and then to all sixty, replacing each stage's copies at once. It waits fifteen minutes at each stage, comparing errors and latency with the old version. Every stage passes, and by 10:00 the release is done. The new version holds on to a little more memory with every request, and at usual traffic, shared evenly, a copy reaches its memory limit after about nine hours. The orchestrator restarts a copy that does, and a restarted copy takes about five minutes to serve again.

What do customers meet at about seven that evening?

<details><summary>A. Nothing they can see, with each restart clearing the memory</summary>

✗ Restarting cures a leak for a while, and a fleet of sixty has room to lose one copy or two. Copies that started within the same hour reach the limit within the same stretch, and that many down at once is more than the room holds.

</details>

<details><summary>B. A failed request now and then, as copies restart one by one through the day</summary>

✗ Independent faults on many machines spread out, and spare capacity is sized for that. This fault counts from the moment a copy starts, and the stages started the copies within a short span of each other, so the restarts bunch together nine hours later.

</details>

<details><summary>C. Errors while a large part of the fleet restarts together</summary>

✓ Right. The copies reach the limit so close together that a large part of the fleet is restarting at once, and for that stretch customers get errors and slow pages.

Stages widen a release fast: the last 54 copies all started at the last widening, and with even traffic each gains memory at the same speed. **Copies that start together and leak alike reach the limit together**: if shares differ by 1% either way, they arrive within about a quarter of an hour, and at five minutes a restart nearly half the fleet is down at once.

Keep one stage running through a full day's peak before the last widening, and alarm on memory that keeps growing on the new version.

</details>

<details><summary>D. Replies that have slowed all day and stay slow</summary>

✗ On a laptop a leak shows up as a machine that gets slower and slower, because nothing kills the program. A copy has a memory limit, and it goes from normal to killed at that limit, so these copies run at their usual speed for most of the nine hours and then stop.

</details>

---

###### 7. Mirroring live traffic to a new order service

A grocery app tests its rewritten order service by having the balancer copy every live request to it and throw its replies away, so customers only ever see the old service's replies. The new service runs with production settings, and like the old one it lets the store number each order and emails a receipt. The same mirror ran on the search service last month, and nobody noticed a thing. A customer places one order at lunchtime.

What does that customer end up with?

<details><summary>A. One order and one receipt</summary>

✗ The mirror is a test, and last month's mirror on search hurt nobody, because search only reads. The balancer throws away the new service's reply after its handler has run, so the order and the receipt it made are already real.

</details>

<details><summary>B. Two orders and two receipts</summary>

✓ Right. Two orders go into the store and two receipts go to the customer, while the app shows one order, the old service's.

**Mirroring drops the reply and nothing else**: the new service gets a whole request and runs its whole handler, and production settings point it at the real store and the real mailer.

Mirror only reads, or point the new service at stubs for everything it writes and sends. Compare what it would have written, and leave the writing to the old service.

</details>

<details><summary>C. One order and two receipts</summary>

✗ Stores have constraints and mailers do not, and an order table usually has a key. The store has nothing to catch here, since it numbers each order afresh, so the second order goes in, and the second receipt goes out.

</details>

<details><summary>D. One order, with the second refused as a duplicate</summary>

✗ A unique key refuses a repeat, and it would here if the id came with the request. Here the store makes the id, so to the store these are two different orders, and it takes both.

</details>

---

###### 8. Adding copies while the flag service is down

A bookshop's new search has been on for every customer for a year through the flag `new_search`, and the old search is still in the code. Running copies hold the flag's rules in memory and refresh them from the flag service each minute, and a starting copy waits briefly for its first fetch. At 14:00 the flag service goes down for an hour, and at 14:20 the autoscaler adds ten copies.

```python
def search(query):
    if flags.is_on("new_search", default=False):
        return new_search(query)
    return old_search(query)
```

Which search do customers get from the ten new copies?

<details><summary>A. The new search, as on the running copies</summary>

✗ Flag clients' documentation promises that they keep working through an outage, and the running copies show it. They keep what they fetched before 14:00. A copy started at 14:20 has fetched nothing, so it has no rule that says on.

</details>

<details><summary>B. An error from each search request</summary>

✗ A call to a service that is down fails, and most clients in your code behave that way. A flag client is built not to: when it has no value, `is_on` returns the default it was handed, and the request carries on down that path.

</details>

<details><summary>C. No search: the ten copies never join the balancer</summary>

✗ Start-up waits and health checks exist so that a copy can hold out for what it needs. A flag client may wait briefly at the start, then gives up and answers with the default, so these copies join the balancer and serve.

</details>

<details><summary>D. The old search</summary>

✓ Right. The ten new copies serve the old search, a year out of use, beside running copies that serve the new one, and a customer's results change from one request to the next.

Whenever it cannot get a value, **a flag evaluation returns the default written in the code**, and a copy that starts during the outage has no rules at all. The `default=False` was right on the day the flag was added. A year later production runs with the flag on, and the old path has had no traffic in that time.

When a flag reaches every customer, finish the job: delete the flag and the old path. Until then, set the default to what production runs now, or let clients start from a local file of the last rules.

</details>

---

###### 9. Turning off the flag while copies run out of memory

An airline's search runs a new fare engine behind the flag `new_fares`, and turning it off is the team's kill switch. At 03:00 a fare import doubles the fare table. An index of that size is over a copy's memory limit by itself. The next hourly build runs on all copies together and goes over it, and the orchestrator restarts every copy it kills. At 03:10 the on-call engineer turns the flag off, and flag changes reach clients in seconds.

```python
index = FareIndex.build(store.fares())

@every("1h")
def refresh():
    global index
    index = FareIndex.build(store.fares())

def search(query):
    if flags.is_on("new_fares"):
        return index.search(query)
    return old_search(query)
```

What do travellers get from search at 03:20?

<details><summary>A. The same failed searches as before the switch</summary>

✓ Right. Search goes on failing as it did before the switch. Travellers see errors, and the switch the on-call engineer reached for has changed nothing.

The flag guards one call inside `search`, and **a kill switch stops nothing that runs before the flag is read**. `FareIndex.build` runs when a copy starts, before it can take a request, and it never asks the flag. Each restart builds the doubled index, passes the limit and dies, so no copy lives long enough to reach the line where `new_fares` is read.

Read the flag before building the index, so that off means the work is never done. Then test the switch with the new path failing at start.

</details>

<details><summary>B. The old search on every copy within a minute</summary>

✗ A flag takes effect in seconds, and the code shows `old_search` waiting behind it. The flag change reaches a client that is running, and these copies die during the build, before any request gets as far as that `if`.

</details>

<details><summary>C. The old search on copies as they restart</summary>

✗ Whatever is new in the settings arrives with a restart, and the flag is off by then. The first line a copy runs is the build, with no flag in it, so the restarted copy runs out of memory the way the last one did.

</details>

<details><summary>D. The old search on the copies still up, and failures on the rest</summary>

✗ Outages often take some copies and leave others, and the old search works on any copy that lives. The fare table is the same for all of them, so each copy's build passes the same limit, and none stays up to serve.

</details>

---

###### 10. Adding a required column before the rolling release

A clinic's booking service, on a relational database, adds `consent_version` to `appointments` as `NOT NULL` with no default, and the migration fills it in for every existing row. The migration runs at 09:00 as the first step of the release, and copies move to the new version from 09:05 to 09:35. The new code sets the column, and the old code has never heard of it. Every test ran on the new code against the new schema.

What do the copies not yet replaced do at 09:10?

<details><summary>A. Show and book appointments as usual</summary>

✗ The rule of thumb says adding is the safe direction, and old code has no line that touches this column. The store is what touches it: an insert from old code leaves the column out, and the constraint refuses that row.

</details>

<details><summary>B. Show appointments, and fail to book new ones</summary>

✓ Right. Old copies still read appointments and every new booking on them fails: for up to half an hour, whether a patient can book depends on the copy that answers.

Reads name the columns they want, and an extra column changes nothing for them, while **at default settings every `NOT NULL` column with no default needs a value in each insert**. The old code gives none for one it has never heard of. The tests never saw this, since all of them ran the new code, which sets the column.

Make it two releases, the expand half of expand and contract. Add the column nullable or with a default, ship the code that sets it to every copy, fill the rows, and only then add `NOT NULL`.

</details>

<details><summary>C. Fail on every query to the table</summary>

✗ Code that no longer matches its table is a familiar failure, and a query that names a missing column does fail. Nothing is missing here: a `SELECT` of the old columns still finds them all, so only one kind of statement breaks.

</details>

<details><summary>D. Book as usual, with the column left empty</summary>

✗ An added column is usually nullable, and an old insert leaves that kind empty with no complaint. This column is `NOT NULL` with no default, and a store at its default settings refuses the row. Only a relational store with strict checking turned off would fill in a value of its own.

</details>

---

###### 11. Reading both session formats in one release

A shop keeps each session in a signed cookie and is moving to a smaller format. An unreadable cookie has always meant a logged-out page, with the cookie left in the browser. The release below reads both formats and writes the new one, and its tests with old and new cookies all pass. It rolls over forty copies in twenty minutes, behind a round robin balancer, and at minute five a customer logs in on a new copy.

```python
def read_session(cookie):
    if cookie.startswith("v2."):
        return parse_v2(cookie)
    return parse_v1(cookie)

def write_session(session):
    return "v2." + encode_v2(session)
```

What does that customer get on the next few pages?

<details><summary>A. Logged-in pages throughout</summary>

✗ The code reads both formats and the tests cover both, so the change goes through review as compatible. Those tests ran the new code, and an old copy still in the pool has only `parse_v1`, which cannot read the `v2.` cookie this customer now holds.

</details>

<details><summary>B. One logged-out page, then logged-in pages from then on</summary>

✗ A format change often costs one login, after which a new cookie settles the matter. The logged-out page leaves the `v2.` cookie in the browser, so the next old copy that answers cannot read it either.

</details>

<details><summary>C. A logged-out page each time an old copy answers</summary>

✓ Right. The customer is logged in on pages from new copies and logged out on pages from old ones, flipping from click to click until the last old copy is gone.

**Reading both formats helps only on the copies already on the new version**, and this login landed on a new copy, which wrote a `v2.` cookie. Round robin sends the next request to whichever copy is next, so an old copy gets a cookie it cannot read.

Split the change in two: first a release that reads both formats and still writes the old one, then, once no old copy is left, a release that writes the new one, with bake time in between.

</details>

<details><summary>D. Logged-out pages on every copy for the rest of the release</summary>

✗ A format that changes for everyone at once logs everyone out on that day, and this one is changing today. New copies read a `v2.` cookie perfectly well, so this customer stays logged in whenever one of them answers, and the failures come only from old copies.

</details>

---

###### 12. Renaming a column while the release rolls

A dating app renames `email` to `contact_email` by expand and contract. Nothing logs an error at any step, and the backfill skips no row.

```text
10:00  migration adds contact_email, empty
10:00  rolling release starts, 40 copies: new copies write email and contact_email
10:05  backfill starts: copies email into contact_email, row by row
10:20  backfill ends
10:30  rolling release ends
Tue    second release reads contact_email
```

Who sees an outdated email after Tuesday's release?

<details><summary>A. Some of those who changed their email before 10:30</summary>

✓ Right. A customer whose change landed on an old copy, after the backfill had passed their row, changed only `email`. From Tuesday the app reads back an address they gave up, and no error marks them.

Until 10:30 some copies still write `email` alone, and **a backfill copies a row once**. If an old copy changes a row the backfill has not reached, the backfill carries the change across. If it changes a row already passed, `contact_email` keeps the value from before, for good. Say 120 changes hit 600 rows evenly through the half hour: the two columns end up differing on about 20 rows.

Start the backfill after the last old copy has gone, and compare the two columns over every row before the read switch.

</details>

<details><summary>B. Nobody, with the two columns matching</summary>

✗ The steps ran in the listed order and nothing failed, which is what a recipe asks. Old copies were still writing `email` alone for the ten minutes after the backfill ended, so changes that landed on them in that stretch never reached the new column.

</details>

<details><summary>C. Everyone who changed their email during the rolling release</summary>

✗ A change made in the middle of a release has the worst timing of any. Most changes survive: one on a new copy writes both columns, and one on an old copy before the backfill reaches its row gets carried across.

</details>

<details><summary>D. People who have not logged in for years</summary>

✗ Batch jobs miss the old corners of a table when they give up early. This backfill walks every row and an unchanged row carries no risk, since `email` holds the same value before and after it passes, so those people get the right address.

</details>

---

###### 13. Running a backfill through the afternoon

A furniture shop is adding `display_name` to its 300-million-row `customers` table, and from 14:00 one worker fills the column, as fast as the primary takes the updates. On staging, with a few thousand rows, the backfill finished in seconds. In production the primary's CPU stays at 40%, and checkout stays fast. Profile pages read from two asynchronous replicas on much smaller machines. At 15:00 a customer changes their address, and the page reloads from a replica.

What address does the page show?

<details><summary>A. The new address, with the replicas untouched by a job on the primary</summary>

✗ The worker is pointed at the primary, and replicas serve only reads. Every update it makes still goes down the replication stream, so each replica has those rows to apply as well, ahead of this customer's save.

</details>

<details><summary>B. The new address, after a save that took minutes</summary>

✗ Bulk jobs slow other writes, which is why they usually get the night. This primary has room: it sits at 40% CPU, and the save returns quickly. The wait comes after the save, on the replica.

</details>

<details><summary>C. An error page while the replica catches up</summary>

✗ An error page is what a struggling database usually gives you, and this replica is far behind. A replica that is behind still serves whatever it has applied so far, so the page gets a normal answer, and that answer is out of date.

</details>

<details><summary>D. The old address</summary>

✓ Right. The page shows the old address with no error, and the customer thinks the save failed and may well enter it again.

Each row the backfill changes is a write the replicas must apply too, and **a job running as fast as the primary takes updates can outrun its replicas** while the primary's own CPU looks fine. The save joins the back of that stream. If the worker writes 20,000 rows a second on top of the usual 1,000 and a replica applies 8,000, the replica is about 98 minutes behind after an hour.

Throttle the backfill on replica lag rather than the primary's CPU, in small batches, and watch the replicas behind every read path while it runs.

</details>

---

###### 14. Two agents saving one phone number in a dual write

A bike-sharing service is moving accounts to a new store. It now saves every change twice, first in the old store and next in the new one, and reads still come from the old store. Both stores take every write, and retries cover the odd blip. Two support agents, on two copies of the service, save different phone numbers for one account in the same moment.

What can the two stores hold for that account afterwards?

<details><summary>A. The same number in both every time, the later one</summary>

✗ Last write wins is most stores' own rule for one record, and it would settle this inside a single store. Here each store sees its own order, and "later" can name one agent's number in the old store and the other's in the new.

</details>

<details><summary>B. The number the old store took last, copied over to the new one</summary>

✗ The old store is still the source of truth, and reads come from it. Nothing copies from one to the other: each copy of the service writes to both on its own, and the writes to the new store arrive in their own order.

</details>

<details><summary>C. One number in the old store and the other in the new</summary>

✓ Right. The old store can end on one agent's number and the new store on the other's, with every write a success. After the switch this customer's phone number changes for no visible reason.

**Each store applies the writes in the order they reach it**, and the two stores share no clock or lock that lines those orders up. One agent's copy can get to the old store ahead of the other and fall behind on the way to the new store. Retries protect against a write that fails, and neither write failed.

Compare the stores row by row before the switch and repair from the old store, or write to the new one from an ordered log of the old store's changes.

</details>

<details><summary>D. The first number in both, with the second write refused as a conflict</summary>

✗ A form that checks versions refuses the second of two saves. These are plain writes to a phone field with no version to compare, so both stores take both and each ends on its own last one.

</details>

---

###### 15. Reporting two weeks of shadow reads with no mismatch

An insurer is moving policies to a new store. The backfill is done, writes go to both stores, and reads still come from the old one. For two weeks every read has also gone to the new store in the background and the two answers have been compared: forty million comparisons across six million policies, and no mismatch. Before the read switch on Friday, the head of claims asks how far that goes.

How many of the six million policies have the comparisons covered?

<details><summary>A. All six million, each about six or seven times</summary>

✗ Forty million is many times six million, enough for six or seven comparisons of every policy if they were shared out. A comparison comes only with a read, and reads are not shared out: they pile up on the policies in use, so some get thousands and others none.

</details>

<details><summary>B. Many of them, with some never compared</summary>

✓ Right. Many policies have been compared and some never have. The clean result covers none of those, and a claim on one of them may be the first read it gets.

**Shadow reads compare what customers ask for**, and a policy nobody opens is never compared. Say reads fall off with popularity, the second policy read half as often as the first: at these figures about 38% of policies are never compared. Even with each read landing at random, all policies equally likely, about 7,600 would be missed.

Before the switch, also compare every row in a full scan, allowing for writes in flight.

</details>

<details><summary>C. All six million, the busy ones far more often</summary>

✗ Busy policies soak up most of the reads, and forty million leaves plenty for the rest. That share falls away fast towards the quiet end of the table, where a policy gets less than one read on average, so many of those are never opened.

</details>

<details><summary>D. Only those written to in the two weeks</summary>

✗ Writes have gone to both stores since the backfill, so recently written rows are where the two stores are likeliest to differ. A shadow read follows reads and has nothing to do with writes: an old policy someone opens gets compared whether or not it was written to.

</details>

---

###### 16. Saving a rule with a blank field

A video platform released a gateway build that reached every region three weeks ago, through five stages over a week, with no errors. One new branch in its rule loader crashes the process when a customer's rule has `max_bitrate` blank, and no rule has ever had it blank. Every gateway keeps the whole rule set in memory, so that it can serve any customer, and watches the rule store, which spans the regions and replicates in seconds. This morning a customer saves a new rule with that field blank, and the API accepts it.

What happens to the gateways over the next minute?

<details><summary>A. Gateways in every region crash, and crash again on restart</summary>

✓ Right. Within seconds gateways in every region crash, and each restart loads the same rule and crashes again. The whole platform is down over one customer's valid save.

The stages proved the build on the rules of that week, and none of those rules reached this branch. The new rule then went out as a config push, and **a config push travels on a path of its own, outside the build's stages**. This one replicated in seconds, and every gateway loads every rule.

Send rule and data changes through stages of their own, with a bake time. Make a reader skip a record it cannot use, and put a flag around a new branch.

</details>

<details><summary>B. Those handling that customer's traffic crash, and the rest serve on</summary>

✗ Keeping customers apart promises that a customer's rule touches only that customer. Every gateway loads all rules, whoever's traffic it handles, so every gateway runs the branch.

</details>

<details><summary>C. One region crashes, and the pipeline holds the rule back from the rest</summary>

✗ Every build here has gone out region by region, through five stages. The rule is not a build, and the pipeline never sees it: the global store has it in all regions in seconds, before anyone could hold anything back.

</details>

<details><summary>D. Each crashes once and comes back clean on restart</summary>

✗ Orchestrators lean on restarts because most crashes do not repeat. This one is written into the data: a restarting gateway loads all rules at start, the blank field is among them, and the branch runs again.

</details>

---

###### 17. Pushing a broken rules file before a routine release

A payment gateway reads its fraud rules from a settings store. In testing you pushed a broken file: every copy logged the error and went on serving with the rules it had. On Monday evening a broken file goes out, and nobody replaces it. On Wednesday morning an unrelated rolling release starts.

```python
rules = parse(store.get("fraud_rules"))

def on_rules_changed(text):
    global rules
    try:
        rules = parse(text)
    except ParseError:
        log.error("bad rules file, keeping the last good one")
```

What happens to Wednesday's release?

<details><summary>A. It finishes, with new copies holding the last good rules</summary>

✗ The log line says the last good file is kept, and the test showed it. It is kept in the memory of the copy that logged the line. A new process has nothing but the store, and the first line it runs parses the broken file.

</details>

<details><summary>B. It finishes, with new copies running no fraud rules</summary>

✗ A check that cannot load is sometimes skipped, and failing open is what some readers are built to do. This one catches nothing at its first line: a parse error at start ends the process, and the copy never serves, with rules or without.

</details>

<details><summary>C. It meets a service already down, every copy stopped by Monday's file</summary>

✗ Outage reports are full of a broken file that stops whoever reads it. The handler here catches the error, and the test showed as much, so since Monday every running copy has gone on serving with the rules it had.

</details>

<details><summary>D. It stalls, with new copies that fail to start</summary>

✓ Right. The first new copies exit at start and the release stalls with old copies serving, while the team looks for a bug in a build that has nothing to do with it.

The handler keeps its rules in the memory of each running copy, and **a last good file held in memory does nothing for a copy that starts later**. A bad push stops nothing until a copy starts, and then every start fails: a release, an autoscaler adding copies, a crash, a machine going away.

Check a file before it goes into the store, alert on the parse error, and save the last good file somewhere a starting copy can read it. Then test a fresh copy starting while the store holds a broken file.

</details>

---

###### 18. Changing a timeout in the settings object

A logistics service runs twelve copies, and the orchestrator fills each copy's environment from a settings object kept outside the template, including `PARTNER_TIMEOUT_MS=5000`. A partner has started hanging, so at 10:00 you change the value to 2000 in the object and save it, and the settings screen shows version 14, saved. You start no release.

Which timeout are the twelve copies using at 11:00?

<details><summary>A. 2000 on some copies, as a restart works through them</summary>

✗ Change the template and the orchestrator starts a release, replacing copies batch by batch. A settings object is a different thing from the template: saving it starts no release, so no copy restarts, and all twelve keep what they have.

</details>

<details><summary>B. 5000 on all twelve</summary>

✓ Right. All twelve are still on 5000 an hour later, and calls to the partner go on hanging for five seconds, though the settings screen shows version 14 saved.

**The environment of a process is set when it starts**, and nothing outside it changes the values after that. The orchestrator read the object when it started each copy, and it has started none since the save. Only a change to the template starts a release, and this object is outside it.

After a change like this, start a release on purpose so every copy restarts, and read the value back from a running copy. A setting you need to change in minutes belongs where the code reads it while it runs.

</details>

<details><summary>C. 2000 on all twelve</summary>

✗ A config file that a program rereads can change while it runs, and this screen says saved. An environment variable has no such path: nothing delivers a new value to a running process, so the code goes on reading the 5000 it started with.

</details>

---

###### 19. Rolling back a release that stores prices in cents

A bakery marketplace keeps `price` in a numeric column, in dollars. Today's release keeps cents in the same column, converting the rows as its first step, and the review passed because the schema does not change. It goes live at 13:00. At 15:00 an unrelated bug turns up in search, and the on-call engineer runs the orchestrator's rollback, which has all thirty copies on the old version within minutes.

What does a customer see for a 19.99 cake at 15:10?

<details><summary>A. 19.99, with the rollback having restored the rows</summary>

✗ A database transaction's rollback undoes all it did, and that is the first meaning of the word most people learn. The orchestrator's rollback shares only the name: it puts the previous template back on the copies and sends no statement to the store, so the rows stay in cents.

</details>

<details><summary>B. 19.99 unless a seller edited the cake after 13:00</summary>

✗ A backup restored over a folder brings back what was untouched and can leave new files behind, and here the sellers' edits are what is new. The conversion was the release's first step and touched every row, edited or not, so this cake holds 1999 either way.

</details>

<details><summary>C. A price a hundred times too high</summary>

✓ Right. The page shows the cake at 1,999.00, a hundred times its price, and every other product is wrong the same way, all from a rollback for a small search bug.

**An orchestrator's rollback brings back the previous template and nothing else**, so the rows still hold cents, and the old code reads whatever is in that column as dollars. Nothing fails, because 1999 is a perfectly good number.

Add a new column for the new meaning, as in expand and contract, so old code keeps a column it can read. Before each release, write down what a rollback would meet.

</details>

<details><summary>D. An error in place of the price</summary>

✗ A validating reader refuses a value of the wrong type, and old code did not write these rows. The type is still numeric and 1999 passes every check the old code has, so it shows the number with no complaint.

</details>

---

###### 20. Reusing a retired flag in a release to eight hosts

A parcel carrier sends its delivery texts from eight hosts, one copy of the service on each, all reading one queue. The flag `burst` once switched on a load-test routine that sends each text over and over. It has been off since 2021, and the ticket to delete the routine is still open. Tuesday's build 412 gives `burst` a new meaning, priority sending, and the pipeline ends with 8 of 8 hosts done. At 09:30 on Wednesday the flag goes on. By 09:40 customers are getting the same text dozens of times, and the send log shows every repeat coming from host 6. The on-call engineer rolls all eight back to build 411, the last good one, and means to look at the flag afterwards.

What do the eight hosts do with texts once the rollback is done?

<details><summary>A. All eight send each text once, as before the release</summary>

✗ Build 411 sent these texts cleanly up to Tuesday, which is why it counts as the last good one. It did that with `burst` off. The rollback brings back the build and leaves the flag on, and in 411 that flag still leads to the routine, so every host starts repeating.

</details>

<details><summary>B. Host 6 repeats texts, and the other seven send each one once</summary>

✗ The repeats have all come from host 6, and the pipeline says all eight got one build, which leaves a fault in that machine as the usual suspect. What sets host 6 apart is its build: it is still on 411. The other seven sent each text once because 412 gives `burst` another meaning, and the rollback takes 412 away from them.

</details>

<details><summary>C. All eight send nothing, with build 411 failing to start</summary>

✗ An old version can crash against settings it was never built for, and this flag went on after 411 was built. Build 411 has known `burst` for years and reads it while it runs, so the value cannot stop a start. Host 6 has been on that build with the flag on since 09:30, up and sending.

</details>

<details><summary>D. All eight repeat each text</summary>

✓ Right. All eight hosts repeat every text, and the rollback has taken the fault from one host to the whole fleet: every customer waiting on a delivery text now gets it over and over.

Host 6 can only have been repeating on a build that still reads `burst` as the routine, so it is still on 411, and the pipeline's 8 of 8 did not show that. The flag has one value for every host, and **the build that reads a flag decides what the flag means**: on 412 `burst` meant priority sending, and on 411 it meant the routine nobody had deleted. A rollback changes the build and leaves the flag on, so all eight now read it the way host 6 did.

Turn the flag off first, since off means each text goes once on both builds. Give a new meaning a new flag, delete a retired routine along with its flag, and before any flag goes on, have every host report the build it runs.

</details>

---

## Further reading

- [Understanding and Detecting Software Upgrade Failures in Distributed Systems](https://doi.org/10.1145/3477132.3483577), 2021: 123 real upgrade failures in 8 widely used systems, about two thirds of them from two versions meeting through stored data or network messages.
- [Gandalf](https://www.usenix.org/conference/nsdi20/presentation/li), 2020: a service that ties fault signals to the rollouts under way and decides whether each may go on, in production at a large cloud for over 18 months.

Prev: [Containers and orchestration](../containers-and-orchestration/README.md) · Next: [Observability](../observability/README.md)
