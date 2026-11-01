# Containers and orchestration

Prev: [Service-to-service security](../service-to-service-security/README.md) · Next: [Releasing changes](../releasing-changes/README.md) · Reviewed October 2026

A container is a process that runs on its machine's kernel with its own view of processes, network and files, under limits on what it may use. An orchestrator runs many of them across many machines: you declare what should run and how many, and it keeps making the real state match, starting replacements rather than repairing what broke. Because the kernel is shared, a kernel flaw can cross from one container to the others, and a container handed a host directory or the runtime's control socket can reach the host.

- **Image**: the read-only files a container starts from, kept in a registry.
- **Tag**: a name for an image in a registry, such as `orders:1.4`. A digest is a hash of the image's content.
- **Node**: one machine in the cluster, with an agent that starts and stops containers on it.
- **Copy**: one running instance of a service. It can hold a helper container that runs first, or a sidecar beside the application.
- **Control plane**: the orchestrator's API, the store that holds its state, the scheduler that places copies and the controllers that act on differences.
- **Health check**: a small request the orchestrator sends a copy on a schedule, with a timeout and a failure count; on this page it is called a probe.
- **Readiness probe**: asks whether a copy should get traffic. Traffic reaches copies through proxies and balancers that follow the orchestrator's list of ready copies, and a copy that fails comes off the list and keeps running.
- **Liveness probe**: asks whether a copy should be restarted. It checks only what a restart could fix, since a copy that fails it is killed and started again.
- **Startup probe**: holds the other two back while a copy starts.
- **Stop signal**: what the orchestrator sends a container to ask it to finish, usually `SIGTERM` on Linux.
- **Grace period**: the time between the stop signal and the forced kill, often 30 seconds.
- **Drain**: emptying a node of its copies for repair or an upgrade. A balancer's draining is another thing, on [Load balancers and proxies](../load-balancers-and-proxies/README.md).
- **Limit**: the most memory or CPU the kernel lets a copy's processes use. A copy with no limit can take from its neighbours.
- **Reservation**: what a copy asks to have set aside on its node, often called a resource request. The scheduler places a copy on a node whose reservations leave room and does not look at real use, so a cluster that looks idle can leave a copy waiting. A placement rule can be required or preferred, and priority lets a waiting copy push out a lower one.
- **Scaler**: the part that sets how many copies run from a signal, usually mean CPU measured against the reservation, between a floor and a ceiling. Scale-down waits a few minutes so that the count does not flap.

A service runs until stopped, and its copies are interchangeable. The copies of a stateful service keep a name and storage across a replacement. A batch job runs to its end and may start again after a failure, so its work has to tolerate that. Name lookups, the network layer and the traffic proxy serve every copy, so a fault there can touch everything at once.

Rollouts and old and new versions side by side are on [Releasing changes](../releasing-changes/README.md), and a restarted copy that takes its share while cold is on [Load balancers and proxies](../load-balancers-and-proxies/README.md).

Trouble starts where the orchestrator's picture of a copy and what the process is doing part ways: during a start, under load, near a limit and when a node falls silent.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- say what is left after a copy restarts, the control plane stops answering, a node falls silent or comes back, or the image registry goes down, and which build each copy runs
- set probes, a stop handler and a start script so that a slow start, a busy evening and a rollout do not cost you requests
- work out what a memory or CPU limit does to the process under it, and where copies stand when a node fills up or is drained
- plan copy counts for a slow partner, a sale at a known hour and a database with a connection limit

###### 1. Editing a config file inside three running copies

A shipping API runs three copies, and at two in the morning all of them reject labels from a new carrier. The on-call engineer opens a shell inside each copy, adds the carrier to a file under `/etc/shipping`, and labels flow again. No volume, meaning storage from outside the container, is mounted on that folder. Nobody gets round to changing the image that week. On Friday one copy's process runs out of memory, and the orchestrator restarts it on the same node.

What do the three copies hold in that file once the restart is done?

<details><summary>A. All three with the new carrier</summary>

✗ The file sits on the node's disk, the node never went away, and restarting a process on a long-lived server leaves its disk as it was. An orchestrator's restart starts a new container from the image, and the old container's edit went with it, so that copy rejects the new carrier again.

</details>

<details><summary>B. None, with the orchestrator restoring all three to the image</summary>

✗ An orchestrator keeps the real state matching what you declared, and all three files now differ from the image. Its loop compares what it was told about, such as how many copies run, and it never reads a file inside a container, so the two copies that did not restart keep the edit.

</details>

<details><summary>C. Two with the new carrier, one without</summary>

✓ Right. The restarted copy holds the file as the image built it and the other two keep the new carrier, which puts about a third of that carrier's labels back to failing on Friday.

**A container's own files last as long as the container**, and an orchestrator's restart starts a new container from the image, even on the same node. What an engineer with a shell wrote inside the old one is gone, and only a volume outlives it. The orchestrator holds no record of the edit, so the fleet now runs two different files.

Put the carrier in the image or in a config store and roll it out, and make that the last step of the incident.

</details>

---

###### 2. Losing the control plane for forty minutes

A food delivery firm runs its order service and forty others on one orchestrator at its default settings, with six copies of the order service. Callers reach it through a proxy on each node that hands each new connection to a copy on its list. At lunchtime the store that holds the orchestrator's state loses its majority, and for forty minutes the API answers nothing. In the on-call channel someone points out that no request passes through the API, so customers will not notice. Ten minutes in, a node holding two of the six copies loses power.

What do customers placing orders get for the rest of the outage?

<details><summary>A. Failures on the share of orders sent to the two dead copies</summary>

✓ Right. About one order in three fails for the rest of the outage, those the proxies hand to the two dead copies.

The API is off the request path, and **a running copy and the proxy in front of it need nothing from the control plane until something changes**. Noticing a dead node, taking its copies off the lists and starting replacements are control plane work, so the proxies keep handing connections to the last list they got, four live copies and two dead.

Give the proxies in front of critical services a health check of their own, so a dead copy drops out with no help from the control plane. Alarm on the API like any other outage, and keep the store's majority across zones.

</details>

<details><summary>B. Orders going through as usual, on the four survivors</summary>

✗ The API is on no request's path, and the proxies sit on the nodes beside the copies. They learn of a dead copy from the control plane, which is not answering, so they keep two dead addresses on their lists and the orders sent there fail.

</details>

<details><summary>C. Orders going through as usual, on six copies, with node agents starting two more</summary>

✗ Each node's agent restarts a crashed copy on its own, and it can do so with the API down. The dead node's agent went with its node, and deciding that two copies belong elsewhere is control plane work, so four copies run while orders still go to six addresses.

</details>

<details><summary>D. Failures on every order, with each node's agent stopping its copies</summary>

✗ A database member cut off from its majority stops serving, and these agents have lost the API they report to. By default an agent leaves its running copies alone when the API goes quiet, so the four live copies keep taking orders.

</details>

---

###### 3. Running one copy of a scheduler so it cannot run twice

The job scheduler of a loyalty points service must not fire twice, so you run it as an ordinary service at the orchestrator's default settings, with the copy count set to `1`. One night a switch fault cuts its node off from the control plane, while the node stays up and still reaches the points database. Some minutes later the orchestrator's timeout for a silent node runs out.

What is running once that timeout has passed?

<details><summary>A. One, the replacement, with the old copy stopped when its node lost contact</summary>

✗ Lease designs stop work once they lose touch with their manager, which is how a count of one stays one there. An orchestrator's agent does not by default: some offer it as an option you turn on, so the old copy runs on and its replacement starts beside it.

</details>

<details><summary>B. Two copies, the old one and its replacement</summary>

✓ Right. Two copies run and jobs can fire twice: the old one on the cut-off node, still reaching the points database, and its replacement on another node.

**The control plane cannot tell a dead node from one it cannot reach**, and for an ordinary service it assumes the first and starts a replacement once a timeout passes. Its order to stop the old copy cannot reach that node, and by default the node's agent does not stop work for losing touch. A copy count says how many the orchestrator wants, and nothing in it stops a second copy from starting. Some orchestrators treat a stateful copy differently and hold its replacement until the old one is proven gone.

Where a job must not run twice, hold a lease and use a fencing token, as on [Distributed locks and IDs](../distributed-locks-and-ids/README.md), or make the job safe to run twice.

</details>

<details><summary>C. One, the old copy, with no replacement until its node answers</summary>

✗ The count says one, and some orchestrators hold a stateful copy back in this way. An ordinary service gets no such care: once the timeout has passed the control plane starts a replacement, with the old copy still running where it cannot see it.

</details>

---

###### 4. Publishing every merge under one tag

The build at a ticketing site publishes each merge to the registry as `tickets:stable`, over the last one. Rollouts are done by hand, and the last was on Monday, when all eight copies started on that tag. The orchestrator's record of the service holds the name as written, and its pull policy is `if-not-present`, which pulls an image only when the node lacks it. By Thursday three more merges have gone through the build. At that day's lunchtime peak the scaler adds two copies, on a node that has never run this service.

Which build do the two new copies start on?

<details><summary>A. Monday's build, the one the other eight run</summary>

✗ The last rollout was on Monday, and in this process a rollout is the one thing that changes what runs. The new copies went through no rollout: they ask the registry for `tickets:stable`, which by now means Thursday's build.

</details>

<details><summary>B. Thursday's build, with the other eight restarted to match</summary>

✗ Two new copies now differ from eight old ones, and an orchestrator exists to remove differences. What you declared is the name `tickets:stable`, all ten run under that name, and the orchestrator does not compare builds, so the eight keep Monday's.

</details>

<details><summary>C. Neither, with the two refused for a changed image</summary>

✗ Images carry checksums, and registries do check them. The check compares a download with the digest the registry gives for that name today, and the two agree, so both new copies start, on Thursday's build.

</details>

<details><summary>D. Thursday's build, three merges past the other eight</summary>

✓ Right. The two new copies start on Thursday's build while the other eight stay on Monday's, and some customers now reach code nobody rolled out.

A tag can move in the registry, and **a node that pulls by tag gets whatever the tag means at that moment**. With `if-not-present` a node pulls only what it lacks, so the eight older copies keep what their nodes fetched on Monday and the new node fetches Thursday's build. Some orchestrators fix a tag to its digest when a service is created, and then every copy stays on one build until an explicit update.

Deploy by digest, or give each build a tag nobody reuses, so a replacement starts exactly what the others run.

</details>

---

###### 5. Bringing an empty node back at noon

A booking API runs nine copies on three nodes in one zone, three on each. A kernel patch takes `node-c` down for the morning, and its three copies are replaced on `node-a` and `node-b`. At noon `node-c` comes back healthy and empty. No deploy is planned this week, and traffic is ordinary.

Where do the nine copies run that evening?

<details><summary>A. All nine on <code>node-a</code> and <code>node-b</code>, with <code>node-c</code> empty</summary>

✓ Right. All nine copies stay on `node-a` and `node-b`, where they went that morning, and `node-c` sits empty. The booking API carries on as it did, with less room if either busy node fails.

**The scheduler places a copy only when that copy has no node**, and a healthy running copy gives it nothing to do. Nine copies were declared and nine run, so no controller has a reason to act. An orchestrator that rebalances on its own evens out zones, and these nodes share one.

If the spread matters, restart the copies one at a time once the node is back, or run a rebalancing job that does this for you, and expect each moved copy to stop and start.

</details>

<details><summary>B. Three on each node again, moved back within minutes</summary>

✗ Putting things back is the orchestrator's job, and three on each is how things stood before the patch. What was declared is nine copies, and spread is no part of that unless a rule says so, so with nine running nothing moves.

</details>

<details><summary>C. Twelve, with the old three restarted on <code>node-c</code></summary>

✗ A rebooted server starts its services again, and these three were on `node-c` this morning. Replacements took their place while it was down, so the node comes back with nothing assigned, and the count stays at nine.

</details>

<details><summary>D. Spread by load, with the busiest copies moved to <code>node-c</code></summary>

✗ A balancer watches load and steers requests, and a scheduler is the part that decides where copies go. The scheduler acts when a copy needs a node and then forgets it, so load on `node-a` never moves a copy.

</details>

---

###### 6. Losing a node during an image registry outage

A news site pulls its images from one private registry, and one morning that registry goes down for two hours. No rollout is planned, so the team shrugs: the article service's eight copies are all up and serving. Half an hour in, a node holding two of those copies fails, and the only node with room for their replacements joined the cluster at dawn and has never run this service.

What happens to the two replacements?

<details><summary>A. They start from an image the orchestrator keeps</summary>

✗ The orchestrator's record names the image of every service, and it can start a copy anywhere. That record holds a name and nothing else. The bytes live in the registry and on the nodes that pulled them, the node that joined at dawn holds none, and both replacements wait.

</details>

<details><summary>B. They wait on the new node while the registry is down</summary>

✓ Right. The two replacements wait on the new node, unable to pull their image, and the article service runs a quarter short on its other six for as long as the registry stays down.

Each node keeps its own store of images and **pulls what it lacks when it has to start a copy**. A running copy reads from its node's disk and never asks the registry, which is why the first half hour passed quietly. The replacement lands on a node with no copy of the image, so it retries with a growing delay and starts once the registry is back.

Treat the registry as a run-time dependency of every node: keep a mirror in the cluster, and alert on failed pulls.

</details>

<details><summary>C. They start with the image taken from another node</summary>

✗ Other nodes hold these exact bytes, and some clusters add a tool that shares images between nodes. Without one, a node gets an image from the registry its name points to, and with the registry down both replacements sit waiting.

</details>

---

###### 7. Adding a copy with no probe set

A currency rates API loads its tables for about 40 seconds at start, and only then opens its port. Nobody has set a probe of any kind on the service, and this orchestrator adds none of its own. At a busy hour you add a fourth copy by hand, and two seconds later the orchestrator's status for it reads `running`. The orchestrator's proxy hands each new connection to one of the copies on its list, and does not retry.

What do callers see during those 40 seconds?

<details><summary>A. Normal replies, with the new copy held off the list until its port opens</summary>

✗ An orchestrator starts and watches this process, and many balancers check a port by default. This one asks only what a probe tells it to ask, and with no probe the new copy joins the list when its process starts, port closed and all.

</details>

<details><summary>B. Refused connections on the share sent to the new copy</summary>

✓ Right. One connection in four is refused for most of the 40 seconds, because the new copy is on the list with its port closed, and callers of the rates API see those refusals as errors.

`running` says a process exists. **With no readiness probe the orchestrator counts a copy as ready as soon as its process starts**, and the proxy's list takes it at once. A proxy that hands over connections and copies bytes does not know what the application has finished loading.

Give the service a readiness probe that passes only once the tables are loaded. The copy then joins the list at about 40 seconds, and the other three carry the traffic until it does.

</details>

<details><summary>C. Slow replies from the share that waits at the new copy's port</summary>

✗ A busy server keeps new connections in a backlog until it gets to them, so a slow reply is what one that is behind gives you. A backlog belongs to an open port. This one stays closed until the tables are in, so the connection is refused at once.

</details>

<details><summary>D. Normal replies, with the new copy restarting over and over, never reached</summary>

✗ A crash loop is what a failed start usually shows on these platforms. A restart needs a process that exits or a failed liveness probe, and this process is alive and loading with no probe set, so it reaches the list with its port still closed.

</details>

---

###### 8. Rolling out a fix after the search index has grown

A product search service loads its index into memory at start and answers its health path only afterwards. When the liveness probe below was set, loading took 20 seconds. The index has grown over the year, and loading now takes about 90. This morning you roll out a one-line fix.

```text
liveness probe
  path:              /health
  initial delay:     30 s
  period:            10 s
  failure threshold: 3
```

What do the new copies do after the rollout starts?

<details><summary>A. They load for 90 seconds and then serve</summary>

✗ The probe was set to watch a running service, and the index loaded fine when it was small. The probe knows nothing of the load: it starts at 30 seconds, fails at 30, 40 and 50, and the copy is killed partway through its load.

</details>

<details><summary>B. They are killed once and stay down, waiting for someone</summary>

✗ A failed step in a pipeline usually stops and waits for you, and a killed copy shows up as failed. A liveness kill ends in a restart: the orchestrator starts the copy again under the same probe, so the load begins from nothing and the kill comes again at about 50 seconds.

</details>

<details><summary>C. They are killed once, and the rollout goes back to the old version</summary>

✗ Pipelines advertise automatic rollback, and a copy that dies on start is the case it is built for. Undoing a rollout is a step something has to take, a pipeline or a setting turned on. Short of that, the new copies go on restarting.

</details>

<details><summary>D. They restart again and again, never finishing the load</summary>

✓ Right. Each new copy is killed at about 50 seconds, partway through a 90-second load, and starts loading again from nothing, so the one-line fix never reaches customers.

**A liveness probe with no startup probe counts from the container's start, whatever the application is doing**: it checks at 30, 40 and 50 seconds, and three failures bring the restart. It was set when the load fit inside that window, and the index grew past it with no change to the probe.

Add a startup probe with room for the largest index you expect, since the liveness probe does not run until the startup probe first passes and a later hang still meets the quick check. A longer initial delay also gets this rollout through, and next year's index outgrows it. Alert on restarts during a rollout.

</details>

---

###### 9. Keeping a one-second liveness probe through an on-sale

A resale ticket API runs eight copies, each with a fixed pool of 50 workers. Its liveness probe calls the route below every 10 seconds, allows 1 second for a reply and restarts a copy after three failures in a row. At a big on-sale, requests on every copy wait about 5 seconds for a free worker, and each one served still succeeds.

```python
@app.route("/healthz")
def healthz():
    return "ok"
```

What does the orchestrator do to the copies during the on-sale?

<details><summary>A. It restarts copies that were still serving</summary>

✓ Right. From about 30 seconds into the on-sale, the orchestrator restarts copies that were serving every request they took. Each restart throws away what that copy held and pushes its share onto copies already full, so buyers lose requests that would have succeeded.

The route does no work, but it runs on a worker like any other, and **a probe served by the request pool waits in the same line as the requests**: about 5 seconds here, against the 1 it allows. A busy copy then fails exactly as a dead one does, and a failed liveness probe can only restart it.

Answer liveness on a thread or port of its own, and give it a generous timeout and failure count. Saying a copy is too busy is a job for a readiness probe or for load shedding, and [Rate limiting and load shedding](../rate-limiting-and-load-shedding/README.md) shows what those cost.

</details>

<details><summary>B. It leaves all eight running</summary>

✗ The route returns `ok` with no work, and a copy serving each request it gets is alive by any measure. Its reply still needs a free worker, about 5 seconds away against the probe's 1, and three such failures get the copy restarted.

</details>

<details><summary>C. It restarts each copy once, and the fresh copies hold</summary>

✗ A restart empties a copy's line, and restarting a stuck process is the first cure anyone reaches for. The line comes from the on-sale's traffic, which is still arriving: a fresh copy fills again within seconds, and its probe fails as the last one did.

</details>

<details><summary>D. It restarts the copy with the longest queue, and the rest hold steady</summary>

✗ On a quiet system a failure stays with the worst machine, and restarting it takes load off the rest. All eight copies wait about 5 seconds for a worker, so all eight fail the probe, and the load from each restart lengthens the wait on the others.

</details>

---

###### 10. Closing the listener when the stop signal arrives

An image resizing API sits behind several proxies, with retries off, that follow the orchestrator's list of ready copies. To shut down cleanly, you wrote the handler below, and it passed in staging, where one test client sends a request every few seconds. At peak traffic, today's rollout shows a moment of refused connections from each copy right after its stop signal, though every running request finishes.

```python
def on_stop_signal():
    server.stop_accepting()
    server.wait_for_active_requests()
    sys.exit(0)
```

Which change stops the refused connections?

<details><summary>A. Raise the grace period to 120 seconds</summary>

✗ A longer grace period gives running requests more time, and lost requests during a shutdown are what it is raised for. It only sets when the kill comes. The port still closes on the handler's first line, while proxies are sending, and those connections are still refused.

</details>

<details><summary>B. Have the readiness probe start failing as the handler runs</summary>

✗ A failing readiness probe takes a copy off the list, and getting off the list is what this copy needs. That update is already on its way from the moment the copy begins to stop, and a probe does not hurry the proxies that apply it, so the port still closes before they catch up.

</details>

<details><summary>C. Keep accepting for a few seconds after the signal, then close</summary>

✓ Right. With the port open for a few seconds after the signal, the proxies have caught up by the time it closes, and callers of the resizing API see no refusals during the rollout.

**The signal and the update to the list go out without waiting for each other**, and a proxy keeps sending to a stopping copy until it applies that update. This handler stops accepting while some proxies still send. The one test client in staging rarely fell inside that gap, and at peak traffic some request always does. Some orchestrators that drive a balancer directly wait for its drain before the signal, and one can hold the signal for a set delay after taking the copy out. With proxies following a list, as here, nothing waits by default.

Set the wait from how long your proxies take to catch up, and check that the wait and your longest request both fit in the grace period.

</details>

<details><summary>D. Exit at once on the signal, with no wait for running requests</summary>

✗ A quick exit shortens the time a stopping copy spends in the way, and it makes a rollout faster. The port closes as early as before, so the refusals stay, and now the running requests are cut off as well.

</details>

---

###### 11. Adding a start script in front of the server

A wallet API has a handler for the stop signal, and sending that signal to the server by hand on your laptop finishes its requests and exits in 2 seconds. To load a secret before start, you add the script below, and it becomes the image's command. The orchestrator's grace period is 30 seconds, and a rollout begins.

```sh
#!/bin/sh
export DB_PASSWORD="$(cat /run/secrets/db_password)"
/app/server --port 8080
```

What happens to a copy when the rollout stops it?

<details><summary>A. The server finishes its requests and exits in about 2 seconds</summary>

✗ The handler passed its test on your laptop, where the signal went straight to the server. Here it goes to the first process, the shell, and the server is only the shell's child, so it hears nothing and its handler never runs.

</details>

<details><summary>B. The script dies at once, and the server is killed with requests running</summary>

✗ In a terminal a shell dies on this signal by default, and when a container's first process ends, the kernel kills the rest. As a first process the shell plays by another rule: a signal it has no handler for is dropped, so it lives on, and the server with it.

</details>

<details><summary>C. The script exits, and the server carries on as an orphan until the kill</summary>

✗ Outside a container a child outlives its parent and runs on. In a container the kernel kills every process once the first one ends, so no orphan could run on, and this shell does not exit at all, since the signal never reached it.

</details>

<details><summary>D. The server runs all 30 seconds, then is killed, its handler unrun</summary>

✓ Right. The server never hears the signal: it runs all 30 seconds of the grace period and is then killed with its handler unrun. Each copy takes 30 seconds to stop, and the wallet API cuts off whatever it was doing.

The orchestrator signals the container's first process, here the shell. **The kernel drops a stop signal sent to a container's first process unless that process has set a handler for it**, and this script has set none. When the grace period ends, the kill reaches every process in the container, and nothing can catch that.

End the script with `exec /app/server --port 8080`, so the server replaces the shell and becomes the first process. A small init process in the image that passes signals on does the same job.

</details>

---

###### 12. Catching a memory error in an export handler

A reporting service runs with a 2 GiB memory limit on Linux nodes that have no swap, and each copy is one process serving many customers at once. To be safe, you turned memory errors into a 413, and a test that caps the process with `ulimit -v` gets that 413 every time. Today a customer asks for an export that needs about 3 GiB.

```python
def handle_export(request):
    try:
        rows = db.fetch_all(request.query)
        return render_csv(rows)
    except MemoryError:
        log.error("export too large", request.id)
        return Response(status=413)
```

What does the customer get?

<details><summary>A. A 413 from the handler, and one line in the log</summary>

✗ Your test returned the 413, and catching the exception is what the language offers. That test capped address space, where the allocation itself fails. Under this limit allocation succeeds, and the process dies when it writes to the memory it was given, with no exception for the handler to catch.

</details>

<details><summary>B. A broken connection, with the copy's other requests lost too</summary>

✓ Right. The customer gets a broken connection, or an error from whatever stands in front of the copy, and no 413 is ever written. The kernel kills the process on its way past 2 GiB, every other customer's request on that copy dies with it, and the orchestrator restarts the copy.

**Memory limits are enforced on the memory a process has used**, not on what it asks for. Asking for memory succeeds, and the kill comes when the process uses memory that would take the copy past the limit and nothing can be reclaimed, so no call returns an error and `except MemoryError` has nothing to catch. `ulimit -v` works differently: it caps address space, so the allocation itself fails and the test gets its 413.

Stream the export row by row, or check its size before you fetch it, and test under a real memory limit.

</details>

<details><summary>C. The export, paid for with free memory on the node</summary>

✗ A node usually has far more free memory than one copy's limit. The limit holds however empty the node is: at 2 GiB the kernel kills the process.

</details>

<details><summary>D. A partial CSV, cut where the limit was reached</summary>

✗ A disk quota stops a file growing and leaves what was already written. The handler returns the CSV only once `render_csv` has finished, and the kill comes before that, so not one byte of CSV is sent.

</details>

---

###### 13. Sizing a worker pool from the CPU count

A PDF rendering API starts its workers with the code below, and each worker loads at start and then holds about 150 MB of its own. Each copy runs with limits of 2 CPUs and 4 GiB, and the team's load test on a small cluster went well. Today is the first deploy to production, whose nodes have 64 cores.

```python
import os

WORKERS = os.cpu_count() * 2 + 1
start_workers(WORKERS)
```

What does the first deploy do?

<details><summary>A. Workers sized to the 64-core node, and a memory kill during start</summary>

✓ Right. The code starts 129 workers, sized to the node's 64 cores, which need about 19 GiB against a limit of 4, so the kernel starts killing processes during start and the first production deploy never settles enough to serve.

A CPU limit is a quota on running time, and **a process under a quota still sees every processor on the node**, so `os.cpu_count()` returns 64 here. The load test passed because a small cluster's nodes give a smaller count.

Set the worker count from an explicit setting, kept beside the limits in the service's config, or work it out from the copy's CPU quota at start.

</details>

<details><summary>B. Workers sized to the 2-CPU limit, and a steady service</summary>

✗ A container gets its own processes, network and files, each a private view of the machine. Its CPU limit is a quota on running time, and `os.cpu_count()` reads the machine: 64 here, so 129 workers where the limit would pay for 5.

</details>

<details><summary>C. Workers sized to the 64-core node, idle ones costing next to nothing</summary>

✗ Idle processes use no CPU, and workers started from one process share memory at first. Each of these holds about 150 MB once loaded, busy or not, so 129 of them need about 19 GiB against a limit of 4.

</details>

<details><summary>D. Far more workers than the limit pays for, crawling under the CPU quota</summary>

✗ Too many workers under a CPU quota is the cost you hear about, and 129 workers on 2 CPUs would crawl. Memory stops them first: the kernel kills at 4 GiB, before any of them has a request to crawl through.

</details>

---

###### 14. Bursting on sixteen threads under a one-core limit

A search suggestions service runs on Linux nodes with 16 cores that sit idle between bursts, and each copy has a limit of 1 CPU. Its runtime keeps 16 threads. The CPU graph, averaged per minute, sits at 30 percent of the limit all day, and most requests finish in about 8 ms, nearly all of it CPU work. Then a small burst of requests wakes all 16 threads at once.

What happens to the copy during that burst?

<details><summary>A. The threads run on idle cores, with the limit settled over the minute</summary>

✗ The node's 16 cores sit idle, and the limit shows as a line on a graph averaged by the minute. The kernel does not settle it by the minute: the quota is spent and refilled every 100 ms, so the threads stop after about 6 ms with those cores free.

</details>

<details><summary>B. The threads share one core's worth, each running slowly and steadily</summary>

✗ Limits get described as speed, and one core shared by 16 threads comes to a sixteenth each. The threads run at full speed until the quota is gone, and then all of them stop until the next refill.

</details>

<details><summary>C. The threads run flat out, then all stop beside idle cores</summary>

✓ Right. All 16 threads run flat out for about 6 ms and spend the copy's whole quota, then all of them wait about 94 ms beside cores doing nothing. Suggestions that usually take 8 ms take around a tenth of a second.

**A CPU limit is a quota of running time per short period**, a tenth of a second by default: 1 CPU means 100 ms of CPU time in each 100 ms, however many threads spend it. Sixteen threads spend it in a sixteenth of the period, and once it is gone every thread in the copy waits for the next period, idle cores or not. The per-minute graph averages over hundreds of periods, most of them quiet, and shows 30 percent.

Bring the runtime's thread count down to about what the limit pays for, or raise the limit, and alert on the throttled time the kernel reports for the copy.

</details>

<details><summary>D. The orchestrator kills the copy for going over its limit</summary>

✗ A memory limit kills, and a CPU limit has the same name. Going over a CPU quota costs running time only: the kernel holds the threads back and the process lives on.

</details>

---

###### 15. Staying under a memory limit on a packed node

A node with 16 GiB runs a dozen copies whose memory limits add up to 30 GiB. A reporting copy on it has a limit of 4 GiB and has held steady at 3 for days. At night a cache rebuild in another copy, still inside its own limit, takes the node's total past what it holds.

Which copies can the node kill?

<details><summary>A. Only the rebuild copy, which pushed the node over</summary>

✗ The rebuild took the node past what it holds, so it takes the blame, and a copy that passes its own limit is the one killed. This rebuild is inside its limit, and the choice goes by what each copy uses, so the reporting copy's 3 GiB can count against it while the rebuild lives on.

</details>

<details><summary>B. None, with the rebuild's next allocation refused</summary>

✗ On a laptop, a program that asks for more than the machine can give may get a failed allocation. On a node with default settings the rebuild's allocation succeeds, the shortage appears when the memory comes into use, and then something gets killed.

</details>

<details><summary>C. Any copy there, the reporting copy included</summary>

✓ Right. Any copy on the node can be killed, the reporting copy included, at 3 GiB and a full GiB under its limit. The night's report can die because another service rebuilt its cache.

The limits here add up to 14 GiB more than the node holds, which some orchestrators allow, and **a limit caps one copy and sets nothing aside for it**. When the node itself runs short, its agent or the kernel has to take memory from somewhere, and who goes first differs: one orchestrator's agent starts with copies using more than they reserved, and the kernel's killer picks by how much a process uses, adjusted by a score the orchestrator can set. Neither asks who is inside a limit or who caused the shortage.

Set a reservation close to real use, since placement then leaves the copy that room, and keep the sum of limits near what the node holds on nodes running work you cannot lose.

</details>

<details><summary>D. None, with the reporting copy moved to a roomier node while it runs</summary>

✗ Virtual machine platforms move running guests off a full host, and the orchestrator is what decides placement. A copy cannot move while it runs: it can only be stopped and started elsewhere, so if the reporting copy is chosen, its running work is lost.

</details>

---

###### 16. Upgrading two nodes that hold all three copies

An insurance quotes API runs three copies on a twelve-node cluster, and its rollout setting allows one copy down at a time. Placement has put two of them on `node-d` and one on `node-e`. Nobody has set a drain limit for this service, and this orchestrator applies none by default. At 03:00 a node upgrade drains `node-d` and `node-e` together.

What does the API have serving while those two nodes drain?

<details><summary>A. Two copies at all times, the rollout setting holding the pace</summary>

✗ One copy down at a time is the only pace written anywhere on this service. It governs rollouts, and a drain does not read it, so all three copies on those two nodes are asked to stop together.

</details>

<details><summary>B. Three copies at all times, with replacements ready before any old copy stops</summary>

✗ A rollout with surge starts a new copy before it stops an old one, and a node upgrade is one more way of replacing copies. A drain stops first: a replacement is created once the old copy has begun to stop, so there is a gap before it is ready.

</details>

<details><summary>C. Three copies, with the upgrade skipping nodes that hold a service's last copies</summary>

✗ The orchestrator knows where all three copies are, and these are the nodes that hold them. With no limit set, a drain does not count what is left of a service: it asks every copy on the node to go.

</details>

<details><summary>D. No ready copy, until the replacements are up</summary>

✓ Right. All three copies are asked to stop together and come off the list, and callers of the quotes API get errors until replacements on other nodes pass their probes.

A rollout's settings pace a rollout, and **a drain asks every copy on a node to leave**. A replacement starts as each one begins to stop, so nothing waits for a new copy to be ready. Some orchestrators pace drains by default.

Set a drain limit on any service that cannot go empty, such as "keep at least two running", and the drain then stops copies a few at a time and waits for the rest. Leave the limit room to let one copy go, since one that lets none leave makes the drain wait for good, and add a spread rule so that copies do not share a maintenance batch.

</details>

---

###### 17. Scaling on CPU while a partner slows down

A hotel search service runs twelve copies under a scaler with a CPU target of 60 percent, a floor of 4 and a ceiling of 40. Its CPU goes to parsing and ranking partner replies, and each request holds one of a copy's 100 threads while it waits on the partner's API. Mid-morning the partner slows from 200 ms a call to 5 seconds, and traffic stays as it was.

What does the scaler do over the next ten minutes?

<details><summary>A. It adds copies up to its ceiling of 40</summary>

✗ A slow service and an overloaded one show the same long waits, and overload is what scaling answers. This scaler reads CPU, and a thread waiting on a partner uses none, so the signal falls below 60 and no copies are added.

</details>

<details><summary>B. It holds at twelve, with traffic unchanged</summary>

✗ Traffic is as it was, and scaling is usually explained through traffic. The scaler reads CPU, which drops as requests finish more slowly, so the count does not hold at twelve.

</details>

<details><summary>C. It adds a few as threads pile up, then holds</summary>

✗ Threads pile up, and 100 of them in use is a process at its full size. A blocked thread sleeps until its reply comes, so the pile adds no CPU, and the first readings after the slowdown are already lower.

</details>

<details><summary>D. It removes copies, heading for its floor of 4</summary>

✓ Right. The scaler removes copies, heading for its floor of 4, while hotel searches pile up, and each removal leaves fewer threads to work through them.

Parsing and ranking replies take the CPU here, and **a thread blocked on a call uses no CPU at all**. At 5 seconds a call, 100 threads finish 20 requests a second on each copy, and the parsing falls with them. Say a copy reserves 1 CPU, a request takes 4 ms of it and each copy gets 150 a second: CPU drops from 60 percent to about 8. A scaler on mean CPU asks for the count that would bring it back to 60, which is below the floor, and each removal cuts what the service can finish.

Scale on what fills first, here threads in use or requests waiting, and set a floor that carries normal traffic through a slow partner. A timeout on the partner's call frees threads sooner than any number of copies.

</details>

---

###### 18. Getting ready for a ten o'clock on-sale

A concert site puts tickets on sale at 10:00 on Friday. Past on-sales went from a quiet 200 requests a second to 6,000 inside 30 seconds, and were over by 10:20. The service runs four copies under a scaler with a CPU target of 60 percent and a ceiling of 100, enough for that traffic once all are running, and a new copy takes minutes to start. You have one change to make before Friday.

Which change keeps the 10:00 buyers served?

<details><summary>A. Raise the scaler's ceiling to 500</summary>

✗ The ceiling is the setting with a number on it, and hitting one is how scaling often fails. This one is high enough already: in the first minutes the copies asked for are still starting, so at 10:00 buyers meet the same four copies at any ceiling.

</details>

<details><summary>B. Raise the scaler's floor to the on-sale's size at 09:45</summary>

✓ Right. With the floor raised at 09:45, the copies are up when the doors open, so the 10:00 buyers land on a fleet already sized for 6,000 requests a second, and the scaler stays on for whatever the estimate missed.

**A scaler acts only after the load has arrived**, and no setting of it shortens a copy's start. Say a new copy takes three minutes, since it may need a new node and then has an image to pull, and say a copy takes 100 requests a second flat out: over the first three minutes the four copies serve about 7 percent of what arrives.

Schedule a higher floor on the scaler for any hour you can name, and keep it past 10:20, since a count set by hand is undone at the next reading. For traffic you cannot name in advance, keep spare copies or shed load while new ones start.

</details>

<details><summary>C. Lower the CPU target to 30 percent</summary>

✗ A lower target keeps more spare room, which is the usual advice for bursty traffic. At 30 percent the quiet fleet grows to 7 copies, which take 700 of the 6,000 requests a second if a copy takes 100, so buyers still get about 12 percent served in the first three minutes.

</details>

<details><summary>D. Have the scaler read CPU every 5 seconds</summary>

✗ The delay in a control loop often lies in how seldom it reads its signal, and 5 seconds cuts that wait to almost nothing. The reading was never the slow part here: if a copy takes three minutes to start, the buyers wait those minutes whichever reading asked for it.

</details>

---

###### 19. Raising the copy ceiling for a sale

The order service of an online pharmacy keeps a pool of 20 connections in each copy, and its readiness probe answers without touching the database. The database accepts 500 connections. At the usual ten copies it shows 200 open, most of them idle, so for a sale you raise the scaler's ceiling from 10 to 40. On the morning of the sale, the scaler takes the service all the way to 40.

What do customers get once all 40 copies are up?

<details><summary>A. Errors on orders that land on copies with no connections</summary>

✓ Right. The first 25 copies take all 500 connections and the last 15 are refused and hold none, so about three orders in eight fail, those sent to the last 15 copies, while all 40 copies stay listed as ready.

Forty copies want 800 connections against a limit of 500, since **each copy opens its own pool and nothing shares out the total**. The limit counts open connections, idle or not. The readiness probe never touches the database, so the starved copies stay on the list.

Check pool size times ceiling against the limit before you raise a ceiling, or put a connection pooler between the copies and the database, so many client connections share fewer real ones.

</details>

<details><summary>B. Normal orders, idle connections not counting toward the 500</summary>

✗ Those 200 mostly sit idle, and an idle connection runs no query. It still holds a slot, since the database counts it toward the 500, so the first 25 copies fill the limit and the rest are refused.

</details>

<details><summary>C. Slower orders, with the late copies waiting in line for free connections</summary>

✗ Pools queue callers when they are busy, and a caller waiting for a connection is a familiar sight. That line exists inside one copy's pool. At the database a connection past the limit is refused with an error, so the late copies have nothing to wait in.

</details>

<details><summary>D. Errors on every order, with the database falling over at its limit</summary>

✗ When connections are refused, the application's error says it cannot reach the database. The limit is there to protect it: connections past 500 are refused while the 500 go on working, so orders on the first 25 copies succeed.

</details>

---

## Further reading

- [DADI](https://www.usenix.org/conference/atc20/presentation/li-huiba), 2020: why downloading and unpacking an image slows a start, and what fetching its blocks on demand changes.
- [Omega](https://doi.org/10.1145/2465351.2465386), 2013: why large clusters outgrow a single scheduler, and what running several over shared state changes.

Prev: [Service-to-service security](../service-to-service-security/README.md) · Next: [Releasing changes](../releasing-changes/README.md)
