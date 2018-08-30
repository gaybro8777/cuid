# cuid
[![Travis-CI](https://travis-ci.org/ericelliott/cuid.svg)](https://travis-ci.org/ericelliott/cuid)

Collision-resistant ids optimized for horizontal scaling and binary search lookup performance.

Currently available for Node, browsers, Ruby, .Net, Go, PHP and Elixir (see ports below -- more ports are welcome).

`cuid()` returns a short random string with some collision-busting measures. Safe to use as HTML element ID's, and unique server-side record lookups.

## Example

ESM:

```js
import cuid from 'cuid';

console.log( cuid() );

// cjld2cjxh0000qzrmn831i7rn
```

Node style:

```js
var cuid = require('cuid');
console.log( cuid() );

// cjld2cyuq0000t3rmniod1foy
```


## Installing

```
$ npm install --save cuid
```


### Broken down

** c - h72gsb32 - 0000 - udoc - l363eofy **

The groups, in order, are:

* 'c' - identifies this as a cuid, and allows you to use it in html entity ids.
* Timestamp
* Counter - a single process might generate the same random string. The weaker the pseudo-random source, the higher the probability. That problem gets worse as processors get faster. The counter will roll over if the value gets too big.
* Client fingerprint
* Pseudo random (`Math.random()` in JavaScript)

## Fingerprints

**In browsers**, the first chars are obtained from the user agent string (which is fairly unique), and the supported mimeTypes (which is also fairly unique, except for IE, which always returns 0).
That string is concatenated with a count of variables in the global scope (which is also fairly unique), and the result is trimmed to 4 chars.

**In node**, the first two chars are extracted from the process.pid. The next two chars are extracted from the hostname.


## Motivation

Modern web applications have different requirements than applications from just a few years ago. Our modern unique identifiers have a stricter list of requirements that cannot all be satisfied by any existing version of the GUID/UUID specifications:

### Horizontal scalability

Today's applications don't run on any single machine.

Applications might need to support online / offline capability, which means we need a way for clients on different hosts to generate ids that won't collide with ids generated by other hosts -- even if they're not connected to the network.

Most pseudo-random algorithms use time in ms as a random seed. Random IDs lack sufficient entropy when running in separate processes (such as cloned virtual machines or client browsers) to guarantee against collisions. Application developers report v4 UUID collisions causing problems in their applications when the ID generation is distributed between lots of machines such that lots of IDs are generated in the same millisecond.

Each new client exponentially increases the chance of collision in the same way that each new character in a random string exponentially reduces the chance of collision. Successful apps scale at hundreds or thousands of new clients per day, so fighting the lack of entropy by adding random characters is a losing strategy.

Because of the nature of this problem, it's possible to build an app from the ground up and scale it to a million users before this problem rears its head. By the time you notice the problem (when your peak hour use requires dozens of ids to be created per ms), if your db doesn't have unique constraints on the id because you thought your guids were safe, you're in a world of hurt. Your users start to see data that doesn't belong to them because the db just returns the first ID match it finds.

Alternatively, you've played it safe and you only let your database create ids. Writes only happen on a master database, and load is spread out over read replicas. But with this kind of strain, you have to start scaling your database writes horizontally, too, and suddenly your application starts to crawl (if the db is smart enough to guarantee unique ids between write hosts), or you start getting id collisions between different db hosts, so your write hosts don't agree about which ids represent which data.


### Performance

Because entities might need to be generated in high-performance loops, id generation should be fast. That means no waiting around for asynchronous entropy pool requests, or cross-process/cross-network communication. Performance slows to impracticality in the browser. All sources of entropy need to be fast enough for synchronous access.

Even worse, when the database is the only guarantee that ids are unique, that means that clients are forced to send incomplete records to the database, and wait for a network round-trip before they can use the ids in any algorithm. Forget about fast client performance. It simply isn't possible.

That situation has caused some clients to create ids that are only usable in a single client session (such as an in-memory counter). When the database returns the real id, the client has to do some juggling logic to swap out the id being used.

If client side ID generation were stronger, the chances of collision would be much smaller, and the client could send complete records to the db for insertion without waiting for a full round-trip request to finish before using the ID.


#### Monotonically increasing IDs

Cuids generated by the same process are monotonically increasing, when less than 10000 cuids are generated within the same millisecond. Generated by different processes, the cuids will still have an increasing value in time if the process clocks are synchronized.

Monotonically increasing IDs are suitable for use as [high-performance database primary keys](http://code.openark.org/blog/mysql/monotonic-functions-sql-and-mysql), because they can be binary searched. Pure pseudo-random variants don't meet this requirement.


#### Tiny

Somewhat related to performance, an algorithm to generate an ID should require a tiny implementation. This is especially important for thick-client JavaScript applications.


### Security

Client-visible ids often need to have sufficient random data that it becomes practically impossible to try to guess valid IDs based on an existing, known id. That makes simple sequential ids unusable in the context of client-side generated database keys.


#### Portability

Most stronger forms of the UUID / GUID algorithms require access to OS services that are not available in browsers, meaning that they are impossible to implement as specified.


# Features of cuids

## Scalable

Because of the timestamp and the counter, cuid is really good at generating unique IDs on one machine.

Because of the fingerprints, cuid is also good at preventing collisions between multiple clients.


## Fast

Because cuids can be safely generated synchronously, you can generate a lot of them quickly. Since it's unlikely that you'll get a collision, you don't have to wait for a round trip to the database just to insert a complete record in your database.

Because cuids are monotonically increasing, database primary key performance gets a significant boost.

Weighing in at less than 1k minified and compressed, the cuid source should be suitable for even the lightest-weight mobile clients, and will not have a significant impact on the download time of your app, particularly if you follow best practices and concatenate it with the rest of your code in order to avoid the latency hit of an extra file request.

## Secure

Cuids contain enough random data and moving parts as to make guessing another id based on an existing id practically impossible. It also opens up a way to detect for abuse attempts -- if a client requests large blocks of ids that don't exist, there's a good chance that the client is malicious, or trying to get at data that doesn't belong to it.


## Portable

The only part of a cuid that might be hard to replicate between different clients is the fingerprint. It's easy to override the fingerprint method in order to port to different clients. Cuid already works standalone in browsers, or as a node module, so you can use cuid where you need to use it.

The algorithm is also easy to reproduce in other languages. You are encouraged to port it to whatever language you see fit.

### Ports:

* JavaScript (Browsers & Node)
* [cuid for Ruby](https://github.com/iyshannon/cuid) - [Ian Shannon](https://github.com/iyshannon)
* [cuid for .Net](https://github.com/moonpyk/ncuid ) - [Clément Bourgeois](https://github.com/moonpyk)
* [cuid for Go](http://github.com/lucsky/cuid) - [Luc Heinrich](https://github.com/lucsky)
* [cuid for PHP](https://github.com/endyjasmi/cuid) - [Endy Jasmi](https://github.com/endyjasmi)
* [cuid for Elixir](https://github.com/duailibe/cuid) - [Lucas Duailibe](https://github.com/duailibe)
* [cuid for Haskell](https://github.com/crabmusket/hscuid) - [Daniel Buckmaster](https://github.com/crabmusket)
* [cuid for Python](https://github.com/necaris/cuid.py) - [Rami Chowdhury](https://github.com/necaris)
* [cuid for Clojure](https://github.com/hden/cuid) - [Hao-kang Den](https://github.com/hden)
* [cuid for Java](https://github.com/graphcool/cuid-java) - [Nilan Marktanner](https://github.com/marktani)
* [cuid for Lua](https://github.com/marcoonroad/cuid) - [Marco Aurélio](https://github.com/marcoonroad)
* [cuid for Perl](https://github.com/zakame/Data-Cuid) - [Zak B. Elep](https://github.com/zakame)
* [cuid for Perl 6](https://github.com/marcoonroad/perl6-cuid) - [Marco Aurélio](https://github.com/marcoonroad)
* [cuid for OCaml](https://github.com/marcoonroad/ocaml-cuid) - [Marco Aurélio](https://github.com/marcoonroad)
* [cuid for Swift](https://github.com/raphaelmansuy/cuid) - [Raphael Mansuy](https://github.com/raphaelmansuy)



# Short URLs

Need a smaller ID? `cuid.slug()` is for you. With 7 to 10 characters, `.slug()` is a great solution for short urls. Slugs may grow up to 10 characters as the internal counter increases. They're good for things like URL slug disambiguation (i.e., `example.com/some-post-title-<slug>`) but **absolutely not recommended for database unique IDs**. Stick to the full cuid for database keys.

Be aware, slugs:

* are less likely to be monotonically increasing. Stick to full cuids for database lookups, if possible.

* have less random data, less room for the counter, and less room for the fingerprint, which means that all of them are more likely to collide or be guessed, especially as CPU speeds increase.

Don't use them if guessing an existing ID would expose confidential information to malicious users. For example, if you're providing a service like Google Drive or DropBox, which hosts user's private files, favor `cuid()` over `.slug()`.


# Questions

### Is this a replacement for GUID / UUID?

No. Cuid is great for the use case it was designed for -- to generate ids for applications which need to be scalable past tens or hundreds of new entities per second across multiple id-generating hosts. In other words, if you're building a web or mobile app and want the assurance that your choice of id standards isn't going to slow you down, cuid is for you.

However, if you need to obscure the order of id generation, or if it's potentially problematic to know the precise time that an id was generated, you'll want to go with something different.

Cuids should not be considered cryptographically secure (but neither should most guid algorithms. Make sure yours is using a crypto library before you rely on it).


### Why don't you use sha1, md5, etc?

A sha1 implementation in JavaScript is about 300 lines by itself, uncompressed, and its use would provide little benefit. For contrast, the cuid source code weighs in at less than 100 lines of code, uncompressed. It also comes at considerable performance cost. Md5 has similar issues.


### Why are there no dashes?

Almost all web-technology identifiers allow numbers and letters (though some require you to begin with a letter -- hence the 'c' at the beginning of a cuid). However, dashes are not allowed in some identifier names. Removing dashes between groups allows the ids to be more portable. Also, identifier groupings should not be relied on in your application. Removing them should discourage application developers from trying to extract data from a cuid.

The cuid specification should not be considered an API contract. Code that relies on the groupings as laid out here should be considered brittle and not be used in production.


### [Submit a Question or Comment](https://github.com/ericelliott/cuid/issues/new?title=Question)


### Credit

Created by Eric Elliott, Author, ["Programming JavaScript Applications (O'Reilly)"](https://ericelliottjs.com/product/programming-javascript-applications-ebook/)

Thanks to [Tout](http://tout.com/) for support and production testing.
