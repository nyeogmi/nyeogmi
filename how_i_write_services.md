Hey!

Here's some quick information on how I write services in Python, for those of you who have fear of the blank page.

This design most likely will translate very well to your favorite non-Python scripting language

I'm pretty sure all this stuff translates straightforwardly to Rust, but I'm paid to write Python, not Rust, so I can't make claims about the battle-testedness of my designs in any other language.

# Stack

## Language

I use Python.

## Editor

I use PyCharm.

## Database

I prefer Redis to other databases:

- It's hierarchical, which is a match for how I think.
- It's very fast.

It has some downsides:

- It flushes once a second, so it can can lose up to a second of data.
- It only scales with RAM, meaning it's very bad for large workloads.
- Its transaction support is somewhat complicated to use -- you can either use Lua stored procedures, optimistic locking, or relatively slow redis-py style locks. 
- redis-cluster is annoying and doesn't work well with redis-py -- stick to manual sharding and automatic replication, if possible

My preferred Redis driver is redis-py.

For long-term storage, PostgreSQL and MySQL are both pretty good and can be used through SQLAlchemy, an awesome ORM. Most cloud database hosts can provide you an upgradable SQL database. Watch out that your queries are efficient -- it's far easier to accidentally write slow queries on SQL than on Redis.

For PostgreSQL, I believe `psycopg` is still the best driver, but I haven't used it for a few years. For MySQL, I recommend `pymysql`, as it's easier to install than `mysqlclient`. (despite having worse performance)

(A sidenote on the Redis lock types -- this is easily the hardest-to-understand part of using Redis. The three have tradeoffs that are worth getting to know; if you can't be bothered, use `MULTI`/`EXEC` if it will work for your use case, and be sure to unit test code that uses `WATCH` since its guarantees can be a little hard to understand. If you prefer to avoid optimistic locking, use Lua if it looks easy and redis-py locks if it looks hard.)

## CI/automatic builds

CI is a low priority for me, so you can postpone this, but it's good to get it working once for a trivial project because then you can copypaste the code.

All CI systems are pretty much the same if you're not doing anything too complicated. GitHub has its own, so use that if you can't pick one.

I recommend trying to get the following working:

- Automatic testing.
- Zipping everything up into a file with requirements.txt included.

## Hosting

You don't need to think about this until your prototype is written.

I use cloud hosting. (currently AWS Elastic Beanstalk, but I can't recommend it -- the experience is a little clunky)

I avoid serverless tools like Lambda because, at least when I last checked, they don't interoperate well with New Relic (my favorite monitoring tool) and because not all of them understand python's requirements.txt .

(You would probably have a great experience on Lambda if you used node.)

The price difference has never been a major factor in my decision whether or not to use serverless. Historically, database bills dominate every other factor when I write web services.

## Server

You don't need to think about this until your prototype is written.

I use Flask on the python side and nginx plus uwsgi on the non-Python side. Your hosting environment will likely set you up with either nginx/uwsgi or Apache/mod_wsgi. I'll caution that I've seen some strange behavior from Apache when the client disconnects early -- it has led to mod_wsgi logging errors that never actually occured in Python.

A third option is to use node/Express along with some kind of message queue (such as Celery), which will allow you to deteriorate gracefully if the backend of your program breaks. Express can handle a lot of requests and most of the attempts to do request handling in Python are pretty clunky and slow by comparison.

However, cloud load balancers such as ELB will get you a lot of the advantage of that at less cost, and it will turn it into your devops guy's problem instead of your problem.

## Migration tracking

You don't need to think about this in your prototype. You may not need to think about this at all, if you use Redis or some other NoSQL database.

# Directory layout

Assuming my product is named `batfriend` I usually lay out my project like this:

## `setup.py`

Stock setup script. If you don't know what this is, you don't need it in your prototype

Used in case I decide to load this project as a dependency of another one.

I frequently end up using Lua, so I usually preemptively write `package_data={"": ["*.lua"]}` in the call to setup().

## `batfriend/__init__.py`

Root module for my project.

I usually include a single decoy export like `DECOY_STRING = "Hello, world!"`

I'll reference this in the first unit test.

## `batfriend/config.py`

I write code that loads environment variables into globals in an ad hoc way.  (I have a helper library for this, but the task is one where you would be better off solving it yourself, because you want the code to be as simple as possible)

## `batfriend/databases.py`

This exports `get_redis(prefix)` which returns a Redis connection suitable for the given prefix. Initially, all calls to this return the same `Redis` instance, meaning that the code looks like this:

```python
from .config import REDIS_URL

def get_redis(prefix: str):
    return Redis.from_url(REDIS_URL)
```

If the project uses SQLAlchemy, I provide a similar function that produces a SQLAlchemy Engine.

## `test/base_test.py`

I define `class TestBase(object): pass`. 

I will eventually write a `setUp` that calls `redis.flushdb()` for every Redis instance used by the app. Each test class based on it will call `super().setUp()` in its own `setUp()` method

## `test/test_trivial.py`

Defines `class TestTrivial(TestBase):`. I write a single test that asserts `DECOY_STRING == "Hello, world!"`

## `basic_config.py`

Configures New Relic and possibly other utilities. (Frequently, logging.) Always the first import loaded when running as a service.

## `app.py`

Imports basic_config, then defines the application that the host's code is looking for. Usually I'll set `application = flask.Flask(__name__)`, then let Amazon's services handle it from there.

I expose the following endpoints by default:

* `/`: displays an ASCII bat. Uses `flask.Response(bat, mimetype="text/plain")` for formatting reasons. I can ping this to make sure the service is running correctly.

* `/status`: alias for `/`. I hit this to make sure the server is running _my_ code. (because `/` is also an endpoint used by many people's sample applications)

* `/sqs`: POST-only endpoint to receive messages from Amazon's queuing system, SQS. I usually check for an environment variable that determines whether I expose this. I call `flask.request.get_json()` to get the text of the queued message. If you don't know what that means, you don't need this yet.

I often expose additional handlers. If I get a lot of them, I may put them in a separate `batfriend/api/` directory.

## `runlocal.py`

Manually loads my code on Flask's web server. This is useful given that Flask-specific debugging is paywalled for PyCharm.

Usually just wraps: 

```python

import basic_config
from app import application

if __name__ == "__main__":
  application.run(host="0.0.0.0", port=8087, threaded=True)
```

# Actual functionality

## Handlers

I always validate user input by unpacking it and making assertions about its type:

```
@app.route("/login", methods=["POST"])
def login():
  request = flask.request.get_json()
  assert isinstance(request, dict)
  assert all(i in request for i in ["username", "password"])

  username: str = request["username"]
  password: str = request["password"]

  assert isinstance(username, str)
  assert isinstance(password, str)

  # ... the remaining code
```

This is verbose (and there's ways to make it less verbose -- protobuf, for instance!) but if I'm just consuming JSON, I just try to make sure all the types are exactly what I expect.

By the time my application code is dealing with objects, I should know all their types and they should not be buried in JSON.

Note that Python 3 has a `typing` module that allows you to write things like this:

```python
  from typing import List, Dict, Optional

  dog: Optional[Dog] = None
  ages: List[int] = {69, 420}
  last_names: Dict[int, str] = {0: "Mars", 1: "Delaney"}
```

## Model code

### Things

My model code usually consists of a large number of models and remote collections, as well as a lot of record types. 

Conceptually:

* A model type represents a single X that lives outside memory. (ex. on Redis somewhere) 
* A record type represents inert data that can be passed between services. (similar to a database row or a tuple)
* A remote collection type represents a collection of Xs which are either a record type or a model type.

I name models after a singular animate noun (ex: `Agent`) If I can't think of an appropriate animate noun, I come up with a verby description of what it does and append `er`. (ex `MessageWatcher`)

I name records after a single inanimate noun. (ex `Message`)

I name collections after a venue or the plural of the thing managed. (ex: `Lobby`, `Agency`, `Messages`) 

### Interface design rules

I follow a few rules to keep performance consistent and maximize testability:

* I try to pass configuration info through the constructor of the type, when possible.
* I have the type grab its database instances using the global functions. (They can be monkeypatched at runtime if I need to mock them in tests.)

I pay a lot of attention to types. Basically, I try to make it really easy to see from the debug output if I got the wrong type, and I try to make type assertions easy to write:

* I avoid `Union` when possible, since it usually makes interfaces much more complicated. It's better to make the caller explicitly convert by calling a separate function.
* I pass config information in the constructor when I get away with it. I'm OK with passing a much wider variety of types to constructors.
* I try to limit argument types to `int`, `str`, `float`, `bool`, `datetime`, and `None` -- as well as `List`s, `Tuple`s, `Set`s, `Dict`s, `Callable`s and record types thereof.
* I represent time as a `float` when I can get away with it, since that works well with Redis's collection types
* I limit return types similarly to how I limit argument types, except that returning a generator is often OK.
* Consuming a generator is OK if there's no other way to get the function to work, but it's often a smell for debugging and will cause your debugger to leap wildly between two points in the same program.

I follow some rules to keep serialization simple:

* I avoid inheritance for anything that might need to be serialized.
* I try to avoid serializing types that aren't record types, and types whose fields are able to hold a reference to a data source.
* If I need to serialize something that is short-lived and never passes through the user, I use `pickle`. 
* If I need to serialize something that runs through the user, I define `load_json` and `dump_json` functions manually on its class, then use JSON or `msgpack`.

### Skeleton (model type or collection type)

My basic model code usually starts out like this:

```python
from batfriend.databases import get_redis
from redis import Redis

class Counter(object):
    """
    Tracks an increasing number in Redis.
    """
    def __init__(name: str)
        assert isinstance(name, str)

        # get the redis instance for this object
        self._redis: Redis = get_redis(name)
        self._name: str = name
    
    # Some models may be represented with multiple Redis keys
    def _count_key(self):
        return "{}:count".format(self._name)

    # actual methods that modify the state
    def increment(self):
        self._redis.incr(self._count_key)

    def increment_by(self, amt: int):
        # whenever you take output from a part of the program that might be 
        # rewritten separately from your code,
        # validate that input!
        assert isinstance(amt, int) and amt >= 0
        self._redis.incrby(self._count_key)
```

My collection code is typically much the same, except it operates on a Redis collection. (usually a set or a zset)

### Patterns for transactions

#### Snapshotting

I typically provide a `view()` method to get a read-only snapshot of the state of a model object.

```python

class Counter(object):
    ...

    def view() -> int:
        return int(self._redis.get(self._count_key()) or 0)
```

If the result type is simple, I just use `int`, `str`, or whatever else is natural.

If it's complicated, I define a record type whose name ends in `Snapshot`

#### Lazy and batch loading

For data that can be efficiently loaded in batches _or_ data that can be efficiently loaded lazily, I provide a context manager that looks like it fetches the data, but actually does other stuff.

The context manager typically manages updates, too. It doesn't handle concurrency -- I use one of the below locking strategies for that, and I use it separately.

A context manager for this might look like this (for batches):

```python
class PhoneNumberContext(object):
    def __init__(self, name: str):
        assert isinstance(name, str)

        self._redis = get_redis(name)
        self._name = name

        self._data = {}
        self._updated = set()

    def __enter__(self): return self
    def __exit__(self): self.commit()

    def _phones_key(self):
        return "{}:phones".format(self._name)

    def preload(self, ids: List[str]):
        assert isinstance(ids, list) and all(isinstance(i, str) for i in ids)

        # load data for all ids
        new_ids = [i for i in ids if i not in self._data]
        if len(new_ids) > 0:
            redis_data = self._redis.hmget(self._phones_key(), *new_ids)
            for i, data in zip(ids, redis_data):
                self._data[i] = Data.load_json(data)

    def update(self, phone_numbers: Dict[str, str]):
        self._data.update(phone_numbers)
        self._updated.update(phone_numbers.keys())  # add to the update collection

    def commit():
        if self._updated:
            self._redis.hmset(self._name, {k: v for k in self._updated})
```

I usually expose this from a remote collection type with a method like this:

```python
class PhoneNumbers(object):
    ...

    @contextmanager
    def transact(self):
        # wrap the contextmanager with `@contextmanager` to get an interface that is more
        # resilient to misuse
        with PhoneNumberContext(self._name) as n:
            yield n
    
```

Sometimes I combine the patterns, and have a `view` function that returns an instance of the type in non-context manager mode, with a `read_only` flag set so that `commit` and `update` will not work:

```python

class PhoneNumbers(object):
    ...

    def view(self) -> PhoneNumberContext:
        return PhoneNumberContext(self._name, read_only=True)
```

#### Pessimistic locking

`redis-py` contains a somewhat sluggish implementation of pessimistic locking which is easy to use.

I often combine it with the context manager pattern to discourage misuse:

```python
class PhoneNumbers(object):
    ...

    @contextmanager
    def transact(self):
        # wrap the contextmanager with `@contextmanager` to get an interface that is more
        # resilient to misuse
        with self._redis.lock(self._lock_key(), blocking_timeout=5, timeout=5) as lock:
            # throws LockError if not acquired
            with PhoneNumberContext(self._name) as n:
                yield n
```

#### Optimistic locking

For optimistic locking, I use the `transact()` function built into redis-py. Internally, it is based on Redis's `WATCH`, `MULTI`, and `EXEC` operations:

```python
class Counter(object):
    ...

    def _transact_optimistic(f)
       self._redis.transact(f, self._count_key())

```

Be sure that the function `f` doesn't make any changes to the Redis instance without first calling `multi`, and try not to let any details of `f`'s work leak out until `f` is finished running -- since you frankly don't know if it will crash and be restarted by `transact()`

I always treat optimistic locking as an implementation detail because the interface is so complicated. That means I make the methods that do it private.

#### Lua scripts

This is more complicated and you probably shouldn't do it in your prototype, but it allows you to write complex conditional logic that's run under Redis's global interpreter lock.  It is probably the most efficient way to do that.

I typically create a `lua` package, then a lua file:

```lua
-- batfriend/lua/transact.lua
local x = redis.call("GET", "test1")
if x == "test" then
  redis.call("DEL", "test2")
end
```

(Note: you're told to make sure Redis knows exactly which keys your Lua script will modify, but in practice this only affects Redis's behavior when you are running under Redis Cluster.)

After writing my Lua script, I use `importlib` to load a `.lua` file from that package containing the code.

```python
# loads batfriend/lua/transact.lua

import importlib.resources as resources
from . import lua

LUA_TRANSACT_EXAMPLE = resources.read_text(lua, "transact.lua")
```

You can then define `self._script` in your constructor

```python
class TransactExample(object): 
    def __init__(self):
        ...

        self._script = self._redis.register_script(LUA_TRANSACT_EXAMPLE)
```

You can then call `self._script` as a function to invoke your script.

### Patterns for worklists

I use worklists extensively to get things done.

Frankly, building some familiarity with Redis is likely to help you more than specific examples, but here's some recommendations:

* Use a ZSET, then insert each job you have with `time.time()` as a key. Every second, fetch the first 100 items and send them to a second "actively in process" worklist implemented with a non-Redis scheduler. Upon success, remove those jobs from the ZSET. This allows you to deprioritize or expedite jobs without calling into the API functionality of a bulky service like RabbitMQ.

* Use HSCAN or SCAN to hit everything -- just call it once a second and store the last cursor, defaulting to 0. This will repeatedly sweep the database at a consistent rate.

I recommend having a separate server whose only job is to do background tasks, which includes worklist stuff.

You probably don't need this in your prototype though.