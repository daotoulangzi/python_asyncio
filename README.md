# python_asyncio
---
```
python 中 asyncio 相关的异步编程 & concurrent.futures.ThreadExecutor、ProcessExecutor 相关异步编程。
1) ThreadExecutor 和 ProcessExecutor 实现异步编程依靠的是任务队列 queue.Queue 以及多线程或多进程技术
2) asyncio 实现异步编程依靠的是 python 中 yield 的语法，主要涉及到 loop, future, task 3种对象
```
### 1、loop
- 事件循环对象，其继承关系 AbstractEventLoop -> BaseEventLoop -> BaseSelectorEventLoop -> _UnixSelectorEventLoop
- `loop` 的主要作用
  - 有一个 `_ready` 属性存储当前 `loop` 对象所有需要执行的调用(`Events.Handle()`对象)
  - 有一个 _stopping 属性
  - `loop` 在执行 `run_until_complete` 或 `run_forever` 时，会一直调用 `_run_once()` 执行 `_ready` 中的调用，直至 `_stopping` 属性值为 `True`

### 2、future
- 期物对象，`asyncio.Future`
- 实现了 `__iter__` 方法，故可以对 `future` 对象使用 `yield from future` 语法；
  - 根据 `__iter__` 方法的实现，可以发现，要获取 `future` 的结果值，需要使用 `send(None)` 驱动。如果 `future` 的状态为 `pending`,则仍然返回该 `future` 对象，且将 `future` 对象的 `_asyncio_future_blocking` 属性设置为 True，在 `task` 的 `_step()` 方法中可以发现,如果 `send(None)` 返回的值 `result` 有 `_asyncio_future_blocking` 属性(表明 `result` 对象是 `future` 对象)，且该属性值为 `True`(表明 `result` 对象的状态为 `pending`)，则将 `_step()` 方法添加到 `result` 对象的回调方法中。即当 `result` 的状态为 `FINISH` 后，调用 `_step()` 将 `result` 的结果值返回。

### 3、task
- 任务对象，`asyncio.Task`
- 在 `__init__` 实例化方法中，将 `_step()` 方法包装成 `events.Handle` 对象，并添加到当前 `loop` 对象的 `_ready` 中。在 `loop` 执行 `run_until_complete` 或者 `run_forever` 时，会循环调用 `_run_once` 执行 `_ready` 中的 `handle.run()` 方法,直至 `loop._stopping` 值为 `True`
