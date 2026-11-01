# WebSockets and real-time delivery

Prev: [Event-driven architecture](../event-driven-architecture/README.md) · Next: [Microservices](../microservices/README.md) · Reviewed October 2026

Real-time delivery gets a server's news to a client soon after it happens, over a connection the client holds open or keeps reopening. Once those connections live for hours on many machines, what each one costs, where it sits and what happens when it breaks decide how the service behaves.

- **Handshake**: the HTTP request that asks a server to upgrade the connection to a WebSocket. Afterwards the connection carries frames of text or bytes, and you decide what goes inside them.
- **Ping and pong**: control frames of the protocol. A ping asks the other end for a pong.
- **Heartbeat**: regular traffic that shows a connection is alive. The ping interval says how often it goes, and the pong timeout how long to wait for the answer. It has to come more often than the shortest idle timeout on the path, the time after which a proxy closes a quiet connection.
- **Socket server**: a server whose job is to hold clients' open connections and pass messages to and from them.
- **Send queue**: what a socket server has accepted for one connection and has not yet written out. Libraries differ, and on this page a queue has no limit and `send` returns at once, unless a question says otherwise.
- **Presence**: who is online, worked out from live connections. One person can hold several, a tab or a device each.
- **Sticky session**: a balancer sending one client back to the server it chose for that client the first time.
- **Room**: a named group of connections that get the same messages.
- **Fan-out**: one message sent on to many receivers.
- **Pub/sub backplane**: a publish and subscribe channel that carries a message from the socket server that got it to the others.
- **Sequence number**: a rising number on each message, which lets a receiver see that one is missing.
- **Replay log**: messages kept for a while with rising ids, so a client that reconnects can ask for what came after the last id it saw. Libraries differ, and on this page a read returns the entries the log still holds above that id. A buffer kept per session grows with the number of people away at once.
- **Draining**: a server taking no new connections while the ones it holds keep running.
- **Reconnect storm**: many clients connecting at once after a server or a network fails. Each should wait a random, growing time first.

Four transports carry the news:

| Way | What stays open | Who can send |
| --- | --- | --- |
| Polling | nothing: the client asks every few seconds | the client asks, and the server answers |
| Long polling | one request, answered when news comes or a timer ends; events between two requests wait only if the server keeps them | the same |
| Server-sent events | one HTTP response that does not end; on a drop the browser reconnects and sends the id of the last event it got, and the server decides what to keep for replay | the server, text only |
| WebSocket | one upgraded connection | both ends, text or bytes |

Proxies and networks on the path treat these transports differently. A proxy with response buffering holds a stream until its buffer fills, so buffering is turned off for streams. After the upgrade a proxy sees one long request, so per-request rate limits, rules and firewalls see the handshake and none of the messages, and a cap on messages per user lives in the socket server. A health check on another path does not test the sockets. Some networks block the upgrade, and libraries then fall back to long polling. Where a server keeps the session in memory, all the requests of that session have to reach that server.

A large message on a socket holds up whatever is behind it. A closing handshake can carry a code, 1001 for going away.

Code in a browser page cannot add its own headers to the handshake, so a token goes in the address, where access logs keep it, or better in the first message. Joining a room needs its own permission check.

The trouble comes when many connections change at once, and when a quiet connection is taken to mean more than it does.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- choose how a page gets its news, and say which number sizes the fleet behind it
- work out what a quiet connection means when a phone goes underground or a server stops, and what a send or a green dot tells you then
- follow a message through several socket servers and a backplane, and bring a client back with what it missed
- keep a room and a fleet working through a phone in a lift, a rollout, a crash and an account that should have lost its access

###### 1. Moving a parcel tracker from polling to a socket

A courier firm has an app that polls every 30 seconds for the status of a parcel, and at the evening peak two million phones have it open. Capacity is planned in requests a second. You move the app to one WebSocket per phone, over which the server pushes each status change, and a parcel's status changes a few times a day.

Which number now decides how many socket servers you need?

<details><summary>A. The status changes pushed each second</summary>

✗ Each push is a piece of work, and work per second is what the old plan counted. If a parcel changes four times a day, the pushes come to about 93 a second, which no server notices, while two million connections sit in memory.

</details>

<details><summary>B. The phones connected at once</summary>

✓ Right. The fleet is sized by how many phones hold a connection at the peak, though almost none of them has anything to say at a given moment, so a plan in requests a second comes out too low while the machines fill up.

**A held connection costs the server memory and a file handle whether or not anything moves on it**: socket buffers, TLS state and your own record of who it is. Say each one takes 50 KB. Two million phones then take 100 GB before the first push, which is four servers at 32 GB each. Polling sent about 67,000 requests a second to do this job, so the CPU has far less to do and memory has become the limit.

Plan in connections held, and measure the memory per connection on your own server with TLS on.

</details>

<details><summary>C. The bytes the sockets carry each second</summary>

✗ Traffic is a line on every cloud bill, and a busy network service is often planned from it. A status change is one small message a few times a day, so the sockets carry next to nothing, and a server runs out of memory for held connections long before its network is busy.

</details>

<details><summary>D. None of them, with a quiet socket costing the server nothing</summary>

✗ An event loop, the code that handles each socket as its data arrives, spends no CPU on a socket with nothing to read, and that is the reason to build a socket server on one. Each connection still keeps its buffers and state in memory, so two million of them fill machine after machine whether or not they speak.

</details>

---

###### 2. Asking for real-time notices on a transit site

A city's transit site is static pages behind a CDN, a shared cache with many edge locations, in front of one small origin server. A new service notice goes up a few times a week, and at the morning peak the home page is open on about 300,000 screens, from phones to the boards in station halls that stay open all day. The product owner asks for real-time notices: a new one should be on every open screen within a minute.

How should the page get its notice?

<details><summary>A. A WebSocket from each open page to new socket servers</summary>

✗ The product owner said real-time, and a WebSocket is built to carry exactly that. Here it means a fleet you do not have, holding 300,000 connections through the morning to carry a message a few times a week, when a delay of a minute is allowed.

</details>

<details><summary>B. A stream of server-sent events from each open page</summary>

✗ The feed runs one way and the notices are short text, which is what server-sent events carry over plain HTTP. A response that does not end cannot be stored and handed to the next screen, so the CDN has nothing to answer from, and 300,000 streams end on one small origin.

</details>

<details><summary>C. A GET on page load, cached for a minute</summary>

✗ One read is all a phone needs when someone opens the page and leaves, and the CDN makes that read cheap. A page left open never asks again, so the boards in the station halls go on showing the notice they loaded that morning.

</details>

<details><summary>D. A poll every 30 seconds, cached for 30 seconds</summary>

✓ Right. A new notice is on every screen within a minute, and the small origin hardly sees the crowd, since each open screen asks for one small URL every 30 seconds and the CDN answers.

**A shared cache answers repeat requests for one URL while its copy is fresh**, so polling costs the origin one request per edge location for each TTL, the time a copy counts as fresh, however many screens ask. With 50 edge locations, say, the 300,000 screens send the CDN 10,000 requests a second, and at most 1.7 a second reach the origin. The longest wait is a TTL plus a poll, 60 seconds, which is the limit you were given.

Set the TTL on the notice's URL and poll for that URL alone. Move to a held connection when the delay allowed falls below what a TTL can give, or when the news comes too often for a poll to carry.

</details>

---

###### 3. Watching a dispatch map when its socket server loses power

A dispatch office follows its couriers on a map page fed by a WebSocket. The socket server sends a ping frame every 30 seconds, and the browser answers each with a pong. The map draws every position it receives, and the page shows a red banner when its connection closes. At 14:00 the machine holding this page's connection loses power.

What do the dispatchers see on that page for the next minute?

<details><summary>A. A red banner at once</summary>

✗ Stopping the server in a test brings the banner up straight away. A server that shuts down sends a close frame first. This machine lost power with its close frame unsent, so the browser has nothing to report to the page.

</details>

<details><summary>B. A red banner within about 30 seconds</summary>

✗ The protocol's own text calls a ping a way to check that the other end is responsive, and this server sends one every 30 seconds. The check belongs to whoever sends the ping. The browser answers the pings it gets and does not count the ones that stop coming.

</details>

<details><summary>C. The map as it stood at 14:00, with no banner</summary>

✓ Right. The dispatchers go on watching couriers who appear to have stopped, on a map that stays as it stood at 14:00, with no banner and no error.

**A machine that loses power sends nothing, so no packet tells the browser that the connection has ended**, and the page hears only what the browser tells it. The pings do not help. The browser answers them by itself and gives the page's code no way to send one or see one, so that code cannot notice that they have stopped.

Send a heartbeat of your own as an ordinary message. With one every 10 seconds and two missed counted as a dead connection, the banner is up within 30 seconds, and the page then opens a new socket.

</details>

<details><summary>D. A brief flicker, then live positions through another server</summary>

✗ The browser's client for server-sent events reconnects by itself, and so do the libraries people wrap around sockets. The browser's WebSocket has no reconnect of its own, and it would need news of the failure before it could start one, so this page sits on the one socket it opened.

</details>

---

###### 4. Stalling one socket server for twenty seconds

A betting app holds 400,000 phones on eight socket servers. During a match, one server hits a GC pause, the runtime stopping the program to clean up memory, for 20 seconds. Its process then carries on from where it was, with its memory and all its connections intact. A phone with no connection opens a new one within seconds. The phones' heartbeat settings:

```text
ping_interval  5s
pong_timeout   10s
```

Where are that server's phones a minute later?

<details><summary>A. On other servers, nearly all of them, over new connections</summary>

✓ Right. Nearly all of the server's 50,000 phones have cut healthy connections and reached the other seven over new ones, though the server came back with everything intact, so a 20-second pause has become a reconnect storm in the middle of a match.

**A phone's timer measures silence and knows nothing of the cause**, so a paused server looks the same to it as a dead one. Every phone sends a ping in the pause's first 5 seconds, so every phone is past its timeout between second 10 and second 15.

Set the timeout above the longest pause you measure and have each phone wait a random time before it reconnects. A longer timeout also means a real failure takes longer to notice.

</details>

<details><summary>B. On that server, over the same connections, with scores 20 seconds late</summary>

✗ TCP holds a connection through a pause, the bytes arrive late, and this server did come back whole. The phones do not stay to find that out: a `pong_timeout` of 10 seconds runs out well inside the pause, and a phone that has given up on a connection opens another.

</details>

<details><summary>C. On that server, apart from the few whose ping fell due in the pause</summary>

✗ Twenty seconds is a short time, and at any moment only some phones are in the middle of a ping. With a ping every 5 seconds, though, all of them send one in the pause's first 5 seconds, and all of those answers are overdue by second 15.

</details>

<details><summary>D. On other servers, with the balancer having moved their connections</summary>

✗ A failed health check is what takes a server out of a balancer's list, and a 20-second pause can fail one. Coming off the list stops new connections from arriving. A balancer cannot hand an open socket to another server, which knows nothing of that connection, so any phone that moves has reconnected by itself.

</details>

---

###### 5. Marking a push as delivered once send returns

A delivery app sends couriers their new jobs over a WebSocket. `conn` is the socket server's connection object and `db` your thin client for the jobs table. On each new connection, the server pushes the courier's jobs that are still marked `new`. A courier drives into an underground car park, where the phone loses its signal, and seconds later a job comes in. Twenty minutes later the phone is back above ground, on a new connection.

```python
def push_job(conn, job):
    conn.send(json.dumps(job))
    db.update("jobs", job.id, status="delivered")
```

What does the courier get when the phone comes back on the new connection?

<details><summary>A. The job, with its row still marked new</summary>

✗ A send on a closed socket fails with an error in some libraries, and that error is how such code learns a client has gone. As far as the server knows this socket is still open, since nothing came from the car park, so `send` returned normally and the next line ran.

</details>

<details><summary>B. No job, with its row marked delivered</summary>

✓ Right. The courier gets no job and the row says `delivered`, so a job nobody has seen sits marked as done and the new connection finds nothing `new` to push.

`send` hands the bytes to a buffer on this machine and returns. A phone that has lost its signal sends no close, so the server still holds an open connection, the buffer takes the data, and TCP goes on resending it with no answer, out of your code's sight. The protocol has no delivery receipt, so **the only proof that a phone holds a message is a message back from the phone**.

Have the phone acknowledge each job by id, mark the row when the acknowledgment arrives, and push everything unacknowledged on a new connection. A job can then arrive twice, when it was the acknowledgment that got lost, so the phone drops an id it already holds.

</details>

<details><summary>C. The job twice, once over each connection</summary>

✗ A sender that retries over a bad link can deliver a message twice. That needs a second send, and the row's `delivered` rules that out: the new connection pushes only what is still marked `new`, and the old connection's bytes do not reach this phone.

</details>

<details><summary>D. The job, from the old connection's buffer</summary>

✗ TCP keeps what it has buffered and retries, which is the reliable part of a reliable transport. It retries on that connection alone, and the phone's new connection is another one, so what was buffered for the old one goes when the old one is dropped.

</details>

---

###### 6. Marking a person offline when their socket closes

A team chat shows a green dot beside everyone online, kept in a shared store that every socket server writes through these handlers. A server closes a connection whose pong is 60 seconds overdue. At 17:59 one server's machine loses power. A colleague connected through it shuts their laptop a few seconds later, before the app has reconnected, and goes home.

```python
def on_open(conn):
    presence.set(conn.user_id, "online")

def on_close(conn):
    presence.set(conn.user_id, "offline")
```

What do teammates see beside this colleague's name that evening?

<details><summary>A. A green dot, all evening</summary>

✓ Right. The dot stays green all evening, with the laptop shut and its owner at home. Teammates go on writing to someone who will not answer.

`on_close` runs on the server that sees a connection close, and this machine has stopped. The pong timer was on it too, the other servers keep no list of its connections, and this code gives the entry no expiry. So the entry says `online` until someone removes it, and nobody is left to do that. Presence has to be a lease, **an entry that lapses unless a live server keeps renewing it**.

Have each server renew an entry per connection, say every 30 seconds with a lease of 90, and let the dot turn offline when no entry is left. Counting connections this way also stops a shut laptop from marking someone offline while their phone is still connected.

</details>

<details><summary>B. An offline dot, from the moment the machine went down</summary>

✗ Clustered stores advertise that they know their members, and this machine's power has gone. The store holds what servers write to it, and the only server that would have written this entry has stopped, so the entry is unchanged.

</details>

<details><summary>C. An offline dot, about a minute after the machine went down</summary>

✗ A pong 60 seconds overdue closes a connection, and on an ordinary day that rule has cleared many a dead phone. It runs on the server holding the connection. With no power on that machine no timer fires, and `on_close` never runs.

</details>

<details><summary>D. An offline dot, once the machine is back up</summary>

✗ A service that rebuilds its state on startup gets a chance to tidy up what it left behind. This server starts with no memory of the connections it held, so it has no list of whom to mark, and the colleague's entry says `online`.

</details>

---

###### 7. Turning on sticky sessions for a two-server chat

An online tutoring site runs its lesson chat on one socket server, and you add a second behind the balancer for the autumn term. Each server sends a message to its own sockets in that lesson. Tutors and pupils start to report that about half of their messages never show up. A colleague has read that sockets on several servers need sticky sessions, and turns them on at the balancer that evening.

What do tutors and pupils report the next day?

<details><summary>A. Every message reaching its tutor or pupil</summary>

✗ Your colleague read it in library guides, which do ask for sticky sessions on several servers. They ask so that the requests of one long-polling session reach one process. A message here still stops at the server holding its sender, and a pupil pinned to the other server never hears it.

</details>

<details><summary>B. Every message reaching them, with all lessons on one server</summary>

✗ In plain English, sticky means everything stays in one place, and that would put every lesson together. The balancer still chooses a server for each new client as it did before, and only repeats that choice for the same client later, so both servers stay in use.

</details>

<details><summary>C. Messages missing only after someone reconnects</summary>

✗ Stickiness brings a client back to its server after a drop, so a reconnect is where its effect shows. These losses need no reconnect: a tutor and a pupil on different servers miss each other's messages on every line they type.

</details>

<details><summary>D. About half of their messages still missing</summary>

✓ Right. The lessons stay as broken as they were, with about half of the messages missing, since the new setting has nothing to do with the fault.

**A sticky session keeps one client on one server**, and a tutor and a pupil are two clients, each pinned on its own. Each client can land on either of two servers, which makes four pairings, and two of them put both on one server, so half the pairs still hear each other, as before the change. What the chat lacks is a way to pass a message between servers.

Publish each message to a pub/sub backplane that every server subscribes to, and have each server send it to its own sockets in that lesson. Keep sticky sessions only where a fallback to long polling needs them.

</details>

---

###### 8. Restarting the pub/sub backplane under open sockets

A sports app pushes live scores over WebSockets from ten socket servers. A score service writes each score to a score log, then publishes it to a pub/sub backplane that keeps no messages, to which all ten servers subscribe. A phone that reconnects sends the last score id it saw and gets what it missed from the log, and that path held through every dropped connection in testing. On Saturday an engineer restarts the backplane during a match. For three seconds its publishes fail, and then the servers subscribe again.

What do phones get of the scores from those three seconds?

<details><summary>A. All of them, from the score log</summary>

✗ The score log holds every one of them, and catching up from it passed every drop in testing. Catching up starts when a phone reconnects with its last id. No phone reconnected, so nobody asks the log.

</details>

<details><summary>B. None of them, with the sockets open throughout</summary>

✓ Right. The phones get none of those scores while their sockets stay open throughout, so a screen can miss a score with no sign that it did.

**Catching up runs when a phone reconnects**, and these phones never disconnected. The break was at the backplane, and each socket from server to phone stayed up. A backplane that keeps no messages reaches only whoever is subscribed at that moment. Say scores 1, 2 and 3 go out and 2 falls inside the break. The phone holds 1 and then 3, score 2 sits in the log, and no code compares them.

Put a sequence number per match on every message. A phone that gets 3 after 1 knows that 2 is missing and asks for it from the log, the way it already does on a reconnect.

</details>

<details><summary>C. All of them, after every phone reconnects</summary>

✗ A health check can mark a server cut off from its backplane as broken, and a broken server's connections drop. This one stays up and goes on holding its phones, since nothing about those connections has changed, so no phone has a reason to reconnect.

</details>

<details><summary>D. All of them, late, from a queue on each socket server</summary>

✗ Brokers hold messages for a consumer that has gone away, and servers often buffer when something they depend on disappears. These publishes failed at the backplane, so the scores never reached any socket server, and there is nothing there to queue.

</details>

---

###### 9. Sending every reaction to every viewer of a live stream

On a live-streaming site, a viewer can tap a heart at any moment, and every tap shows up on the screen of everyone watching, through their sockets. Last month's biggest broadcast had 2,000 viewers on four socket servers, which coped. Next week's final is booked for 20,000, with viewers expected to tap as often as they do now, so you order ten times the servers.

What does each socket server send each second at the final, compared with last month?

<details><summary>A. About what it sends today</summary>

✗ Load usually follows users, so a capacity plan multiplies by ten and expects each server to stay level. A tap here reaches the whole audience, so more viewers means more taps and also more copies of each tap, and that grows far faster than the servers.

</details>

<details><summary>B. Less than today, with the backplane doing the fan-out</summary>

✗ A pub/sub diagram shows one arrow going in and a whole room coming out. The backplane moves a tap between servers. The last step, one frame down each viewer's socket, is still written by the socket server holding that viewer.

</details>

<details><summary>C. Many times what it sends today</summary>

✓ Right. Each server sends about ten times what it sends today. The order for ten times the servers leaves every one of them far past what it coped with last month.

**Taps grow with the audience and so does the list of screens for each tap**, so frames grow with the square of the audience. If each viewer taps once every ten seconds, last month's 2,000 viewers made 200 taps a second and 400,000 frames, 100,000 per server. At the final, 20,000 viewers make 2,000 taps and 40 million frames, 1 million per server.

Stop sending each tap. Count them on the server and send each viewer one frame a second with the total, which makes frames grow with the audience alone.

</details>

---

###### 10. Doubling the socket servers on one backplane channel

A team-messaging company runs 40 socket servers for 200,000 rooms of a few people each. Every message goes out on one pub/sub channel, every server subscribes to it, and each server passes a message on to any of its own sockets in that room. The servers now run at 85 percent CPU, nearly all of it spent on messages. Phones connect evenly, and you add 40 more servers.

What does each server's CPU do once the new ones hold their share of the phones?

<details><summary>A. It falls to about half</summary>

✗ Twice the servers and half the load on each is how adding servers is taught, and the phones per server do halve. A server passes on only the few messages for rooms it holds, so sending to phones is the small part of this CPU. Reading the channel is the large part, and every server still reads every message.

</details>

<details><summary>B. It rises, with each server now talking to twice as many others</summary>

✗ Mesh designs pay for every peer, since each node talks to all the others. These servers never talk to one another: they read one channel holding the same messages as before, and the extra copies are the backplane's work.

</details>

<details><summary>C. It falls to about half, until the backplane becomes the limit</summary>

✗ A shared part is usually what gives out once the servers have had their relief. Here each server's reading stays where it was, so the relief never comes, while the backplane's work doubles at the same moment.

</details>

<details><summary>D. It falls by a few points</summary>

✓ Right. If reading takes 70 of the 85 points, each server falls to about 78 percent, so forty new machines buy almost nothing, and the backplane now works twice as hard.

The phones per server halve and the frames go with them, but **every server reads every message on the channel**, so the reading does not move. In a room of three people, at most 3 of the 80 servers need any given message.

Give each room its own channel, or split rooms over a set of channels, so a server subscribes only to rooms it holds sockets for. A new server then takes phones and rooms and the messages that belong to them.

</details>

---

###### 11. Sending two messages across a reconnect

A shop's support chat runs on several socket servers joined by a backplane, and the agent's screen shows each message as it arrives. A load test sent 10,000 messages from one phone, and they came out in the order they went in, every time. On a train at the evening peak, a customer types "Cancel my order". A few seconds later the phone's Wi-Fi drops, the app reconnects through another socket server, and the customer types "Wait, keep it".

In what order can the agent's screen show the two messages?

<details><summary>A. Cancel, then keep, every time</summary>

✗ A WebSocket runs on TCP, libraries cite that when they promise order, and the load test bore it out. That promise covers one connection, while the reconnect put the second message on another connection through another server, with nothing holding it back for the first.

</details>

<details><summary>B. Keep alone, with the cancel lost</summary>

✗ Data in flight on a connection that drops can be lost, and that is the risk most people plan for. The Wi-Fi stayed up for seconds after the cancel was typed, long enough for it to reach its server, and the server passes it on whatever then becomes of the phone, so the agent gets both.

</details>

<details><summary>C. Either order, with "Wait, keep it" on top at times</summary>

✓ Right. The agent can see either order, so "Wait, keep it" can arrive above "Cancel my order", and an agent acting on the last line cancels an order the customer wants.

**TCP orders the bytes of one connection**, and these messages used two connections and two servers, each working through what it holds at its own pace. Say the messages are typed 5 seconds apart. With the first server 10 seconds behind at the peak and the second keeping up, they arrive reversed. With the first server 1 second behind, they arrive as typed. The load test could not show it, since its 10,000 messages shared one connection.

Have the phone number its messages, and have the receiver hold or sort them by that number, or pass a room's messages through one place that gives them their order. The number has to outlive the connection, or the reconnect resets it.

</details>

---

###### 12. Loading the seat map, then opening the socket

A cinema's booking page draws its seat map from a GET and keeps it current over a WebSocket, which pushes each seat as it sells. On the night a blockbuster goes on sale, seats sell several times a second.

```javascript
const seats = await fetch("/shows/42/seats").then(r => r.json());
drawMap(seats);

const socket = new WebSocket("wss://tickets.example/shows/42");
socket.onmessage = (event) => markSold(JSON.parse(event.data).seat);
```

What does a customer's map show a minute after opening the page in that rush?

<details><summary>A. Some sold seats still marked free</summary>

✓ Right. Some sold seats are still marked free, and they stay that way while the page is open. A customer can pick a seat that has gone and be refused at the end.

**A seat sold after the GET read the table and before the socket subscribed is in neither of them**: the map says free, the seat's one push went out before this page was listening, and no later message mentions it. If 5 seats sell each second and the gap is 300 ms, the rush leaves about 1.5 wrong seats on every map opened.

Open the socket first and hold what it sends, then GET the map with a version, and apply only the updates newer than that version. The booking itself still checks the store, since another customer can buy the seat while this map still shows it free.

</details>

<details><summary>B. Every sold seat marked, as the updates come in</summary>

✗ The socket does carry each sale once it has subscribed, and that is what keeps the map live while the customer picks. A sale before that moment went to the sockets open at the time, and this page was not among them.

</details>

<details><summary>C. Every sold seat marked, with the earlier sales arriving over the socket</summary>

✗ Some libraries catch a returning client up, so a server that knows what a page missed does exist. It has to be told what the client last saw. This page tells it nothing, and the server cannot know how old the map behind a new connection is.

</details>

<details><summary>D. Seats that flip between sold and free as updates cross the GET</summary>

✗ Two channels racing is the danger people know, with an older reply landing on a newer update. This code cannot do that: it waits for the GET to finish before it opens the socket, so no update can arrive ahead of the map, and updates only ever mark a seat sold.

</details>

---

###### 13. Replaying from a five-minute log after a night offline

A hospital's staff chat keeps the last five minutes of each room in a shared replay log, with ids that only rise. The app loads the full history from the database when it starts, and on a reconnect it sends the id it saw last. A ward clerk closes a laptop at 18:00 with the chat open in a tab, and opens it at 08:00.

```python
def on_resume(conn, room, last_id):
    conn.subscribe(room)
    for message in replay_log.read(room, after=last_id):
        conn.send(message)
```

What does that tab show for the night's messages?

<details><summary>A. None of them, only new ones</summary>

✗ An id older than anything in the log matches nothing, the way a lookup by key finds nothing. `read` asks for ids above the one it is given rather than for that id, so every entry the log holds qualifies, and the tab gets those.

</details>

<details><summary>B. The night's last five minutes, directly under last evening's messages</summary>

✓ Right. The tab shows the night's last five minutes directly under last evening's messages, with no mark between them, so the clerk reads a chat with 14 hours missing as a chat that is complete.

The log holds five minutes and hands back what it holds above the id, and an id from 18:00 is below all of it. There is no error, and **the reply has the same shape as the reply to a drop of a few seconds**. If a room gets 6 messages a minute, the night holds 5,040 of them and 30 come back. The handler sees only the entries, so it cannot tell the two replies apart.

Have the log report its oldest id as well, compare it with the id the client sent, and when the client's is older, tell it to load the room from the database.

</details>

<details><summary>C. A full reload of the room, after the log rejects the old id</summary>

✗ Some log brokers can report an error on a position they no longer hold, so a client knows to start again. This `read` answers with the entries it holds above that id, and the handler has no branch that could order a reload.

</details>

<details><summary>D. All of them, in order</summary>

✗ Sending the last id is how a client asks for everything since, and after a short drop the tab does catch up completely. The log can return what it still holds, and by 08:00 that is the last five minutes.

</details>

---

###### 14. Broadcasting each bid to a room in a loop

A car auction sends each bid to everyone in the room from one socket server process. Every bid starts its own `broadcast` task, a separate run of the function scheduled alongside the others, and in this library `conn.send` waits while that connection's send queue is full. Of the 300 bidders in a room, one is on a phone in a lift, where the connection has stopped taking data and its send queue has filled.

```python
async def broadcast(room, bid):
    for conn in room.connections:
        await conn.send(bid)
```

What do the other bidders see as the next bids come in?

<details><summary>A. Every bid on time for all of them</summary>

✗ Async code is sold on nobody waiting for anybody, and the server's other tasks do keep running. A task still stops at each `await` until that send finishes, and this task has the rest of the room's list behind the connection that stalled.

</details>

<details><summary>B. Every bid late, at the phone's pace</summary>

✗ One slow connection holds back everything that waits on it. Each bid runs in a task of its own and walks the list in order, so bidders ahead of the phone get every bid at once, and only those behind it wait.

</details>

<details><summary>C. Every bid on time for the others, with the phone's connection closed</summary>

✗ Closing a connection whose queue stays full is the usual cure for a slow client. That takes code or a setting of its own, and this `broadcast` holds neither: the send goes on waiting at the phone, with the rest of the list unreached.

</details>

<details><summary>D. Bids on time for those listed before the phone, and none for the rest</summary>

✓ Right. Bidders listed before the phone get each bid on time and those after it get none, so if the phone is at place 121, 120 bidders see the auction move and 179 bid against a price that has stopped.

**The loop sends to one connection at a time and waits for each**, so at the stalled connection the send does not finish, the task never reaches the rest of the list, and each new bid's task stops at the same place.

Give each connection its own send queue and its own task to write it out, so `broadcast` only adds to queues and a stalled phone delays itself. Close a connection whose queue stays full, and let it come back through the replay log.

</details>

---

###### 15. Streaming prices to a phone on a ferry

An exchange app streams 20 price updates a second to each phone. In office tests on Wi-Fi, every phone showed prices under a tenth of a second old. On a ferry, a customer's connection can carry about one update a second, and it stays open all afternoon.

What does that customer's screen show by late afternoon?

<details><summary>A. The latest price, with the updates in between skipped</summary>

✗ Video on a weak link drops its quality and stays live, and streams over UDP skip what they cannot carry. A WebSocket runs on TCP, which skips nothing, and the send queue drops nothing either, so the updates arrive in turn, each one further behind than the one before.

</details>

<details><summary>B. A price from hours earlier</summary>

✓ Right. The screen shows a price from hours earlier as if it were current, on a connection that is up and still updating, so the customer decides on an old price.

**TCP delivers every byte in order**, and the send queue, which has no limit here, keeps every update, so the phone reads them in the order sent, one a second, while 20 a second go in. After four hours the queue on the server holds about 270,000 updates, and the price on screen is about 3.8 hours old. Office Wi-Fi kept up, so the queue was always empty in the tests.

For a feed like this one, keep only the newest update for each price in the queue, often called conflation, so a slow phone gets current prices less often.

</details>

<details><summary>C. A lost connection, after sends start to fail</summary>

✗ A buffer that fills makes sends fail in plenty of code you have seen. `send` adds to a queue that keeps growing, and the link stays up carrying its one update a second, so no send fails and nothing closes.

</details>

<details><summary>D. The price from when the link fell behind, frozen there</summary>

✗ A slow app shows a spinner and stops updating, which is what falling behind usually means to its user. This link still carries one update a second, so the number on screen keeps changing, through prices from further and further back.

</details>

---

###### 16. Restarting twenty socket servers one at a time

A game's chat runs on 20 socket servers behind a balancer using round robin. A rollout restarts them one at a time, a few minutes apart, each after a drain of 60 seconds. Players stay connected for hours, and the app reconnects within seconds when cut. A player connects shortly before the rollout begins, to the first server in line.

How many times does that player's connection get cut during the rollout?

<details><summary>A. A few times, on server after server</summary>

✓ Right. The player is cut a few times in one rollout, so the chat breaks several times for the same person. If each reconnect lands on any other running server, a player on the first server is cut about 2.65 times on average.

A drain stops new connections and waits for running ones to end, and a connection that lasts for hours does not end in 60 seconds, so the server stops and cuts what it still holds. **The reconnect goes to the next server in turn, most likely one still to be restarted**, which puts the player ahead of the rollout again.

Start a spare restarted server first and send reconnects only to servers already restarted, which cuts each connection once. Close connections a few at a time through the drain, with close code 1001, rather than all together at its end.

</details>

<details><summary>B. Never, with the drain waiting for the connection to end</summary>

✗ Draining means running work finishes, and with requests that takes a second or two. This connection runs for hours and the drain lasts 60 seconds, so the server stops when the time is up, and the player is cut with it.

</details>

<details><summary>C. Once, when its own server restarts</summary>

✗ Each server takes its turn once, and this player's server is first in line. The cut sends this player to another server, most likely one that has not been restarted, and that server's turn is still to come.

</details>

---

###### 17. Losing one socket server of twenty at the evening peak

A delivery marketplace holds a million phones on twenty socket servers, 50,000 each, and the capacity plan counts the connections a server can hold. On a new connection, the server checks the token's signature and loads the user's rooms from a database. At the evening peak one socket server crashes, and its phones reconnect within seconds.

What takes the hardest hit in the first minute?

<details><summary>A. Nothing, with the other nineteen each taking a few more phones</summary>

✗ Twenty servers exist so that one can fail, and the nineteen do have memory to spare. The room loads behind each handshake appear nowhere in that plan, and every phone on the crashed server needs one from the database within seconds.

</details>

<details><summary>B. The other nineteen servers' memory</summary>

✗ Connections take memory and the capacity plan counts them, so a pile of new ones is what it warns about. Each of the other nineteen gets about 2,600 more, so a server at half of its memory goes to about 53 percent.

</details>

<details><summary>C. The database behind the new connections</summary>

✓ Right. The database takes 50,000 room loads within seconds, which leaves the phones waiting to finish connecting and slows whatever else uses it.

Spare capacity on a socket server is counted in connections held, and a crash spends little of it. **Making a connection is a separate load from holding one**, and an ordinary evening spreads it thin. If the phones come back over 5 seconds, that is 10,000 loads a second, against say 40 on an ordinary evening and 2,000 that the database can do. With no retries the queue clears in 25 seconds. A phone that times out and tries again adds to the queue, which is how a rush becomes a lasting overload.

Cap how many new connections a server handles at once and refuse the rest with a delay. Keep room lists where loading them does not touch the database, and plan for connections made as well as held.

</details>

---

###### 18. Disabling a support agent's account at 14:00

A bank's support tool shows agents incoming customer chats over a WebSocket. Login gives a 15-minute token, the page renews it in the background for its HTTP calls, and the socket server checks it on the handshake. An agent leaves the company at 14:00, with the tool open on a laptop, and an administrator disables the account. From that moment renewal fails and every HTTP call the page makes gets a 401.

What does the still-open page get at 15:00?

<details><summary>A. New customer chats, as it did before 14:00</summary>

✓ Right. At 15:00 the page still gets new customer chats, an hour after the account went off. Someone who has left the bank goes on reading customers' messages while every HTTP call the page makes fails.

A token is checked when something reads it, and **the socket server read this one once, on the handshake**. Frames carry no token, and nobody has told the server about 14:15 or about the account. The 401s come from the HTTP API, which reads the token on every call, so they prove nothing about the socket.

Store the token's expiry time with the connection and close the socket at that time, at 14:15 here at the latest, so that the page has to come back with a fresh token. Publish a disabled account on a backplane too, so that every socket server closes that person's connections at once.

</details>

<details><summary>B. Nothing after 14:15</summary>

✗ A token is good until its expiry and no longer, and the last token issued before 14:00 runs out by 14:15. That holds where a token is read, and after the handshake the socket server has no reason to read it, so 14:15 passes with no event on this connection.

</details>

<details><summary>C. Nothing after 14:00</summary>

✗ The administrator's screen says the account is disabled, and the page's HTTP calls stopped at that minute. That change reached the systems that ask on every call. The socket server checked once, at the handshake, and the chats keep coming.

</details>

<details><summary>D. Only the chats the agent had open at 14:00</summary>

✗ Rooms are often authorized when they are joined, which would keep what was open and stop what is new. This page gets incoming chats over its one connection, whose rights were settled on the handshake, so a new chat arrives exactly as an old one does.

</details>

---

## Further reading

- [Metastable Failures in Distributed Systems](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s11-bronson.pdf), 2021: how a system stays in a bad state after its trigger has gone, with retries and backlogs among what keeps it there.
- [Metastable Failures in the Wild](https://www.usenix.org/conference/osdi22/presentation/huang-lexiang), 2022: 22 metastable failures from 11 organizations, read from their public incident reports.
- [The φ accrual failure detector](https://doi.org/10.1109/RELDIS.2004.1353004), 2004: a detector that reports a level of suspicion on a continuous scale instead of a yes or a no.
- [Leases](https://doi.org/10.1145/74850.74870), 1989: leases as a fault-tolerant mechanism for keeping file caches consistent across machines.

Prev: [Event-driven architecture](../event-driven-architecture/README.md) · Next: [Microservices](../microservices/README.md)
