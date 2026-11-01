# HTTP caching and CDNs

Prev: [DNS, TCP and TLS](../dns-tcp-and-tls/README.md) · Next: [Load balancers and proxies](../load-balancers-and-proxies/README.md) · Reviewed October 2026

HTTP caching keeps a copy of a response so that a later request can be answered without going back to the server that made it, and a CDN is a network of such caches placed close to users. A response says in its headers who may keep it and for how long, and a browser and a CDN can each hold a copy of one page at the same time.

- **Origin**: your own server or servers, which build the responses; everything in front of them stores or passes on what they send.
- **Edge location**: one of a CDN's many sites, close to the users it serves, answering them from what it has stored and going to the origin for the rest.
- **Shared cache and private cache**: a shared cache, such as a proxy or an edge location, serves many users from one store; a private cache, a browser's, serves one.
- **Hit, miss and hit rate**: a hit is a request answered from a stored copy, a miss is one that has to go onward, and the hit rate is the share of requests that hit.
- **TTL** (time to live, which the HTTP specification calls the freshness lifetime): how long a stored copy may be used with no request to the origin. A shared cache takes it from `s-maxage`, `max-age` or `Expires`, in that order, and a browser from the last two.
- **Fresh and stale**: a copy is fresh inside its TTL and stale after it.
- **Validator**: what a response carries to identify its version: a date in `Last-Modified` or a content tag in `ETag`. A date counts whole seconds, so it cannot tell apart two saves inside one second.
- **Revalidation**: a request that sends a stored copy's validator and asks whether the copy still holds. The origin answers 304 with no body, or 200 with the whole response, so a 304 saves the body and still costs the request.
- **Cache key**: what a cache looks up a stored response by: at the least the method and the address.
- **Vary**: a response header naming request headers that have to match as well before a stored copy is reused; `Vary: Accept-Encoding`, for one, keeps a compressed copy apart from a plain one, and `Vary: Origin`, named for the request header that says which site is asking, keeps replies written for different sites apart.
- **Request coalescing**: when several requests miss on one object at the same moment, a cache sends one fetch and holds the others, to answer them from its result; also called request collapsing.
- **Purge**: an order from a CDN's operator to drop stored copies, by address, by a tag the origin attached to its responses, or for a whole site. Some CDNs can mark copies stale instead, so that the next request revalidates them.
- **Shield**: a second tier of cache between the edge locations and the origin. The locations ask the shield, and only the shield asks the origin.

The rules for caches travel in the `Cache-Control` response header, as directives such as these:

| Directive | What it tells a cache |
| --- | --- |
| `max-age` | the copy's TTL in seconds, for any cache |
| `s-maxage` | the copy's TTL in seconds for shared caches, in place of `max-age`; browsers leave it alone |
| `no-store` | no cache may keep any part of this response |
| `stale-while-revalidate` | a stale copy may be served for this many seconds while the cache refreshes it behind the reply; the refresh starts only with a request inside those seconds |
| `stale-if-error` | a stale copy may be served for this many seconds when an error is met |

A CDN also has settings and rules of its own, and makers differ on them. Unless a question says otherwise, a CDN here keys on the method and the whole address, has no shield and follows HTTP as written.

Most of this bites after a change: the origin holds one version, a copy somewhere holds another, and the headers set long before decide who sees which.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- read the headers on a timetable, a report, a renamed script or a file on four servers and work out who stores it, for how long and when it asks the origin again
- decide what belongs in the key, from a click id to a country, and follow an account page, a session cookie and a rule that caches everything through a shared cache
- work out what reaches the origin and what people get back, from a refusal during a bad deploy and a launch to a crowd opening a personal page at the same moment
- get an old schedule, a logo, a wrong result or a mistaken redirect out of the caches that hold it, and know which of them you can reach

###### 1. Sending a response with no freshness headers

A council's site serves its bus timetable with a `Last-Modified` date a year old and no `Cache-Control` or `Expires`. A resident loads the page one evening, and later that evening the site publishes a new timetable. The resident's browser runs with its default settings, and the next morning the resident comes back through a bookmark, with no reload.

What does the browser do the next morning?

<details><summary>A. It downloads the new timetable whole</summary>

✗ In most projects caching is something you switch on with a header you chose, and this site chose none. A response with no stated TTL can still be stored, and the browser gives this one an estimated TTL of its own. The morning visit is answered from that copy, with no request for the new timetable.

</details>

<details><summary>B. It asks the site whether the date has changed</summary>

✗ `Last-Modified` exists for that question, and browsers send the date when they revalidate. They revalidate a stale copy, though, and this copy is one day into a TTL estimated at a week or more. It is still fresh, so the date in `Last-Modified` goes unused.

</details>

<details><summary>C. It shows the old timetable and fetches the new one behind it</summary>

✗ The copy is out of date by now, since the site published a new timetable last night, and a refresh behind the page would put that right. The browser has no way to know this. It counts its copy as fresh, and with no `stale-while-revalidate` on the reply no refresh starts.

</details>

<details><summary>D. It shows the old timetable and sends no request</summary>

✓ Right. The browser sends nothing to the site and shows the old timetable from its cache. The resident reads the old times with no sign that a newer timetable exists.

This reply states no TTL, and **HTTP lets a cache estimate one**, usually as a fraction of the time since the page last changed. The fraction the specification calls typical is 10%, which turns a date 365 days old into about five weeks. One browser engine caps the estimate at a week, and the morning visit falls well inside that too.

Give every page a TTL you chose. For a timetable that changes on a known day, a short `max-age` with `Last-Modified` kept on means browsers ask often and download only when the date moves.

</details>

---

###### 2. Marking a dashboard no-cache

An accounting product with cookie sign-in marks its report dashboard `Cache-Control: no-cache`, with an `ETag`, because the figures should stay out of other people's caches. Customers' offices reach it through a corporate proxy that can read their traffic and is set to store everything HTTP allows. One morning the first accountant in an office opens the report.

After that first request, what does the proxy hold and do?

<details><summary>A. It holds a copy and reuses it for a few seconds</summary>

✗ A `max-age` of a few seconds is the usual way to keep figures nearly current. `no-cache` sets no seconds at all, and a copy of any age needs a check before reuse, so the next accountant's request goes to the origin even a moment later.

</details>

<details><summary>B. It holds nothing and passes each request on in full</summary>

✗ Sample code pastes `no-cache` wherever a page has to stay out of caches, though `no-store` is the directive that forbids a copy. This proxy keeps the report, and later requests go out with its `ETag`, so an unchanged report comes back as a 304 with no body.

</details>

<details><summary>C. It holds a copy and checks with the origin before each reuse</summary>

✓ Right. The proxy keeps the report in its store, the very cache the product wanted to stay out of, and asks the origin before each reuse.

**`no-cache` forbids reuse without a successful revalidation and does not forbid storing**, while `no-store` forbids any cache to keep a copy and `private` forbids a shared one to.

Send `no-store` where no copy may exist anywhere, or `private` where a browser's copy is fine and a shared one is not. Keep `no-cache` for pages you want checked cheaply, and treat none of these as a privacy guarantee, since a directive only asks caches to behave.

</details>

<details><summary>D. It holds a copy for that accountant alone</summary>

✗ `private` lets one person's browser keep one person's copy, and this page does not send it. The proxy has one store for all its users, and `no-cache` names no user. The stored report sits under its address, and the next accountant's request finds it and sets off the same check.

</details>

---

###### 3. Hashing asset names under a cached page

A shop gives its script a name that changes with its content, `/app.v1.js` today, with a one-year TTL, and loads it from a plain script tag. The page has a one-hour TTL, and no shared cache sits in front of the origin. A deploy publishes `/app.v2.js` with a page that names it, and removes the old script from the origin. One customer's browser stored the page ten minutes before the deploy, and has since dropped the script to make room.

Which page and script reach that customer on a visit twenty minutes after the deploy?

<details><summary>A. The old page, running the new script</summary>

✗ Recipes for hashed names promise that a deploy reaches everyone, and a new name forces a fetch in a browser that sees it. This browser never sees that name: the old page is fresh, so it comes straight from the cache, and the only name it holds is `/app.v1.js`.

</details>

<details><summary>B. The old page, and a not-found for its script</summary>

✓ Right. The browser shows the old page, asks for `/app.v1.js` and gets a 404. The script never runs, and the customer sees a page without it until a reload or the end of the page's hour.

The browser learns a script's name from the page, so **a new script name reaches a browser only inside a new page**. Here the page is fresh, and the browser reuses it without asking.

Keep old script files on the origin for at least the longest TTL of a page that names them, here an hour, and remove them on a later deploy.

</details>

<details><summary>C. The new page, after a refresh the failed script forces</summary>

✗ A page's own code can reload it when a script fails to arrive. A plain script tag has no such logic: the failed request leaves the page as it was, still fresh, and nothing asks for a newer one.

</details>

<details><summary>D. The new page and the new script</summary>

✗ The deploy replaced the page on the origin, and the origin is where this page came from. The browser has no reason to go there: its copy has thirty minutes of TTL left, and a fresh copy is used without asking, so the deploy goes unseen until that hour ends.

</details>

---

###### 4. Copying a download page to four servers

A software vendor serves its download page from four servers behind a balancer that sends each request to any of them. Their web server builds the `ETag` from the file's size and its modification time, and a deploy copies the same bytes to the four servers a few seconds apart, each file stamped with the time of its own copy. One shared cache in front holds one stored copy of the page and revalidates it on each read.

What do most of the cache's revalidations bring back?

<details><summary>A. A full 200 from each server once, then a 304 every time</summary>

✗ Once every server has answered, the cache has met all four tags, and `If-None-Match` can carry a list of them. This cache holds one copy and sends one tag, and each 200 replaces it, so what it learned from the last server is gone by the next, and most answers stay a 200.

</details>

<details><summary>B. A 412 for a failed condition</summary>

✗ A failed condition can bring a 412 on a write, when `If-Match` guards an update. A revalidation is a read that carries `If-None-Match`, where a different tag is the ordinary case, and the answer to it is the whole page with a 200.

</details>

<details><summary>C. A full 200 with the whole page</summary>

✓ Right. Most revalidations bring back a full 200: the cache keeps downloading a page it already holds byte for byte, and revalidating saves little of the traffic it was meant to save.

**A content tag is whatever the server computes**, and this one holds a size and a modification time. The size is one number and the modification times are four, so identical files carry four different tags, and the cache's one tag matches only when the balancer picks the server that last answered with a 200. If each read goes to a server picked at random, about three revalidations in four get a 200.

Build the tag from the bytes, or have the deploy set one modification time on every copy. Then check it from outside, by asking each server for the same file and comparing tags.

</details>

<details><summary>D. A 304 with no body</summary>

✗ Explanations of `ETag` often define it as a hash of the content, and these four files are byte for byte the same. This server hashes nothing: its tag holds the file's size and modification time, and with four copies written seconds apart, a tag from one server matches only that server's reply.

</details>

---

###### 5. Adding a click id to every link of a summer mailing

A travel firm mails a summer campaign, and each link carries a click id unique to its recipient, as in `/summer?click=…`. The landing page ignores the id, so every recipient gets the same HTML, with an hour's TTL. The plain `/summer` address is already stored at the edge from the site's own traffic. The mail goes out, and recipients start clicking.

What reaches the origin as the clicks come in?

<details><summary>A. Nothing, with the stored <code>/summer</code> page answering every click</summary>

✗ Your web framework routes `/summer?click=…` to the `/summer` handler whatever follows the question mark, and a copy of that page is already stored. A router reads the path, and this key holds the whole address. Each link is an address the cache has never stored, so the plain copy answers none of them.

</details>

<details><summary>B. One fetch, then every later click served from the edge</summary>

✗ The HTML behind every link is the same, but the cache never compares HTML. The lookup uses the key, and the next recipient's id makes a different one. The first fetch's copy waits for a second click on that one link.

</details>

<details><summary>C. One fetch from each edge location</summary>

✗ Each edge location holds its own copies, and that is the usual way one page turns into several fetches. The ids multiply here and the locations do not matter: with one location or five, each new id misses wherever it lands.

</details>

<details><summary>D. A fetch for each recipient's first click</summary>

✓ Right. Every recipient's first click misses and goes to the origin, which builds the same page again and again through the peak of the campaign.

The cache key here is the whole address, and **a different query string is a different address**. Each id gets its own entry and its own fetch, and the copy it leaves is read again only if that recipient clicks twice.

Have the CDN leave the click id out of the key, and the clicks find the stored `/summer` copy and cost no fetch while it is fresh. Read the id from the page's own script or the edge's logs, since a request answered from the cache no longer carries it to your server.

</details>

---

###### 6. Varying on the User-Agent header

A news site reads `User-Agent` to sort each visitor into phone or desktop and sends the matching HTML, so its front page now goes out with the headers below. Each edge location gets a few hundred visitors a minute.

```http
Cache-Control: max-age=60
Vary: User-Agent
```

What happens to the front page's hit rate after the change?

<details><summary>A. It falls, with many copies hardly anyone reads</summary>

✓ Right. The hit rate falls. A page that almost never reached the origin now reaches it for each value a location has not met that minute, while each location holds many copies that almost no second visitor reuses before the minute ends.

**`Vary` makes the cache compare the header's whole value**, and the cache does not know what a phone is: two strings that differ by one digit are two requests, and each needs its own stored copy. Browsers send a great many different strings, one for each mix of browser, version and platform. Say each location gets 300 visitors a minute, whose browsers draw on 20,000 different values, with the ten commonest bringing about a quarter of the visitors and the rarer values thinning out gradually: the hit rate then goes from nearly every request to about a third. How far the rate falls depends on how many values there are and how much traffic each gets.

Have the edge write a device class into a header of its own and, on a CDN that applies `Vary` to a header it adds itself, send `Vary` on that. That leaves two copies and a hit rate almost where it was.

</details>

<details><summary>B. It falls by half, with one copy for phones and one for desktops</summary>

✗ The design has a copy for phones and a copy for desktops, with half the traffic each. Two copies would barely cost anything at this traffic, since each is fetched once a minute and answers the rest. This cache keeps one copy per value, and with many values each copy meets few readers before its minute ends: the loss comes from those thinly read copies.

</details>

<details><summary>C. It holds, with two stored copies</summary>

✗ Your code reads this header and gets one of two answers, so to you the header has two values. The cache runs none of your code. It stores a copy per string it receives, and a browser's string carries its version and its platform, so the visitors at one location split across far more than two.

</details>

<details><summary>D. It dips a little, with similar values grouped</summary>

✗ These strings differ by little, often a digit or two in a version. The cache compares them whole, with no rule here to group them, so each new value goes to the origin, and many visitors send a value that no earlier request in that minute carried.

</details>

---

###### 7. Choosing a country at the origin

On a concert site, the origin picks the price and the legal text from the visitor's country, read from a `Visitor-Country` header the CDN adds to each request. The reply carries the caching header below and no other. One edge location serves both Germany and the US, and a visitor from each arrives, the US visitor a minute after the German one.

```http
GET /tour HTTP/1.1
Visitor-Country: DE

HTTP/1.1 200 OK
Cache-Control: max-age=300

... 40 EUR, German terms ...
```

What does the US visitor see?

<details><summary>A. A price of 45 USD and the US terms</summary>

✗ The origin decides per visitor, and every request it receives gets the right country. The request stops at the edge, since the key is the address and the German visitor's page is stored under that address.

</details>

<details><summary>B. A price of 40 EUR and the German terms</summary>

✓ Right. The US visitor gets the German page from the edge, and the origin never hears of that visitor: a buyer in the US reads a price in euros and legal terms that are not theirs.

**The origin's choice runs on a miss only**, and its result is stored under the cache key for everyone who asks at that location until the TTL ends. The CDN adds the `Visitor-Country` header itself, and it plays no part in the lookup, since neither the key nor a `Vary` names it.

Put the country in the key, or, on a CDN that applies `Vary` to a header it adds itself, send `Vary: Visitor-Country` on every reply from that address. Giving each country its own address, as in `/de/tour`, does the same.

</details>

<details><summary>C. A redirect to a page for the US</summary>

✗ The edge knows the country, since it adds the header itself, but it uses the header for nothing. Only the origin reads it, and the origin is not asked: the edge answers from the fresh page under this address.

</details>

<details><summary>D. The German terms with a price of 45 USD</summary>

✗ A price could be fetched by a request of its own, apart from the page around it. This reply holds the price and the terms in one stored response, and a hit hands over the whole of it, with its 40 EUR.

</details>

---

###### 8. Sending a personal page with a short TTL

To ease the evening peak, a phone company's account page gets `Cache-Control: max-age=60` and no other caching header. Requests carry a session cookie. In testing with one account, every page came back right. That evening two customers open `/account` through one edge location, ten seconds apart.

Which page does the second customer get?

<details><summary>A. Their own account page</summary>

✗ Each request carries its customer's session cookie, and HTTP has a rule of this kind for the `Authorization` header: a shared cache does not reuse such a response unless the response opts in to shared reuse. A cookie has no such rule, and the second request matches the stored page by its address and never reaches the code that reads the cookie.

</details>

<details><summary>B. Their own account page after a check</summary>

✗ A check before every reuse is what `no-cache` asks for, and a short TTL is a milder form of the same care. `max-age=60` asks the opposite: for sixty seconds the copy may be reused on its own. Ten seconds in, the cache has no reason to contact anyone.

</details>

<details><summary>C. The first customer's account page</summary>

✓ Right. The second customer reads another person's account: the edge answers with the page it stored for the first customer, and the origin never sees the request.

The page has a TTL and no `private`, and **HTTP has no rule that a cookie stops a shared cache**, so the response is stored. The key here is the address alone, which makes the second request a hit. The one test account could not show this, since each page it received was its own.

Send `private` on anything built for one person, and move the short TTL to the parts of the site that are the same for everyone.

</details>

<details><summary>D. The page's shell, with the personal parts blank</summary>

✗ A blank shell would let a personal page be stored safely, with the account filled in afterwards. A cache does not edit what it stores: this one kept the first customer's response as it came and hands it over with every detail in it.

</details>

---

###### 9. Setting a session cookie on a home page with a five-minute TTL

A retailer's home page sets a session cookie for each new visitor and has a five-minute TTL. Visitors reach one edge location. The first new visitor of the morning gets the reply below.

```http
HTTP/1.1 200 OK
Cache-Control: max-age=300
Set-Cookie: sid=1

<html>...home page...</html>
```

What does the next new visitor's browser receive with the page?

<details><summary>A. A new cookie of its own from the origin</summary>

✗ Tutorials say a response with `Set-Cookie` is not cached, and some CDNs do skip such replies by default. HTTP makes no such rule, and it says outright that the header does not stop caching. This CDN has no rule of its own, so the request is a hit and never gets as far as the origin to issue `sid=2`.

</details>

<details><summary>B. The cookie <code>sid=1</code>, the first visitor's</summary>

✓ Right. The stored reply is handed over with its headers, `Set-Cookie: sid=1` among them. Two browsers now hold one session, and whatever the site ties to it, such as a cart, is shared between two strangers.

HTTP states that `Set-Cookie` does not inhibit caching, and **a stored response is reused whole, headers included**. Makers differ here: some CDNs skip such replies, and some strip the header and store the page. On a CDN with no such rule, each new visitor receives `sid=1` until the five minutes are up.

Keep `Set-Cookie` off responses with a shared TTL. Issue the session from a separate request marked `private`, or leave the home page uncached for visitors who have no session yet.

</details>

<details><summary>C. The page with no cookie on it</summary>

✗ A session cookie is one person's and the page around it is everyone's, and some CDNs strip the header before storing. This CDN has no rule about `Set-Cookie`, so the reply stays as the origin sent it, and the next visitor's copy carries `sid=1`.

</details>

<details><summary>D. The cookie <code>sid=1</code>, which the browser then ignores</summary>

✗ Cookies come with many binding rules, by domain, by path and by secure transport. None of them ties a cookie to the first browser that got it. The browser sees a `Set-Cookie` for this site on a reply from this site, stores it and sends `sid=1` from then on.

</details>

---

###### 10. Adding a CDN rule that caches everything for two hours

A traffic surge is slowing a furniture retailer's origin, so an operator adds a CDN rule: cache every response, with a two-hour TTL. The order page goes out with `Cache-Control: private`, and the team counts that page as safe. Then two customers open their orders, one after the other, through one edge location.

```http
HTTP/1.1 200 OK
Cache-Control: private
```

What does the edge do with that order page?

<details><summary>A. It passes it through unstored</summary>

✗ The specification says a shared cache must not store this response, and the team counts on that. The edge is run by its operator, and this rule tells it to disregard the header, so the directive is never applied.

</details>

<details><summary>B. It stores it and checks with the origin before each reuse</summary>

✗ Revalidating personal pages is a common habit. This rule sets a two-hour TTL and the page carries no `no-cache`, so inside those hours the copy goes out and the origin is not asked.

</details>

<details><summary>C. It stores one copy per customer</summary>

✗ `Vary` keeps copies apart for one header, and this page differs by customer. This reply names no `Vary`, which leaves copies to the key, and this key is the address alone. Whatever tells these customers apart is not in the lookup, so one entry answers them both.

</details>

<details><summary>D. It stores it and serves it to other customers</summary>

✓ Right. The edge stores the first customer's order page and answers the second customer with it, so one customer reads another's order, for as long as the two hours run.

`private` has no effect here, since **an edge rule can override the origin's headers** and this one does. The key is the address alone, which leaves one entry to answer both customers.

Scope such a rule to paths you know are public, so that it never reaches a page built for one person.

</details>

---

###### 11. stale-if-error when the origin answers 403

An airline's fare page goes out with `Cache-Control: max-age=300, stale-if-error=86400`. A bad firewall deploy makes the origin answer 403 to the CDN for an hour, starting at the moment the stored copy's five minutes end. The team's resilience checklist has this directive ticked.

Which reply do customers get during that hour?

<details><summary>A. The 403 from the origin</summary>

✓ Right. Customers get the origin's 403 for the whole hour. The fare page is gone for them, though the CDN still holds a stale copy with most of its window left.

**The specification of `stale-if-error` counts as errors only 500, 502, 503 and 504**, the replies of a server that failed. A 403 is an answer from a server working as told, and the cache passes it on. If the origin had answered 503, customers would have got the stored fare page.

Alert on a jump in 403s from the origin, since this directive will not cover them. Where a page must outlast a bad deploy, give it a longer TTL, and remember that a cache may already have dropped a page that is rarely read to make room.

</details>

<details><summary>B. The stored fare page for the whole hour</summary>

✗ In plain English a 403 during an outage is an error, and the directive is named for errors. Its specification lists four statuses and all of them are in the 500s. A 403 is not among them, so the stale copy stays where it is.

</details>

<details><summary>C. A CDN error page</summary>

✗ The CDN sits between the customer and a failing origin, and a failure there usually shows up as the CDN's own page. Here it has received a complete, well-formed reply and has no error page set, so it hands that reply to the customer, 403 and all.

</details>

---

###### 12. Serving a not-found page before a product launch

When a fan checked a phone maker's launch page five minutes before the launch, it returned 404 with no `Cache-Control`. A shared cache in front gives a default of ten minutes to any reply it may store that names no TTL. At the launch, the page goes live on the origin and marketing sends out the link.

```http
HTTP/1.1 404 Not Found
Content-Type: text/html
```

Which reply do fans get in the minute after the launch?

<details><summary>A. The new page</summary>

✗ The page exists on the origin, but this request never gets there. A 404 is one of the statuses HTTP lets a cache store by default, and this cache gives it ten minutes. Five of them are still to run when the launch comes.

</details>

<details><summary>B. The new page after a short delay</summary>

✗ New content is often described as spreading across a network in moments. Nothing here spreads: the origin has the page and tells no cache about it. The cache finds a fresh 404 under the address and answers with it.

</details>

<details><summary>C. The 404 for the early fan, the new page for everyone else</summary>

✗ On a server each request gets its own answer, and the 404 went to one fan. A shared cache keeps one entry for the address, whoever filled it. Every fan who asks through it gets what the early fan's check left there.

</details>

<details><summary>D. The 404 left by the early fan's check</summary>

✓ Right. Every fan going through that cache gets the stored 404 for about five minutes. The launch link reads as broken while the page is live on the origin.

**HTTP lists a 404 among the responses a cache may store with no stated TTL**, and this cache's default then applies. The ten minutes run from the early check, so the page shows five minutes after the launch.

Find out what your CDN does with a 404, since makers range from a few minutes to some seconds to no storing at all. Then send `no-store` on the 404 for an address that is about to go live, or purge the address as part of the launch.

</details>

---

###### 13. Sending a purge while the origin fleet is rolling out

A broadcaster's deploy hook purges its schedule page as soon as a deploy starts. The page has a one-day TTL, the rollout takes a few minutes, and meanwhile half the origin servers still have the old build.

```text
start    deploy begins, hook purges the page, CDN answers success
next     first reader at one edge location: fetch reaches an old server
later    rollout finishes, every server has the new build
```

After the rollout finishes, which page does that location serve?

<details><summary>A. The new page, after a check with the origin</summary>

✗ A CDN can mark a purged copy stale, and the next request then revalidates it. The fetch stored a new entry with a TTL of its own, a whole day, so the location counts it as fresh and serves the old build's page.

</details>

<details><summary>B. The new page, once the last server has updated</summary>

✗ From the deploy tool's side the rollout has a clear end, with every server green. The edge cannot see it: the location holds a fresh copy, so it asks nothing and learns nothing when the last server changes.

</details>

<details><summary>C. The old page for a full day from that fetch</summary>

✓ Right. The location keeps serving the old schedule for a whole TTL from that fetch. Viewers there see it for about a day after every server has the new one.

A purge clears the copies stored when it takes effect, and **a fetch afterwards stores whatever the server it reached returned, with a new TTL**. Here that server still had the old build.

Make the purge the last step of the rollout, after the last server reports the new build.

</details>

<details><summary>D. The old and new pages in turn, by server</summary>

✗ A balancer hands out old and new in turn during a rollout, one request at a time. The location asks once and stores the reply, and later readers all get that one stored page.

</details>

---

###### 14. Purging everything at peak traffic

A large shop changes its logo, and at lunchtime an operator sends a purge that deletes every stored copy of the whole site, rather than the logo's address. The CDN has no shield, and the lunchtime traffic keeps arriving at every location. Only the logo has changed: the other pages and images are as they were.

What must the origin be ready to serve in the minutes after?

<details><summary>A. Full downloads of whatever each location is asked for</summary>

✓ Right. The origin serves a full download of each object every location is asked for. At the lunchtime peak, most of what it sends out again has not changed by a single byte.

**A purge that deletes leaves nothing to revalidate**: the location has no copy and no validator, so its next request is a plain fetch. Say 50 locations are each asked for 5 objects, 1 of them changed: deleting brings 250 full downloads, and marking stale instead brings 50 full downloads and 200 answers of 304.

Purge the logo's address. If a whole site has to go, mark it stale where the CDN allows it and pick a quiet hour, with a shield in place where you have one.

</details>

<details><summary>B. A conditional request per object, each answered 304</summary>

✗ Revalidation is what normally happens to an old copy, and it is cheap when nothing changed. A check needs a copy, and this purge deleted them. With nothing stored there is no validator to send, so each location asks for the whole object.

</details>

<details><summary>C. A short rise that the CDN smooths out</summary>

✗ Standing between users and the origin is what a CDN is for, and an everyday purge passes without anyone noticing. With no shield here, each location's fetch goes straight to the origin, and the load lasts as long as objects with no stored copy are being asked for.

</details>

<details><summary>D. One fetch per object, shared by all the locations</summary>

✗ Request coalescing turns many requests for one object into one fetch. It works inside one cache, and each location is its own: with every location coalescing, the origin still gets about one fetch per location for a page none of them holds. Only a shield brings that down to one fetch in all.

</details>

---

###### 15. Purging a wrong result from the CDN

A sports league publishes its table with the header below. A wrong result goes up, the origin is corrected, and an operator purges the table's address from the CDN. Opened in a private window, the table now looks right. A fan who loaded it an hour ago, with the wrong result showing, follows a link to it again.

```http
Cache-Control: max-age=86400, s-maxage=60
```

What does that fan see?

<details><summary>A. The corrected table, fetched through the CDN</summary>

✗ The private window's check went through the CDN, and the purge clearly worked there. This browser holds a fresh copy and never goes that far, so the corrected table waits at the CDN.

</details>

<details><summary>B. The wrong table, read from the fan's own browser</summary>

✓ Right. The fan's browser shows the wrong table from its own cache and sends no request. The league has already corrected the result, and this fan keeps reading the old one for most of another day.

**A purge reaches the caches the operator runs**, and a browser's cache is not one of them. Its TTL comes from `max-age`, as `s-maxage` binds shared caches only, and a fresh copy is reused without asking anyone.

For data that may need correcting, give browsers the short TTL and the CDN the long one, as with a `max-age` of a minute and a long `s-maxage`, since the CDN's copy is the one you can purge. Tag every page with the results it shows and purge by tag, since a purge by address clears only that one address.

</details>

<details><summary>C. The wrong table for up to a minute, then the corrected one</summary>

✗ The shortest TTL in the header is one minute, and the CDN stands between the origin and every fan. That minute binds shared caches only, while the browser follows `max-age`, which gives its copy a day, and only an hour of that has gone.

</details>

<details><summary>D. The corrected table after a check with the origin</summary>

✗ Press reload and a browser checks with the server. This fan followed a link, and for that a fresh copy needs no check, so the browser shows what it has.

</details>

---

###### 16. Sending a permanent redirect by mistake

A software firm means to point its pricing address at a sale page for one day, and by mistake the server answers `/pricing` with the permanent redirect below, and no other caching header. The next morning the team removes the redirect, purges `/pricing` from the CDN and leaves the sale page up. From the office, on machines that never followed it, the address shows the pricing page again. A visitor who followed the redirect that day, on a browser with default settings and nothing cleared since, types `/pricing` again three days after the removal.

```http
GET /pricing HTTP/1.1

HTTP/1.1 301 Moved Permanently
Location: /sale
Date: Mon, 12 Oct 2026 09:00:00 GMT
Last-Modified: Sat, 04 Jul 2026 09:00:00 GMT
```

Where does that visitor land?

<details><summary>A. On the pricing page, straight from the CDN</summary>

✗ The redirect is gone from the server, the purge emptied the CDN, and the office's machines show the pricing page. Neither step touches a browser's own store, and the office's machines had nothing in theirs. This visitor's browser holds the redirect and answers the typed address from it, so the request never sets out for the CDN.

</details>

<details><summary>B. On the pricing page, after the browser checks its stored redirect</summary>

✗ The redirect carries a `Last-Modified` date, and that is the validator a browser sends when it revalidates. A validator comes into play once a copy is stale. The estimate that browsers make for this redirect still has time to run, so no check goes out.

</details>

<details><summary>C. On the sale page, with no request for <code>/pricing</code></summary>

✓ Right. The browser finds the redirect in its own cache, sends nothing for `/pricing` and goes to the sale page. This visitor keeps missing the pricing page while newcomers and the office read it.

**A redirect is a stored response like any other**: HTTP lets a cache reuse a 301 that names no TTL on an estimate of its own, and a fresh copy is used with no request. Browser engines set that estimate differently. One gives such a redirect no limit, one gives it a year, and one works from `Last-Modified`, at a tenth of its age up to a week, so all three still count this copy as fresh three days on.

Send a redirect you may want back as a temporary one, or give the 301 a short `max-age`, before it goes out. Once a browser has stored it, nothing you change at `/pricing` reaches it, since the browser no longer asks there. The sale page is the one address those visitors still request, so put a link to the pricing page on it.

</details>

<details><summary>D. On the browser's error for too many redirects</summary>

✗ Mixed-up redirects end on that error when two rules keep sending a browser back and forth. A loop needs the chain to come back, and this one stops at its first step: the stored redirect leads to `/sale`, which is still up and answers with its page.

</details>

---

###### 17. Coalescing requests for a page marked private

A festival's app shows each customer their tickets on one page, `/my-tickets`, through a CDN that uses request coalescing. A session cookie tells customers apart, and each reply carries `Cache-Control: private`. Building it takes the origin about a quarter of a second, and yesterday's test, one account at a time, showed each account its own tickets in about that time. At noon a notification goes out, and forty customers at one edge location open the page at the same moment, the first requests it has had today.

What do the customers behind the first request get?

<details><summary>A. Their own tickets, after a longer wait</summary>

✓ Right. Each customer gets their own tickets and nobody sees anyone else's, so the only cost is time: those behind the first request wait longer than the quarter of a second the test showed.

The cache starts holding them before it has any reply, because `private` only arrives on that reply. Once it does, **a reply marked `private` cannot answer the held requests**, and the cache has to send them on. Some caches do that one at a time, each new fetch holding those behind it, and the fortieth customer waits about ten seconds. Others hold them only until the first reply, which adds about one more quarter of a second.

Have the CDN skip the cache for an address that is always personal, by a rule, so that no request is held. Keep `private` on the page either way, since a reply marked only `no-cache` or `max-age=0` can still be shared with held requests.

</details>

<details><summary>B. The first customer's tickets</summary>

✗ Request coalescing exists to answer many requests from one fetch, and forty requests for one address are exactly that case. `private` bars a shared cache from reusing this reply for anyone else, held requests included, so the first customer's tickets go to the first customer alone.

</details>

<details><summary>C. Their own tickets, in the usual quarter of a second</summary>

✗ Pages marked `private` stay out of a shared cache's store, and the test came back in a quarter of a second. The cache reads that header only on the reply, so it holds the others until then, and the test had nobody behind it to hold. A cache that remembers such a reply can skip the hold for a while, but this page has had no request today.

</details>

<details><summary>D. A 503 from the CDN</summary>

✗ A 503 is what people see when a CDN cannot get a reply for them, and the one reply these requests were held for cannot be used for them. The CDN still has each request and a working origin behind it, so it sends each one on and that customer's tickets come back.

</details>

---

## Further reading

- [Cached and Confused](https://www.usenix.org/conference/usenixsecurity20/presentation/mirheidari), 2020: a large-scale study of what happens when an edge and an origin read one path differently, and private pages end up stored.
- [Your Cache Has Fallen](https://doi.org/10.1145/3319535.3354215), 2019: a study of fifteen web caching solutions and of how a single error reply can come to be stored for everyone asking for a page.

Prev: [DNS, TCP and TLS](../dns-tcp-and-tls/README.md) · Next: [Load balancers and proxies](../load-balancers-and-proxies/README.md)
