---
layout: post
author: alex
title:  "Loading 7M items to Neo4j with and without UNWIND"
categories: cartography performance cypher neo4j
date: 2020-07-19 11:53:00 -0700

---

Is it faster for your application to write large amounts of data to Neo4j in one large, batched query, or in many, much smaller ones? 

One official Neo4j blog post on query tuning prefers the former, saying you should "[batch your writes]((https://neo4j.com/blog/cypher-write-fast-furious/#batch-cypher-writes))" at the application layer using one large `UNWIND` query. On the other hand, that same blog post also says

>  A number of small optimized queries always run faster than one long, un-optimized query.

It advises devs to "[avoid long cypher queries (30-40 lines)](https://neo4j.com/blog/cypher-write-fast-furious/#long-cypher-queries)". What does this mean though? It's very possible to write fast queries that are 30 lines long and slow queries that are 2 lines long. How much is "a number"? Why 30 lines and not 20?

I thought the explanations weren't written as clear as they could've been, so now it's time for an experiment: Let's compare loading large amounts of data to Neo4j in (1) a batched way using `UNWIND` versus (2) multiple smaller queries to find out for ourselves.

# Experiment setup

Let's make some fake data, for starters let's create some objects to represent 70,000 AWS principal to S3 bucket pairs. 

```python
def make_fake_data(items_to_make):
    res = []
    for _ in range(items_to_make)
        res.append({
            'principal_arn': f'arn:aws:iam::1234:role/{randint(0,2400)}',
            'resource_arn': f'arn:aws:s3:::{randint(0,4000)}'
        })
    return res

# 70,000 fake items in the list
full_data = make_fake_data(70000)
```
Note that this code sets an upper bound of 2400 unique AWS roles and 4000 unique S3 buckets.


# Approach 1 - `UNWIND`

## The code
```python
def unwind():
    query =  """
        UNWIND {Mapping} as mapping
        MERGE (principal:AWSPrincipal{arn:mapping.principal_arn})
        MERGE (resource:S3Bucket{arn:mapping.resource_arn})
        MERGE (principal)-[r:CAN_READ]->(resource)
        """
    print(f"UNWINDing {len(full_data)} items")
    neo4j_session.run(
        query,
        Mapping=full_data,
        )
```

This batches up the data in our Python application and writes it all to the graph. Let's set up our timer and run the experiment!

## Results

```python
# Start with an empty DB
neo4j_session.run('match (n) detach delete n')

# Run the experiment
unwind_time = timeit.timeit(unwind, number=1)
print(f"--> unwind = {unwind_time} seconds")
```

Let's run the code:

```
➜  python unwind_vs_each.py
UNWINDing 70000 items
--> unwind = 1.086337526 seconds
```

1 second! Pretty good! Let's see how a non-batched, multiple-smaller-queries approach does.

# Approach 2 - Multiple smaller transactions

## The code

```python
def each():
    query =  """
        MERGE (principal:AWSPrincipal{arn:{principal_arn}})
        MERGE (resource:S3Bucket{arn:{resource_arn}})
        MERGE (principal)-[r:CAN_READ]->(resource)
        """

    print(f"Performing for-each on {len(full_data)} items")
    for d in full_data:
        neo4j_session.run(
            query,
            principal_arn=d['principal_arn'],
            resource_arn=d['resource_arn'],
        )
```

The key difference here is that the Python code iterates through each item and loads each one as a separate transaction. 

It was hard for me to decide on a hypothesis because

- Approach 2 could be slower because of there is a lot of overhead in setting up and tearing down each individual transaction.

or

- Approach 2 could be faster because that Neo4j blog post says many smaller queries are faster than large ones.

Let's find out the answer.

## Experiment results

This involves the same setup as last time but with a different function:

```python
# Start with an empty DB
neo4j_session.run('match (n) detach delete n')

# Run the experiment
foreach_time = timeit.timeit(each, number=1)
print(f"--> for-each result = {foreach_time} seconds")
```

## Result

I ran the above and it did not finish. I let it run for at least **15 minutes** before I got sleepy and went to bed. When I woke up and checked, it was crashed and I have no idea how far along it got or how long it took to get to this point:

```
Performing for-each on 70000 items
Traceback (most recent call last):
  File "/Users/alex/.virtualenvs/env7/lib/python3.7/site-packages/neobolt/direct.py", line 408, in _send
    self.socket.sendall(data)
  File "/usr/local/opt/python/Frameworks/Python.framework/Versions/3.7/lib/python3.7/ssl.py", line 1034, in sendall
    v = self.send(byte_view[count:])
  File "/usr/local/opt/python/Frameworks/Python.framework/Versions/3.7/lib/python3.7/ssl.py", line 1003, in send
    return self._sslobj.write(data)
BrokenPipeError: [Errno 32] Broken pipe

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "unwind_vs_each.py", line 82, in <module>
    each_time = timeit.timeit(each, number=1)
  File "/usr/local/opt/python/Frameworks/Python.framework/Versions/3.7/lib/python3.7/timeit.py", line 232, in timeit
    return Timer(stmt, setup, timer, globals).timeit(number)
  File "/usr/local/opt/python/Frameworks/Python.framework/Versions/3.7/lib/python3.7/timeit.py", line 176, in timeit
    timing = self.inner(it, self.timer)
  File "<timeit-src>", line 6, in inner
  File "unwind_vs_each.py", line 59, in each
    aws_update_tag='1595116803'
  File "/Users/alex/.virtualenvs/env7/lib/python3.7/site-packages/neo4j/__init__.py", line 502, in run
    self._connection.send()
  File "/Users/alex/.virtualenvs/env7/lib/python3.7/site-packages/neobolt/direct.py", line 388, in send
    self._send()
  File "/Users/alex/.virtualenvs/env7/lib/python3.7/site-packages/neobolt/direct.py", line 414, in _send
    self.server.address))
neobolt.exceptions.ServiceUnavailable: Failed to write to defunct connection Address(host='localhost', port=7687) (Address(host='127.0.0.1', port=7687))
```

Interesting! This is the same error observed in Cartography issue [#170](https://github.com/lyft/cartography/issues/170), which we've had a hard time reproducing. I would have thought that multiple smaller writes would avoid weird socket problems like this because there's less data to write in each interaction, but anyway, this is a topic for another day.

So for this experiment, I observed my code running for at least 15 minutes, which is already **900 times** slower than approach 1's `UNWIND` method (900 seconds in 15 minutes vs the 1 second it took for `UNWIND` to run).

# Let's try 7,000,000 items

Ok, we've already learned that batching data at the application layer with `UNWIND` is way faster than individual transactions, but can it handle loading millions of items? I'll need to first adjust the fake data to allow 240,000 AWS roles and 400,000 S3 buckets.


```python
def make_fake_data(items_to_make):
    res = []
    for _ in range(items_to_make)
        res.append({
            'principal_arn': f'arn:aws:iam::1234:role/{randint(0, 240000)}',
            'resource_arn': f'arn:aws:s3:::{randint(0, 400000)}'
        })
    return res

# 7,000,000 fake items in the list
full_data = make_fake_data(7000000)
```

Now let's run the experiment again...

```
➜  Desktop python unwind_vs_each.py
UNWINDing 7000000 items
--> unwind = 107.39662291500001 seconds
```

Under 2 minutes to process 7 million items!


# Conclusion

So there you have it: batched `UNWIND` is much faster than for-each. At least 900 times faster.
