# DNS, TCP and TLS

Prev: [Performance and capacity](../performance-and-capacity/README.md) · Next: [HTTP caching and CDNs](../http-caching-and-cdns/README.md) · Reviewed October 2026

Before one machine can send a request to another by name, three protocols do their work: DNS turns the name into an address, TCP sets up a connection to that address, and TLS proves who is at the far end and encrypts what follows.

| Protocol | What it does | What it hands to the next step |
| --- | --- | --- |
| DNS | looks a name up, in a cache or at the servers that hold it | an address |
| TCP | sets up a connection and carries bytes, retransmitting what is lost | one ordered run of bytes in each direction |
| TLS | shows the server's certificate, agrees keys and encrypts the bytes | a private channel for the HTTP request |

- **Resolver**: the software that turns a name into an address. A stub resolver on each machine asks a caching resolver, which asks name servers, the servers that hold a domain's records, and keeps their answers.
- **TTL** (time to live): how long a resolver may keep an answer. A resolver may cap it, and some can serve expired answers when the name servers are out of reach. A "not found" answer is cached too, for a TTL of its own.
- **CNAME**: a record that makes one name an alias of another, so a lookup carries on at the target.
- **Search list**: the domains a stub resolver can add to a short name, with a threshold on dots deciding which names count as short.
- **Source port**: the port a client picks from a range for one connection. Two addresses and two ports together identify a connection.
- **TIME_WAIT**: the state a side enters on closing a connection first, keeping its addresses and ports for a while so that late packets cannot reach a new one.
- **Congestion window**: how many bytes a sender may have on the way and not yet acknowledged, one window per round trip (the time from sending a packet to hearing its answer). The receiver advertises a limit of its own, the receive window, and a long path needs a larger window to fill its link.
- **Slow start**: a new connection begins with a small window and roughly doubles it each round trip.
- **Congestion control**: the sender's rule for sizing its window to what the path can take. A loss-based one treats a lost packet as a sign of a full path; a model-based one sets its rate from measurements of the path.
- **Reset**: a packet that ends a connection at once; the side that gets it sees an error on its next call.
- **Stream**: one request and its reply inside a shared connection. HTTP/2 runs many streams over one TCP connection; QUIC, a transport carried in UDP datagrams, runs many with ordering kept per stream.
- **Certificate chain**: the server's own certificate, called the leaf, then one or more intermediates, each signed by the next, up to a root.
- **Trust store**: the list of root certificates a client accepts. A root has validity dates like any certificate, and a runtime or an application may keep a list of its own, apart from the host's.
- **Session ticket**: what a server gives a client after a handshake, good for up to seven days, so that a later connection can resume with no certificate sent.
- **Early data**: a request a resuming TLS 1.3 client sends in its first flight, its opening messages, before the handshake has finished, saving a round trip.

Unless a question says otherwise, a resolver neither shortens a TTL nor answers past its end, and a TCP stack keeps Nagle's algorithm and delayed ACK on and path MTU probing off.

The trouble mostly sits below the code that made the call: a cache you do not run, a connection that still counts as open, a kernel queue, a certificate in memory.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- work out what a faster link buys on a long path, and what a network in the middle can read of an encrypted call
- plan the move of a site or a database to a new address, and set TTLs, search settings and leftover records around it
- find what a stalled or failing connection is waiting for, from a burst of connects and two small writes to a lossy line, a tunnel, a quiet hour and an early close
- renew a certificate and install its whole chain, see what an expiry at midnight touches and turn on early data, knowing which clients keep working

###### 1. Buying a faster link for customers on another continent

A shop's API runs in Europe and serves customers in Australia. Each call opens a new connection with TLS and fetches a 100 KB reply, traffic is light, and the customers' lines are fast, well above what such a reply needs. Customers say the shop is slow, so the operations lead raises the shop's own link from 1 Gbps to 10 Gbps.

How long do those customers wait for a reply after the upgrade?

<details><summary>A. A tenth of the old wait</summary>

✗ Size over bandwidth is the sum on every download page, and for a large file on a slow line it gives the right answer. Nearly all of this wait is round trips across the world, and a tenfold link takes none of them off.

</details>

<details><summary>B. About half the old wait</summary>

✗ A call has two parts, the handshakes and the transfer, and a faster link speeds up a transfer. On a new connection the transfer is round trips as well: the sender puts out a small window, waits to hear back and doubles it, and 10 Gbps leaves that waiting as long as it was.

</details>

<details><summary>C. About the wait they had on the old link</summary>

✓ Right. Customers in Australia wait as long on the 10 Gbps link as they did on the 1 Gbps one.

The wait on a new connection is counted in round trips, and **a link's speed does not change how long a round trip takes**. The TCP handshake costs one, the TLS 1.3 handshake one more, and slow start spreads the reply over several, since the window starts small and doubles each time. If the round trip is 280 ms and the first window is ten packets of 1,460 bytes, the 100 KB reply (taken as 100,000 bytes) needs three round trips of sending. The whole call then needs five, about 1.4 seconds on either link.

Take round trips away instead: keep connections open between calls, which skips both handshakes, and serve from a machine closer to the customers, which shortens each trip.

</details>

<details><summary>D. Longer than on the old link</summary>

✗ A fast link feeding a slow one builds a queue where the speeds meet, and traffic on such a path can get slower after an upgrade. These lines are well above what a 100 KB reply needs, so the reply's small flights find nothing to queue behind, and the wait does not grow.

</details>

---

###### 2. Making an HTTPS call on a hotel's network

A travel app calls an HTTPS API with the booking id in the path, such as `/bookings/84731`. Its TLS client is an ordinary one, with no privacy extensions. A guest opens a booking on the hotel's Wi-Fi, and the hotel's operator logs all it can of the traffic.

What does the hotel's log hold about that call?

<details><summary>A. The API's name and address, and the traffic's sizes</summary>

✓ Right. The log shows which API the guest called, at which address, how much went each way and when. The hotel can tell this guest uses this travel service, and it cannot tell which booking was opened.

TLS protects what it wraps, and **it wraps neither the packets that carry it nor the message that starts it**. Every packet shows its destination address. The client's first message carries the server name, which the server needs in order to pick a certificate, and a lookup over plain DNS shows that name as well. TLS 1.2 and TLS 1.3 both encrypt the path, the headers and the body, and both leave the length and timing of the records in view.

Treat the name, the address and the sizes as visible to anyone on the path. TLS lets a sender pad its records to hide sizes, and an extension can encrypt that first message, though the packets still show the address.

</details>

<details><summary>B. The full URL with the booking id, and no body</summary>

✗ A URL is what everyone calls the address of a request, and an address is the part a network can see. The path belongs to the HTTP request, which travels inside TLS: `/bookings/84731` never appears in the bytes the hotel's log can read.

</details>

<details><summary>C. Scrambled bytes that name no server</summary>

✗ A padlock in the address bar marks the whole connection as secure, and encrypted traffic is unreadable. The encryption starts only after the first messages of the handshake. The app's first message comes before it, in the clear, and it names the API.

</details>

<details><summary>D. The server's address, with no name beside it</summary>

✗ A packet is routed on its address alone, and the name is used once, to find that address. The TLS client sends the name again in its first message, so that one address can serve several names, and the hotel's log picks it up there.

</details>

---

###### 3. Lowering a record's TTL on the morning of a move

A florist moves its site to a new address. Its record has had a one-day TTL for years, and the old server stays up through the move. You follow the checklist that morning.

```text
09:00  you set the record's TTL to 60 s
09:01  the provider's panel shows a TTL of 60 s
09:05  you change the record to the new address
09:06  the panel shows the new address
```

How long after 09:05 can visitors still land on the old server?

<details><summary>A. About a minute, the new TTL</summary>

✗ The panel has shown 60 s since 09:01, and the panel is right about the record. That 60 s travels only on answers fetched after 09:00. A resolver that asked shortly before 09:00 holds an answer marked good for a day, and its visitors go on reaching the old server long after the minute is up.

</details>

<details><summary>B. Almost no time, once the panel shows the change</summary>

✗ The usual word for a DNS change is propagation, and the panel says done at 09:06. Name servers push nothing out: they answer whoever asks, and a resolver holding an unexpired answer does not ask.

</details>

<details><summary>C. Up to two days, the time DNS takes to spread</summary>

✗ Help pages for domains have long warned of a day or two for any DNS change. No such timer exists in the system. Each resolver holds one answer until its TTL ends and then asks the name servers directly, and here no answer can be older than a day.

</details>

<details><summary>D. Up to a day, the old TTL</summary>

✓ Right. Visitors can land on the old server until about 09:00 the next morning, and the 60 s on the panel does nothing for them.

**A resolver goes by the TTL on the answer it already holds**, and it learns of a new record only when it asks again. Lowering the TTL at 09:00 changes what the name servers hand out from then on, and a resolver that fetched the record shortly before 09:00 still holds it for a day.

Lower the TTL at least one old TTL ahead, here the day before, so that every cached answer carries the short one by the time you change the address. The move then settles within 60 s, on resolvers that keep to the TTL.

</details>

---

###### 4. Keeping a connection pool when the address changes

A payment service keeps a pool of connections to its database, opened by name, and reuses them, opening a new one only when one dies. The record's TTL is 60 s. At 14:00 you point the name at a new host and leave the old host running. All the pool's connections stay healthy for the next hour.

Where do the service's queries go at 15:00?

<details><summary>A. To the new host, the TTL long past</summary>

✗ The TTL ran out by 14:01, and any lookup after that returns the new host. This pool makes no lookup: its connections are open, it opens a new one only when one dies, and none has, so nothing in the service has asked the name again.

</details>

<details><summary>B. To the old host, on the connections already open</summary>

✓ Right. At 15:00 every query from the payment service still goes to the old host, and the database you meant to retire is the one taking its writes.

**A name is looked up when a connection is opened**, and from then on the connection is tied to two addresses and two ports and never looks the name up again. The 60 s TTL governs the lookup's answer and leaves open connections alone.

Give connections a maximum age, so that the pool replaces them and looks the name up each time. Some runtimes look a name up once at start and keep the address, and their new connections reach the old host too, so check how your runtime caches lookups before you switch the old host off.

</details>

<details><summary>C. To both hosts, the pool moving over one connection at a time</summary>

✗ Pools with a maximum connection age drift across in this way, retiring one connection after another. This pool retires nothing: it replaces a connection only when one dies, so every connection is still the one opened before 14:00.

</details>

<details><summary>D. Nowhere, the pool cut off when the record changed</summary>

✗ Database moves get maintenance windows for fear of this very cut. A record change sends nothing to anyone. The connection runs between two addresses and two ports, the old host is still up, and the queries keep flowing over it.

</details>

---

###### 5. Losing the name servers of a news site and its archive

A publisher runs a news site and its archive on its own web servers, with both names at the same DNS provider, and both are visited steadily. You set the news name's TTL with failover in mind, so that a move to its standby settles fast. One afternoon the provider's name servers become unreachable, and they stay that way for an hour. They hold these two records.

```text
name                       type  TTL
news.publisher.example     A     30 s
archive.publisher.example  A     1 day
```

Half an hour into the outage, which site can most visitors still reach?

<details><summary>A. The archive, on an address fetched before the outage</summary>

✓ Right. Half an hour in, the archive still resolves for most visitors. The news site, with the short TTL you chose for fast failover, has stopped resolving for nearly all of them.

**A cached answer can be renewed only at the name servers**, and those are now out of reach. The news name's 30 s entries run out in the first minute of the outage. If visits are steady, about 98 of 100 resolvers still hold the archive's one-day entry half an hour in. A resolver that serves expired answers would keep both names working, and only some resolvers have that option.

A TTL is both how long a move takes to settle and how long a name survives its name servers. Pick the shortest your real moves need, rather than the shortest the provider allows.

</details>

<details><summary>B. Both sites, on the last addresses their resolvers held</summary>

✗ An HTTP cache can keep serving its last copy while the origin is away, and some resolvers have an option to do the same. A resolver without that option drops an entry when its TTL ends, so the news name's 30 s entries are gone in the first minute, and its visitors get a failed lookup with the web server still up.

</details>

<details><summary>C. The news site, on its standby's address</summary>

✗ Failover guides recommend the shortest TTL the provider allows, since it lets you react in seconds. Reacting needs a name server to hand out the standby's address, and here there is none, so the news name's entries run out within the minute and nothing replaces them.

</details>

<details><summary>D. Neither site, with no name server left to answer</summary>

✗ A lookup ends at the name servers that hold the record, and those are what went away. A resolver goes to them only when its entry has run out, and most entries for the archive's name have hours left, so most of its visitors get their answer from the cache and reach the site.

</details>

---

###### 6. Calling an outside API through a search list

A service in a cluster calls a partner's API by the name `api.partner.example`, exactly as written here. It makes 200 calls a second, and each call looks the name up afresh through the host's stub resolver, which has these settings.

```text
search list                 svc.cluster.local  cluster.local  corp.example  lan.example
dots for a complete name    5
```

What does the host's stub resolver send for one lookup of that name?

<details><summary>A. One query, for <code>api.partner.example</code> as written</summary>

✗ The name has dots in it, and it is the partner's real, full name. The stub goes by the count: two dots against a threshold of five make this a short name, so four other names are asked about before this one.

</details>

<details><summary>B. The name as written, then the search list only if it fails</summary>

✗ A fallback is tried after the exact thing has failed, and a short name you type at a prompt gets filled in that way. That order applies to a name at or above the threshold. Under it, the list goes first and the name as written comes fifth.

</details>

<details><summary>C. A failed query for each search domain, then the name as written</summary>

✓ Right. One lookup tries four names that come back not found and then the one that works. Each call to the partner's API starts with four questions about names nobody ever created.

A name with fewer dots than the threshold counts as short. **The stub tries a short name with each search domain in turn and only then as written**: the first attempt is `api.partner.example.svc.cluster.local`, and `api.partner.example` itself is the fifth. With two record types asked on each lookup, 200 lookups a second make 2,000 queries a second, 1,600 of them failures.

Write the name with a final dot, `api.partner.example.`, which makes it complete and leaves one query per record type. A threshold of 1 also leaves one query per record type for this name, but it changes how every name with one to four dots is tried on the host.

</details>

<details><summary>D. One failed query on the first search domain, then an error to the caller</summary>

✗ In most code, not found is a final answer: a missing key ends the call. To a stub it means only that this one attempt does not exist. After `api.partner.example.svc.cluster.local` fails, it moves to the next domain and keeps going until a query gets an answer, and the fifth does.

</details>

---

###### 7. Leaving a CNAME behind after closing a CDN account

A retailer ran a spring campaign on a subdomain, a CNAME to a hostname at a CDN, a content delivery network that serves pages from its own machines. Setting it up took one form in the CDN's console: you picked a free hostname, typed the subdomain, and the pages went live. The campaign ends, and you close the CDN account, which frees the hostname. The CNAME stays in the zone, the domain's list of records, where the campaign left it.

Who can put pages under that subdomain from now on?

<details><summary>A. Nobody, until the retailer points the name elsewhere</summary>

✗ A link to a deleted page stays dead until someone fixes it, and the hostname behind this subdomain has been freed. The CNAME stays in force, though: the record is in the zone, and each lookup carries on at that hostname for whoever holds it next. The failure on closing day lasts only while that hostname has no new holder.

</details>

<details><summary>B. The retailer's team alone</summary>

✗ The zone belongs to the retailer, and no one else can change a record in it or prove the domain is theirs. This CDN's form asks for no such proof, and it asks the next customer what it asked you. The record in your zone still sends every visitor to the target, and the target's holder picks what comes back.

</details>

<details><summary>C. The CDN alone, with its own page on every visit</summary>

✗ Visit the subdomain right after closing the account and the CDN's own page is what you get. That page stands in only while the hostname has no holder. The next customer who fills in the form takes the hostname, and that customer's pages replace it.

</details>

<details><summary>D. The next customer to take that hostname</summary>

✓ Right. Whoever takes the hostname next serves what they choose under the retailer's own subdomain, so the retailer's customers can get a stranger's pages at an address that looks like the shop's.

A CNAME makes a name an alias and the lookup carries on at the target, so **the name shows whatever the holder of its target serves**. Providers differ on who can become that holder: one that demands proof of owning the pointing domain stops a stranger from taking the name, and one with no such check, like this CDN with its one form, lets the name go to anyone.

Delete the CNAME before you release what it points at, and compare the CNAMEs in your zones with the accounts you still hold to find the ones already left behind.

</details>

---

###### 8. Opening a new connection for every call to one backend

A report worker opens a new TCP connection for each call to one internal API, at one address and port, and closes it itself when the reply is in. It makes 2,000 calls a second, each over quickly. After some seconds, new connections start failing on the worker.

What has the worker run out of?

<details><summary>A. Bandwidth on the worker's network link</summary>

✗ When an office line fills up, calls crawl and lose packets, and each one still gets attempted. These connections fail on the worker at the moment of connecting, and short calls to one API leave the link with room.

</details>

<details><summary>B. Source ports, tied up by connections it closed</summary>

✓ Right. The report worker has no source port left for that address and port, all of them held by connections it has already closed, and each new call fails at connect.

One client address talking to one address and port can tell connections apart only by source port, and **the side that closes first keeps each port in TIME_WAIT** for about twice the longest packet TTL, which the standard puts at 2 minutes, so 4 minutes. If the range holds 28,232 ports, 2,000 calls a second use them all about 14 seconds in.

Keep a pool of open connections and reuse them, so that 2,000 calls a second share a handful of source ports. Where a call must have its own connection, letting the API close first moves TIME_WAIT off the worker.

</details>

<details><summary>C. Connection slots at the API, full of open sockets</summary>

✗ All 2,000 connections a second land on the API, each with a socket of its own. This worker closes each connection as soon as its reply is in, so the API holds only the calls in flight. What outlasts those calls sits on the worker, the side that closed first.

</details>

<details><summary>D. File descriptors, held by sockets it closed</summary>

✗ "Too many open files" is the usual failure of a busy socket program. A socket gives its descriptor, the handle the process holds for it, back at close. If each call lasts 0.05 s, about 100 descriptors are in use at any moment, far from any limit, while the closed connections go on holding something else.

</details>

---

###### 9. Seeing connect times cluster near one and three seconds

At peak, the clients of an API record how long their TCP connects take. No balancer sits in the path, and clients see no refused connections.

```text
connect time   share of connects
under 10 ms    most
near 1 s       some
near 3 s       some
```

What happened to the connections that took a second?

<details><summary>A. The server answered each one a second late, busy with handlers</summary>

✗ At peak the server is busy, and its handlers most of all. The kernel answers a connection request, with no handler involved, and work that slows down under load would smear the delays across a range, not gather them near 1 s and 3 s.

</details>

<details><summary>B. The client paced itself to a fixed rate of new connections</summary>

✗ Connection pools do throttle themselves. A pacing limit makes each wait depend on how many connections are ahead, and that gives a spread set by the pool's rate. These waits sit at the same marks, 1 s and 3 s, and nothing about pacing puts them there.

</details>

<details><summary>C. Their first packet got no answer and was sent again</summary>

✓ Right. Each of those connection requests got no reply, and the client sent it again a second later and found room. Callers at peak lose that second before any handler has seen them.

A kernel holds arriving connections in a listen queue until the server accepts them. When the server falls behind, **a full queue can drop a request with no reply**. The client's retransmission timer starts at one second and doubles, so a request lost again goes out at 3 s, then 7 s, from the first send, whatever the server's speed. Some systems can be set to refuse on overflow instead, and clients would then see errors in place of delays.

Make the burst fit: accept connections faster and raise the queue's limit, and add capacity if the peak outruns both.

</details>

<details><summary>D. The server refused them, and the client quietly tried once more</summary>

✗ A closed port refuses at once, and TCP does plenty of retransmitting on its own. A refusal is not among the things it retransmits. It reaches the calling code as an error straight away, and these clients logged none, so no connection was refused.

</details>

---

###### 10. Sending two small writes and then a read

You write the client for an in-house protocol, on a long-lived TCP connection. `read_reply` reads until a whole reply has come in. The server logs 1 ms for each call and the round trip is 1 ms, yet each call takes about 40 ms at the client.

```python
def call(sock, header, body):
    sock.send(header)
    sock.send(body)
    return read_reply(sock)
```

What is the second write waiting for?

<details><summary>A. The server's handler, reading the first part</summary>

✗ The header goes out first, and a handler on the server reads what arrives. This one logs 1 ms, and it cannot start until the body has arrived, so nothing on the client is waiting for it. The handler is the one waiting, for a body that has not left the client.

</details>

<details><summary>B. A full packet's worth of data to send</summary>

✗ TCP does gather small writes into fuller packets, and this body is smaller than a packet. A full packet is one of two ways out. The other is an acknowledgment for the earlier data, and once it comes the body leaves however small it is.

</details>

<details><summary>C. The operating system's next scheduling tick</summary>

✗ A delay of a round 40 ms on every call comes from some timer, and schedulers run on timers. Put the header and body into one write, and the same hosts with the same scheduler finish the call in about 2 ms.

</details>

<details><summary>D. An acknowledgment the receiver is holding back</summary>

✓ Right. The body sits in the client's stack until the header is acknowledged, and the server's stack is delaying that acknowledgment. A call that needs about 2 ms of work and travel takes about 40.

**A sender holds a small write while earlier data is unacknowledged, and a receiver delays an acknowledgment** in the hope of sending it along with a reply. Here each waits for the other: the handler cannot reply without the body, and the body cannot leave without the acknowledgment. Only the receiver's timer ends it, and that timer is where the 40 ms comes from.

Put the header and body together and send them in one write, which brings the call to about 2 ms. Turning off Nagle's algorithm, the rule that holds the small write, does the same, and it sends more small packets for every other write on that connection.

</details>

---

###### 11. Losing one percent of packets on a long path

An archive replicates to a second site over one long-running connection, on a 1 Gbps path with 100 ms round trips and receive window to spare. Its stack uses classic loss-based congestion control. One morning a failing line card in one of the path's routers starts to lose 1 percent of the packets, and the link's dashboard shows 99 percent delivered.

What happens to the connection's transfer rate?

<details><summary>A. It falls to a small share of the link</summary>

✓ Right. The rate falls to about a seventh of one percent of the 1 Gbps link. At that speed the archive's copy at the second site falls far behind.

**A loss-based sender cuts its window on each loss** and regrows it by about one packet per round trip. At 1 percent loss the next cut comes long before the window has grown back. Say each packet carries 1,460 bytes: filling this path takes 8,562 packets on the way, the window settles near 13, and 13 packets per 100 ms is about 1.5 Mbps. The 1997 paper on the macroscopic behaviour of this algorithm gives an upper bound of the same size.

Replace the line card, and when a long transfer slows down, look at the loss rate rather than the share delivered. Where a lossy path cannot be fixed, use a model-based congestion control, which is designed to hold its rate at a loss of this size.

</details>

<details><summary>B. It drops by about one percent</summary>

✗ The dashboard says 99 percent delivered, and 1 percent is a small figure to lose. The lost packet is the small part of the cost. Each loss cuts the window, and getting it back takes many round trips of 100 ms at one packet each, so the rate lands far under 99 percent of the link.

</details>

<details><summary>C. It holds, the lost packets retransmitted along the way</summary>

✗ TCP retransmits what is lost and delivers every byte, which is what reliable means. The retransmission repairs the data and leaves the window cut. With a cut this often, the sender never gets back to filling the path, and the archive's data arrives complete and very slowly.

</details>

<details><summary>D. It falls to nothing, the connection dropped for loss</summary>

✗ Calls drop on flaky networks, and a failing line card makes a flaky network. TCP gives up on a connection only after one packet has been retransmitted many times in a row without success. At 1 percent loss a retransmission nearly always gets through, so this connection stays up.

</details>

---

###### 12. Sending large responses over a new tunnel

A logistics firm moves its internal traffic through a new tunnel, and a firewall along the way drops all ICMP error messages, the ones routers use to report trouble, as policy. Its servers mark every packet do-not-fragment, which forbids a router to split one. Connections open, pings pass and short JSON replies arrive. Then a client asks for a 2 MB export, and the reply sits there, with the connection open and the client still waiting.

What happens to the full-size packets of the export reply?

<details><summary>A. The server learns the path's size and sends smaller ones</summary>

✗ Stacks find a path's packet size on their own, and a new tunnel is what that discovery is for. The discovery rests on a report from the router at the tunnel's mouth. That report is an ICMP message, the firewall drops it, and the server goes on sending the old size.

</details>

<details><summary>B. They arrive, and the client's buffer holds the reply back</summary>

✗ A 2 MB export can fill a client's receive buffer, and a full buffer stalls the sender. This client is reading and has an empty buffer. None of the export's full-size packets has reached it, so it has nothing to hold back.

</details>

<details><summary>C. They are dropped on the way, and the server is told nothing</summary>

✓ Right. Each full-size packet is dropped at the tunnel, and the report that would say so is one of the messages the firewall drops. The export never arrives, and the client goes on waiting on an open connection.

The path MTU is the largest packet that every link on a path can carry, and **a packet above the path MTU that is marked do-not-fragment cannot cross**. Normally a router reports this by ICMP and the sender lowers its size. Without the report, the server sees only a missing acknowledgment and retransmits the same size. Pings and short replies stay under the limit, which is why they pass. The sender gives up with a timeout in the end, once its retries run out.

Let the "packet too big" messages through the firewall, or turn on path MTU probing at the servers, which finds a size that fits after a few expired timers.

</details>

<details><summary>D. They pass slowly, paced by a limit on the tunnel</summary>

✗ Paid links do come with rate limits, and the new tunnel is what changed. A limit would let some of the data through. The path is not slow: short replies arrive at full speed, full-size packets never arrive, and the export stays at zero bytes.

</details>

---

###### 13. Sending the first query after a quiet hour behind a firewall

A booking service holds open connections to its database in a pool, through a firewall that forgets a connection after 30 minutes idle, tells neither end and drops any later packet of that connection. You turned TCP keep-alive on, which sends small probes over an idle connection, and left its first probe at the usual two hours, and the query's read has no timeout. After the lunch rush a connection sits idle for an hour, and then a customer books a table.

What does your code see on that first query?

<details><summary>A. The write fails at once with a closed-connection error</summary>

✗ A write to a server that has gone away often fails on the spot, as you have seen against a stopped process. That failure needs a packet coming back to say so. No packet has come here: the firewall told neither end, so the write goes into the local buffer like any other.

</details>

<details><summary>B. The write succeeds, then a long wait for the reply</summary>

✓ Right. The write returns success, and then your code waits for a reply that is not coming. The customer booking a table hangs on a connection the firewall forgot half an hour after the rush.

The first keep-alive probe is two hours away, so after one idle hour none has gone out and neither end knows anything has changed. **A write only puts bytes in the local send buffer**, and it succeeds whether a path exists or not. TCP then retransmits those bytes into the firewall until it gives up, which the standard puts at no less than 100 seconds and one kernel's default at about 15 minutes.

Put a timeout on the read, so that the query fails after, say, 5 s, and have the pool check a connection when it lends one out. A first probe at 10 minutes also works here, since the probes keep the firewall's entry alive.

</details>

<details><summary>C. The usual answer, as both ends are still up</summary>

✗ Neither the service nor the database has closed anything, and each still lists the connection as open. The firewall between them has forgotten it and drops the query's packets, so no reply comes back.

</details>

<details><summary>D. A fresh connection, opened after keep-alive found the dead one</summary>

✗ The setting is called keep-alive, and you turned it on to catch this very thing. It has sent nothing yet: the first probe is due at two hours and the query comes at one. The pool has heard nothing either, so it lends out the dead connection as a good one.

</details>

---

###### 14. Rejecting an upload and closing the socket

A photo-sharing API reads the headers of a 200 MB upload, finds the account over its quota, writes a 413 reply (body too large) with a clear message and closes the socket in one call. The client's code writes the whole body first and reads only after that, and it is still writing. Both ends run ordinary TCP stacks.

What does the client's code get from that upload?

<details><summary>A. A connection reset, and no 413 to read</summary>

✓ Right. The client's write fails with a connection reset soon after the close. This code reads only after writing, so it never reaches the 413, and the customer gets a network error in place of the clear message about the quota.

**A full close ends both directions**, so the chunks still arriving have nowhere to go, and the server's stack answers them with a reset. On the client, a reset makes the next write fail and, by the standard, flushes what the stack had queued, so a read after that write may find nothing.

Close only the sending direction after the 413, a half-close, and keep reading and discarding until the client closes. The client then writes its 200 MB and reads the 413. On the client, read while you write, so that the reply can stop the upload early.

</details>

<details><summary>B. The 413, after the server has read and dropped the rest</summary>

✗ Some HTTP servers read and discard the rest of a body before they close, for exactly this case. This one closed in one call straight after writing, so no code is left to read the rest of the 200 MB.

</details>

<details><summary>C. The 413 at once, its write stopped by the reply</summary>

✗ The 413 was written before the close, and TCP delivers what it is given. The bytes reach the client's stack, and they stop nothing there: the code is inside a write and does not look at incoming data until the body is done. The next thing it hears about is the reset.

</details>

<details><summary>D. A write that hangs, nobody reading at the far end</summary>

✗ A peer that stops reading leaves its receive window at zero, and a writer blocks on that for as long as the peer stays open. This peer closed the socket, and its stack answers arriving data with a reset instead of a closed receive window, so the write returns an error.

</details>

---

###### 15. Losing one packet under many streams

A mobile banking app loads ten resources at once over one HTTP/2 connection on a poor network, with pieces of all ten mixed together on the wire. One TCP packet early in the burst, holding a piece of the first resource, is lost, and the packets after it arrive.

What do the other nine responses do while that packet is retransmitted?

<details><summary>A. They carry on, each on its own stream</summary>

✗ Streams are what HTTP/2 added to stop responses queuing behind each other, and each one has its own id. Those ids live above TCP, which sees one run of bytes. The nine other responses' bytes sit behind the gap in that run, and the app gets none of them.

</details>

<details><summary>B. They start again from their first byte</summary>

✗ A download that fails in a browser often starts again from zero. TCP keeps what it has received and retransmits the one missing packet, so no response is asked for again. Once that packet lands, the bytes that were held back go up to the app all together.

</details>

<details><summary>C. They are cut off, and the connection reopened</summary>

✗ A poor network does drop connections, and the app is on one. One lost packet is routine for TCP: the sender retransmits it and the connection stays up. Nothing is reopened, and the nine responses are still on their way.

</details>

<details><summary>D. They wait, though their bytes have arrived</summary>

✓ Right. The later packets reach the phone and stay in its TCP stack until the lost packet is retransmitted. In the banking app all ten resources stall, nine of them on a piece of the first.

**TCP hands bytes over strictly in order and knows nothing of streams**, which to it are all one run of bytes. This is head-of-line blocking: everything behind a gap waits, and HTTP/2 leaves it in place.

On a lossy path, move the traffic to QUIC, so that a lost packet holds back only its own stream. That stream still waits for its retransmission.

</details>

---

###### 16. Using early data on a checkout request

A food-delivery API accepts TLS 1.3 early data on every route, to save repeat customers a round trip. A customer taps to order, and the app's stack sends this request in its first flight. The network falters before any reply reaches the phone, and the stack sends the request again on a new connection.

```http
POST /orders HTTP/1.1
Host: api.delivery.example
Content-Type: application/json

{"dish": "pad thai"}
```

What can the order handler receive for that one tap?

<details><summary>A. One order, the repeat stopped by TLS</summary>

✗ TLS rejects a repeated record inside a connection, and a server can allow each session ticket only once. Those checks stop at the connection and the ticket. The resend comes on a new connection as a new and valid request, and TLS passes it up to the handler like any other.

</details>

<details><summary>B. Nothing, orders held for a finished handshake</summary>

✗ Handshake diagrams number their steps, and application data comes last. Early data is the exception: the server takes the POST from the first flight with the handshake still unfinished. This API passes it straight to the handler.

</details>

<details><summary>C. The same order twice, once from each try</summary>

✓ Right. A customer who tapped once can get two meals: the handler can receive the order from the first flight and again from the resend.

Early data travels before the handshake has finished. **TLS gives no guarantee against a second delivery across connections**, and the resend comes on a new connection, with nothing for TLS to match it against. If the server processed the first flight and only the reply was lost, the handler gets two orders. If the first flight never arrived, it gets one.

Keep early data to requests that are safe to repeat: answer a POST that arrives in early data with 425 (Too Early), and the app sends it again after the handshake. Or have the app put an idempotency key on each order, an id that lets the handler spot a second arrival, which also covers retries that have nothing to do with TLS.

</details>

<details><summary>D. A refusal, no POST being allowed as early data</summary>

✗ The standard for HTTP over early data tells clients to keep unsafe methods such as POST out of it. That rule binds a client that follows it, and this one has sent the POST. Refusing takes a rule on the server, and this API has none, so the request goes through.

</details>

---

###### 17. Installing a certificate without its intermediate

You renew a partner API's certificate and install only the new leaf file. The API opens fine in your laptop's browser. Next the partner's backend connects, from a locked-down network where it can reach your API and nothing else, using a plain TLS library and its list of roots.

What does the partner's backend get on its next connection?

<details><summary>A. A clean handshake, as in the browser</summary>

✗ Your laptop's browser opened the API with this same leaf installed. The partner's library has your leaf and its roots, no root signed your leaf, and the chain between them is missing.

</details>

<details><summary>B. A failed handshake, and no call gets through</summary>

✓ Right. The handshake fails, since the library cannot link your leaf to a root, and the partner's calls stop.

A client has to build a path from the leaf up to a root in its trust store, and **the server has to present the part of the certificate chain the client lacks**. A leaf on its own leaves a gap at the intermediate. A client that kept that intermediate from another site, or that fetches it from the issuer's address written in the leaf, can close the gap by itself, so a check that passes on such a client shows only that this client could. The partner's library builds from what it is given and its roots, with no other source it can reach.

Install the leaf and the intermediate together, and check each renewal with a strict client such as the partner's library, not only with a browser.

</details>

<details><summary>C. A clean handshake, with the leaf trusted on its issuer's name</summary>

✗ The leaf names the intermediate that issued it, and the partner's list holds roots only. The signature on your leaf was made with the intermediate's key, and neither your server's reply nor that list supplies it, so the library has nothing to check the signature with.

</details>

<details><summary>D. A warning in its log, and the request carrying on</summary>

✗ A browser meets a certificate problem with a warning page and a button to carry on. A backend has nobody to press it. The library treats a chain it cannot complete as a failed check and ends the handshake, so the request is never sent.

</details>

---

###### 18. A certificate that expires at midnight

At a delivery tracker, the server read its certificate file when it started last month and has been running undisturbed since. A renewal job wrote the new file last week and reported success. The old certificate runs out at midnight. Drivers' apps hold long-lived connections open from the evening shift, and through the night more apps connect, new ones with no session ticket to resume with.

What happens at 00:01 to the open connections and to new handshakes?

<details><summary>A. Open connections drop, and new handshakes fail</summary>

✗ Status pages describe an expiry as a site going down all at once. Midnight sends nothing down an open connection. The certificate was checked at that connection's handshake, a running connection does not check the date again, and the evening's drivers go on with the keys they agreed.

</details>

<details><summary>B. Open connections carry on, and new handshakes pass on the renewed file</summary>

✗ The job reported success, and the new file is on the disk. A file on the disk is not what this server sends: it read its certificate last month and still presents that one from memory. Each new app gets a certificate past its date and ends the handshake.

</details>

<details><summary>C. Open connections drop, and new handshakes pass on the renewed file</summary>

✗ A login token stops working at its expiry, and the next login gets a fresh one. A certificate's date is applied during a handshake, not along a running connection, so the drivers on shift carry on. The new apps meet what this server loaded at start, and that is the old certificate.

</details>

<details><summary>D. Open connections carry on, and new handshakes fail</summary>

✓ Right. Drivers on connections from the evening shift carry on, and an app that connects at 00:01 fails its handshake on the certificate the server read last month.

**A server presents the certificate it has loaded, whatever the file now holds**, and a client checks it during the handshake. After the handshake a TLS 1.3 connection never sees the server's certificate again, and an older one sees it again only if it runs a new handshake inside the connection, so an open connection need not drop at the date. A client holding a usable session ticket can resume with no certificate sent, which is why the new apps, with none, are the ones that fail.

Make the renewal job reload the server, and check what the server presents at its port rather than what the job reports.

</details>

---

## Further reading

- [The Macroscopic Behavior of the TCP Congestion Avoidance Algorithm](https://doi.org/10.1145/263932.264023), 1997: a simple bound on the rate a loss-based sender reaches at a given loss and round trip, and the cases it does not cover.
- [Cache Me If You Can](https://doi.org/10.1145/3355369.3355568), 2019: what longer and shorter DNS TTLs do to latency, measured on real zones.
- [An Argument for Increasing TCP's Initial Congestion Window](https://doi.org/10.1145/1823844.1823848), 2010: the case for letting a new connection start with a window of ten segments.
- [The QUIC Transport Protocol](https://doi.org/10.1145/3098822.3098842), 2017: the design of QUIC and what running it at internet scale showed.

Prev: [Performance and capacity](../performance-and-capacity/README.md) · Next: [HTTP caching and CDNs](../http-caching-and-cdns/README.md)
