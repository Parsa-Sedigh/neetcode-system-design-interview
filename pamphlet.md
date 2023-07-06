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
so that it can protect multiple services or APIs. Therefore we have to implement that rate limiter as a **separate service**. 

## 2 - Design TinyUrl
## 3 - Design Twitter
## 4 - Design Discord
## 5 - Design Youtube
## 6 - Design Google Drive
## 7 - Design Google Maps
## 8 - Design a Key-Value Store
# 9 - Design a Distributed Message Queue