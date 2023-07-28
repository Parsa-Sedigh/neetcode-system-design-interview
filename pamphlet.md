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

**Note: A billion is a giga(so `1 billion bytes = 1 GB`).**

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
Url shortening service like `bit.ly`.

We're gonna be focusing on the core functionalities of a URL shortening service.

### Background
long url: https://neetcode.io/courses/system-design-for-beginners/1

short url: https://tinyurl.com/9a3v32bp

When we open up the short url, it's gonna redirect us automatically to the long url.

### Functional requirements
We want to be able to map long URLs to short URLs and then using those short URLs, we should be automatically redirected to the long url.

Obviously the path in the short url should be relatively short(like `9a3v32bp`), that's the whole purpose. So while it would be
even more user friendly that that path in the short url to actually be a meaningful word like `happy` or ..., we might not always be able to
do that, we might need to have some jargon or some scrambled characters as path.

- So let's say the restriction for us is that we want that random path to be 8 characters to keep it user-friendly.
- to make it simple, let's say we don't include the case where users are able to specify their own string which could be user-friendly, so
sth like (the word `happy`), because it's not too different for our design
- in theory, a user could also delete a URL OFC only if they own it. So we need to keep track of the ownership of these URLs. But for simplicity,
we won't consider the case where users can actually delete those URLS, even though it doesn't really change our design too much
- our URLs do expire, maybe by default it's 1 year. So every URL expires after 1 year but maybe users can specify a longer time period that they want
their URL to expire like 5 years, 10 years. So we do need to think about this in terms of a long-term mindset. We don't want to run out of resources.

**Q:** A last clarifying question would be to ask: what would happen if multiple users had the exact same long url and they were trying to map it to a
short url? Should it always map to the **same** short url? What difference would it make if a user is trying to shorten a URL
but there's already a shorten version of that, why not just give them the same short URL?

**A:** Well remember we do have the concept of ownership. What if I set my url to expire but somebody else had the same long URL shorten to
the same shortened url? Then the shorten URL expires but the other user wanted that shorten version to expire in a future date. So even if the
same long URL has already been shortened, we create a unique short URL every single time.

The answer of this question is gonna affect the outcome of our design.

### Non-functional requirements
- obviously we want this to be as available as possible and as fault-tolerant as possible. So if components of our design end up failing,
we do not want the entire system to shut down.
- the point of a short url is to make it more user-friendly so we definitely don't want to add additional latency, because at that point you might
stick with the long url, if the short url is gonna be so much slower. So we want minimal latency as possible.
- the scale that we're gonna be dealing with: in the real world, with url shortner, what do you think we're gonna be doing more? writes or reads?
In this case, a write will create a short URL and a read would be **using** a short URL(and getting what the long url it maps to, is?). So obviously
reads are gonna be a lot more frequent, because you're probably not gonna create a short URL just to use it one time, that would mean
a one to one ratio, but reads are probably gonna be a lot more frequent
You can already guess this, but to clarify this with your interviewer maybe they say: we're gonna have a ratio of 1 to 100, so 100 reads for every write.
- how many urls are we actually gonna create?
This is generic enough service, we could be dealing with a lot of short URLs. Let's say the number that your interviewer gives you is sth like
**1 billion short urls created per month**. This is a very large number. The first thing that's going through our mind is we only have
8 characters, are we going to run out of space? To ensure that we don't, we should use as many digits or characters as possible for every single
position. Let's say we can use digits 0-9 and a-z and A-Z , that gives us `62` characters to choose from and `62^8` is gonna be a very large 
number(this would be hard to calculate on a back of an envelope or in your head). But remember 60^2 is 3600 and we know: 3600 = (60^2)^4 = 60^8 .
We know 3600^2 is gonna be at least 9 million(because 1000^2 is a million and you have 3^2 as well here, so 3600^2 is gonna be a million).
Now 1M^2(a million squared) is roughly a trillion. So `1T` is quite a lot of possibilities we have with short urls. If we're creating `1B`(one billion)
per month, we're not gonna run out for `12B` per year. So we would have at least almost 100 years. So this is pretty dang good. We don't have to
worry about that.

One billion writes per month it's gonna be around 33 Million writes per day and for each second, it's gonna be: 33M/100000s and the result is gonna
be roughly 330 writes per second and if we round this, it's gonna be 300 or you can round up to **400 writes per second**.

**Note:** We consider a day roughly as 100000 seconds.

`400 writes per second` is not a ton, but still a pretty large amount. Now if we have a ratio of 1:100 w/r and the number of writes per second was
300, we're gonna have `300 * 10^2 reads per second`. So we definitely have a lot more reads and this number for reads is actually quite a lot of
reads.

So we're gonna need to be able to handle this scale and try to identify the bottlenecks.

- **storage**: When we talk about storage, the writes are clearly more important. If we have `1B` writes per month, how many urls are we storing
per month? 
A: We know that `https://tinyurl.com` is kinda a base domain, it could be anything, but we probably don't need to store the base domain for every
single URL, because clearly it's gonna be the same. We just store those 8 characters and that's gonna be 8 bytes. So does that mean we have
8 billion bytes? Or in other words 8 GB? No, we might need to store more than just those 8 characters. We might need to store some more metadata.
Of course, we're gonna store the long URL that the short version corresponds to. So how many characters could that long url be? Probably on average,
100 characters for storing it would be fine, but it could be a lot larger. Could be 1000 characters if there's some super long URL.
But let's keep the average case in mind(100 chars, so 100 bytes).

So we have the total of about `108` bytes per user. But maybe we want to store additional metadata, maybe userId and ..., let's say 1000 bytes per URL.
As we'll see though, most likely it would actually be significantly less than this on average.

**Note:** 1 character = 1 Byte

**Total storage:** If we have 1000 bytes per URL and we have `1B` URLs created per month, we're gonna have `1000 * 1B` that's gonna be a trillion,
so 1TB which is a terra byte. So a terra byte per month. This is reasonable, it's a lot smaller than the scale of a social media network.
So we're not storing that much data when you think about it. A terrabyte per month over the course of 10 years, will end up being a pedabyte,
not that bad.

**Note:** A trillion bytes is a terra byte.

### High level design
Q: What type of storage solution would we want to use?

A: We talked about how we have a large amount of data, but it's not super large. The big thing for us is actually the amount of reads that we're
gonna be doing to that storage solution. We're not gonna be writing frequently. So how are we gonna be able to handle this scale?

We can use some type of NoSQL DB, since they're specifically designed to be able to handle a large amount of scale. Now they don't come with the
**ACID** properties and they're definitely not relational but we're not gonna be storing a lot. At the very core, we're just gonna be mapping a 
long url to a short url and storing them together with maybe additional metadata sth like the userId and ... , but it's not like we're gonna
be doing a bunch of joins and complex queries. We don't need those ACID rules, we don't need atomic transactions and actually with **BASE** properties
with NoSQL, we have **eventual consistency**, so when we create a URL, it might be that somebody reads one of our DB replicas and maybe that DB
replica does not have the most up-to-date URL just yet. That's fine. Because we know eventually all of our replicas will be consistent, it's not like
we're gonna be writing super frequently, eventually the replica will have that up-to-date url and it might take a few seconds and that's perfectly
fine, but after all the replicas have the URL, it will be working, it will be able to handle that scale.

---

The main responsibility of this NoSQL DB is gonna be to store those URLs.

#### Main user scenarios
##### Creating
The main use scenarios we're gonna be doing is a user is going to possibly create a URL. They're gonna hit some server and suppose we have
some type of way of generating a URL. So let's say we have another server besides that mentioned server and that server is named `url-generator`
and then from the server(not the url-generator service), write to the DB and that DB may not be a single node DB, OFC there could be replicas and
it could be distributed and all that and that would be the scenario where a user is writing or creating a small URL given a long URL

![](./img/2-1.png)

##### Reading
User hits the server and given some short url, we want to find in the DB what is the corresponding long URL and return that to the user and then the
user will be redirected to that long url.

Since we know we're gonna be doing things very read-heavy, OFC we want the generated tiny URLs to be persisted, but it's probably good
for us to have some sort of caching layer in between. So server first hit the cache and see if the tiny URL is there, because
most likely, there's gonna be a few tiny URLs that are used a lot more frequently than others. So having that subset of URLs in the cache,
will create a much better performance and it will speed the redirect up a lot.

**Note:** This cache should OFC be an in-memory cache because that's kinda the whole point of making it faster than reading from disk on our DB.

### Design details
We haven't talked about what the req would like look to the server and we especially have not talked about how the URLs will be generated?
Also we'll talk about how to scale this, but there are a few other points that are more important for this problem than the scale.

---

Q: For the caching that we have, what type of algorithm could we have? and what size would cache actually be?

A: We talked about how `1TB` of URLs will be created per month. Now OFC the reads will be 1000 * 1TB, but that doesn't mean that we're gonna have
that many **unique URLs**, that doesn't mean we need `1PB` of data for the amount of URLs that are gonna be read per month or in some type of time period.
**So a reasonable size for our cache would be sth like `256GB`**. This number is reasonable for memory, though it might be pretty expensive, but for a 
system like this, it's not the end of the world. Our cache could be distributed, we could implement sharding(for cache) and
OFC having it(cache) replicated as well.

Q: What type of algorithm would we use for our cache? What type of eviction policy would we use?

A: We could try `LFU`(least frequently used) or `LRU`(least recent used). `LRU` would be a bit better because we could have some tiny URLs
that are used really frequently like a million times, but then after a while nobody is using them anymore, maybe there was some type of trend
or ... , but that URL has not been used for a long time but has a large count, then it would end up staying in the cache and it's kinda useless,
it's just taking up space. So we prefer LRU, because we only want the active URLs to be in the cache, so we can avoid the cache misses.

**So the scenario would be:**

A user is trying to find the long URL given some short URL(**user wants the long url by giving us the short URL, because he wants to be redirected to
the original url which is the long one**). The server will first read from the cache, if we have a cache hit, we will have the long url and we return
it to the user. If we don't, we have to go read from the disk(database) and that will end up taking longer.

#### How the URL redirection would actually work?
What would happen if the user types `tinyurl.com/9a3v32bp`, yeah they will be redirected but how does that work?

The response that we'll get from sending a req to the short url, will have a status code of 301, meaning a resource has been moved - the resource
does exist but it's not gonna be found at the original url(which in this case is the short url) and in the response headers, there's a header
called `location` which will contain the location of the resource(long url). With that header, the browser will redirect us to the value in the
`location` header automatically.

What type of status code could we get?

We could get 301 or 302. The main difference is 301 tells you that this resource has been permanently moved to the URL specified in the `location` header of
response. So what your browser will do is it will cache this information. It will say: Ok the short URL will always map to the long url, so next time
you type the short url, the browser not even gonna ask the short url related server(specified in the url we asked) to return us the long url,
the browser knows automatically the long url related to this short url you typed. So after the first req, all subsequent reqs will be as if you're
typing in the long url, because it'll happen immediately by reading from cache. This was for 301.

With 302, it implies that this(short url that you typed) has **temporarily** been moved to the value in the location header of response. So it(browser)
won't cache it and every single time you type the short url, it will ask the url shortner server what's the long url? Maybe by requesting
the long url next day(with the same short url), the long url would be different(that's why the server responds with 302 - because it's temporarily).

In this case, it makes sense to use 301 in the response status code. But the reason 302 might be better is if for some reason, tinyurl wanted every req
to go through them(tinyurl servers) maybe for analytics purposes or sth like that.

#### How are we gonna generate these URLs? How are we gonna generate an 8 character string?
For each character we have 62 possibilities. How can we make sure we do it in a way that we don't have collisions? Because **one of the problems with
hashing is we could have two different stings that map to the same 8 character string that is used to create the tinyurl short url(collision situation)**.

So a problem with hashing is having 2 different input but the generated hash could be the same.

**With hashing we can also guarantee that the hashed string is going to be exactly 8 characters long.**

One thing we can do is we can generate all possible strings of length 8, or at least most of them and store them in a DB. So maybe our url generator
will have a DB and it will have all the keys(the extension path of tinyurl - the path in the short url). This is surprisingly simple but it 
actually works, we will not have collisions if what we do is the user makes a req to create a tinyurl. We need to find some key that has not been
used yet, so that we can say: that key maps to the url that the user specified. So our server will just fetch one from the URL generator service
(at the high level).

Note: Our URL generator service can just be a DB, if we pre-populate it, or we could have a service running that is populating the DB which holds
the keys(paths of tinyurl) at some point as we start to run out of keys, because we don't necessarily need to generate all `62^8` immediately.
To make this even faster, since we might have to be reading from disk(DB) and OFC we're not gonna need to read all the keys at the same time, we just need
a subset of them and it doesn't really matter which subset we take as long as they have not been used yet, so our URL generator service can have a
cache inside of it where it can just load some keys into memory(subset of keys in DB), it can: 
1. **periodically** do it
2. or as it starts to run low on the remaining keys in cache, it can make a single bulk req to get a bunch of keys from DB and load it into memory(cache)
and then it will be able to use those keys.

With this cache, it will be able to use those keys a lot faster. Everytime we generate a short url, we won't have to read from disk(DB) and this will
make things faster.

When we do take that subset of keys from db and load it in memory, remember that this url generator service may not be a single instance server,
this entire service could have multiple url generator services and remember **the whole thing we're trying to avoid, is we do not want to
reuse a key**, so when we take that subset of keys, we want to mark them as used in our DB. For example our DB has a column named **key** and another one
called **used**

| key | used |
|-----|------|
|     |      |
|     |      |

and everytime we fetch keys from db into cache, we wanna make sure that they are unused keys. If a key has been used though, we don't necessarily
need to delete it, we can still keep it in the DB as long as we mark it as used. The reason we do that is after a URL expires, we can then start
reusing it, we can mark it as unused and this helps us from running out of keys.

**About the problem on concurrency:** What would happen if multiple URL generator instances were reading the same key? What if two users are
making a req to create a tinyurl and at the same time, they both end up with the exact same key(they both get the same key that we generated and have
put in DB), that would obviously be a problem. Because one tinyurl can not map to two long urls. So one of these users would not get what he
asked for, things would break, the functionality would not work for that user on that given request.

How can we prevent this?

What type of DB would help us prevent this? SQL DB or NoSQL?

It's not like we have a lot of data here, it's not like we have a relational data, so we might not need SQL, but don't forget
that SQL comes with ACID properties and in this case, which one of these properties would help us in terms of the concurrency issue?

A: Atomic and isolation properties are gonna help. Because the transaction that we're gonna be doing with our DB is reading a key that has not been used
and in the same transaction, we're gonna be marking that key as used. So this transaction is atomic(it's all or nothing), it's gonna execute
everything or it's not gonna execute anything and remember two(or more) of these transactions could happen concurrently, but they're gonna appear
as if they were happening one by one, that's what **isolation** means. So one of them is gonna completely finish(we're gonna query an unused
key and then mark it as used) and then maybe the next transaction is gonna execute and they're gonna get an unused key and since the previous
transaction has completed, it can not possibly be the one that the first user got, because they already marked it as used.

Consistency for ACID means we follow the constraints like primary keys, foreign keys and ... .

So use SQL DB for keys DB.

### Fault tolerance
What about the fault tolerance of a URL generator service? What if we have multiple instances, one of them loads a bunch of keys into the memory,
it crashes? It marked those keys as used, but they never end up getting used(all the keys in the cache will be marked as used in the DB).

In this case, we kinda see a tradeoff between loading the keys in the cache because of speed, but if we interact with the DB which is slower,
but we do get the ACID properties. Remember we have 300 writes/second, we should be able to handle this with the DB.

An even better thing to do would be to use the in-memory cache but to prevent concurrency issues, we implement some type of locking to make sure
that a single key can not possibly be used by multiple users, it can't even be read by multiple users. So on a request, as soon as we identify
the key(path segment of tinyurl) for that req, we will lock that row in in-memory store.

![](./img/2-2.png)

### How would we be able to handle deletes? How handle a URL expiring?
First of all how would we even detect if a url expired?

One way to do it would be to have a cleanup service. It may not happen immediately. So if a url is set to expire at a particular time,
it's not the end of the world if it takes maybe 5 extra minutes for it to be deleted(you can discuss the duration here). 

So a cleanup service which periodically(maybe every 5m or an hour) reads from the URL DB and filters them by the expiration date and reads all the rows
that have been expired, what it will do is it first take the key and goes to the keys DB(the SQL DB) and frees it, so it will 
mark that key as **unused**, so that it can possibly be used again. It will also delete that URL from the URL DB, since we don't need to
store the long URL anymore that the key was originally mapped to.

So the cleanup service does it's work asynchronously, it doesn't need to happen on every req that is made to the server by the user.

### How would we handle the scale with this design?
Our design is implemented such that it can actually handle the scale that we need to.

The easiest thing to do would just be to introduce a load balancer in between the client and the server. Because we could have multiple instances
of the server and we could load balance them and OFC we will have multiple replicas of the URL DB and actually not just replicas, we might need to
partition the data of URL DB, so we could obviously have a load balancer between the servers and the URL DBs.

How would we partition the URL DB?

The easiest way would be to hash the user id and using consistent hashing, if we have multiple partitions of the URL DB, we can guarantee that the same
user will be hitting the same partition and all the URLs for that particular user id will be stored on a single partition and this should be able to
handle scaling up and scaling down.

We could also introduce a load balancer layer in between servers and the cache.

The thing you definitely should not miss for this problem is that you would need a NoSQL for the URL mapping storage(URL DB).

**Q:** What data would we have in that NoSQL DB(URL DB - in the middle of the pic)?

**A:**
- The user id: assuming that the user needs to signup or we have some type of key to identify a user before they actually start creating URLs.
- long url: 
- path segment of tinyurl(key): The key is enough for the server to generate the entire url, because it can just use the base url(tinyurl.com/<key>)
- expiration date: Which is needed by the cleanup service
- other fields like creation time of that url and ...

This was one way of designing tinyurl.

**So the URL DB(s) is gonna be NoSQL and the keys DB is gonna be SQL.**

## 3 - Design Twitter
## 4 - Design Discord
## 5 - Design Youtube
## 6 - Design Google Drive
## 7 - Design Google Maps
## 8 - Design a Key-Value Store
## 9 - Design a Distributed Message Queue
How they can be implemented? How can we scale them?

### Background
There are many kinds of message queues and they had a lot of evolution over the years which kinda blurred the differences between them.

#### one-to-one pattern

Talking about more of a traditional message queue, sth like rabbitmq which started out as sort of a one to one mapping. So you produced messages on
one side and then consume them on the other side. So it was just one system creating messages and adding them to the queue and then one consumer
taking those messages. So kinda how a queue DS works in code(currently rabbitmq supports more than just this, this is just history).

So this was the one-to-one pattern.

#### pub-sub pattern
Things get more powerful when talking about the pub/sub pattern which is also a pattern that can be applied to code, but in terms of distributed
systems, they're also helpful and have additional complexity with them.

At a high level, we have publishers and subscribers. But these two are actually part of **our** application. So we could have some servers
as publishers and possibly they are payment transaction servers, like everytime we receive a payment, we have our application servers that do sth
with that payment info. Maybe these servers can't process the payments immediately. So we want to process them asynchronously, because if we can't
handle the traffic of like the payments, but we keep receiving payments, then eventually our servers might miss some of the payments or the servers might
crash and not be able to process them. If it takes a few extra seconds to process them asynchronously, that won't be the end of the world.

So what we do instead is everytime the servers receive a payment, we send those payments to our pub/sub message queue system. Now there are topics 
are sort of entities within the pub/sub system. So first of all, the messages that we get, will be persisted in the message queue, in 
some type of persistence storage they'll be stored so we definitely will not lose them and then eventually, messages will end up going to
subscribers and those subscribers may be the ones that actually process the payments.

Note: The subscribers will get those messages **as needed** meaning if they can't process those messages immediately, it's OK, because 
the messages can still be stored within the pub/sub system.

Note: It's up to subscriber to which topic(s) it consumes. One of them could receive messages from multiple topics, one of them only receives
messages from one topic. This also adds some decoupling to our system, where the publishers don't need to know about the subscribers. Only the
pub/sub system needs to know about publishers **and** subscribers. This also allows for additional **scaling**.

subscriber is pretty much the same as consumer.

- In kafka: producers -> consumers 
- publishers -> subscribers

Note: Kafka technically is not a message queue, it's more for event streaming, but the lines are sorta blurred between the message queue and
event streaming systems, they support a lot of the same functionality, for example kafka does support fanout, persistent storage(messages can be
retained in kafka for a certain period of time).

### Functional requirements
1. we want to support **fanout**. Meaning with our message queue, we want the same message to be received by multiple subscribers a.k.a consumers.
2. being able to **retain** messages. But in our case, we're gonna simplify and only retain the messages until they're delivered.
3. we want to support **at least once delivery**. Why this is important? Because we want every single message that's received by our pub/sub system
to definitely deliver that message at least once. Because if we can't guarantee that, which means some of the messages, are never going
to leave aka, some of the messages are gonna be lost and we definitely don't want that to happen because that's one of the points behind using a
message queue. There are other types of delivery with message queues, things like **at most once delivery**. Why that would be important?
Because if we don't want the same message to be sent multiple times, because you don't want to end up with duplicate messages. With payments that
could be important, - because you don't want the same payment to be processed multiple times. So you want a guarantee at most once delivery, but
if we have at most once delivery but not at least once delivery, that brings us to the problem where a message might get dropped. So
there's also a third type called **exactly once delivery** which means every message will be received and get sent out exactly once, so we can
guarantee that every message will be delivered exactly once, so we won't end up with duplicate messages and we won't end up with messages that
get dropped, but we're not going to support this in our design just for simplicity. This one is actually a pretty complicated problem to solved and
there are a lot of tradeoffs that can be made when trying to design this.

**Note:** The at least once delivery is the most important type.

![](./img/9-1.png)

### Non-functional requirements
1. needs to be **scalable**. Ideally we should be able to horizontally scale our message queue. If we add more and more publishers and more and more
subscribers, we should be able to scale out our system to be able to handle the traffic.
2. we want **persistent storage**, we do not want the messages to be lost(this was already in the functional requirements, but it's worth mentioning here
as well)
3. we want high throughput

### High-level design
Remember that publishers are servers that our application owns, but they're not necessarily a part of the message queue that we're designing, but OFC
they're a very important part because the message queue doesn't really do anything if we don't have publishers and subscribers.

First of all remember publishers are going to be sending messages into the message queue and then subscribers are going to be receiving messages
from the message queue and somewhere in between, the messages are going to be stored to make sure that we never lose them.

So at a high level, we know we're going to have some servers as **publish forwarders** that are part of the
message queue(blue ones at the top of the message queue) and another layer of servers as **subscriber forwarders**(purple ones).

The idea is: As publishers create messages, they will send the messages into our pub/sub system(in other words, the message queue) and they'll be
received by the publish forwarders. Depending on number of messages that we're sending(producing), we may need to scale the publish forwarders.
But remember the **same** message produced by a publisher, might get sent to **multiple** subscriber publishers. So it's important to
be able to independently scale the subscriber forwarders which will be sending the messages to the subscribers. Maybe the same messages needs to
get sent to a bunch(multiple) subscribers. So we'd need to scale subscriber forwarders **more** than we scale the publish forwarders. Or OFC we could have
the opposite case where we have a bunch of publishers but all the messages are getting sent to a **single** subscriber forwarder and we only have
one subscriber. Then we need to scale publish subscribers more than forward subscriber.

As those messages come into the publish forwarders, those messages will then be stored somewhere, possibly a DB, which stores all of the messages
and as soon as a message is stored(persisted), we know the message should never get lost at this point. So typically what's done is 
the pub/sub system(actually a publisher forwarder which is part of the pub/sub system) will then send a message back to that publisher, telling it that
the message you just sent me with the related id, has been stored. So basically one of the publish forwarders(the same one that received the message
in the first place) would send an acknowledgment(ack) to the publisher saying: Your message has been stored, you don't have to worry about it anymore, you
don't have to try to send it to me again, because if the message was never received, we know the publisher doesn't want the message to get lost, so it
will probably try to send it again. This is all part of the **fault tolerance** that the message queue should provide.

After the publishers receives the ack, then it can send different messages(if it has any).

Generally, sending messages by publishers is done over **HTTP**.

We don't get into the security, but probably we wouldn't want anybody to just be able to send a message into the message queue for obvious reasons like
being able to take our servers down, store too much data and ... .

Once those messages are stored, subscriber forwarders can read that data(messages in the DB) and then send it to their relevant subscribers and then
once a subscriber has seen a message, then they will send an acknowledgment back to the pub/sub system and once a particular message has been read
by all of the subscribers, then we probably don't need to store that message anymore. If a message has been acked by every relevant subscriber,
then we can probably remove that message from storage. This will ensure that we support **at least once delivery**. Because if we guarantee that
everyone(subscriber) has already seen the message, then we know we're safe to delete it and we won't end up taking an infinite amount of storage.

#### Secondary data storage
We might need a secondary data storage system and this could be for metadata. Specifically we might want to store information on how many
subscribers we have.

Note: What are topics in message queue?

**Topic:**

A topic is a way to organize messages. So when we receives messages into our pub/sub system, we may have a single topic for payments for example.
All payment info(messages) will get sent to this topic. If we ended up having a very large amount of messages stored in the messages DB, we probably
want to shard it based on the topic, if possible. But even a **single** topic may have too many messages in the messages DB.

**Subscription:**

When messages are created and sent by publishers, they will go to a topic(they could go to multiple topics, if the publisher decides that, but anyways,
they will be organized based on a topic). Now when messages are consumed and received by subscribers, they will go through a subscription.
One of the major things about a subscription is it's used to fanout information from a topic(to fanout the messages).

For example if we wanted two specific subscribers to receive messages from the payments topic, we could create two subscriptions for the same topic.
So we have two subscriptions for that topic and each of those subscribers(servers) listens to a single subscription. So this was a way to
fanout the messages and also this is a way to guarantee that each message is received by both of the subscriptions and both of the subscribers.
Maybe one of the subscribers is for processing the payment and one of them is for fraud detection. We definitely don't want the messages to
only be received by one of them, we want **both** of them to receive and this is a way to do that.

These topics and subscriptions are sort of abstracted.

We'll store which topics and subscriptions exist in the metadata DB and this will allow us do this:

When a message has been acknowledged by the subscriber that it received the message, we can go to the metadata DB and see that a certain topic
has for example two subscriptions and we can record: Ok, one of the subscriptions has already seen the message(since it acknowledged it), but another
subscription has not seen the message, it hasn't sent the acknowledgment yet, so at this point we would know we can't delete that message yet from the
messages DB, but maybe as soon as the other subscriber sends the acknowledgment, then we see that both of the subscriptions on that topic have
seen that particular message, so then we could remove that message from the messages DB. It doesn't need to be stored anymore because all of the
relevant subscribers(subscriptions of that topic) have already seen it.

Ideally we'd also want the messages to be FIFO. So ideally we'd like to have ordering with the messages. So in the same order that the messages
are received from publishers and then stored into the messages DB, is the same order that we'd like to deliver the messages.
So at a high level, the publishers can create some type of id which is tied to the timestamp that those messages were received and those messages can be
stored in the messages DB using that id.

Since we don't need to do complex joins, we can probably just stick with a NoSQL DB for storing the messages, because a message is typically
like a JSON object. Probably a relational DB would work just as well here. A reason to go with a SQL DB in this case would be for the
ACID properties. In this case, if we possibly store the messages and just auto increment the id, then we can kinda be sure that the messages will
be stored in the order that they were received and in that case, the **isolation** property of ACID would be important, because we will be having
multiple publish forwarders concurrently writing to the messages DB and we want to write those messages in an isolated way.

There are other policies we can have for retention:
- we could design our system such that we can store messages even if every subscriber has received those messages. We could set like a 7day policy
even after every subscriber has acknowledged them.
- or we could set a 7day policy where even if some subscribers have **not** acknowledged them, we will still those messages anyway

So we can have flexibility in message queues including pub/sub and real message queue systems generally support this functionally.

**Q:** What about replication of our messages DB? When we take a message and we just store it one time in a single DB and then we send the acknowledgment to the publisher saying: hey, we guarantee
we're going to deliver this message or at least it will never be lost. How can we make that guarantee if we're just storing it in a single DB?
What if the DB crashes? What if the data center explodes? All of the data is definitely going to be lost.

**A:** So probably we should replicate the messages DB among additional database instances. But how many times should we replicate it?
Probably at least 3 is a good rule of thumb. Especially if these 3 are located in different regions of the world.

Q: When should we send the acknowledgment to the publisher? If we take that message and we store it in just a single DB replica and 
we know it eventually is gonna replicate it, but it might not! What if the other replicas crashed or sth happens, at what point can we
say we safely tell the publisher we acknowledged that we received the message and it's been stored? Should we just store it in a single replica?

A: Well we should definitely store it in at least one replica, but should we wait until it's been stored in 2 replicas? Or should we wait
until it's been stored in all of the replicas? Because this is a tradeoff between latency and fault tolerance. If we only store it in a single
replica before sending the acknowledgment, then we're lowering the latency but that's also not as good for fault tolerance, because we have not
stored it, we have not replicated that yet, so at some point the DB that has the last message could crash or explodes and we didn't replicate
it but we still sent the acknowledgment, so that's not good. Maybe we can wait, we can increase the latency, we can wait a little bit, until
that message has been stored at least in 2 replicas(2 times). This is good for fault tolerance. Because we know we replicated it in two different
DB replicas. We could wait even longer on average until it's been replicated 3 times. So this is a tradeoff worth discussing.

Depending on the throughput of our system and what we're trying to optimize for, we might be willing to be flexible on the number of replicas to store
the message before sending an ack to the publisher.

#### Pull vs push based delivery
When subscribers receive messages, we can do it in two different ways:
- pull based: The subscriber will be pulling messages from our pub/sub system. So the subscriber will be the one initiating the request, it will say:
I'm ready to proces more messages, let me make a request, let's see if there's any messages ready. One obvious downside is that what if
there aren't any messages ready, then this subscriber just made a request for no reasons and it doesn't even know when the next message
will be ready, it could not be ready for hours. What's gonna happen in the meantime? Well, depending on the arcadance that this subscriber makes reqs on,
the frequency could be every 5s, that's gonna be a lot of wasted reqs, or we could make it a lot slower we could make it 30s and then the subscriber
will send a new req. The downside of this case is that it may increase **latency**, there might be a new message ready, but we won't see it for another
29s because we have to wait. So obviously this would be slower but the benefit would be that the subscriber can process data when it sends the req
for pulling messages. So the latter case might be suitable for **batch-based processing**.
- push based delivery: The message will be sent from the subscriber forwarder to the subscriber as soon as the message is ready. The downside here is
what if the subscriber is not ready? Well that's fine, when the subscriber forwarder will send the message, it will never receive an acknowledgment
from the subscriber, therefore maybe 10s later it will try to resend the messages. The downside here is that if it just keeps resending the message
and this subscriber is not ready, that's also a lot of wasted time and if too much data is sent, it could end up overwhelming(overloading)
the subscribers, though they'll probably never crash, it's just that they'll just keep getting sent messages and they can't really do anything with
them until they're done processing the current messages. The good thing about this is it can minimize the **latency**. So if we're going for 
**stream-based processing** where we want the messages to be sent to the subscribers as soon as they're ready, this is the best thing.

Generally, you don't need to decide on which delivery type(push vs pull). Because most message queue systems will support both pull and push based
message delivery.

This was a high-level design of a message queue pub/sub based system.

If we have persistent data, you definitely end up needing a DB type system. Kafka which is an event processing message queue, ends up storing
messages in sth called a write a head log(WAL). The correct term for kafka is message broker. But the line is kinda being blurred between kafka
and traditional message queues.

Note: In between publishers and pub/sub system, generally there will be a load balancer because those publishers could be located all around the world
and we might want to load balance them and send their req(sending messages) to the nearest publish forwarder and similarly with the
subscriber forwarders and the subscribers as well(we would have a load balancer there as well).

Note: For us to store info about what subscribers we have(metadata store), which subscriptions we've created, how many subscribers we have, which topics 
we have like the relationships between them, we might want to use a highly available key-value store, sth like zookeeper or etcd and even more
importantly, with this metadata DB, we probably want another service sth like a control plane or a coordinate service to manage
that, like manage the state of the subscriber(subscribing) forwarders, the coordination between assigning subscribers and subscriber forwarders
and manage the health of our messages DB(Msgs DB), maybe this service can determine how to shard the data in messages DB and partition them as we need to
and possibly this would also manage any req we receive to update the state of our message queue for example if we want to change it from
pull based to push based and ... .

![](./img/9-2.png)