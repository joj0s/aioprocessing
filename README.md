aioprocessing
=============
[![Build Status](https://github.com/dano/aioprocessing/workflows/aioprocessing%20tests/badge.svg?branch=master)](https://github.com/dano/aioprocessing/actions)


`aioprocessing` provides asynchronous, [`asyncio`](https://docs.python.org/3/library/asyncio.html) compatible, coroutine 
versions of many blocking instance methods on objects in the [`multiprocessing`](https://docs.python.org/3/library/multiprocessing.html) 
library. To use [`dill`](https://pypi.org/project/dill) for universal pickling, install using `pip install aioprocessing[dill]`. Here's an example demonstrating the `aioprocessing` versions of 
`Event`, `Queue`, and `Lock`:

```python
import time
import asyncio
import aioprocessing


def func(queue, event, lock, items):
    """ Demo worker function.

    This worker function runs in its own process, and uses
    normal blocking calls to aioprocessing objects, exactly 
    the way you would use oridinary multiprocessing objects.

    """
    with lock:
        event.set()
        for item in items:
            time.sleep(3)
            queue.put(item+5)
    queue.close()


async def example(queue, event, lock):
    l = [1,2,3,4,5]
    p = aioprocessing.AioProcess(target=func, args=(queue, event, lock, l))
    p.start()
    while True:
        result = await queue.coro_get()
        if result is None:
            break
        print("Got result {}".format(result))
    await p.coro_join()

async def example2(queue, event, lock):
    await event.coro_wait()
    async with lock:
        await queue.coro_put(78)
        await queue.coro_put(None) # Shut down the worker

if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    queue = aioprocessing.AioQueue()
    lock = aioprocessing.AioLock()
    event = aioprocessing.AioEvent()
    tasks = [
        asyncio.ensure_future(example(queue, event, lock)), 
        asyncio.ensure_future(example2(queue, event, lock)),
    ]
    loop.run_until_complete(asyncio.wait(tasks))
    loop.close()
```

The aioprocessing objects can be used just like their multiprocessing
equivalents - as they are in `func` above - but they can also be 
seamlessly used inside of `asyncio` coroutines, without ever blocking
the event loop.


What's new
----------

`v2.0.0`

- Add support for universal pickling using [`dill`](https://github.com/uqfoundation/dill), installable with `pip install aioprocessing[dill]`. The library will now attempt to import [`multiprocess`](https://github.com/uqfoundation/multiprocess), falling back to stdlib `multiprocessing`. Force stdlib behaviour by setting a non-empty environment variable `AIOPROCESSING_DILL_DISABLED=1`. This can be used to avoid [errors](https://github.com/dano/aioprocessing/pull/36#discussion_r631178933) when attempting to combine `aioprocessing[dill]` with stdlib `multiprocessing` based objects like `concurrent.futures.ProcessPoolExecutor`.


How does it work?
-----------------

In most cases, this library makes blocking calls to `multiprocessing` methods
asynchronous by executing the call in a [`ThreadPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#threadpoolexecutor), using
[`asyncio.run_in_executor()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.BaseEventLoop.run_in_executor). 
It does *not* re-implement multiprocessing using asynchronous I/O. This means 
there is extra overhead added when you use `aioprocessing` objects instead of 
`multiprocessing` objects, because each one is generally introducing a
`ThreadPoolExecutor` containing at least one [`threading.Thread`](https://docs.python.org/2/library/threading.html#thread-objects). It also means 
that all the normal risks you get when you mix threads with fork apply here, too 
(See http://bugs.python.org/issue6721 for more info).

The one exception to this is `aioprocessing.AioPool`, which makes use of the 
existing `callback` and `error_callback` keyword arguments in the various 
[`Pool.*_async`](https://docs.python.org/3/library/multiprocessing.html#multiprocessing.pool.Pool.apply_async) methods to run them as `asyncio` coroutines. Note that 
`multiprocessing.Pool` is actually using threads internally, so the thread/fork
mixing caveat still applies.

Each `multiprocessing` class is replaced by an equivalent `aioprocessing` class,
distinguished by the `Aio` prefix. So, `Pool` becomes `AioPool`, etc. All methods
that could block on I/O also have a coroutine version that can be used with `asyncio`. For example, `multiprocessing.Lock.acquire()` can be replaced with `aioprocessing.AioLock.coro_acquire()`. You can pass an `asyncio` EventLoop object to any `coro_*` method using the `loop` keyword argument. For example, `lock.coro_acquire(loop=my_loop)`.

Note that you can also use the `aioprocessing` synchronization primitives as replacements 
for their equivalent `threading` primitives, in single-process, multi-threaded programs 
that use `asyncio`.


What parts of multiprocessing are supported?
--------------------------------------------

Most of them! All methods that could do blocking I/O in the following objects
have equivalent versions in `aioprocessing` that extend the `multiprocessing`
versions by adding coroutine versions of all the blocking methods.

- `Pool`
- `Process`
- `Pipe`
- `Lock`
- `RLock`
- `Semaphore`
- `BoundedSemaphore`
- `Event`
- `Condition`
- `Barrier`
- `connection.Connection`
- `connection.Listener`
- `connection.Client`
- `Queue`
- `JoinableQueue`
- `SimpleQueue`
- All `managers.SyncManager` `Proxy` versions of the items above (`SyncManager.Queue`, `SyncManager.Lock()`, etc.).


What versions of Python are compatible?
---------------------------------------

`aioprocessing` will work out of the box on Python 3.5+.

Gotchas
-------
Keep in mind that, while the API exposes coroutines for interacting with
`multiprocessing` APIs, internally they are almost always being delegated
to a `ThreadPoolExecutor`, this means the caveats that apply with using
`ThreadPoolExecutor` with `asyncio` apply: namely, you won't be able to
cancel any of the coroutines, because the work being done in the worker
thread can't be interrupted.
