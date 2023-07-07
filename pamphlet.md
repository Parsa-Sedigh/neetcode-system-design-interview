## 0 - How to Approach
Do we even need to scale the DB? Maybe just a regular sql single instance DB is enough(it depends on the problem).

It's all about scoping out the problem with your interviewer.

When it comes to scoping and really understanding the requirements of the problem, we can typically categorize the requirements
in 2 buckets: functional vs non function requirements.

1. functional: The actual features that we're implementing. How do we want the application to function? What do we want to happen
   when we like a tweet? For example a single user should not be able to like a tweet multiple times. So everytime somebody likes a tweet,
   we should store their userId in a DB, so we kinda remember that. About twitter news feed: Every single user should have their unique
   news feed that we need to generate.
2. non-functional: Pretty much everything that is not about the function(how the app functions). Namely things around scalability. A lot of tutorials
   teach how to build a twitter clone but:
   - **A)** it won't have all of the features(functionalities) of the real twitter
   - **B)** it **definitely** will not be able to handle the scale of the real twitter.
     In twitter, possibly we would have 100M daily active users(you need to ask the interviewer about
     this). Typically non-functional requirements are the difficult of system design interviews. Things that fall into the scalability: throughput(100M
     daily active users and 1000s of reqs per second hitting the DB, falls into throughput), storage capacity.
     If we have 100M of users, first of all we're gonna have a lot of reqs going to our DB(throughput), but we also need to store a massive
     amount of data and a single instance DB theoretically can work **but** even a single query on that much data is gonna go very very slow. It could
     take seconds, 10 seconds. So first we need to understand how do we even store that data itself and another set of non-functional requirements comes
     from the **performance**.

When we talk about performance:
- latency
- availability: **#non error / #req . So total number of non error responses divided by the number of total requests, generally gives us the availability**
  and we want this to be as high as possible.

We need to **clarify** these with the interviewer. You shouldn't make assumptions without clarifying what we're trying to design. You don't want to
just jump straight into the design and starting solving the problem and it turns out you solved the wrong problem. Maybe we want 99% or 99.9% which are
the percentage of reqs that will not return an error.

It's impossible to have a 100% availability.

Five nines of availability(99.999) is sth like **20 or 30 minutes of downtime per year**, so getting that low isn't a huge issue like if twitter goes
down for 20 minutes per year, that's not end of the world, but we can add another 9 to get about 5 minutes of downtime per year. But the more
availability from this point, might not be worth the cost.

To make queries faster, we can partition this DB for the read/write latency to be lower. Maybe we only care about a subset of latency, maybe we want
the reads to be faster, we want them to be less than 1second, but maybe we have more tolerance for slower writes like 2, 3 seconds.

When we can't meet every requirement, we have to make intelligent tradeoffs, we have to sacrifice sth that's not super important to gain the
core functionality.

For quantifying these metric and requirements, in interview you don't have the time to get and calculate the exact value that you're looking for like exact
throughput, so what we do to get around that, is to do **back of the envelope calculations**.

Ex) We know twitter has 100M daily active users and let's say each of them reads about 100 tweets per day but maybe for writing a tweet on average,
only a 10th of these people make a tweet per day. So for each person, we have 0.1 tweets, so how many tweets total do we expect to be written?
Probably 10M writes per day and 10B reads per day.

So:

10B reads per day and 10M writes per day.

Quick calculation for throughput for reads and writes:

reads we expect per second: 10 B / seconds in a day which is 86000 ~ 100k => 10 B / 100 k = 100,000 reads/second

seconds in a day: 60 * 60 * 24 = 86000

### Basic numbers
1ms = 1s/1000

1 us = 1s / 10^6

1 ns = 1s / 10^9

1 MB = 10^6 bytes

1 GB = 10^9 bytes (used for persistent storage)

1 TB = 10^12 bytes

1 PB = 10^15 bytes (15 zeroes)

[latencies](colin-scott.github.io/personal_website/interactive_latency.html)

cpu cache(1us) faster than main memory(100ns) faster than disk.

SSD reads(disk) are a lot slower than main memory - about 100times slower.

Sending data over network is even slower than reading from disk. Sending data in the same data center is faster than sending data over the
internet all around the world is even slower.

## 1 - Design a Rate Limiter
Let's say we have 3 retries per hour for entering password on our phone. The reason for using rate limiter here is **security**. In this case,
we're not using it for saving resources on our phone!

Also on youtube we can only upload 20 videos per 24 hours. Why youtube does this?

This is probably not for security reasons. It's because of cost and availability. Why cost? Because with videos, cost is an important factor.
Why availability? Because someone can upload massive amount of videos and it will take up all of the storage and crash youtube.

So the **context** in rate limiter is important. So depending on context, the rules for rate limiting change. Maybe 100 comments per day but
20 videos per day.

Also about comments on youtube: We can make a million comments. You may see a lot of bots writing comments, so this is a UX issue and also a security
issue. We can use rate limiter here as well.

An application like youtube could have rate limits in many places.

Overall the main goal that we're trying to achieve is to prevent users or possibly even bots, from making too many reqs in some time period.

Note: First step of interview would be to clarify the functional requirements with the interviewer. Because rate limiting can be really open-ended.

**First question to ask:** Are we designing a rate limiter for some backend API? 

This(backend API) is most commonly where rate limiters are implemented, especially in distributed systems.

Let's assume the answer is yes.

**Second question:** Is it just for a single API? Like maybe just a single microservice that we own and we're trying to protect just this single
microservice.

Most likely the answer would be no but the reason we even entertain this thought is because the easiest way we could do a rate limiter is
inside of the application. In other words, in the microservice we can add more code to that and implement rate limiting in the code, like in a
hash map. So keeping track of the rules in the code itself and keeping track of the reqs in some memory data structure like a hashmap.

Since the answer is no, we're trying to design this rate limiter such that it will be able to protect multiple microservices or APIs and it should be reusable
so that it can protect multiple services or APIs. Therefore, we have to implement that rate limiter as a **separate service**.

Note: Each of these APIs(services), for example maybe there's a separate uploading api and comments api, each service here is not gonna be a single
instance api, there could be multiple instances of the upload API. But all of them need to be able to communicate with the **same rate limiting service**.
So these instances need to be in sync and this is another reason why we can't implement the rate limiting inside each of these instances. Because
suppose I upload one video, I end up hitting the first upload service, then we upload a second video, I end up hitting the other upload service.
So in theory I can bypass the rate limiter by accidentally hitting other instances of the APIs, that's why we can't implement the rate limiter at
the API level(in each API), we have to implement it at some kind of shared service. So that shared service knows I have uploaded 2 videos but each of
those individual upload services will think I've uploaded one in each.

So the reasons of implementing a shared rate limiter service:
1) should be reusable and protect multiple services
2) all of the microservices should communicate with the shared rate limiter service to be in sync with the current state of how much req a user
has made in total

**Note:** When the rate limit has been reached, the rate limiter should not allow the APIs to actually fulfill the req and just tell the user
that they're being throttled(that they can't continue to send req because the rate limit has been reached). To be more helpful, we can
let them know what is that time period. Maybe it's 24 hrs. So wait this much longer until you send another req.

In context of HTTP, when the rate limit is reached, we can send a `400` error because it's a client error, there's nothing wrong with our servers,
their functioning completely fine, it's just that the client is sending too many reqs, so they're being throttled now. Specifically the error code
for rate limiting is `429`.

### What about client side rate limiting?
It's easy to implement. In some client side code(js, java in case of mobile app), we could have some code that doesn't even send the req
in the first place. This would be ideal because it's simple and easy to implement and we probably should have client side code that does that anyway.

**Q:** What would happen if we didn't have a rate limiter on the backend and we just implement it client side?

A: In theory the user could be able to reverse-engineer API reqs and continue to send those and bypass the client side rate limiter. They might
not even be using our application in the first place, we have some public API endpoints and they can hit it just by themselves maybe with a CURL
req.

**Last functional requirement:** How do we identify users?

It depends on the context of our app. Are we just gonna use the userId? If we use this approach, the users have to signup before they send a req.
What if that's not the case? What if a user does not even have a userId? On youtube that doesn't make sense because you do have to have
an account to upload and comment but in other sites, it could. So we probably want our rate limiter to be general enough since it's gonna be serving
many possible APIs in many possible applications. Like google implements a rate limiter, they want it to not only serve youtube but also
server google search. So we wanna make it as general as possible. So we should be able to support many ids. We could even use IP address which would
be the most general thing to use.

But we're not gonna focus on a particular case, because there could be tradeoffs with IP address if we were talking about youtube or sth like that.

We wanna design a general rate limiting service that could be used in many contexts.

### Non-functional requirements
in this context, the important Non-functional requirements are:
- performance
- throughput
- storage
- availability

This is about things like performance. **So not what our rate limiter should do but how well it should perform or how it should do what it's doing?**

Because think about it: With every req that the user makes now, with our rate limiter sitting in between or being involved in that req,
isn't that gonna slow down the req? Because every req has extra overhead now.

So interviewer would ask about latency requirements but we would already think that we want latency to be as low as possible. Because we don't want
to add another 1 second to every req now just because we have a rate limiter. We want to pretend like the rate limiter doesn't even exist and
latency is probably the most important non-functional requirement in this context.

It's also worth talking about throughput if we have a rate limiter that's being used by a lot of services. In theory, it should be able to handle
a lot of throughput. In general, this means that we should try to design it such that it is horizontally scalable.

Another important one is storage. 

Q: What are we gonna storing? How much are we gonna be storing?

A: First of all we have rate limit rules. We talked about how a single application can have multiple rules but we're also dealing with
potentially multiple applications and all of them will have their own rules. But this is really not the bottleneck. Do you think how many
rules a single application like youtube possibly has?

10, 100, it's not a lot. So we don't have to worry about the rules part being the bottleneck for storage.

What about the number of users that we're serving? Because in our rate limiter service, for every single user, we have to identify them let's 
say with their IP address and then for that user, we have to map them to possibly the number of reqs that they've made so far so that
we know should we throttle them or should we allow their req to go through? So how much data would we end up storing for each user?

Well just at a high level without even knowing what we're gonna be designing here, an IP address has 4 bytes and let's say to store the number of
req that they've made in some time period, takes 128 bytes. So for each user, we have potentially 132 bytes and let's say we have a very large app with
one billion users, so we would have `132 bytes * 10^9 = 132 GB`. This is the amount of data that we need to possibly store for the rate limit counts
for each user(not the rules).

**Note: A billion is a giga.**

It's feasible to fit 132 GB into memory. This is the takeaway from this portion.

Last non-functional requirement is availability. In theory our rate limiter should be as available as possible, like the highest degree(five nines 
of availability 99.999).

Q: What if our rate limiter goes down? How should our application handle that?

A: There are a couple of terms called fail-open and fail-closed.

- **Fail-open** means that if sth in our system goes down, our entire system including the APIs should function as if the rate limiter(the part
that failed) never existed.
- **Fail-closed** means that if one part of our system goes down, the whole system will go down and we return 500 error in case of req, to the user.
So we're not gonna allow any req.

Q: Now which one of these makes more sense in the context of a rate limiter? Should we fail-open or fail-close?

A: If we fail-open, then most users will still be able to use youtube for example. Most users will still be able to use the app normally.
But a small subset of users possibly who are trying to abuse the app, will also be able to get through. In the context of youtube, it might be
ok for a small portion of time. Maybe the rate limiter only goes down for 5m and people will able to spam comments, probably not the end of the world.
But if we fail close, then we take down the entire system. Users can't make any reqs so this would affect all 1 billion of our users. Depending on the
application, we probably would choose to fail-open 

Tip: Most likely though availability wouldn't be a big part of your rate limiter interview.

Note: You should aim to get all the requirements from your interviewer within 5 to 10 minutes. through throughout the interview, you can continue to ask
your interviewer clarifying questions about small details.

If you don't even have the correct requirements, you're gonna end up creating the wrong design. So you want at least have the big requirements clarified
at the beginning.

### high level design
Let's propose our high level design to make sure that we have the general picture of the problem, correct and then after we done that, then
we can kinda dive into the details.

First suppose a user makes a req. The first thing that they hit, is our rate limiter service. So it kinda serves as a reverse proxy. Because that rate
limiter service is not gonna have our application logic like our APIs do, instead, that is gonna decide whether we should allow the req
to our APIs and then API can fulfill that req, or we should not send the req to the API in the first place. Because the whole point is to not
overload the APIs even though deciding whether we should rate limit or not, should be as fast as possible.

Next we need to have some type of storage to store all of our rules. These rules are defined by us as developers and these have to be persistent, most
likely in a DB whether it's SQL or noSQL probably doesn't matter because remember the scale of our DB doesn't matter, we're not gonna be storing
that many rules.

The rate limiter is mainly gonna be reading these rules. So suppose user is trying to upload a video, the rate limiter will check the rule
for uploading videos and get: Oh, they can only upload 20 videos per 24 hours.

Q: Now how does the rate limiter know how many videos the user has uploaded in the last 24 hours?

A: We can have an in-memory key-value store, sth like redis or memcached and one of the reasons we would do this is because the rate limiter 
is not only gonna be reading from that in-memory key-value store, but it's also gonna be writing. So suppose a user uploads a video for the
first time. Then the rate limiter writes that the user has uploaded one video in the last 24 hours. We wanna be as quick as possible. We know
writes typically take longer than reads and we know memory is faster than persistent storage like disk and we know that most likely the amount of
data that we're gonna be storing can probably fit into memory(redis, memcached).

We have a bunch of API microservices and the rate limiter will be the one deciding where to forward the req to? and whether it should do that at all?
In the case that the req is allowed(which we know that from reading from key-value store), we forward the req to the API and then the API will
fulfill the req through the rate limiter and send it back to the client. But in the case where we can't do this, we would just send a res to the client
saying they've been rate limited(429 code) and our APIs would never even know that the user made the req in the first place.

![](./img/1-1.png)

Having some type of rate limiter in between the user and the actual APIs is just one approach. We could just send the req to the API itself and then
the api has application logic to read from the in-memory key-value store directly and maybe in this case we don't even need to have
rules in the first place(so no need of storing rules in DB, rules are defined within the code of the API). **But** remember we're trying to design
this to be as reusable as possible.

### Design details
The most important detail is what type of algorithm are we gonna use for rate limiting?

Remember we have some type of rule like `100req/min`. But what does that mean?

One simple algo we could use is based on clock like every one minute time period, we only allow at most 100 reqs and then when the next minute
starts, based on the clock that we're using, we will allow another 100 reqs. Now in theory, a user right before one minute is ending(like one second
before a minute is ending), could send 100 reqs and then right after the one minute passed(like 1 second after that previous 100reqs), we know
there's a brand new minute, therefore he can send another 100reqs. So in the span of 2 seconds, they were able to send 200 reqs. That kinda broke
our rate limiting rule. Now obviously this is a problem, but it may not be a huge problem because yeah they sent 200reqs in 2 seconds but now they
have to wait 59 more seconds until they can send another 100reqs. So maybe over a long period of time, our rate limiter will be pretty accurate, but maybe
we don't want these types of **spikes** that I mentioned, in the first place.

But before we get to how we kinda fix this spike and use a possibly better algorithm, the benefit of this algo which is called `fixed window` algo
for rate limiting(because each window is fixed - when a minute ends(in this case), we get a branch new window), since we're storing these counts inside
of key-value store in memory, this is a simplistic approach to take. We don't have to keep track of exactly when each request was made?
We don't care exactly what time they happen, we just care they happened in this minute(window) and this would be simple to store in our
key-value store. So we would only need to include the minute for all 100 of reqs and the count of them.

Q: What would be a better algo that would be more accurate?

It's called the **sliding window**. Anytime a user makes a req like right before a minute is over, we would look at the last 60 seconds, but
if a user is making a req at the next minute, unlike the fixed window where we just disregard everything that happened before this minute,
we actually include current second and still the last 60 seconds. So everytime a second passes, we shift our window over by one.
So this is much more accurate and pretty much always enforce the defined rule. But the downside here is that we will have to store the 
**exact timestamp** that that req occurred. We can store it using a unix timestamp but **each timestamp is gonna be 32 bites or 4bytes**. So a 100
req is gonna take 400bytes which is **per user** and that could get higher depending on what type of rate limit rule that we're using.
So this is the tradeoff here.

Even though the fixed window is less accurate, it's less cost-intensive but the sliding window is much more accurate, it requires much more resources
and is more complex to implement. Because as our rate limiter reads from the key-value store, we would only be reading the last 60 seconds using
that sliding window and then possibly we would delete the rest of the data that is stored at the time that we read from it and deleting that would
save us some space but still another operation that we have to do.

### implementation
With redis, this can be accomplished with **sorted sets** even though it's a key-value store, you can still map a single key to a set of values in sorted
order. We sort it based on timestamp.

There are many more algos when it comes to rate limiting like:
- token bucket: Instead of using windows, it uses a concept of a refill array.
- sliding window counter: is a balance between the fixed window and sliding window

### The data schemas that we would use for in-memory data store
The key would be userId or IP address and we map it using for example the fixed window algo, to some count and then the rate limiter would enforce
that count using the specified rule and when a user makes a req, we increment that count. But this count applies to only the current minute(like
12:01)and when the next minute starts(12:02), we want to reset the count. With redis we can do this easily with an `expire`. We can set data in
redis to expire, so it will kinda automatically clean this up for us and then we will have reset count to 0 and then we start incrementing. So
we can set it to expire every 60 seconds or expire exactly at the next minute.
`key(userId, IP address): count`

#### Rules DB schema
- id: Every rule needs an id to be able to identify that rule. When a req hits the rate limiter maybe for uploading a YT video, the rate limiter needs to
be able to find that particular rule for uploading YT video. 
- api: The rate limiter might also need to know which api it should forward the req to? So we need a field named(api)
- op: also which type of operation are we gonna hit on that API(are we gonna upload a vid?, are we gonna comment? or ...) and we use `op` 
as the name of the DB column or field
- time_unit: Also, a column named time_unit that this corresponds to. Which could be a second, minute or hour depending on how we implement this. Also 
- request: we wanna store the number of reqs allowed. For example in 100req/min, we store 100 in this field and the minute will be stored in the
`time_unit` entry.

DB schema for rules:
```
id: string
api: string
op: string
time_unit: string
reqeust: string
```

### Scaling the in-memory data store
When it comes to scaling the in-memory data store, if we needed to store terabytes of data in memory, we would probably have to have multiple
instances of our in-memory data store(scale horizontally), but OFC a user should be able to access the **same data** every single time. If we have
our in-memory store and we sharded it, so we have like half the data stored in one instance and half on the other, if this user's data
is stored on the second in-memory store instance, hit should be able to hit that instance **every single time**. Now what algo could we
use to accomplish this? Probably **consistent hashing**.

There's also another bottleneck and that has to do with latency of our req. We want to minimize the latency. OFC about reading and writing to memory, we can't
do much better than what we have. But what about reading from rules DB? It's so much slower than reading from memory. This is our bottleneck.
How we can improve it?

If we **need** the rules to be stored persistently, we can't prevent having a DB as those API services are defined and we add rules to the rules DB,
we'll have to write to that DB, but in theory, rules won't be created super frequently, not nearly as frequently as we will be reading 
from the rules DB, so one thing that we can do here is have a middle layer in between the DB and the rate limiter and it will be our **cache**.
Periodically, we will take rules that are written to the DB and copy them to our cache and OFC it's gonna be some in-memory cache because we 
the reads to be as quickly as possible. So we will kinda replicate those rules DB to be in-memory but we also get the fact that they are persistent(since
we have them in a DB persisted on disk), but we also get quick reads.

So we as developers, as we define or update(or in general write) rules to the DB, we need some kind of **background process** that will also cache them
and for that we can have some type of **worker** that is responsible for doing that, OFC the DB can't directly write sth to other place, so periodically we
would read from DB and update the cache. We can do this pretty periodically to make sure that the rate limiter isn't reading stale rules and remember
that we're not gonna be updating the rules super frequently, so that worker that periodically updates the cache from rules DB should be good enough and
we can scale that worker horizontally as well if we really needed to.

![](./img/1-2.png)

## 2 - Design TinyUrl
## 3 - Design Twitter
## 4 - Design Discord
## 5 - Design Youtube
## 6 - Design Google Drive
## 7 - Design Google Maps
## 8 - Design a Key-Value Store
## 9 - Design a Distributed Message Queue