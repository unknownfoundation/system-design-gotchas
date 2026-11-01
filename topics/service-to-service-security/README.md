# Service-to-service security

Prev: [Microservices](../microservices/README.md) · Next: [Containers and orchestration](../containers-and-orchestration/README.md) · Reviewed October 2026

Service-to-service security is how a call between two of your own services settles three things: who is calling, whether that caller may do this, and on whose behalf it acts. Addresses, certificates, tokens, keys and rules each answer part of that, at a gateway, in a proxy or inside a service.

- **Service identity**: the name a caller proves, such as `orders`, given to a workload rather than to a machine.
- **Orchestrator**: the system that runs copies of services across machines, starting, stopping and moving them as it needs to.
- **Certificate authority**: the issuer of certificates. A firm's internal one signs its services' certificates.
- **Certificate chain**: a service's own certificate, then any intermediates, each signed by the next, up to a root.
- **Trust store**: the list of root certificates one side accepts.
- **Mutual TLS**: TLS in which the client presents a certificate as well as the server, and each side verifies the other's. In an accept-both mode, often called permissive, used while callers move over, a service also takes plain connections.
- **TLS termination**: ending TLS at a proxy, which then opens its own connection to the backend.
- **Access rule**: a rule in a policy engine, such as a service mesh or a cluster's network layer, that says which callers may reach a workload, by name or by address.
- **Token**: a set of claims signed by an issuer: who it is about (`sub`), what it allows (`scope`) and when it ends (`exp`). The issuer publishes a key set, and each token's `kid` names the key that signed it.
- **Audience**: the `aud` claim, which names the service or services a token was issued for.
- **Bearer token**: a token that works for whoever holds it, with nothing else to prove.
- **Gateway**: the single entry point that outside requests pass through on the way to the services.
- **Confused deputy**: a service with wide access that uses it on a request from someone who has less.
- **Secret store**: a service that holds secrets for others.

Mutual TLS runs either in a sidecar, a proxy beside each service, or in the service's own TLS library. Tokens come in two kinds. A self-contained token is checked with the issuer's public key. The other kind is checked by asking the issuer whether the token is still active, often called introspection.

A service can learn who is calling in six ways:

| Way | What the service gets | When it is checked |
| --- | --- | --- |
| Source address | the address the connection comes from | on each connection, by a network rule or the service |
| Mutual TLS | the caller's certificate, with its name | in the handshake, as the connection opens |
| Token | signed claims in a header | on each request, by the service that receives it |
| Shared key | a secret in a header | on each request, against the service's own copy |
| Signed request | a signature over parts of the request | on each request, recomputed by the receiver |
| Header from a gateway | a field such as `X-User-Id` | at the gateway |

A client that turns off verification to get past a certificate error still encrypts, and it no longer knows who it reached. A verifier fixes in advance the algorithms it accepts and where keys come from, and takes neither from the token. A check that runs only when a token is present passes a call with none. A secret deleted from a repository or a store after a leak goes on working until the side that checks it stops accepting it.

It bites at rollouts and rotations, on the second hop behind a proxy, and wherever a service acts for someone else.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- say what a certificate, an address or a second hop tells a service about its caller, before you turn on mutual TLS, write an allow list or attach a packet capture to a ticket
- follow a caller's rights through forwarded calls, an export, a queue and a new proxy, and work out what each stop still allows
- place each check at the gateway, in the service or in a policy engine, and predict what a first rule or a silent policy service does to traffic
- plan the change of a shared key, a signing key or a root across a fleet that keeps running, and say how long that fleet can run with its certificate authority down

###### 1. Turning on mutual TLS between all the services

A food delivery company turns on mutual TLS between all its services: every copy now gets a certificate from the internal certificate authority, and the services' code, which has always served whatever reached it, stays as it was. A week later an analyst building a refund report points a new job, which has its certificate like everything else, at `POST /refunds` on the payment service, having meant to read `GET /refunds`.

What happens to the job's call?

<details><summary>A. The handshake fails, and the job gets no connection</summary>

✗ Diagrams draw mutual TLS as a lock between two named services, and these two were never drawn together. The handshake asks only whether the job's certificate chains to a root in the trust store. It does, so the connection opens and the call goes in.

</details>

<details><summary>B. The payment service answers 403</summary>

✗ A 403 is how a service says no, and courses teach authentication as the first step on the way to permission. Mutual TLS delivers the job's name and decides nothing with it, and the payment service's code never compares that name with anything.

</details>

<details><summary>C. The refund goes out</summary>

✓ Right. The handshake passes and the payment service runs `POST /refunds`, and a customer gets money back that nobody meant to return.

Mutual TLS proves that the caller holds the key for a certificate from an authority you trust, and it gives the caller a name. **What a caller may do is a separate decision**, made by an access rule or a check in the service, and the rollout added neither. Inside one firm every job has a certificate, so the certificate alone draws no line between a report job and the payment service.

Add access rules that name each caller and the routes it may use, starting with the routes that move money. In a rollout plan, count mutual TLS as the step that makes those rules possible.

</details>

<details><summary>D. The call returns 200, with the refund held for a person to approve</summary>

✗ Refund screens in support tools ask a second person to approve, and that step is what most people have seen around refunds. It lives in the support tool, and this job calls the route directly, so `POST /refunds` issues the refund with no queue in between.

</details>

---

###### 2. Allowing the order service by its addresses

A ticketing company runs everything on one orchestrator. The billing service accepts calls only from an allow list: the six addresses the order service's copies held at its last redeploy, a month ago. Analytics jobs start and stop on the same cluster all day. This morning the order service redeploys, and its copies come up on new addresses.

Whose calls does billing accept after the redeploy?

<details><summary>A. The six order copies, at their new addresses</summary>

✗ Service discovery keeps a name pointed at its copies wherever they run, and the list was written for the order service. It holds six addresses and no name, so an order copy at a new address is refused like any stranger.

</details>

<details><summary>B. Jobs on the six freed addresses, and no order copy</summary>

✓ Right. Billing refuses the redeployed order copies and accepts whichever workloads now hold the six addresses. Orders stop reaching billing, and an analytics job that lands on a freed address gets in.

On an orchestrator, **an address belongs to whichever workload holds it at that moment**. Each new copy takes one from a shared range, and one that an old copy gives up goes back for the next workload.

Name callers by service identity, which stays with the workload wherever it runs, and write the rule against that name. Keep address rules for what they do well: a coarse fence between networks.

</details>

<details><summary>C. Nobody's, with all six addresses gone for good</summary>

✗ A fixed server kept its address for years, and a retired one stayed unused. On an orchestrator an address returns to the shared range when its copy stops, and the list lets in the next workload to get one of the six.

</details>

<details><summary>D. Any workload on the cluster, analytics jobs included</summary>

✗ Everything on the cluster is inside the private network, the usual picture of who is trusted. Billing's list is narrower than that: a job on any address outside the six is refused, and only one that happens to hold one of them gets in.

</details>

---

###### 3. Capturing packets on a backend to chase a slow upload

An insurer's mobile app sends every request to its API over HTTPS, with the customer's token in the `Authorization` header. A balancer ends TLS, and its config forwards to the backends across the private network with `scheme: http` and addresses like `10.0.3.7:8080`. Chasing a slow claim upload, an engineer runs a packet capture on one backend and attaches the file to the ticket, which the whole support team can read.

What does the file hold of the customers' requests?

<details><summary>A. Encrypted bytes, with only addresses and ports readable</summary>

✗ The app sends everything over HTTPS, and for the trip from phone to balancer that holds. The capture was taken past the balancer, where each request arrives as the balancer forwarded it, on an HTTP port with no encryption left.

</details>

<details><summary>B. Readable paths and headers, with the bodies encrypted</summary>

✗ A router on the path reads addresses and ports and leaves the contents alone, and HTTP headers are routing data of a kind. HTTP on port 8080 seals nothing: the headers and the claim details travel as one readable stream.

</details>

<details><summary>C. Readable requests, with the balancer having removed each token</summary>

✗ Authentication is among the front's jobs, and a token has done its work once it has been checked. The app sends its token in the `Authorization` header, and the balancer forwards it with the rest, so the tokens are in the file.

</details>

<details><summary>D. Whole requests in readable text, tokens included</summary>

✓ Right. The file holds whole requests in readable text, with customers' tokens and the contents of their claim uploads, and the support team can now read them on a ticket.

**TLS protects the hop it runs on**, here from the app to the balancer. The balancer decrypts each request to forward it, and port 8080 carries the result as plain HTTP. A private network limits who can reach that hop and does nothing about who can read a capture made on a machine inside it. A token in that file works until it expires.

Encrypt the hop behind the balancer, so a capture shows sealed bytes. Treat any capture from inside as customer data, and strip tokens and bodies before it leaves the machine.

</details>

---

###### 4. Checking that a peer's certificate chains to your own authority

A bank's transfer service writes to its ledger service over mutual TLS. Every service, and every staging copy, gets its certificate from the one internal certificate authority. `discovery.pick` returns the address of a live copy of a service, and `TLSContext` is the bank's thin wrapper around its TLS library. The client connects like this, and a test against a server with an expired certificate fails as it should.

```python
def open_ledger():
    address = discovery.pick(config["ledger_service"])
    tls = TLSContext(cert="transfer.pem", key="transfer.key", verify="required")
    tls.load_trust_store("internal-ca.pem")
    return tls.wrap(socket.connect(address))
```

On Friday a config change meant for a test environment points the transfer service at `ledger-staging`.

What happens to the next transfer's ledger write?

<details><summary>A. The client ends the handshake over the name on the certificate</summary>

✗ A browser shows a full-page warning for a wrong name, and that page is where most people learned what a certificate check does. A browser knows the name you typed. `tls.wrap` gets a socket with an address behind it and no name at all, so `ledger-staging` is compared with nothing and the handshake completes.

</details>

<details><summary>B. The write lands on staging and returns success</summary>

✓ Right. The write lands on `ledger-staging` and returns success. The transfer is recorded in a ledger nobody reads, and the real ledger never hears of it.

Verifying a certificate is two checks: **its chain leads to a root in the trust store, and its name is the one you meant to reach**. `verify="required"` runs the first, and staging's certificate comes from the same internal authority, so it passes. The second needs a name to compare, and this code passes none. The expired certificate test passed because it exercised the first check alone. TLS libraries differ here: some refuse to set up a client until you give a name or switch verification off, and others do the chain check and stop unless told more.

Pass the expected name, `ledger`, to the TLS layer even when you connect by address, and add a test against a server with a good chain and the wrong name. Separate roots for staging and production stop this mix-up a second way.

</details>

<details><summary>C. Staging ends the handshake over the production certificate</summary>

✗ Staging and production have separate names, separate configs and often separate teams. They share one authority here, and staging checks the client's certificate against that authority's root, which is the check `transfer.pem` passes everywhere.

</details>

<details><summary>D. The write succeeds, with a name warning in the client's log</summary>

✗ Compilers and linters turn a doubtful check into a warning and carry on. With no name passed in, the name check never runs, so it has nothing to warn about and the log shows an ordinary connection.

</details>

---

###### 5. Reading the partner's name off the connection

A logistics firm bills its partners per API call. Each partner connects with its own client certificate, and the backend reads the name straight off the connection. This quarter a balancer that reads HTTP moves in front of the backends: it ends TLS, checks the partner's certificate against the firm's list and re-encrypts to the backend, with mutual TLS on both legs. The code stays as it was.

```python
def handle(request):
    partner = request.connection.peer_certificate().subject
    db.execute("INSERT INTO usage (partner, path) VALUES (?, ?)", partner, request.path)
    return route(request)
```

What does the `usage` table hold after a day of partner traffic?

<details><summary>A. Each partner's name, as before</summary>

✗ Mutual TLS is on for every leg, and each partner still presents its certificate as before. Each leg has its own peer, and on the second one that peer is the balancer, so the partners' names never reach this line of code.

</details>

<details><summary>B. An empty partner on every row</summary>

✗ Once TLS ends, the partner's certificate is gone, and on a plain second leg `peer_certificate()` would indeed have nothing to return. This second leg runs mutual TLS, so the backend does hold a certificate, and it is the one the balancer presented.

</details>

<details><summary>C. The balancer's name on every row</summary>

✓ Right. Every row names the balancer: all the partners' calls land on one account, and the firm cannot split this month's bill between them.

Each leg here is a handshake of its own. The partner presents its certificate to the balancer and it stops there, because **a connection knows only the peer at its other end**. On the backend's connection that peer is the balancer, and `peer_certificate()` returns what the balancer presented.

Have the balancer send the checked name in a header it overwrites on every request, and have the backend trust that header only on connections whose peer is the balancer.

</details>

<details><summary>D. The partner's and the balancer's names joined on every row</summary>

✗ `X-Forwarded-For`, the header that lists a request's addresses, grows by one address at each hop, and proxies write that list on purpose. A handshake shows one peer, and here its subject is the balancer's alone.

</details>

---

###### 6. Writing a rule by name while plain connections are still accepted

A courier firm is moving to mutual TLS one team at a time. While the teams move, its dispatch service accepts both plain and mutual TLS connections, and its policy engine runs every rule on every call. Routing, whose move is booked for next week, still connects in plain. On Monday a security review gets what it asked for: an access rule on dispatch that allows only `routing` and `tracking` by the service name in their certificates.

What happens to routing's next call to dispatch?

<details><summary>A. Dispatch refuses it</summary>

✓ Right. Dispatch refuses the call, and routing, a service the rule names, loses its access a week before its move.

A rule by name reads the name in the caller's certificate, and **a plain connection has no name to match**. An allow rule lets in what matches, so routing is refused. The accept-both mode decides which connections open, and the rule then decides which calls go ahead. Engines differ here: some run their rules on plain calls, as this one does, and others skip the rules for plain connections, so the rule stops nobody.

Move the callers first, watching the count of plain connections for each one, then write the rules. Once the last caller has moved, turn off the accept-both mode, so no call can arrive without a name.

</details>

<details><summary>B. Dispatch serves it, as a caller the rule names</summary>

✗ The rule has `routing` on it by name, and routing has called dispatch for as long as anyone remembers. The rule reads that name from a certificate, a plain connection has none to read, and the call matches neither entry.

</details>

<details><summary>C. Dispatch serves it under routing's name, read from its address</summary>

✗ The platform's registry knows which workload holds which address, so the name is known to something on the cluster. This rule reads the name in the certificate, and routing has presented none, so the registry's answer never comes into it.

</details>

<details><summary>D. Dispatch ends the connection at the handshake</summary>

✗ Once a service takes only mutual TLS, a plain client gets no further than that. Dispatch is still accepting both, so routing's connection opens as it did last week, and the refusal comes later, from the rule, on the call itself.

</details>

---

###### 7. Allowing the private range on a settings service

The settings service at a booking firm has no route from outside, and its admin endpoint accepts any call from the private range. The logo service, which runs in that range, downloads each partner's logo from the address the partner enters in its profile. An audit finds that the rule cannot tell a logo fetch from any other call in the range, and the team has to close that finding this quarter.

Which change closes the audit's finding?

<details><summary>A. A tighter firewall on the firm's outside edge</summary>

✗ An audit finding that involves partners points at the outside, and the edge firewall is the control every review asks about first. The settings service already has no route from outside. A logo fetch starts at the logo service, inside that firewall, so no rule at the edge ever sees it.

</details>

<details><summary>B. A narrower range on the endpoint that still holds the logo service</summary>

✗ A range that wide is the first thing a reviewer circles, and narrowing it is a small, clean change. A rule on addresses goes on asking where a call starts, and a range that still holds the logo service accepts its fetches as it did before.

</details>

<details><summary>C. A filter in the logo service on the text of each address</summary>

✗ A check in the logo service's own code ships this week, and it can list the settings service's name. The text of an address does not settle where it leads, since one endpoint can go by several names and numbers, so a filter on text leaves the rule on the range as open as it was.

</details>

<details><summary>D. An access rule by service identity that leaves the logo service off</summary>

✓ Right. With a rule by service identity that leaves the logo service off, the admin endpoint refuses that service's calls whatever address it was given, and partners' logos go on arriving.

An address rule sees **where a call starts and not who is calling**. That trust by network location holds only while each workload in the range decides its own calls, and the logo service fetches the address a partner fills in. A rule by identity checks which name is on the caller's certificate, and that name stays the same whatever address the fetch was pointed at.

Require a certificate on the admin endpoint, and list only the services that administer settings. Send the logo service's fetches through an outbound path that refuses private addresses after the name resolves, since it can reach more than this one endpoint.

</details>

---

###### 8. Sending a notification token to the reports service

One issuer at a media company signs every service's tokens with one key, and each token names the service it was issued for in its `aud` claim. `tokens.decode` is the company's thin helper around a token library: it returns the claims once the signature and expiry pass. The reports service handles a call like this.

```python
def handle_report(request):
    claims = tokens.decode(request.bearer_token, ISSUER_PUBLIC_KEY)
    audit.record(claims["sub"], "reports.latest")
    return reports.latest()
```

A notification job holds a token issued for the notification service, and a wrong base URL in its config sends its next call to reports.

What does the reports service do with that call?

<details><summary>A. Answers 401, over the audience in the token</summary>

✗ Token libraries do refuse on audience once they are told which one to expect, and most people meet the check through a framework that sets it up for them. This helper is given no audience, and the handler never reads `aud`, so the two values are never compared.

</details>

<details><summary>B. Returns the report</summary>

✓ Right. The reports service returns the latest report to a job that was never meant to have it, and the audit line records the job like any other caller.

**A signature proves who issued a token and that nobody changed it, not who the token is for**: with one issuer and one key, a notification token is signed exactly as a reports token is. The audience is what ties a token to a service, and the token standards require that check wherever one issuer serves several recipients. Libraries differ in what they do with an `aud` when no expected value is passed, and this helper checks none.

Pass `"reports"` as the expected audience on every decode, and add a test in which a token for another service is refused.

</details>

<details><summary>C. Answers 403, over the job's missing report rights</summary>

✗ The status tables teach that a valid token without the rights gets a 403, and the job has no business with reports. A 403 takes a check of rights, and this handler goes from `decode` to `reports.latest()` with nothing between them.

</details>

<details><summary>D. Returns only the reports the notification job owns</summary>

✗ A token has a subject, and many APIs return only what the subject owns. `reports.latest()` takes no caller at all: the subject goes to the audit line, and the job gets what anyone would get.

</details>

---

###### 9. Forwarding the customer's token to every service

On a travel site, the front service receives each customer's token and forwards it unchanged on its calls to payment, loyalty and the ad service. Each of them validates it before serving. Tokens last fifteen minutes. An audit finds that the ad service writes request headers to a debug log, which forty analysts can read.

What does a token in that log still allow in its fifteen minutes?

<details><summary>A. Any call the customer could make, payment included</summary>

✓ Right. For the rest of its fifteen minutes, a token in that log allows whatever the customer could do, payment included, and forty analysts can read it.

All three services accept one forwarded token, so they share one audience, and **a bearer token works for whoever presents it**. The copy in the ad service's log opens payment as readily as it opens the ad service. A short lifetime limits how long a copy is good for and leaves alone what it is good for.

Have the front service use token exchange, trading the customer's token for one with a single service as its audience and only the rights that call needs. Stop writing `Authorization` headers to logs at the source.

</details>

<details><summary>B. Calls to the ad service alone</summary>

✗ A session on one site does not open another, and the ad service is where this token turned up. The token is not tied to the ad service: it is the same one payment and loyalty took on the same request, and they would take it again.

</details>

<details><summary>C. Nothing away from the front service's connection</summary>

✗ TLS carried the token here, and its encryption keys belong to that one connection. A plain token is a string that is good apart from any connection, and the front service's forwarding already moves it across connections all day.

</details>

<details><summary>D. Reads across the site, with payment calls refused</summary>

✗ Some login pages ask for a second step before money moves, in front of a person. That step lives on the screen, and the payment service validates this token and serves, with no way to tell the log's copy of the token from the one the front service sent.

</details>

---

###### 10. Picking another region on the export form

At a retail chain, regional managers see only their own region's invoices in the app: its backend refuses another region's page. A new export service makes spreadsheets. Its form asks for a region, and it asks the invoice service for that region's invoices in a call that carries only the region, over mutual TLS, where an access rule lists the export service as allowed to read invoices. One manager, in a hurry, picks the neighbouring region on the form.

What does the spreadsheet she downloads hold?

<details><summary>A. Nothing: the access rule refuses the call</summary>

✗ The app refuses her for that region, so a refusal for this case exists somewhere. It lives in the app's backend, and the export never passes through it. The rule on the invoice service asks about the export service, which it lists.

</details>

<details><summary>B. An empty sheet, after the invoice service filters by her region</summary>

✗ Filtering by region has to live somewhere, and the invoice service holds the invoices. The call it receives carries a region and the export service's name, with nothing about the manager in it, so it has nothing of hers to filter by.

</details>

<details><summary>C. Her own region's invoices, whatever she picked</summary>

✗ She is signed in when she fills in the form. The export service takes the region the form sends and passes it on, and nothing replaces what she picked with what she may see.

</details>

<details><summary>D. The neighbouring region's invoices</summary>

✓ Right. The sheet holds the neighbouring region's invoices, and a manager the app refuses has another region's figures in a download.

The export service is a confused deputy, with wide access that it uses for whatever region the form names. The invoice service sees mutual TLS and a name on its rule, and **a check on the calling service does not cover the person it acts for**. Her region is checked only in the app, and the export never passes through it.

Check her region in the export service before it calls, or have it send a token that names her, so the invoice service can apply the smaller of the two sets of rights.

</details>

---

###### 11. Adding a clerk to billing during a live token

The staff portal at a hospital group signs people in through an issuer, which gives each a 60-minute token listing their roles. Every service checks tokens with the issuer's public key. At 10:00 an administrator adds a clerk to billing, and the directory shows the change at once. At 10:05 the clerk, who signed in at 9:50 and has been told to start right away, opens the invoice screen.

What does the invoice service do with the clerk's request?

<details><summary>A. Serves the invoice screen</summary>

✗ Courses teach roles as a lookup on each check, from user to role to permission, and the directory does hold the new answer. The invoice service never asks it, since it reads the roles from the token, and that token was signed at 9:50 with no billing in it.

</details>

<details><summary>B. Sends the clerk back to the login page</summary>

✗ A server-side session can be reloaded when an administrator changes a user, and the next click then lands on the login page. This token lives in no table to reload: its signature is good and its `exp` is 10:50, and neither has changed, so nothing sends the clerk anywhere.

</details>

<details><summary>C. Answers 403, as it did before the change</summary>

✓ Right. The invoice service answers 403 and the clerk, told to start at once, is refused. A token issued after 10:00 would carry the role, and this one runs to 10:50.

The service reads the role from the token, and the token was signed before the clerk had it. Checking with a public key means no call to the issuer, and the directory's change never reaches the service. **A role in a token is a fact about the moment of sign-in**, in both directions: a clerk moved out of billing at 10:00 would keep this screen until 10:50.

Have the portal sign the clerk in again after a role is added, and set the token's lifetime to what a role change can bear. For actions that matter, ask the issuer whether the token is still active, at the cost of a call on each check.

</details>

<details><summary>D. Asks the issuer about the role, then serves the screen</summary>

✗ A cache that misses goes back to its source, and here the role is missing from the token. The roles a token lists are the whole answer, and this service checks it with the public key, with no call out to the issuer.

</details>

---

###### 12. Signing requests to the ledger through a new proxy

At a card processor, the settlement job signs every call to the ledger with an HTTP message signature, and the ledger recomputes it with the shared key from the request as it arrives. That has worked for a year of direct calls. This week a new proxy that reads HTTP goes in front of the ledger for a switch between two copies of it, and an engineer compares a request as it leaves the job with the same request as it reaches the ledger.

```http
POST /entries HTTP/1.1
Host: ledger.internal
Signature-Input: sig=("@method" "@path" "@authority" "content-digest")
Signature: sig=:qLxv...:

POST /entries HTTP/1.1
Host: ledger-blue.internal
X-Forwarded-Host: ledger.internal
Signature-Input: sig=("@method" "@path" "@authority" "content-digest")
Signature: sig=:qLxv...:
```

What does the ledger do with the job's calls?

<details><summary>A. Refuses each of them on the signature</summary>

✓ Right. The ledger refuses each call. Settlement stops the moment the proxy goes in, though the job, the key and the entries have not changed.

With a signed request, **the signer and the verifier each build the signed text from their own copy of the message**. The job signed `@authority`, which HTTP/1.1 carries in `Host`, as `ledger.internal`, and the ledger builds its text from `ledger-blue.internal`, so the two signatures differ. The ledger cannot tell a proxy's rewrite from any other change to the request, and refusing is the check doing its job.

Have the proxy keep the original `Host`, or have the ledger verify against its own public name from config instead of the header. Before a proxy goes into a signed path, run a signed call through it in staging.

</details>

<details><summary>B. Accepts them, reading the host from `X-Forwarded-Host`</summary>

✗ Backends do read `X-Forwarded-Host` to rebuild the URL a client used, and the right value is sitting there. It is outside what was signed, and any caller can write it. This ledger builds its text from the `Host` it received, so the forwarded value never enters the comparison.

</details>

<details><summary>C. Accepts them, on a body that has not changed</summary>

✗ A signed file is checked over its contents, and this body arrived byte for byte. This signature also covers the method, the path and the authority, which is the `Host`, so a request whose body is intact and whose `Host` has changed still fails the comparison.

</details>

<details><summary>D. Never sees them, with the proxy refusing what it cannot verify</summary>

✗ Gateways carry authentication, and the front of a signed API is an obvious place for a check. This proxy holds no shared key and has no check to run: it rewrites the `Host` and forwards, and the failure happens at the ledger.

</details>

---

###### 13. Queueing a year-end pack with the customer's token

An accounting product builds a year-end pack for each customer in the background. The web service puts a job on a queue with the customer's short-lived access token in it, and a worker later calls the documents service with that token. Jobs usually wait a few seconds. On the last day of the tax year the queue backs up by three hours.

What happens to a job that reaches a worker three hours later?

<details><summary>A. The worker gets a fresh token and builds the pack</summary>

✗ Login sessions in the app renew quietly while a customer works, so an expired token rarely shows. That renewal takes a refresh token, a longer-lived credential for getting new access tokens, and a client that holds it. The job carries only the access token, so the worker has nothing to renew with.

</details>

<details><summary>B. The documents service accepts the token and the pack is built</summary>

✗ The customer was allowed to ask when they clicked, and the job only carries that out later. The documents service has no record of that moment: it compares the token's `exp` with its own clock when the worker calls, three hours on.

</details>

<details><summary>C. The worker builds the pack under its own service identity</summary>

✗ Workers usually have credentials of their own for their housekeeping, and background work often runs under them. This worker sends the token that came in the job, and nothing in the call falls back to anything else when it is refused.

</details>

<details><summary>D. The documents service refuses the token and the job fails</summary>

✓ Right. The documents service refuses the token and the job fails, and the customer's pack is not built.

**Expiry is checked when a token is presented**, here by the worker three hours on, and the time spent queued counts. Say the token lasts fifteen minutes: every job that waits longer fails, though its token was fine when it went on the queue.

Store in the job who asked and what for, and have the worker ask the issuer for a token when it runs, under its own identity and naming that customer, with the issuer deciding whether this worker may act for that customer. Set any lifetime against the longest wait the queue can reach, not the usual one.

</details>

---

###### 14. Mistyping an order number behind a checking gateway

The gateway at an online pharmacy validates each customer's token and checks for the scope `orders:read` before it routes to the order service, which looks an order up by its number. A customer checking on order 58213 mistypes it in the app's search box as 58231, which belongs to someone else.

What does the customer see?

<details><summary>A. Another customer's order, in full</summary>

✓ Right. The customer sees another customer's order in full, and at a pharmacy that is a stranger's medicines on someone else's screen.

The token is good and the scope covers reading orders, so the gateway routes the call: **a scope says what kind of thing a caller may do and not to which object**. To a gateway 58231 is only a string in a path, and who owns that order is a fact in the order service's data, so that is the one place this check, called object-level authorization, can run.

Have the order service compare the order's owner with the token's `sub` on every read, and answer not found when they differ. Test each route with two customers' tokens, and leave the gateway to the token and the scope.

</details>

<details><summary>B. A not-found message</summary>

✗ List screens are filtered to the signed-in customer, and this search box sits in the same app. That filter belongs to the list, and the call for 58231 asks for one order by number, which exists.

</details>

<details><summary>C. A 403 from the gateway</summary>

✗ The gateway does authorization, and it did check a scope. The token gives who the customer is and what kind of call they may make, and who bought order 58231 is a fact in the order service's data.

</details>

<details><summary>D. The other customer's order, with name and address masked</summary>

✗ Support screens and logs often mask personal details for anyone but the owner, and a pharmacy has every reason to. Masking is code that someone writes against the owner's id, and it needs the same comparison that nobody here makes, so the order comes back as stored.

</details>

---

###### 15. Trusting the user header the gateway adds

A credit union runs a gateway that validates each customer's token, then adds an `X-User-Id` header before it forwards the request, and the statement service takes its user from that header. An analyst's reconciliation job, running on the same private network, calls the statement service by its internal name and fills `X-User-Id` from a table of accounts closed last year.

What does the statement service do with the job's call?

<details><summary>A. Answers 401 for the missing token</summary>

✗ At the gateway every request carries a token, and a bank's statements are guarded by one. The token stops there, and the statement service has no code that reads one, so a call that arrives without it looks no different.

</details>

<details><summary>B. Returns the closed accounts' statements</summary>

✓ Right. The statement service returns the closed accounts' statements, one set for each id the job sends, to a job nobody gave that access.

The check happened at the gateway, and **a header a gateway adds is text that any caller can write**. A call that skips the gateway skips the check and arrives looking the same.

Replace the header with something the service can check: a signed token with the statement service as its audience. Add an access rule on service identity, so that only the gateway and named jobs can call it at all.

</details>

<details><summary>C. Returns only what an anonymous visitor could see</summary>

✗ A call with no token is anonymous on a public site, and this job presents none. The statement service never asks for a token: it reads `X-User-Id`, the header is there, and it looks those accounts up.

</details>

<details><summary>D. Returns nothing for the closed accounts</summary>

✗ Login systems end a user's sessions when an account closes, and these closed last year. The statement service keeps no sessions: it takes an id from a header and looks up statements, and those outlive the account for as long as the bank must keep them.

</details>

---

###### 16. Writing down the fraud service's access to stock

The stock service at a warehouse firm has run for two years with no access rules: mutual TLS is on, and checkout, the picking app and the reports job all call it. A new fraud service needs it too, and a security review asks for that access to be written down, so one Tuesday afternoon you add a rule to the policy engine in the cluster's network layer.

```yaml
action: ALLOW
to: stock
from: [fraud]
```

What happens to the calls to the stock service once the rule lands?

<details><summary>A. Fraud's are added, and the others carry on</summary>

✗ Allow adds to what is allowed in plain English, and on a firewall whose default is to accept, a new rule leaves everyone else alone. This engine's first allow rule turns the service's open door into a list, and checkout, the picking app and the reports job are not on it.

</details>

<details><summary>B. All are logged, with none refused</summary>

✗ New policy often starts out in an audit mode, and rollout guides recommend it. That mode is something you ask for in the rule or the engine. This rule says `action: ALLOW` and nothing else, so the engine enforces it as soon as it lands.

</details>

<details><summary>C. All are refused, fraud's included</summary>

✗ A form rejects a record with missing fields, and this rule names no method and no path. A rule matches on the fields it does name, so `from: [fraud]` covers every call fraud makes to the stock service.

</details>

<details><summary>D. Fraud's get through, and the others are refused</summary>

✓ Right. Fraud's calls get through, and calls from checkout, the picking app and the reports job are refused, which stops the warehouse's other work on stock.

With no rules the workload is open. **In a cluster's network layer, the first allow rule on a workload refuses every call that no rule lists**, and the engine has no idea who was calling before. Some service meshes work the same way, and one whose global default is allow leaves the other callers alone until that default changes.

Write the first rule from a log of who calls today, including jobs that run only once a month, and list them all. Run it in a log-only mode, where the engine has one, through a full cycle of those jobs, and make sure that removing the rule does not depend on a call the rule would refuse.

</details>

---

###### 17. Choosing what the check does when the policy service is silent

Services at a hardware shop ask one central policy service before each call goes ahead, and today you decide what the check does when that service does not answer within its timeout. Two routes come first: `POST /refunds`, which pays money back to a card, and `GET /catalog`, which serves the prices the public site already shows, and which partners' ordering systems read all day and cannot sell without.

What should each route do when the policy service does not answer?

<details><summary>A. Refuse on both</summary>

✗ Security checklists say a check fails closed, refusing when it cannot decide. On the catalog that rule turns a policy service outage into partners who cannot sell, and all it protects is prices the public site shows anyway.

</details>

<details><summary>B. Allow on both</summary>

✗ Availability targets count every dependency against you, and the policy service is one more of them. Allowing refunds means that each time the policy service goes silent, refunds from callers no rule would pass pay out too.

</details>

<details><summary>C. Refuse refunds and allow the catalog</summary>

✓ Right. Refunds are refused and the catalog is allowed: a wrong yes on a refund pays out money, and a wrong yes on the catalog shows prices that are already public.

On each route, **what a check should do with no answer depends on what a wrong yes costs against what a wrong no costs**. A refund that waits is late, and a refund paid in error may be gone, so refunds fail closed. Partners who cannot read the catalog stop selling, and a call let through by mistake learns nothing new, so the catalog can fail open.

Set the choice route by route, and write the reason beside it where a reviewer can see it. Alert whenever a check falls back, and test both routes by stopping the policy service.

</details>

<details><summary>D. Answer both from the last cached decision, however old</summary>

✗ A client that holds its last answer stays up and still has an answer to give. A decision with no age limit goes on saying yes after a permission was removed, and on refunds that pays money.

</details>

---

###### 18. Changing the key two services share

The loyalty service and the checkout service share one API key: checkout sends it in a header on every order, and loyalty compares it and adds the customer's points. A customer whose call fails loses that order's points. Each runs a dozen copies, and each copy polls the secret store for the key on its own five-minute timer. Security wants the key changed this afternoon.

Which plan lets every customer keep their points?

<details><summary>A. Write the new key to the store at a quiet hour</summary>

✗ A quiet hour has fewer calls to lose, and a change window is where risky work usually goes. The timers run the same at night: copies on the two sides disagree for up to five minutes, and each order that meets a mismatch in that time loses its points.

</details>

<details><summary>B. Write the new key, then restart every copy of both at once</summary>

✗ Restarting is how config gets picked up, and it does make each copy read the key. Two dozen copies do not come back in one instant, so old and new still meet while they do, and a copy that is down takes no calls at all.

</details>

<details><summary>C. Move checkout to the new key, then move loyalty</summary>

✗ Deploy guides move one side first, and checkout is the side that sends. Loyalty is still comparing against the old key, so every call from a moved copy of checkout is refused, and the gap between the two steps makes the refusal certain.

</details>

<details><summary>D. Move loyalty to accept both keys, then checkout, then drop the old key</summary>

✓ Right. At no moment does a copy of checkout send a key that a copy of loyalty refuses, and every customer keeps their points.

Each copy changes over at its own poll of the secret store, on both sides, and a call fails when its caller and callee disagree. **Accepting two keys on the callee's side removes that disagreement**, as long as you wait out a full timer after each step.

If loyalty's check holds one key slot, first deploy a version with two, with no key change in it. Before each step, confirm that every copy has moved, and drop the old key only when loyalty's log shows no call still using it.

</details>

---

###### 19. Rotating the token signing key at noon

On a video platform, an issuer signs the service tokens, which last fifteen minutes. Each verifying copy loads the issuer's public keys and keeps them for an hour.

```python
def verify(token):
    keys = cache.get_or_load("issuer_keys", issuer.fetch_keys, ttl=3600)
    key = keys.get(token.header["kid"])
    if key is None:
        raise Unauthorized()
    return tokens.verify_signature(token, key)
```

At noon you add a new key to the published set, keep the old one in it, and the issuer starts signing with the new key.

What do the verifiers do with new tokens in the hour after noon?

<details><summary>A. Every copy accepts the new tokens</summary>

✗ The set is public and the token names its key, which is what a rollover is built around. Publishing changes the issuer's set and nobody's copy of it, and a copy that loaded its keys before noon does not find that `kid`, so this code refuses.

</details>

<details><summary>B. Some copies accept them, and others refuse them</summary>

✓ Right. Copies that loaded the set before noon refuse the new tokens and copies that have reloaded since accept them. One token gets a different answer depending on which copy it reaches.

**A verifier finds the key by `kid` in the copy of the set it holds**, and here that copy can be up to an hour old. Each copy's hour started when that copy last loaded, so if the loads are spread evenly, about half refuse at half past twelve and none by one o'clock. One standard has the verifier fetch the set again when it sees an unfamiliar `kid`, and this code does not.

Publish the new key first, wait out the cache's TTL, then start signing with it, and keep the old key in the set until the last token signed with it has expired. Have verifiers reload on an unknown `kid`, with a limit on how often.

</details>

<details><summary>C. Every copy refuses them for the whole hour</summary>

✗ A TTL of an hour is one number in the config, the same on every copy. Each copy's hour began at its own last load, so some copies hold the new key minutes after noon and accept what the others refuse.

</details>

<details><summary>D. Every copy accepts them and refuses the older tokens</summary>

✗ Rotating a signing key is often taught as a way to end every live token at once. You kept the old key in the set, so older tokens still verify for their fifteen minutes. The only refusals come from copies that have not reloaded, and they refuse the new tokens.

</details>

---

###### 20. Changing the internal root authority

An airline is replacing its internal root. An automatic agent renews every copy's certificate twice a day. Trust stores ship inside the images, and this week's deploy, which carries the new root, reaches the services team by team between Monday and Thursday. On Monday at 09:00 you switch the authority to issue under the new root, which the old root has not signed.

What do calls between services do this week?

<details><summary>A. Calls fail between some pairs of services, and the pairs change through the week</summary>

✓ Right. As copies renew, their new certificates are refused by any peer whose image does not yet hold the new root. The pairs that fail shift all week as both sides move.

A certificate under the new root has no chain on an old store, since **a peer accepts only a chain that ends at a root in its own trust store**. Renewals spread the new certificates within about half a day, and the deploy spreads the new root by Thursday. Having the old root also sign the new one, called cross-signing, can bridge the two, and only for the verifiers that build the path through it.

Trust both roots everywhere first, and check what each fleet's store holds. Then issue under the new root, and retire the old one only when no live certificate chains to it.

</details>

<details><summary>B. Every call fails from 09:00 on Monday</summary>

✗ A root sits at the top of every chain, and every certificate in this fleet hangs from the one being replaced. Certificates already issued keep their old chain, and the old root stays in every store, so nothing fails at 09:00, and the failures begin as copies renew.

</details>

<details><summary>C. Calls go on as before</summary>

✗ The agent renews twice a day and nothing ever happens, and the same authority is doing the issuing. Every renewal so far ended at a root all peers held. These end at one that a peer on last week's image does not have, and that peer ends the handshake.

</details>

<details><summary>D. Calls go on, with the new certificates logged as untrusted</summary>

✗ A browser lets a person click past an untrusted certificate, after a warning. A service has nobody to click: a verifying client or server that meets a chain to an unknown root ends the handshake, and the call never starts.

</details>

---

###### 21. Writing the runbook for the certificate authority

One internal certificate authority issues the service certificates at a payments firm. Each certificate lasts 24 hours, which security chose so that a stolen key is soon worthless, and an agent beside each copy asks for a new one when eight hours are left. Copies start at all hours, so renewals run all day. The authority's alert opens a ticket for the next working day. You are writing its runbook, and the first line gives on-call's time to bring it back.

How long can the authority stay down before calls between services start to fail?

<details><summary>A. None: calls fail as soon as the authority stops</summary>

✗ Diagrams of TLS draw the authority beside the handshake, and a browser can ask about a certificate while it connects. These peers check the chain against the root in their own trust stores and send the authority nothing, so calls carry on, and the first thing to go unanswered is the next agent's renewal.

</details>

<details><summary>B. Eight hours, when the first certificates run out</summary>

✓ Right. The line says eight hours. Calls between running copies go on until then, and a ticket for the next working day can reach someone long after they have started to fail.

A peer checks a certificate against the root in its own trust store, so calls do not need the authority. Renewals do, and since they run all day, **the oldest certificate in the fleet has only the time left at renewal**: a copy whose agent was about to ask holds eight hours, and one that renewed moments before holds nearly 24. From hour eight more copies run out, and peers refuse them on each new connection, until by hour 24 no copy has a good certificate. A connection already open goes on until it closes, since the check runs at the handshake.

New copies are stuck from the first minute, because a copy's first certificate comes from the authority too. A crashed copy is not replaced, and a fix cannot roll out.

Page on failed renewals at any hour. Have agents ask early enough that the time left at renewal is longer than the slowest restore you have drilled, and keep a way to restore the authority that needs no certificate from it, since past hour eight the tools that restore it can hold expired ones too.

</details>

<details><summary>C. 24 hours, when the first certificates run out</summary>

✗ Security gives 24 hours as the length of a certificate, and a copy fresh from its renewal does hold nearly that much. Renewals run all day, so when the authority stopped, some other copy's agent was about to ask, and that copy's certificate runs out eight hours later. Hour 24 is when the last one goes.

</details>

<details><summary>D. No limit: calls go on, and only new copies are refused</summary>

✗ Calls do not pass through the authority, and every running copy already holds a certificate to present. Each of those certificates has an end date, and the agent's way past it is a request the authority no longer answers. Once the date arrives, peers end the handshake with that running copy as they would with any other expired certificate.

</details>

---

## Further reading

- [The most dangerous code in the world](https://dl.acm.org/doi/10.1145/2382196.2382204), 2012: how confusing TLS library interfaces left certificate validation broken in many security-critical applications and libraries.
- [The Confused Deputy](https://dl.acm.org/doi/10.1145/54289.871709), 1988: the short story of a compiler, a billing file and where the name for this failure comes from.
- [The Protection of Information in Computer Systems](https://doi.org/10.1109/PROC.1975.9939), 1975: the design principles behind least privilege and basing access on permission rather than exclusion.

Prev: [Microservices](../microservices/README.md) · Next: [Containers and orchestration](../containers-and-orchestration/README.md)
