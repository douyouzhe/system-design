# Design an URL shortener
Examples: bitly, tinyurl

Disclaimer: this is based on my knowledge only, I do not have information about how the above apps are built. 

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Clarification](#clarification)
2. [Requirements](#requirements)
3. [Design Basics](#design-basics)
4. [Components Deep Dive](#components-deep-dive)
-----------------------

## Architecture Diagram
![architecture](/case-study/tiny-url/tiny-url.jpg)

-----------------------

## Clarification
* Examples
* Length of the shortened URL
* Characters allowed
* Traffic volume
* Read/write ratio
* Rate limiting
* Automatic redirection
* time-to-live
-----------------------

## Requirements
### Functional
1. get a short URL when giving a long URL
2. redirect short URL to long URL

### Non-Functional
1. very low latency
2. very high availability

-----------------------
## Estimates

* DAU: 10 million DAU
* Each DAU uses it 2 times a day on average
* Write QPS: 10 million * 2 / 24 / 3600 =  230
* Peak QPS: 230 * 2 = 460
* Assume Read / Write ratio 10:1 - Read QPS: 2300
* Assume TTL 1 year and each shortened URL takes 100 bytes of storage: 10 million * 2 * 365 * 100 bytes < 1TB
* Assume allowed chars in [0-9, a-z, A-Z] = 62 
* 62^n> 10 million * 2 * 365  ->  n = 6 is enough
-----------------------

## Design Basics
![basics](/case-study/tiny-url/1665715726497.jpg)

* The most intuitive solution for a URL shortening service will be using a Hash function because *A hash function is any function that can be used to map data of arbitrary size to fixed-size values*. Such transformation can be done via many algorithms, like MD5 and SHA-n. The fundamental process involves a set of operations like shift, rotate, padding and bit level manipulation. ([AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) is an interesting algorithm to know about). Some of that can be irreversible and asymmetrical depends on the application. For example, in the crypto world, we do not want the input to be reverse-engineered. As proved in the estimates session above, 6 Characters will be more than enough for scale we planned, we can alway increase the length if needed. The problem is MD5 returns a 128 bits output and SHA-1 returns a 160 bits output and this is more than what we needed. The naive solution will be of course just take the first 6 Characters of that hashed value. However, since we are mapping a large space into a smaller one, hash collision will be inevitable, that is more than one long URLs will produce the same short URL and this will be an issue if some pass that short URL to us and hope to get the original long URL back.

    <p align="center">
    <img src="/case-study/tiny-url/first.jpg" width="700">
    </p>

* Can we add some randomness you may ask? Yes, for sure we can add some salt. salt is a random string that you add to an input word, to generate a different hash that with the word alone. If we can add a random salt then ideally we can generate a different short URL every time (Please check with the interviewer because they might require the same result for each long URL then this will break that). One way to avoid this is to check the database before storing the short URL, to ensure that it doesn’t already exist and retry if it does. Bloom filter in Redis support that functionality very well.

    ```
    First, let’s add a handful of usernames as a test:
    > BF.ADD usernames fredisfunny
    (integer) 1
    > BF.ADD usernames fred
    (integer) 1

    Now, let’s run some test versus the Bloom filter.

    > BF.EXISTS usernames fred
    (integer) 1
    > BF.EXISTS usernames fred_is_funny
    (integer) 0
    ```

    This solution might be good enough for a single machine scenario but it will not scale.
    First, every server instance will be interacting with Redis, which will increase the load on Redis. Also, the Redis here becomes a single point of failure, which is a bigger issue. If Redis goes down we will have no way to recover since it is in-memory unless you want to do frequent persistence. To solve this, we can add more Redis instances as well and this will give us higher availability and better performance. But then we will lose this centralized source of truth and again we fall back to the original problem of having duplicates. 

* This problem of generating short URL can be transformed to a different problem: ID Generator. It is like generating a n Characters unique ID for each long URL and store it to later retrieval. Hence, if we can figure out how to generate Unique IDs in a distributed system, similar solution can work for our question. 
    * UUID: it is generally considered as a safe way to come up with IDs. Quote: *the annual risk of a given person being hit by a meteorite is estimated to be one chance in 17 billion, which means the probability is about 0.00000000006 (6 × 10−11), equivalent to the odds of creating a few tens of trillions of UUIDs in a year and having one duplicate. In other words, only after generating 1 billion UUIDs every second for the next 100 years, the probability of creating just one duplicate would be about 50%. However, these probabilities only hold when the UUIDs are generated using sufficient entropy. Otherwise, the probability of duplicates could be significantly higher, since the statistical dispersion might be lower. Where unique identifiers are required for distributed applications, so that UUIDs do not clash even when data from many devices is merged, the randomness of the seeds and generators used on every device must be reliable for the life of the application. Where this is not feasible, RFC4122 recommends using a namespace variant instead.*
    * Snowflake ID: Snowflakes are 64 bits in binary. (Only 63 are used to fit in a signed integer.) The first 41 bits are a timestamp, representing milliseconds since the chosen epoch. The next 10 bits represent a machine ID, preventing clashes. Twelve more bits represent a per-machine sequence number, to allow creation of multiple snowflakes in the same millisecond. The final number is generally serialized in decimal. We can tune the number of bits for each section to suit our need.
    * Assign a range of IDs: We can also partition all the 58 billion possible short URLs into sections and give it to a instance. For example, we can use round robin (%n) if we have n servers. But this will make it very hard to add or remove servers. Even simpler, we can just assign continuous range for example, 0 to 10k-1 to Server 1 and 10k to 20k-1 to server 2 and so on. We just need a external controller to control this process of assignment. We can even go with consistent hashing if you like, but normally we do not need that many servers and that level of complexity for a URL shortener. 

* Purge outdated URLs

    One of the good-to-have features is to clean up outdated URLs so that we can re-use them.
    
    * Redis TTL: we can use the built-in TTL feature in Redis to automatically purge outdated records, but again this introduces dependency on using Redis as a centralized datastore.
    * Active DB scan: we can have scheduled DB scan to purge outdated records.
    * Lazy: we can check the time_created timestamp whenever the records are fetched and decide whether to purge or not.
    * *note: they can be used together* 
* Rate limiting
    
    This service will be prone to DDoS attack and we should have a rate limiter (another design question we will cover later). Some designs include a User Service so that the usage of our service can be limited. Each request will include a token specifically for the user and we can monitor the rate. 
* Analytics - WIP

    most visited domain - cache - longer TTL
    response code 301 vs 302
-----------------------

## Components Deep Dive

### Short URL Service
ID generator

### Token Service
assign ranges
zookeeper with DB


-----------------------
## Reference

https://en.wikipedia.org/wiki/Hash_function

https://en.wikipedia.org/wiki/Advanced_Encryption_Standard

https://www.enjoyalgorithms.com/blog/design-a-url-shortening-service-like-tiny-url

https://medium.com/@souravgupta14/url-shortener-with-zookeeper-aa38174c598b

https://redis.com/blog/bloom-filter/

https://www.techtarget.com/searchsecurity/definition/MD5

https://infosecscout.com/md5-salt-hash/

https://stackoverflow.com/questions/1155008/how-unique-is-uuid

https://en.wikipedia.org/wiki/Snowflake_ID