---
title: "What's a ULID"
date: 2020-06-27T14:35:31-03:00
draft: false
tags: ["development","python"]
---

This time I will talk about someting I found out very recently, and I found interesting to share this information as it might be handy for developers to use when the use case fits.  
So this time I will introduce the `ulid`. Spoiler alert, it's an ID or at least a data structure that can be used to represent an ID. So why do we have another way of expressing an ID, I won't get into too much details into that but yes there are many ways of defining ids, it can be numerical, using UUIDs, etc. So my motivation is that I found useful the properies of ulid for some specific use cases.  

### Background
I just want to give some background how I came across ulid and why I found it handy to use. I regularly find myself browsing through code from open source projects, even if I am not familiar with the language the projects are written, I always take a peek at the code. I have this obsesion to know how things work under the hood, which has helped me to solve many tricky issues operating open source software.  
So one day I was looking at [Prometheus](https://prometheus.io/) data folder and I found the following

```bash

# Looking at files and folders in the Prometheus data folder
$ ls -l
total 84
drwxr-xr-x 3 nobody nogroup  4096 May 29 21:00 01E9H3RBHR7NP48DNJJK99SRVN
drwxr-xr-x 3 nobody nogroup  4096 Jun  1 03:00 01E9PX50124DREREKAB8PDASV5
drwxr-xr-x 3 nobody nogroup  4096 Jun  3 09:00 01E9WPHK3Q7Y58K8X49AW5JRQR
drwxr-xr-x 3 nobody nogroup  4096 Jun  5 15:00 01EA2FY7JBZP2D6119TXG3EFWG
drwxr-xr-x 3 nobody nogroup  4096 Jun  7 21:00 01EA89ATAJHX36BMBAVRRG44XF
drwxr-xr-x 3 nobody nogroup  4096 Jun 10 03:00 01EAE2QEVTWR7Y4EKWEYYXEXP5
drwxr-xr-x 3 nobody nogroup  4096 Jun 12 09:00 01EAKW3Z3T1N2K0FQWVPTZ5T7T
drwxr-xr-x 3 nobody nogroup  4096 Jun 14 15:00 01EASNGHSNXG01MGREADK79EY2
drwxr-xr-x 3 nobody nogroup  4096 Jun 16 21:00 01EAZEX4YK3E0ACSPXETHG953Z
drwxr-xr-x 3 nobody nogroup  4096 Jun 19 03:00 01EB589TB6X24Q2JWFE5Z2RQWE
drwxr-xr-x 3 nobody nogroup  4096 Jun 21 09:00 01EBB1PEFAZGT5VG2MWK53ACAK
drwxr-xr-x 3 nobody nogroup  4096 Jun 23 15:00 01EBGV31HBX0BJGS1WFCWWEA7S
drwxr-xr-x 3 nobody nogroup  4096 Jun 25 21:00 01EBPMFNV6FQ4299NFMSAQTVWV
drwxr-xr-x 3 nobody nogroup  4096 Jun 26 15:02 01EBRJE3ABRFZBHET8ZJ34F5VX
drwxr-xr-x 3 nobody nogroup  4096 Jun 27 09:00 01EBTG2M659D94WBXTAZXEYR4C
drwxr-xr-x 3 nobody nogroup  4096 Jun 27 15:00 01EBV4NRBW7WXBFHSQ2DZ3EPSZ
drwxr-xr-x 3 nobody nogroup  4096 Jun 27 15:00 01EBV4NSMXJ3SRJ6J38Z4J7CAZ
drwxr-xr-x 3 nobody nogroup  4096 Jun 27 17:00 01EBVBHFM05VRE1B3QXGDTNE8B
drwxr-xr-x 2 nobody nogroup  4096 Jun 27 17:00 chunks_head
-rw-r--r-- 1 nobody nogroup     0 Apr 10 22:17 lock
-rw-r--r-- 1 nobody nogroup 20001 Jun 26 14:28 queries.active
drwxr-xr-x 3 nobody nogroup  4096 Jun 27 17:00 wal

# Then looking into one of this weird folders. It is where the 
# tsdb (time series database) structure is stored.
$ ls -lh 01EBVBHFM05VRE1B3QXGDTNE8B
total 860K
drwxr-xr-x 2 nobody nogroup 4.0K Jun 27 17:00 chunks
-rw-r--r-- 1 nobody nogroup 845K Jun 27 17:00 index
-rw-r--r-- 1 nobody nogroup  279 Jun 27 17:00 meta.json
-rw-r--r-- 1 nobody nogroup    9 Jun 27 17:00 tombstones
```
Ever since I looked into that folder I wondered about the logic behind the design. Then some day I was looking trought the Prometheus code and I came across the term ulid. Off course I quickly googled the term and found out what it was, then the whole directory structure stated to make sense. I won't get into details of that as it is a subject matter that probably could get 2 or 3 blog posts itself. But I will get into details what ulid are and it might give developers some perspective to solve some problems in a clever way like promtetheus developers did.


# What is it then?

The term is ulid and it's long name is `Universally Unique Lexicographically Sortable Identifier`. [ulid spec](https://github.com/ulid/spec) is hosted in github if you would like to know more in details.  
But in plain words it is a 128 bit ID, represented in a string using [Crockford's Base32 encoding](https://en.wikipedia.org/wiki/Base32#Crockford's_Base32).  
It is compossed by a timestamp which is the fist 48bits and a secuence of random 80 bits giving a 26 character string.  
We can see how the representation is split
```
 01AN4Z07BY      79KA1307SR9X4MV3

|----------|    |----------------|
 Timestamp          Randomness
   48bits             80bits
```
The timestamp is 48 bits unix timestamp with milisecond precision encoded in Base32, which will generate monotonically the ids. When there are 2 IDs being generated in the same milisecond there should be 80 bits of randomness to prevent collisions in the ID, which has a very little chance that 2 IDs will collide and making them unique. Also the spec mentions how to keep the randomness part sortable, however this will depend how it is implemented in the library we are using and we need to research how the library provides the source of randomness, I have seen some libraries providing a function or method to pass the random function if you don't trust that the library implementation is a good source of randomness. Since the first 48bits is a timestamp and the way it's encoded it makes the ID's lexicographically sortable which give some unique benefits. Another benefit is that it doesn't contain special characters making it web and url safe.  
This make a perfecly good alternative to UUID which is one of the most popular formats, giving some extra properties along the way.

### Benefits
#### Lexicographically sortable
This is the main benefit, making the IDs sortable through time. Not only that you can sort by the order of creation but the ulid has the timestamp enconded in it when it was created, this might come handy when creating an object or event you can use the ulid to also store the creation time.  
The timestamp precision is milisecond which could be used for most use cases.

#### Unique
Like UUIDs, ulid are unique. The random component gives 80 bit of randomness per milisecond, which means there are 1.21e+24 possible IDs per milisecond making it very unlikely to have collisions.

#### UUID Compatible
Both UUID and ulid are represented by 128 bit, making them compatible when storing them at rest. This is an advantage to leverage existing methods to store UUIDs, for example some databases support UUID as a data type or they can be extended to support storing UUID as data type, which makes it convenient to reuse the UUID data types to store ulid.

#### Web Safe
Since it's a string without special characters, they can be used safely in urls or even html, also there are 26 characters compared to the 36 chars from UUIDs.

### Disavantages
#### Milisecond precision
The sortability is only guaranteed with a milisecond precision, while the spec recommends how to implement it to provide some guarantees on sortability, it is not bulletproof and it depends on implementation.  
If the use case requires hard guarantees with sub milisecond precision, ulid won't help much. But most use cases for web applications milisecond precision is good enough.


#### Timestamp embeded on the ID
This is an advantage and a disadvantage at the same time. Why? It depends on the use case. While it is great to have the timestamp embeded in the ID and handy for some use cases, when you need a pure random ID having something like the timestamp could be an anti pattern. We need to be careful with this feature as it might expose sensitive information in the ID so we need to be aware of the use case before using it.

#### No RFC
There are no standards defining ulids, this is just a spec that is hosted in github, which means that someone created the concept and shared it to the world. While UUIDs are an industry standard and they have their own [rfc4122](https://www.ietf.org/rfc/rfc4122.txt), UUIDs can be used for interoperability between vendors. While ulid doesn't have an RFC which might be a problem when you need to interoperate with vendors. Although, a ulid can be converted and represented as a UUID when needed.

#### Implementations might diverge
This is related to the previous disadvantage, since this is a spec and not backed by an industry standard, implemetation is subject to interpretation and some implementations might have slightly different behavior than other. Off course having an RFC is not guarantee that implementations will be the same, in general RFC are topics already discussed by a group of people/vendors/companies making it a standard and making it easy for interoperate.

#### No tools to manipulate ulids
There are libraries for all major languajes, but ulid is still missing tools to manipulate ulids in the operating systems, all major operating systems like Mac or linux already have some kind of tool for generating UUIDs like `uuidgen` or something similar. I haven't found tools to generate ulids out of the box without having to install a programing laguage and the library to generate the ulid.


### Use cases

This is not an exhaustive list of use cases, but some common use cases it can be used for. Most use cases are related to the objects/events which need to be sorted through time when precision doesn't require sub milisecond.  
Some examples:  

- blog posts
- new feeds
- picture history
- revision/version control of files
- Events in a distributed system
- Time series blocks (Prometheus use case)

I wanted to point out a use case that catches my attention and probably would have saved some problems with race contitions when sending messages in an async messaging plattform and still have events that need to be ordered. In an distributed system sending messages/events to other systems and attaching a ulid to the message would help prevent or detect race conditions, it won't guarantee that my messages arrived in order but at least I would know that I got a message older than a message that I already received. Helping detect and remediate the situation that might have generated a race condition. Well this is not a bulletproof approach but whould helped to detect some of this situations.

#### Example

As an example I would show how to generate a ulid in python but other languajes should be pretty similar. The example I used `python-ulid` and `iPython` interpreter.

```python
Python 3.7.5 (default, Nov  1 2019, 02:16:32)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.16.1 -- An enhanced Interactive Python. Type '?' for help.

In [1]: from ulid import ULID

In [2]: ULID()
Out[2]: ULID(01EBVTVMGED7CVDDE8A0JDTQKZ)

In [3]: ULID()
Out[3]: ULID(01EBVTVP8HHF971ZMB2RQFQW8W)

In [4]: ULID()
Out[4]: ULID(01EBVTVQQ6K47WC9ZN9F3T6EBZ)
```

As we can see the values generated where sortable in time.  
Now we can generate a ulid and extract the timestamp from it
```python
Python 3.7.5 (default, Nov  1 2019, 02:16:32)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.16.1 -- An enhanced Interactive Python. Type '?' for help.

In [1]: from ulid import ULID

In [2]: ulid = ULID()

In [3]: ulid.timestamp
Out[3]: 1593293283.947

In [4]: ulid.milliseconds
Out[4]: 1593293283947

In [5]: ulid.datetime
Out[5]: datetime.datetime(2020, 6, 27, 21, 28, 3, 947000, tzinfo=datetime.timezone.utc)
```
And at last, I will show how easy is to swap from ulid to UUID, and UUID to ulid.

```python
Python 3.7.5 (default, Nov  1 2019, 02:16:32)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.16.1 -- An enhanced Interactive Python. Type '?' for help.

In [1]: from ulid import ULID

In [2]: uuid = ulid.to_uuid()

In [3]: ulid.from_uuid(uuid)
Out[3]: ULID(01EBVTW8KB5CWRHV0Q2BY5WSQA)

In [4]: ulid = ULID()

In [5]: ulid
Out[5]: ULID(01EBVV37GHRTGN0PKZ0VBS3SPM)

In [6]: ulid.to_uuid()
Out[6]: UUID('0172f7b1-9e11-c6a1-505a-7f06d791e6d4')

In [7]:  uuid = ulid.to_uuid()

In [8]: ulid.from_uuid(uuid)
Out[8]: ULID(01EBVV37GHRTGN0PKZ0VBS3SPM)
```


# Conclusion

With this post I wanted to share something I learnt by digging how things work under the hood, and I wanted to share it because this might give other developers more tricks to solve issues in a different way or make their implementations much easier or simpler.  
I know ulid won't replace UUIDs because of the fact that both probably compliment each other and one covers use cases the other doesn't. As I mentioned a few times already it would depend on the use cases. I would still use UUIDs when I need them to be completely random and being random is a must or need. But I would use ulid when the use case is not sensitive to having the timestamp embeded in the ID, then I would choose to generate them with ulid generator, you never know when it would be useful, and if the ulid was generated and then stored as UUID you can always recover the timestamp from the UUID if you stored it as UUID. Since they are compatible there would be some benefits to generating the UUID from a ulid generator and  convert to UUID.  
And as I already mentioned I would definitely use it for use cases where we need some object ID ordered in time or sortable, when we need to also the timestamp of an event I would also choose the ulid. 