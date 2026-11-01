# Reliability and disaster recovery

Prev: [Load testing](../load-testing/README.md) · Next: [System design](../system-design/README.md) · Reviewed October 2026

Reliability is how much failure a service may have and how it keeps serving through a loss; disaster recovery is how it gets back after a loss too large to ride out. It takes a target, copies kept apart, backups that restore, and a plan for moving traffic and data.

- **Target**: the figure you hold a service to over a window, also called a service level objective (SLO); the number it is judged on is the indicator (SLI), and a contract's promise to customers, the agreement (SLA), sits below it.
- **Error budget**: the failure the target allows. It acts only through a rule agreed in advance, such as stopping feature releases while it is spent.
- **Zone and region**: a zone is one site or group of sites in a cloud, and a region is a set of zones in one part of the world.
- **Failure domain**: what fails together: a machine, a rack, a zone, a region, one provider, one account's credentials.
- **High availability**: keeping on serving through a small loss, with nobody having to act.
- **Recovery point objective (RPO)**: how far back in time the data may go after a recovery.
- **Recovery time objective (RTO)**: how long the service may be down before it is back.
- **Write-ahead log**: the store's own record of every change, kept in segments, as storage-level changes (a physical log) or as row values (a logical log); archived as it grows, it lets a restore replay changes made after a full backup, also called a base backup; replaying up to a chosen moment is point-in-time recovery.
- **Logical dump**: the store's tables written out as SQL or rows, loadable into any server of that store.
- **Base backup**: a copy of the store's own files at one moment, which the write-ahead log can carry on from.
- **Point-in-time recovery**: restoring a base backup and replaying the log up to a chosen moment.
- **Control plane and data plane**: the control plane creates and changes resources, such as starting a server or editing a DNS record; the data plane handles the traffic those resources serve.
- **Static stability**: a design that keeps working with what is already running when something it depends on is impaired.
- **Failback**: moving the service back to the original site after a failover to another.
- **Data residency**: a rule on where a kind of data may be held, usually a country or a group of countries.

A standby is what waits in a second region. These are the usual kinds:

| Kind | What runs in the second region | What must happen after the loss |
| --- | --- | --- |
| Backup and restore | nothing but stored backups | build the store and the fleet |
| Pilot light | a replica of the data, with the fleet switched off | promote the replica, start the fleet |
| Warm standby | a replica and a small fleet that serves | promote the replica, grow the fleet |
| Active-active | a full fleet serving customers every day | shift the lost region's customers, and whatever its writes need |

A DNS failover moves clients only as resolvers and clients ask again. Services in series multiply their availabilities, so their downtime adds up, and a replica copies a wrong delete as faithfully as a good write. The survivors of a loss take its traffic with cold caches, retries and re-copying on top.

It bites in what a plan does not rehearse. A recovery leans on login, runbooks, pipelines, emergency access and status pages, and each can go down with what it repairs. A drill can be a talk through the plan, a dry run outside production, or a site cut off for real. The code that handles a failure rarely runs: a 2014 study of 198 failures in five systems found that simple testing of error handling could have prevented most of the catastrophic ones.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- weigh an outage, a fifth nine or a second zone against a target, and follow a bad input through redundant copies
- say what a backup gives back after a lost region, a dropped table or a slow fault, what a stalled or silent backup job leaves you with, and what the restore will lean on, and choose the check for bytes a disk returned wrong
- pick a standby, then follow its fleet, its writes and the way back to the first region when a site is lost, and go through what two regions still share
- start a platform from nothing, name every place a region-bound field can be found, and follow a renamed metric into a nightly delete job

###### 1. Two twenty-minute outages under a 99.9 target

A food delivery service has a target of 99.9 percent of requests over 30 days, worked out from its balancer's logs. The team's wiki carries the usual table, which turns 99.9 percent into 43 minutes a month. This month the service went fully down twice, for 20 minutes each: on Monday at 4 a.m., when almost nobody is ordering, and on Thursday at 12:30, the peak of the lunch rush. A slide in the incident review adds them up as 40 of the 43 minutes.

How much of the month's error budget did each outage spend?

<details><summary>A. An equal share each, 20 minutes of the 43</summary>

✗ The table on the team's wiki turns the target into minutes, and the slide adds them up that way. Those minutes match the budget only while traffic stays flat. Monday's 20 minutes met a trickle of orders, Thursday's met the lunch rush, and the balancer's log counts each request that failed.

</details>

<details><summary>B. Far more on Thursday than on Monday</summary>

✓ Right. Thursday's outage spent far more of the budget than Monday's. The slide's 40 of 43 minutes says little about what is left, and Thursday alone can have used up the whole month's budget.

A balancer's logs count requests: the target is the share of requests that succeeded, and the budget is the 0.1 percent that may fail. **A budget counted in requests is spent by the requests an outage meets**, whatever its length in minutes. Take 4 a.m. at a tenth of the day's average and the lunch peak at three times it. Monday then costs what 2 average minutes cost, and Thursday what 60 cost, against a budget worth about 43. The table's minutes are right only for flat traffic, or for a target that counts bad minutes instead, as some contracts do.

Report the budget in failed requests, and plan risky work for the hours when an outage would meet the fewest.

</details>

<details><summary>C. A share on Thursday, and nothing on Monday</summary>

✗ At 4 a.m. hardly anyone is awake to complain. The few orders placed then failed and sit in the balancer's logs, so Monday spent a little of the budget.

</details>

<details><summary>D. Less on Thursday, with the lunch crowd's phones retrying until they got through</summary>

✗ Phones retry, and a customer who got through after the outage still had lunch. A failed attempt stays in the balancer's log whatever a later one does, and every retry inside those 20 minutes is one more failure on Thursday's count.

</details>

---

###### 2. Funding a fifth nine for a phone app

A bank's customers use its app on their phones, on the train and in the shop, over mobile networks. The API behind the app holds 99.99 percent, measured at the API's own edge. A proposal asks for a third region and a year of one team's time to reach 99.999 percent. The head of digital asks what customers will get for it.

What will customers notice once the API reaches 99.999 percent?

<details><summary>A. About as many failed screens as before, on any ordinary day</summary>

✓ Right. A failed screen comes about as often as before, and the year of work reaches customers only as a rare outage that gets shorter.

A call succeeds only if the phone's network and the API both do their part, and **the weaker of the two sets what the customer sees**. If the mobile network fails one call in 100, then of 10,000 calls about 101 fail with the API at 99.99 percent and about 100 fail at 99.999 percent. What the fifth nine does shorten is a full outage: at flat traffic the budget is about 53 minutes of API downtime a year, and it becomes about 5.

Before you fund it, measure failures from the phones as well as at the edge. Retries and screens that work from saved data cut the failures a customer on a train sees.

</details>

<details><summary>B. Far fewer failed screens, all day</summary>

✗ Each nine cuts the API's failures tenfold, and the proposal is pitched on that. If the network fails one call in 100, the API is a small part of what fails: the 10,000 calls lose about 101 today and about 100 afterwards.

</details>

<details><summary>C. Fewer failed screens at the evening peak, with quiet hours unchanged</summary>

✗ Reliability work is often about load, and the evening peak is when an API is most strained. A fifth nine is not tied to any hour, and on a train at any hour the customer's own network is what fails their screens.

</details>

---

###### 3. Adding a second zone to a 99.9 percent service

A ticketing service runs in one zone and has averaged about 99.9 percent. Last year's incident reviews name four outages of about equal length: a bad release, a certificate left to expire, a database failover that hung, and one network loss in the zone. You get approval to run a second copy of the whole service, database included, in a second zone, and the way the team runs it stays as it is. The finance director has read that two independent copies at 99.9 percent make six nines.

How much of last year's downtime would the second zone have prevented?

<details><summary>A. Nearly all of it</summary>

✗ The formula is taught in design courses, and it is right for copies that fail independently. Three of the four causes here reach both zones at once, so the downtime they caused stays.

</details>

<details><summary>B. About half of it, with each outage reaching one zone's customers</summary>

✗ Balancers spread customers across zones, so a failure inside one zone reaches only that zone's share. Only the network loss happened inside a zone. The release, the certificate and the database reach every customer in both, and those stay whole outages.

</details>

<details><summary>C. None of it, with twice the machines to fail</summary>

✗ More machines mean more machine failures, and each added part in a chain lowers the total. These copies stand side by side: traffic leaves a zone whose network is lost, so that one outage goes away.

</details>

<details><summary>D. Only the downtime from the zone's network loss</summary>

✓ Right. The second zone would have prevented about a quarter of last year's downtime, the part from the network loss. The service moves to around 99.925 percent, a long way from six nines.

Six nines comes from **multiplying failure rates, which is right only for copies that fail independently**. A bad release goes out to both zones, a certificate that expires on one hostname expires for both, and a database failover that hangs stops the servers in both. Each of those causes has a failure domain larger than a zone, and this high availability covers only the loss of a zone.

Go through your incident list before you buy redundancy, and put each cause in its failure domain. A release that reaches one zone at a time would shrink the outage from a bad release.

</details>

---

###### 4. A search string that crashes one copy of twelve

A second-hand marketplace runs its search on twelve copies across three zones, behind a balancer that retries a failed request on another copy. A new listing brings a rare sequence of characters that crashes the search parser whenever it appears in a query. A seller checks the listing in the app, gets a blank page and taps search again.

What happens to other customers' searches over the next few minutes?

<details><summary>A. They carry on, with one copy of twelve restarting</summary>

✗ One copy lost in twelve is the case redundancy is drawn for, and a balancer routes around a copy that dies. This copy died from what it read, and the balancer treats that as a failed request, so it hands the same string to the next copy.

</details>

<details><summary>B. They carry on, with the balancer refusing that string after the first crash</summary>

✗ A balancer can notice failing backends, where passive health checks count their failures. They count by backend, never by what a request held, so the balancer marks a copy unhealthy and sends the string on to a healthy one.

</details>

<details><summary>C. They fail more and more often, as copies go down in turn</summary>

✓ Right. One seller tapping search is enough to crash the copies one after another, and other customers' searches fail more and more often.

The twelve copies run one parser, so the string crashes any copy it reaches, and the balancer's retry carries it there. Three zones make no difference, since each holds the same code. Redundancy covers a copy that dies on its own, and **a poison request kills whichever copy reads it**. Say the balancer tries two further copies, a restart takes a minute and the seller taps every 20 seconds: nine copies are down at once, and the 3 left carry all the traffic for as long as the seller keeps tapping.

Stop retrying a request after a copy dies on it, and give each customer a small subset of the copies, known as shuffle sharding, so a poison request reaches only that subset.

</details>

<details><summary>D. They all fail at once, from the first crash</summary>

✗ All twelve copies run the same code, and a shared bug is the textbook way for copies to fail together. Each copy crashes only when the string reaches it, so searches on the other copies go on working for a while.

</details>

---

###### 5. Losing the region while the night's backup is on its way

A clinic's booking system runs in one region and ships a nightly backup to a second.

```
01:00  snapshot of the store taken in region A
01:10  transfer to region B starts
06:30  transfer to region B ends
```

On Wednesday at 05:00, region A is lost for good.

Which moment can the store in region B be restored to?

<details><summary>A. 01:00 on Tuesday</summary>

✓ Right. The store goes back to 01:00 on Tuesday, the last snapshot whose transfer ended. The clinic loses about 28 hours of bookings, more than the day a nightly backup suggests.

After a lost region, **the RPO counts from the newest backup that has reached the other region**, not from the newest one taken. Wednesday's snapshot was in region A and went with it, and the part that reached region B cannot be restored.

State the RPO as the gap between backups plus the transfer. To bring it down to minutes, ship the write-ahead log to region B as it is written.

</details>

<details><summary>B. 06:30 on Tuesday</summary>

✗ The backup report shows 06:30 as the time Tuesday's job finished, and from then on the backup was safe. Its data stopped earlier, at the 01:00 snapshot, so the bookings made between 01:00 and 06:30 are not in it.

</details>

<details><summary>C. 01:00 on Wednesday</summary>

✗ Wednesday's snapshot was taken four hours before the loss, and the backup tool lists it as taken. It was held in region A, and its transfer had not ended, so region B has only part of a file.

</details>

<details><summary>D. No moment at all, with the backups spoiled by the half-sent file</summary>

✗ A half-written file can ruin a good one when it overwrites it. A nightly backup arrives as a new file beside the older ones, so Tuesday's is still whole next to the broken transfer.

</details>

---

###### 6. Shortening a restore that ran for six hours

A marketplace's 2 TB store takes a full backup at 02:00 every night and archives its write-ahead log all day. The team's quarterly drill, run at 03:00, restores in about 50 minutes, and the plan gives an RTO of one hour. Last night the store's disks failed at 23:30, after a busy day of orders, and the same restore took six hours. The team has budget for one change before the next quarter.

Which change shortens the next restore the most?

<details><summary>A. Archiving the log every minute</summary>

✗ Planning documents state RPO and RTO together, and a fresher archive is the first request after a bad restore. That setting decides how little you lose, while the log to replay since 02:00 stays as long as it was, and so does the restore.

</details>

<details><summary>B. Taking a base backup every few hours</summary>

✓ Right. More frequent base backups shorten it the most, since most of the six hours went on replaying the day's log, and they bring the marketplace close to the hour its plan gives.

A restore loads the last base backup and replays every change made since: **the longer since that backup, the longer the restore**. The drill at 03:00 replayed one hour of log, and the failure at 23:30 had 21 and a half. Say replay runs at an even rate: the two restores then split into about 35 minutes of loading and about 15 minutes of replay for each hour of log. With a base backup every 2 hours, the worst restore falls to about 65 minutes.

Base backups cost storage and load on the store, so choose the gap from the RTO. Run the drill shortly before a base backup is due and put that figure in the plan.

</details>

<details><summary>C. Copying the backup over a link twice as fast</summary>

✗ Moving the data is the part of a restore anyone watches, with its progress bar on the screen. Say loading takes about 35 of the drill's 50 minutes: a link twice as fast then saves at most about 17 minutes of six hours, and the replay stays.

</details>

<details><summary>D. Running the drill every month</summary>

✗ What gets drilled gets better, and reviews ask for more drills after a slow recovery. A monthly drill at 03:00 repeats the same 50 minutes, and even one at the end of the day would measure the six hours without taking any of them away.

</details>

---

###### 7. Rolling a nightly dump forward from the archived log

A school's records store runs two jobs. At 01:00 a logical dump writes every table out as SQL, and all day a second job archives the store's physical write-ahead log. At 14:03 a migration drops the wrong table. You load last night's dump onto a new server and set recovery to replay the archived log up to 14:02.

How far does the new server get?

<details><summary>A. To 14:02, with the table back</summary>

✗ Backup plus log is how point-in-time recovery is always told, and this store has both. The log has to continue the very files it was written against, and a loaded dump has other files, so replay has nothing to start from.

</details>

<details><summary>B. To 14:02, after a slow run that turns each change back into SQL</summary>

✗ Change data capture tools read a log and emit rows, which shows a log can be translated. They work alongside the store that wrote the log. Recovery from a physical log has page numbers and no rows to turn them into, and on this new server those pages hold something else.

</details>

<details><summary>C. To 01:00, the moment of the dump</summary>

✓ Right. The new server stops at the dump: the table comes back as it was at 01:00, and the school's 13 hours of changes since then are out of reach through this route.

A physical write-ahead log records changes to pages of the store's files, and loading a dump writes the same rows onto other pages, in another order. An entry that says to change a place in a page has no such place to change, so **replay needs a base backup taken at file level, which the log can carry on from**. The kinds differ here: a store whose log records row changes can replay that log over a dump, from a position the dump writes down.

Keep the dump for moving data, and add a file-level base backup, so the archive has something to roll forward. Then try a restore to a chosen minute.

</details>

<details><summary>D. To 14:02, with the tables scrambled by changes landing on the wrong pages</summary>

✗ A log entry names a page and not a row, and on other files that page holds something else, so a change applied there would wreck it. Recovery checks first: one store stamps its log with the identity of its files and refuses a log from another, so replay stops before a change lands, and the tables stay as the dump wrote them.

</details>

---

###### 8. Tightening permissions on the log archive during a launch

An events platform runs a primary store that takes heavy writes through a product launch and archives each finished segment of its physical write-ahead log to object storage. In the launch's first week someone tightens the permissions on that storage, and every upload starts to fail. The failures go to a log read only by the backup's owner, who is on leave for the rest of the month.

What state is the primary store in by the end of the month?

<details><summary>A. Refusing writes, with its disk full of log</summary>

✓ Right. The store refuses writes, its disk full of log segments it is not allowed to delete, and the platform stops saving events because of a backup job.

While a physical log is archived, **each segment stays on the disk until the archive has taken it**, since a missing segment would stop any restore at that point. Every failed upload leaves one more segment behind. If the launch writes 400 GB of log a day and 1 TB is free, the disk fills in two and a half days. What comes next differs by maker: one store shuts down until space is freed, another stays online for reads and refuses changes.

Alert on the age of the oldest segment waiting to be archived, and send that alert to whoever is on call.

</details>

<details><summary>B. Serving as usual, with a gap in the archive</summary>

✗ Log rotation works that way on a web server: old files go when they age out, and some logical logs expire on a timer too. A store archiving its physical log keeps the archive free of gaps: a segment stays on the disk until it is taken, and the disk fills.

</details>

<details><summary>C. Serving slowly, with each commit waiting on an upload</summary>

✗ Commits wait for their copies under synchronous replication, and an archive holds a copy too. Archiving runs in the background on finished segments, so commits go at full speed through all of this, right up to the moment the disk has no room.

</details>

---

###### 9. Reporting a nightly backup's result to monitoring

A nightly job backs up a shop's store and reports its result to monitoring. Since a change to the store in March, `db_dump` has stopped with an error partway through every run.

```bash
#!/usr/bin/env bash
db_dump --all shop | gzip | storage put "backups/shop-$(date +%F).sql.gz"
report_status "$?"
```

What has monitoring shown for the nights since March?

<details><summary>A. A red run each night, from the error in <code>db_dump</code></summary>

✗ A command that fails makes its job fail, and exit codes exist to say so. `db_dump` exits with an error, but after a pipeline `$?`, the shell's record of the last status, holds only the last command's, and `storage put` finished cleanly.

</details>

<details><summary>B. No report at all, with the script stopped at its first error</summary>

✗ Scripts often open with `set -e`, which stops them when a command fails. It judges a pipeline by the pipeline's own status, here that of `storage put`, so even with it the line after the pipeline runs, and `report_status` sends the upload's success to monitoring.

</details>

<details><summary>C. A red run each night, from <code>storage put</code> refusing a short file</summary>

✗ A storage service checks what it receives, down to checksums and size limits. It has no idea how long a backup should be, and a stream that ended early is a valid upload, so the short file is stored like any other.

</details>

<details><summary>D. A green run each night, with part of the store in the file</summary>

✓ Right. Monitoring has shown green every night, while each file holds only what `db_dump` got through before it stopped, and the shop has made no full backup since March.

By default **a shell pipeline's exit status is the status of its last command**, here `storage put`, which uploaded whatever came down the pipe, and `$?` holds that status. The compressor, `gzip`, closed a valid file over the shortened stream.

Fail the job when any stage fails: the `pipefail` option, in a shell that has it, makes a pipeline report a failure from any of its commands. Compare each file's size and row counts with the night before's, and restore a recent backup on a schedule.

</details>

---

###### 10. Choosing where the next backup copy goes

A tutoring platform keeps its store and its snapshots in one cloud account, and an hourly job copies the snapshots to a region on another continent. Last month a teardown script meant for a staging project was nearly run under the production admin role. The review asks for one change, so that a script like that cannot cost the firm its data.

Which change do you make?

<details><summary>A. Copy the snapshots to a third region as well</summary>

✗ More copies further apart is the rule of thumb for any backup, and a third region survives the loss of two. Every one of them sits in this account, so one run of the teardown reaches the third as well.

</details>

<details><summary>B. Copy them to a separate account that production can write to and not delete from</summary>

✓ Right. A separate account puts a copy out of the production admin role's reach, and the same script, run by mistake, leaves the tutoring platform something to restore.

The role that can delete the store in one region can delete the snapshots on the other continent with the same call, because the admin role's reach is the whole account. Credentials are a failure domain like a zone or a region, and **a copy protects only against a failure it does not share with the original**. These snapshots share the account and its admin role with the store.

Keep the backup account's credentials separate and give them to few people, and turn on write-once retention, which also stops a role that could overwrite a copy. Then try a restore from that account.

</details>

<details><summary>C. Keep the snapshots for a year instead of a month</summary>

✗ Long retention is the cure for a fault found late, and compliance guides ask for it. This fault arrives in one run of a script: a snapshot kept for a year is deleted by the same call as one kept for a month.

</details>

<details><summary>D. Encrypt the snapshots with a key only the backup job holds</summary>

✗ Encryption heads most backup checklists, and it stops anyone reading the contents. Deleting needs no key, so the encrypted snapshots go in the same teardown.

</details>

---

###### 11. Taking payments back to 02:00 so the stores agree

A shop keeps orders in one store and payments in another, each with its own backups, and it charges cards through a payment provider. A storage fault hits both stores at 11:00 on Tuesday. Orders comes back from its nightly backup, taken at 02:00, and payments comes back to 10:55 from its archived log. On the call, someone points out that the stores now disagree, and proposes taking payments back to 02:00 as well, so that they match.

What does that do to the customer who paid by card at 09:00?

<details><summary>A. Their charge is undone, with the purchase gone from every record</summary>

✗ A rollback inside a database undoes a transaction, and after the restore to 02:00 both stores say the purchase never happened. The charge sits at the provider, outside both, and no restore sends the provider anything.

</details>

<details><summary>B. Their money comes back within days, once the provider finds no matching order</summary>

✗ Providers settle and reconcile, and banks reverse charges on request. A provider never sees the shop's orders, so the charge stands until someone asks for a refund, and the shop now holds nothing that says to ask.

</details>

<details><summary>C. Their card stays charged, with no record of it at the shop</summary>

✓ Right. The customer's card stays charged, and the shop now holds neither the order nor the payment, so they have paid for something nobody at the shop can find.

**A restore changes what your stores hold and nothing outside them**, and the charge lives at the provider and the customer's bank. Restored to different moments, the two stores disagree about everything from 02:00 to 10:55, and that disagreement is useful: each payment row with no order marks a customer to put right. Taking payments back makes the stores agree with each other and removes the shop's only record of those charges.

Leave payments at 10:55 and rebuild the missing orders from those rows and the provider's records before the shop reopens. Write down in advance which store leads after a restore.

</details>

<details><summary>D. Their purchase stands, with the order rebuilt from the charge</summary>

✗ Payments reached 10:55, so the 09:00 charge could feed an order rebuilt after the fact, and that is how a reconciliation runs. The proposal takes that row away first, which leaves nothing at the shop to rebuild from.

</details>

---

###### 12. A cleanup job that blanked tax numbers for six weeks

A payroll firm keeps 7 days of write-ahead log, daily backups for 30 days and monthly backups for a year. Today a customer notices an empty tax number. A cleanup job released six weeks ago has been blanking the tax number on a few hundred older customer rows each night.

Which repair do you run?

<details><summary>A. Restore the whole store to the newest backup from before the job</summary>

✗ The backup from before the job holds those older customers' tax numbers, and restoring is what backups are for. Restoring the whole store also takes back six weeks of salaries, addresses and new customers, so payroll would run on old figures.

</details>

<details><summary>B. Restore the whole store to the minute before the job first ran, using the log</summary>

✗ Point-in-time recovery is sold as reaching any moment. It reaches as far back as the log does, which is 7 days here, and the job began six weeks ago. Even a longer log would throw away the same six weeks of good changes.

</details>

<details><summary>C. Restore yesterday's backup to a side server and copy the tax numbers across</summary>

✗ Runbooks start from the latest backup, and a side server is the right place to read it. Yesterday's backup has had six weeks of blanking done to it, so it still holds a number only for rows the job reached last night.

</details>

<details><summary>D. Restore the last monthly backup from before the job to a side server and copy the tax numbers across</summary>

✓ Right. The monthly backup still holds the tax numbers the job blanked, and filling in only those leaves six weeks of salaries, addresses and new customers where they are.

Restoring a whole store takes back everything else as well, so a slow fault needs a repair row by row from a clean copy of the blanked column. Only the monthly tier reaches back past the job: **when retention is shorter than the time it takes to notice a fault, the clean copy has aged out**.

Set your longest retention by how long a quiet fault can go unseen, and count empty tax numbers from outside the application, so the next one surfaces in days.

</details>

---

###### 13. Restoring from the second region with the backup key

Your backup job encrypts each file before it leaves the store, and the files are copied hourly to a second region. `keys.get` is your thin client for the key service, and `storage.read` reads from the region it runs in. The quarterly drill runs this script from an operations server in region A and restores into region B, every time with no error. Region A has now been lost for a day, and the on-call starts the same script from a server in region B.

```python
KEY_REGION = "region-a"

def restore(path, target):
    key = keys.get("backup-key", region=KEY_REGION)
    data = decrypt(storage.read(path), key)
    target.load(data)
```

What does the on-call get from the script?

<details><summary>A. An error at the key lookup, and nothing restored</summary>

✓ Right. The script stops with an error at `keys.get`, nothing reaches the store in region B, and the files stay unreadable for as long as region A is gone.

**A backup is only as available as everything needed to read it**, and here the files left the failure domain of region A while the key stayed in it. Each drill ran from region A with region A up, so the lookup answered and `storage.read` read region A's own files, which left both the key and the copies in region B untested.

Keep the key in a service that also runs in region B, or encrypt the data key with a second key held there. Run the drill from region B with region A cut off.

</details>

<details><summary>B. A restored store, up to the last hourly file</summary>

✗ This script has restored without an error every quarter, and the files are sitting in region B. Every one of those drills asked region A for the key while it was up. Today that call has nowhere to go, so the script stops before it reads a file.

</details>

<details><summary>C. A restored store, with a copy of the key kept among the backup files</summary>

✗ Backup tools often store the data key, itself encrypted, beside the data. This script has no such step: it asks `keys.get` for the key, in region A. A key stored that way would still need a second key to open it.

</details>

<details><summary>D. A store full of rows that are still encrypted</summary>

✗ Encryption at rest is handled below the store, and whoever holds the key reads plain rows. Here the job encrypted each file itself, and `decrypt` needs the key from region A, so no row reaches the store.

</details>

---

###### 14. Picking a standby for a filing service

An accounting firm runs a filing service, and its plan for the loss of a region gives an RPO of one minute and an RTO of eight hours. In the drill, run shortly before the night's backup was due, building the store and the fleet from files held in the second region took about five hours. The budget review asks for the least the firm can keep in that region and still meet the plan.

Which standby do you build?

<details><summary>A. Nightly backups, stored in the second region</summary>

✗ Backups are the cheapest thing to keep in another region, and the drill's five hours sits well inside the RTO. A nightly backup can miss most of a day of filings, against an RPO of one minute.

</details>

<details><summary>B. A replica in the second region, with the fleet switched off</summary>

✗ Replication is the usual answer when someone says one minute, and this pilot light would meet both figures. It keeps a second store running all day to save restore time that an RTO of eight hours does not ask to save.

</details>

<details><summary>C. Nightly backups and the write-ahead log, stored in the second region</summary>

✓ Right. Stored backups and a shipped log meet both figures with no server running in that region until the day it is needed: the firm loses under a minute of filings and is back in about the five hours the drill took.

The two figures are bought apart: **the RPO decides how the data travels and the RTO decides what must already be running**. Shipping the write-ahead log as it is written loses only what has not yet left, and storing that log needs no running store. A five-hour restore inside an eight-hour RTO means nothing has to be running.

Alert when the newest log in the second region is more than a minute old. Time the restore again as the store grows, and pay for a replica when it nears eight hours.

</details>

<details><summary>D. A replica in the second region, with a small fleet running</summary>

✗ A warm standby would be back in minutes, and it is the kind most diagrams show for losing a region. The replica and the small fleet run and bill all day, for speed this plan does not ask for.

</details>

---

###### 15. Growing the standby fleet to full size

An online pharmacy runs a warm standby of 20 servers in a second region, with a scaler that can take it to production's size. Both regions are built from the same templates, and every release goes to both. Last spring the team got its server limit raised for a drill at 150 servers, production's size then, and the standby grew to that. Production has since doubled to 300, with its limit raised on request along the way. Tonight production's region is lost, and the scaler starts.

Where does the standby fleet end up?

<details><summary>A. At 300 servers, with those over the limit billed at a higher rate</summary>

✗ Metered services usually charge more for going over. A server limit is a permission: a launch over it is refused, and the fleet stays where it stopped.

</details>

<details><summary>B. At 300 servers, in about the time the drill took</summary>

✗ The templates match, every release goes to both regions and the last drill went well. The limit is no part of a template, and that drill went only to 150, half of tonight's load.

</details>

<details><summary>C. At 300 servers, after a much slower climb</summary>

✗ Large scaling steps are slow, and 300 servers take more launches than 150. Speed is not what stops this fleet: a refused launch stays refused, however long the scaler goes on asking.

</details>

<details><summary>D. Short of 300 servers, at the limit set for that region</summary>

✓ Right. The fleet stops short of 300 servers at that region's limit. Part of the pharmacy's traffic gets errors for as long as the limit stands.

**Clouds set server limits for each account in each region**, and the limits sit outside your templates. The raises on the way to 300 were granted in production's region, and the standby's limit stayed where the drill left it.

Raise limits in the standby region whenever you raise them in production, and grow the standby to today's full size in a drill. A limit does not reserve servers, and a region can run out of capacity, so for a fleet you cannot lose, reserve that too.

</details>

---

###### 16. Counting on the scaler when a zone is lost

A video service runs 300 servers across three zones, exactly what its evening peak needs. If a zone is lost, the scaler is to replace its 100 servers in the two zones left. In each drill an engineer terminates one zone's servers by hand, and the replacements are serving in six minutes. Tonight, at the peak, a power failure takes out one zone.

How long can the two zones left run short of servers?

<details><summary>A. Until the lost zone returns, with the scaler holding to its three zones</summary>

✗ Scalers keep zones balanced, and this one has run on three for as long as anyone remembers. A scaler places replacements in the zones it can reach once launches work, so the wait is tied to those launches and not to when the third zone comes back.

</details>

<details><summary>B. Far longer than six minutes, with launches slow or refused</summary>

✓ Right. The shortage can last far longer than six minutes, since the 200 servers left must carry a peak sized for 300 until launches succeed, and viewers get errors or stalled streams for as long as they do not.

Starting a server goes through **the control plane, which a lost zone can impair**. It has to find a host, prepare storage and network and issue credentials. That work is more complex than serving, and one maker designs it for lower availability than its data plane. Other customers in that zone ask for replacements at the same moment, and the zones left can run out of free capacity. The drill terminated one customer's servers while all of that was healthy, so its six minutes measure only the good case.

Build for static stability: run enough in each zone that any two carry the peak, here 150 a zone and 450 in all, so a lost zone needs no launch at all. Keep the scaler for the recovery afterwards.

</details>

<details><summary>C. About six minutes, as in every drill</summary>

✗ The drills gave six minutes each time, and the scaler has never been late. Those drills ran with launches healthy and free capacity in the region. A real zone loss promises neither, so six minutes is what the best case took.

</details>

<details><summary>D. About six minutes, plus the time the health checks take to notice</summary>

✗ Detection is the step drills tend to leave out, and it adds a little. The larger unknown comes after it: six more minutes holds only if every launch goes through, and in a real zone loss they can be slow or refused.

</details>

---

###### 17. Serving from two regions for six months

A shop has served from two regions for six months, each taking half the customers and reading from its own replica. A login writes a session to the store, whose one primary sits in region A. The slide for the board says active-active, and the team has stopped rehearsing failover, a promotion run by hand. This afternoon region A is lost.

What do customers routed to region B get over the next few minutes?

<details><summary>A. The shop as usual, with region A's half moving over</summary>

✗ Region B has carried half the customers for six months. Its reads were local all along, and every order still went to region A, so it has nowhere to send them today.

</details>

<details><summary>B. Pages and orders, with the orders held in region B for later</summary>

✗ Some designs queue a write locally and send it later. This shop has no such queue: each write goes straight to the primary, the connection fails, and the customer's order fails with it.

</details>

<details><summary>C. Pages that load, with logins and orders failing</summary>

✓ Right. The shop can be browsed and nothing can be bought: customers in region B keep loading pages from their own replica, and logins and orders fail.

**Serving from two regions says where reads land, not where writes land**, and here every write goes to the primary in region A. With it gone, region B can take no write until its replica is promoted, which is the failover the team stopped rehearsing. Stores that do take writes in both regions pay in another way: writes sent to the nearest region have to be reconciled, and writes assigned to a home region by key still need a failover for that key.

Keep rehearsing the promotion, and test it by cutting region A off.

</details>

<details><summary>D. An error on every page, with no replica serving without its primary</summary>

✗ A replica follows its primary and takes every change from it. It goes on serving what it holds, growing older by the minute, so pages load from region B's replica.

</details>

---

###### 18. Moving back to the first region the next morning

A ticket seller failed over to its second region last night when the first lost power, with about two seconds of changes not yet shipped. Nine hours and many orders later, the first region is back, and its store starts up with all its disks intact. The runbook's failback page says to repeat the failover steps in the other direction.

What has to happen to the first region's store before it can take writes again?

<details><summary>A. It gives up its last two seconds, or is rebuilt, and then follows the second region</summary>

✓ Right. The first region's store has to give up its last two seconds, or be rebuilt, and then follow the second region like any replica. Failback is a job of its own, and the runbook's one line does not cover it.

The two stores forked at the failover: the first holds two seconds the second never saw, and the second holds nine hours of orders. Since **a store can replay a log only from a point it shares with the log's writer**, the first store is rewound to the fork, or rebuilt from the second when no rewind is possible. It then catches up, and you can switch back at a quiet hour with nothing lost, since a planned switch waits for the copy.

Write the failback page on its own: find the fork, save the two seconds for someone to review, time the rebuild, and pick the hour.

</details>

<details><summary>B. Nothing, with the second region's changes following once it is primary</summary>

✗ The first region was the primary before the night began, its disks are intact, and the runbook treats the failover as a detour. The nine hours of orders are only in the second region, so taking writes now leaves two stores that each hold sales the other lacks.

</details>

<details><summary>C. It replays the nine hours from the second region on top of what it holds</summary>

✗ A replica that was away catches up from the log, and this store was away for nine hours. It holds two seconds of changes the log never mentions, so the log's next entry does not fit what is on its disk. The fork has to go before any replay.

</details>

<details><summary>D. The two stores merge row by row, newest timestamp first</summary>

✗ Multi-leader stores reconcile copies with last write wins. These two share a log in one direction and have no merge, and a merge by timestamp would keep or drop a paid order by how two clocks happened to read.

</details>

---

###### 19. Restarting an index service after four years of rolling restarts

A storage platform runs its index service on 30 nodes, and the service tracks where every object lives. A starting node fetches its signing key from the secret store, and the secret store keeps its encrypted records only as objects on the platform and reads them there on each request. For four years the index service has only ever restarted one node at a time, each back within two minutes. This morning a mistyped maintenance command removes more nodes than the service can run without, and it stops answering. The command is undone, and all 30 machines boot.

What does the index service do once the machines are back?

<details><summary>A. A slow climb, one node after another, with answers from the first node on</summary>

✗ A cluster usually comes back gradually, each node adding capacity, the way it does during a rolling restart. The first node here never finishes: its key lookup needs the secret store, and the secret store's records can be found only through the index.

</details>

<details><summary>B. Hours of checks, with all the nodes answering together at the end</summary>

✗ Long full starts are real: a 2017 report from one object store's operator says the checks on metadata took longer than expected. This service never gets as far as those checks. Waiting cannot help, because each part is waiting on another, and someone has to break the loop first.

</details>

<details><summary>C. It answers within about two minutes, the time one node takes</summary>

✗ Two minutes is what every restart on record took, and the nodes start in parallel. Each of those restarts had 29 other nodes answering, so the key lookup worked. Today no node is answering, and the lookup cannot finish.

</details>

<details><summary>D. No node comes up, with each one waiting on a key it cannot fetch</summary>

✓ Right. No node comes up: each waits for its signing key, the secret store waits to read its records, and those records can be found only through the index. The platform stays down with every machine running.

The loop has been there for years: a rolling start finds other nodes answering and every lookup works, while **a start from nothing runs a path no rolling restart ever ran**.

Break the loop, for example with a sealed copy of the key on each node's own disk, and bring a copy of the whole stack up from nothing on a schedule.

</details>

---

###### 20. Pinning European candidates' rows to a European region

A recruiting platform must keep European candidates' personal data in Europe, so a routing layer pins their rows to a store in a European region, and that store's backups stay there too. App servers run in Europe and the US, each close to its users. Central logging and the error tracker run in the US. An auditor asks where a candidate's email address can be found.

Which places can hold a candidate's email address?

<details><summary>A. The European store</summary>

✗ The routing layer puts every row in one region, and rows are where an audit starts. The store's backups hold the same rows, and logs and error reports hold what requests carried, so one store is too short a list.

</details>

<details><summary>B. The European store, its backups, and the US log and error stores</summary>

✓ Right. Expect it in the European store, its backups, and the US log and error stores, and tell the auditor so: a candidate's email can reach the US with any logged request or failed call.

Pinning rows fixes where one table's rows live. Personal data also travels in requests, and whatever the app logs or the tracker captures lands where those run: a failed sign-up's body, a URL with an email in it, a variable in a stack trace. **Data residency has to follow every copy**, including search indexes, caches and analytics exports as well as logs.

Follow one request from end to end and write down each place it leaves a trace. Then run logging and error tracking in each region, or strip personal fields before anything leaves.

</details>

<details><summary>C. The European store and its backups</summary>

✗ Backups are the one copy everyone remembers, and here they stay in Europe. Logs and error reports are copies too, and they run in the US, so the list still leaves out places the email can be.

</details>

<details><summary>D. The European store and its backups, with the US servers only passing the address through</summary>

✗ App servers are often built to keep no state. A server that keeps no state still logs and reports its failures to central systems in the US, so an address can stay there after the request has gone.

</details>

---

###### 21. Deleting idle servers by their request counts

A video platform runs 2,000 long-lived servers behind its balancers, and each one takes requests all day. A nightly job deletes idle servers, and in eight months it has removed only machines somebody forgot. `metrics.query` is your thin client for the metrics store, which holds each server's counts over time: it returns a list of hourly counts and fails with an error of its own if the store answers with an error. Today a new version of the metrics agent, the program on each server that reports those counts, reaches the whole fleet, and from now on it reports `http_requests` as `http_requests_total`. The team moves its dashboards and alerts to the new name the same day, and nobody opens the job.

```python
for server in cloud.list_servers():
    counts = metrics.query("http_requests", server=server.id, days=30)
    if sum(counts) == 0 and server.age_days > 30:
        cloud.delete_server(server.id)
```

What does the job do to the fleet from here on?

<details><summary>A. Tonight's run deletes every server</summary>

✗ From today nothing writes `http_requests`, and that is the name the job asks for tonight. The store keeps what was already written under it, so tonight's window holds 30 days of counts up to the rollout, and every total is far above zero.

</details>

<details><summary>B. A few servers go each night, the quietest first</summary>

✗ A window sliding over old counts shrinks every total a little each night, and a quiet server has the least to lose. A total reaches zero only when its last count leaves the window, and all 2,000 servers wrote their last one today, so the quiet ones and the busy ones reach zero in the same run.

</details>

<details><summary>C. A run about a month from now deletes every server</summary>

✓ Right. For about a month nothing looks different, and then one night's run deletes all 2,000 servers, each of them still taking requests.

A metrics store files each count under the name it was written with, so the rename starts a new series and leaves the old one where it is. Every night `http_requests` still has counts in the 30-day window, fewer each time, until about a month after the rollout the last of them leaves. **A window with no counts in it is an ordinary question to a metrics store**: it answers with success and an empty list, the client has no error to report, and `sum` of an empty list is 0. The job then reads no data as no traffic, and with no limit on how many servers one run may delete, it goes through the whole list.

Have the job first query a server you know is busy and stop if it reads zero, and treat an empty answer as a fault, never as zero. Cap what one run may delete at a small share of the fleet, and delete in two steps, stopping a server and removing it days later, so a wrong run can be undone.

</details>

<details><summary>D. A run about a month from now stops with an error, and every server stays</summary>

✗ Asking for something that is not there fails in plenty of APIs, with a 404 for a missing record, and this client fails on any error the store returns. The store has nothing to object to: it already holds the name, the window is a valid one, and a result with no counts in it comes back as a success, the way a `SELECT` that matches no rows does. No error reaches the loop, and the job goes on to the delete.

</details>

---

###### 22. Finding a balance one bit off in all three copies

A water utility keeps its billing data on one primary and two replicas, in rows held in pages on disk. Replication runs over TCP, with its checksum on every packet. A customer rings about a balance of 510.18, and last month's statement says 182.50, with nothing paid or charged since. Stored in cents, the two figures differ by one bit. The primary and both replicas hold 510.18, and so do the last four nightly backups. The only write to that row since the statement came from the reminder job, which reads a row, sets its reminder date and writes the whole row back. The maker's test on the primary's disk then finds it returning wrong bytes from a few sectors, with no read error.

Which check would have reported the damage?

<details><summary>A. TLS on every connection between the copies</summary>

✗ TCP's own check is a 16-bit sum, where two flips can cancel out, and a 2000 study of Internet traffic estimated that it misses an error in roughly 1 in 200 million packets. TLS adds a cryptographic check on top. That check starts when the primary hands over its bytes, and the job's write sets out already saying 510.18, so TLS delivers the wrong balance to both replicas intact.

</details>

<details><summary>B. A checksum on each log record</summary>

✗ Stores checksum their log records and verify them on replicas, and the log is how this damage reached the replicas. The primary computes that checksum over the job's write, which already says 510.18, so each replica finds the record sound and applies it.

</details>

<details><summary>C. A monthly test restore of the newest backup</summary>

✗ A restore is the one test that proves a backup can be used. It shows the backup holds what the primary held: the restored store starts cleanly and serves 510.18, and the drill passes.

</details>

<details><summary>D. A checksum on each data page</summary>

✓ Right. A checksum on each data page would have reported the damage when the primary read that page for the reminder job, before the job wrote anything, and both replicas would still have held 182.50 to repair the customer's row from.

The store computes that checksum when it writes a page and verifies it each time it reads the page back. The disk sits inside that span, and **a check covers only the bytes between where it is computed and where it is verified**. TLS, the log checksum and a restore all start after the wrong value is in memory. Once the job wrote the row back, 510.18 was an ordinary change, and replication and backups copied it like any other. In a 2008 study of storage systems with 1.53 million disks, block checksums caught more than 400,000 mismatches over 41 months.

Turn page checksums on where a store leaves them off, and alert on the failures they report. A bit flipped by the primary's memory or CPU before the page is written gets a valid checksum, so for figures that must be right, have the application store its own checksum beside the value and check it on read.

</details>

---

###### 23. Running two regions with no shared database

A travel app serves Europe from region A and Asia from region B. Each region has its own fleet and its own store, and the two share no database. The sign-in service was built first, in region A, runs only there, and both regions use it. A customer's app holds a token that lasts an hour, signed so that each server can check it with a public key the server keeps, and the app renews it in the background once ten minutes are left. In each quarterly drill region A is cut off for ten minutes, and region B's servers go on answering with no rise in errors. This morning region A is lost for the day.

What do customers in Asia get over the next hour?

<details><summary>A. The app as usual, with only new sign-ins refused</summary>

✗ A signed token is checked with no call out, and every drill has shown region B's servers holding steady. That lasts as long as the token does. Each app's renewal has to reach the sign-in service in region A, and every token left unrenewed runs out within the hour.

</details>

<details><summary>B. Customers signed out a few at a time, starting about ten minutes in</summary>

✓ Right. For about ten minutes signed-in customers in Asia notice nothing. Then they start dropping out a few at a time, and within the hour all of them have been sent to a sign-in service that is gone, while region B's fleet and store stay healthy.

While a signed token lasts it needs no call: each server checks it with the issuer's public key. Renewing it is a call, and that call goes to region A. Tokens renew with ten minutes left, so when region A goes each has between ten and sixty minutes to run, and the ten-minute drill ended before the first one ran out. The two stores are separate and the regions still share a failure domain, because **whatever both regions need to keep serving can stop them together**. A secret store, a feature-flag service or a provider's control plane that runs in one place can do the same.

List what a request, a token renewal and a server start each call, and where each one runs. Issue tokens from both regions, and run a drill that outlasts a token.

</details>

<details><summary>C. Nothing for most of the hour, then every customer signed out together</summary>

✗ Each token lasts an hour, and that hour is the cover a plan counts on if sign-in goes. It started at each token's last renewal, which differs from app to app, so the oldest tokens have ten minutes left and the rest end one after another.

</details>

<details><summary>D. Tokens accepted past their hour, with nobody signed out until sign-in returns</summary>

✗ Some systems do fail open when a dependency goes down, and none of this is the customer's fault. The expiry check compares the time in the token with the server's clock, and whether sign-in is up plays no part in it, so a token past its hour is refused.

</details>

---

## Further reading

- [Simple Testing Can Prevent Most Critical Failures](https://www.usenix.org/conference/osdi14/technical-sessions/presentation/yuan), 2014: 198 failures in five distributed systems, and how most of the catastrophic ones came down to error handling that simple tests would have caught.
- [When the CRC and TCP Checksum Disagree](https://dl.acm.org/doi/10.1145/347059.347561), 2000: traces of Internet packets, and an estimate of how many damaged packets the TCP checksum lets through.
- [An Analysis of Data Corruption in the Storage Stack](https://www.usenix.org/conference/fast-08/analysis-data-corruption-storage-stack), 2008: more than 400,000 checksum mismatches across 1.53 million disks over 41 months, and how they cluster.
- [Metastable Failures in the Wild](https://www.usenix.org/conference/osdi22/presentation/huang-lexiang), 2022: 22 failures from 11 organisations that outlasted their triggers, and what kept them going.
- [Availability in Globally Distributed Storage Systems](https://www.usenix.org/conference/osdi10/availability-globally-distributed-storage-systems), 2010: a year of availability data from one large storage system, with models of how placement and replication choices change it.

Prev: [Load testing](../load-testing/README.md) · Next: [System design](../system-design/README.md)
