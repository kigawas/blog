+++
title = "Reconsider the performance benchmarks in Python web development"
date = "2020-11-12"
cover = ""
tags = ["Python", "web", "FastAPI", "Sanic", "Golang", "benchmark"]
description = "A few words on common misunderstandings and performance tips of web development benchmarks"
showFullContent = false
+++

A few words on common misunderstandings and performance tips of web development benchmarks.

## Overview

As of 2020, there are many web benchmarks and more web frameworks even if we just focus on Python community. Among those benchmarks, the most prestigious and reliable one would be [TechEmpower benchmarks](https://www.techempower.com/benchmarks/#section=data-r19&hw=cl&test=fortune&l=zijzen-1r). Undoubtedly, [asyncio](https://docs.python.org/3/library/asyncio.html)-based web frameworks prevail on performance in theory and in reality, however, we are still curious about **how** performant those frameworks are in a more pragmatic environment.

If you have compared Python and Go frameworks in TechEmpower benchmarks, you might have noticed that some Python frameworks even perform [better than Gin](https://www.techempower.com/benchmarks/#section=data-r19&hw=ph&test=fortune&l=zijmrj-1r), the most famous and widely used Go web framework. Is that real? Or is that just **theoretical**? Well, in this article, we'll do some fact-checking on this, in a more **practical** way.

## Common misunderstandings

### People may be misunderstanding the word _performance_

You might have found lots of claims on performance from lots of frameworks, such as:

> It features a Martini-like API with much better performance -- up to 40 times faster. -- _Gin_, 2020
>
> Fast and unfancy HTTP server framework for Go (Golang). Up to 10x faster than the rest. -- _Echo_, before 2016
>
> High performance, extensible, minimalist Go web framework. -- _Echo_, 2020
>
> Screaming-fast Python 3.5+ HTTP toolkit integrated with pipelining HTTP server based on uvloop and picohttpparser. -- _Japronto_, 2020

Let's ponder: what do they mean "fast" or "high performance"? Being able to handle one request in very short time? Or being able to handle **more** requests per second? Unfortunately, I don't think all those guys have the same understanding on "performance", but here let's clarify:

If you can handle more requests per second, you are more performant. We can define more formally: more requests/responses per second, or RPS, means higher performance.

### `Asyncio` does not improve speed, but it does improve throughput

Some would assume that `asyncio` or [coroutines](https://www.python.org/dev/peps/pep-0492/) can make Python run faster, but this is a common fallacy. Coroutines have never made or promised to make your favorite Python program run faster, instead, they do make your program run **asynchronously**.

> Synchronous/asynchronous program and blocking/non-blocking IO are orthogonal concepts.
>
> You can still use a blocking IO in an asynchronous coroutine. A more realistic example may be reading file synchronously from your hard disk in a coroutine.
>
> Coroutines are special functions, which can be paused and resumed without losing current states by some supervisor, or "event loop". In Python 3.5 and before, coroutines were implemented by generators.

Let's take this example:

```python
async def test():
    print("before 1")
    await coro1()
    print("after 1")

    print("before 2")
    await coro2()
    print("after 2")

    return "result"
```

Say your `test` coroutine will run 10 times, and let's number them as `test_1` ... `test_10`. If `test_1` stops at some time-consuming `coro1`, `asyncio` will select `test_2` and call it. The same is for `test_3` ... `test_10`. When we are in `test_5` and stops before `coro1`, at the same time, `await coro_1()` in `test_1` returns a result, `asyncio` will go back to `test_1` to continue the execution.

> This is a simplified model, for more details, keep the [official documentation](https://docs.python.org/3/library/asyncio.html) under your pillow.

If you only run `test` for one time, `asyncio` does not help much. It'll even cost more resources since there are event loops and schedulers under the hood. But thankfully, in a typical web server, functions to handle HTTP requests are invoked over and over again. In these coroutines, we'll probably spend much time on retrieving data from DB frequently and updating data in DB less frequently.

This is a perfect fit for introducing `asyncio`. You may have heard [ASGI](https://asgi.readthedocs.io/en/latest/introduction.html) somewhere:

```python
async def application(scope, receive, send):
    event = await receive()
    data = await read_data_from_db()
    await send({"type": "websocket.send", ...})
```

The coroutine `receive` transforms raw HTTP requests into Python dictionaries:

```python
{
    "type": "http.request",
    "body": b"Hello World",
    "more_body": False,
}
```

Calling the coroutine by `await send(...)` will send the response to client, as you imagine, the coroutine `application` is called every time when there comes a request from a client.

Now we are confident to say `asyncio` makes your web server handle more requests, but it cannot make your web server handle one request faster - in effect, it gets slightly slower in each request.

## Performance tips

### `Uvloop` is literally dope

`Asyncio` does help, but actually it's just a [reference implementation](https://www.python.org/dev/peps/pep-3156/). By leveraging [`Cython`](https://github.com/cython/cython) and [`libuv`](https://github.com/libuv/libuv), [MagicStack](https://github.com/MagicStack) implemented a faster event loop called `uvloop`:

![uvloop performance](https://raw.githubusercontent.com/MagicStack/uvloop/master/performance.png)

Based on `uvloop`, we also have a fast ASGI server implementation called [`uvicorn`](https://github.com/encode/uvicorn).

### "Hello world" JSON does not tell the whole story

Most benchmarks are focusing two aspects mainly: JSON serialization and database read/write. JSON serialization is typically a CPU-bound task in contrast database r/w is an IO-bound task. Often we'll have this JSON:

```json
{ "message": "Hello, World!" }
```

And test like this if we choose a Bottle/Flask-like way to define routes:

```python
from awesome_framework import JSONResponse, App

app = App()

@app.get("/")
def handler():
    return JSONResponse(dict(message="Hello, world!"))
```

If you only look into JSON serialization, [Japronto](https://github.com/squeaky-pl/japronto) has incredible performance: [133859 RPS on Azure D3v2 instances](https://www.techempower.com/benchmarks/#section=data-r19&hw=cl&test=json&l=zijmrj-1r) and this is twice Gin or Beego written in Golang.

A Python novice may be excited, but as veterans let's inspect again: is this what we really want? What is the approximate percentage of JSON serialization in our yet another awesome web service? Can we profile this?

Probably the percentage is pretty low, in fact, the most time-consuming task would be connecting the database, retrieving or updating data and close the connection. And the network-IO will cost you most of time, no matter how fast you serialize JSONs.

> No one refuses better performance. If you want extreme JSON serialization speed in Python, check [`orjson`](https://github.com/ijl/orjson).

### Using ORM (or at least raw SQLs) is closer to reality

In a more practical scenario, we are always integrating some ORM into the web framework. There are also many async or sync ORMs in Python community, such as:

- [gino](https://github.com/python-gino/gino), async
- [orm](https://github.com/encode/orm), async
- [tortoise-orm](https://github.com/tortoise/tortoise-orm/), async
- [SQLAlchemy](https://docs.sqlalchemy.org/en/14/), async and sync
- [Django ORM](https://docs.djangoproject.com/en/3.1/topics/db/), sync
- [Pony ORM](https://github.com/ponyorm/pony), sync

Whatever ORM we choose, we'll write queries similarly:

```python
class User(Model):
    id = Column(Integer, primary_key=True)
    name = Column(String)

    @classmethod
    async def get_user_by_name(cls, name: str):
        # in async orm
        return await cls.filter.where(name=name)

    @classmethod
    def get_users(cls):
        # in sync orm
        return cls.filter.all()
```

When we call these query functions, our ORM will do the hardest task for us: converting queries to raw SQLs. As you know, this is pretty sluggish and knocks down performance badly, especially in Python.

> I didn't test very rigorously, but it may make our server 30% slower approximately.

If ORM is so slow, what about writing raw SQLs directly?

### Raw SQLs outperform ORM, but we write ORMs mostly

In Python community, most popular database clients are just a thin wrapper over some C implementation, such as `psycopg2`/`psycopg2-binary` for PostgreSQL or `mysqlclient` for MySQL. Prevalent reasons include efficiency, security, code reuse, etc.

However, in asynchronous Python, there is another story. Connecting a database is mainly about setting up a TCP socket, sending some binary data and getting the response, in addition, there may be some concurrent connections in a pool. This is akin to what we have discussed above - an HTTP server.

Do you remember `uvloop`? What if we introduce `uvloop` into our database connector just like `ASGI`? Well, we are not the first to come up with this notion and there already exists a PostgreSQL connector called `asyncpg`.

> MagicStack is also the author of `asyncpg`.

If you see its performance at the first time, you may be astonished:

![asyncpg performance](https://raw.githubusercontent.com/MagicStack/asyncpg/master/performance.png)

For [how](http://magic.io/blog/asyncpg-1m-rows-from-postgres-to-python/) did they achieve this:

> It soon became evident that by implementing PostgreSQL frontend/backend protocol directly, we can yield significant speed improvements. Our earlier experience with uvloop has shown that Cython can be used to build very efficient libraries. asyncpg is written almost entirely in Cython with careful buffer management and highly optimized data decoding pipeline.

Most benchmarks are combining web frameworks with database connectors and performance are tested by executing raw SQLs instead of ORM queries.

This looks okay, but to what extent are we writing raw SQLs and getting them executed in a connector? At least for me, I only write raw SQLs in ORM **only** when there are performance problems I have to resolve, but in most cases, something like `cls.filter.where(name=name)` is way more familiar to me.

Everyone blames ORM when they get into performance trouble. Is there any method to get performant as promised in the `benchmarks.md`?

### Let's bake queries in ORM to get around bottlenecks

As mentioned above, converting an ORM query to its corresponding SQL costs lots of time. According to the [Pareto principle](https://en.wikipedia.org/wiki/Pareto_principle), we normally spend most of time on a few queries, thus, it's a natural intuition to **cache those queries** instead of parsing ORM queries to SQLs over again.

> The cached queries are called [baked queries](https://docs.sqlalchemy.org/en/13/orm/extensions/baked.html).
>
> This technique is different from caching the results from database - you only cache the generated SQLs so there is no need to worry about the consistency.

Let's see an example from [gino](https://python-gino.org/docs/en/master/how-to/bakery.html).

> Gino is an async ORM based on [SQLAlchemy Core](https://docs.sqlalchemy.org/en/13/core/) and asyncpg with rather decent usability and performance.

```python
class User(db.Model):
    __tablename__ = "users"

    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String)

    @db.bake
    def getter(cls):
        return cls.query.where(cls.id == db.bindparam("uid"))

    @classmethod
    async def get_by_id(cls, uid: int):
        return await cls.getter.one_or_none(uid=uid)
```

With baked queries, you can get around the main bottleneck of parsing ORM queries into SQLs, in most cases this would be a huge performance enhancement:

```bash
$ wrk --latency -t20 -c50 -d15s http://0.0.0.0:8080/api/users/1  # pure ORM

Running 15s test @ http://0.0.0.0:8080/api/users/1
  20 threads and 50 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.57ms    8.38ms 178.15ms   76.60%
    Req/Sec   165.40     79.13   440.00     82.63%
  Latency Distribution
     50%   12.12ms
     75%   16.06ms
     90%   19.62ms
     99%   39.17ms
  49473 requests in 15.06s, 7.03MB read
Requests/sec:   3284.77
Transfer/sec:    477.96KB

$ wrk --latency -t20 -c50 -d15s http://0.0.0.0:8080/api/users/1/bake  # with baked queries

Running 15s test @ http://0.0.0.0:8080/api/users/1/bake
  20 threads and 50 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    10.53ms   10.43ms 156.56ms   96.66%
    Req/Sec   211.44     69.41   560.00     76.27%
  Latency Distribution
     50%    9.07ms
     75%   11.34ms
     90%   14.57ms
     99%   61.73ms
  63338 requests in 15.06s, 9.00MB read
Requests/sec:   4206.11
Transfer/sec:    612.02KB

$ wrk --latency -t20 -c50 -d15s http://0.0.0.0:8080/api/users/1/raw  # raw SQL

Running 15s test @ http://0.0.0.0:8080/api/users/1/raw
  20 threads and 50 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     9.12ms    6.35ms 125.16ms   93.34%
    Req/Sec   233.71     69.49   520.00     71.32%
  Latency Distribution
     50%    7.80ms
     75%   10.08ms
     90%   13.39ms
     99%   38.28ms
  69947 requests in 15.05s, 9.94MB read
Requests/sec:   4647.83
Transfer/sec:    676.30KB
```

### It's time to rethink benchmarks

If you've read everything above, you probably agree that we should run benchmarks with an ORM. Let's design some test vectors:

- FastAPI/Sanic + gino, raw SQL
- FastAPI/Sanic + gino, without baked queries
- FastAPI/Sanic + gino, with baked queries
- Gin + raw SQL

Let me omit the [details](https://gist.github.com/kigawas/f97c245efedb286ebfe1deb979a217cb) here lest Go manias be irritated. In conclusion, the performance (RPS) ranking would be:

- Sanic > FastAPI ~= Gin
- Raw SQL > baked queries >> ORM
- Gin, Raw SQL ~= FastAPI, baked queries

Sanic performs really well, but if you need automatic request parameter validation and OpenAPI documentation, FastAPI would be a better (or perhaps the only) choice.

## Conclusion

Asynchronous Python community is growing rapidly and we are always encountering new libraries and frameworks. With blazing-fast development efficiency and comparable performance to NodeJS or Golang, asynchronous Python will win greater popularity in web development.

## Trivia

### Gin's annoying 406 errors

If you set `wrk -c` to `200`, there may come many HTTP 406 errors when testing Gin. A simple solution is to set the concurrency to `50`.

### Wise pool size setting

`DB_POOL_MAX_SIZE` should be greater than concurrent wrk connections to avoid frequent DB connection acquirements.
