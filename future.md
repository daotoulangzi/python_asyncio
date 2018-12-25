### asyncio.Future
---
#### Future 的作用
- 负责终止 loop 的循环。
  - loop 停止循环的唯一条件为 `loop._stopping = True`
  - 而将 loop 的 _stopping 设置为 True 的活，一般是 future 负责干。
  - 比如官方 sample 里
    ```
    import asyncio
    
    # 定义一个协程
    async def slow_operation(future):
        await asyncio.sleep(1)
        future.set_result('Future is done!')

    # 获得全局循环事件
    loop = asyncio.get_event_loop()
    # 实例化期物对象
    future = asyncio.Future()
    asyncio.ensure_future(slow_operation(future))
    # loop 的 run_until_complete 会将 _run_until_complete_cb 添加到 future 的完成回调列表中。而 _run_until_complete_cb 中会执行 loop.stop() 方法
    loop.run_until_complete(future)
    print(future.result())
    # 关闭事件循环对象
    loop.close()
    ```
   - 可以发现，loop 事件循环执行最外层一般是 Future 期物对象，期物会包裹其他 task 对象，在 future 对象的状态为 PENDING 时，会一直对 future 包裹的 task 对象调用 `send(None)` 方法，直到报出 StopIteration 方法。期物包裹的每一个 task 对象在执行完成后都会执行回调，如果期物包裹的所有 task 均执行完毕，则执行期物的 `future.set_result()` 方法，此方法不仅会将期物的状态设置为 FINISHED,还会执行期物的完成回调方法，执行 `loop.stop()` 方法。此时 loop 停止循环。
   
#### Future 的关键属性和方法
- 属性
  - _state = _PENDING  # 表征状态，5 颗星
  - _result = None  # 存储结果值，5 颗星
  - _exception
  - _loop  # 5 颗星
  - _source_traceback
  - _asyncio_future_blocking = False  # 用以在 task._step() 方法中判断是否为期物，以及期物是否已执行完毕，重要指数 5 颗星
  - _log_traceback
  
 - 核心方法
  - `__iter__` 方法  # 重要指数，5 颗星
    ```
    def __iter__(self):
      if not self.done():
      # 如果期物的状态非 FINISHED，则将其 _asyncio_future_blocking 属性设置为 True,并且 yield self，这个设置是告诉 task，期物未执行完毕，期物的值应该在期物执行完毕后再获取。
          self._asyncio_future_blocking = True
          yield self  # This tells Task to wait for completion.
      assert self.done(), "yield from wasn't used with future"  # 此处用于在期物没有执行完下强行获取期物值的情况。
      return self.result()  # May raise too.
    ```
  - `set_result` 方法，4.5 颗星
    ```
    def set_result(self, result):
      if self._state != _PENDING:
          raise InvalidStateError('{}: {!r}'.format(self._state, self))
      self._result = result  # 给期物的结果值属性赋值
      self._state = _FINISHED  # 设置期物的状态
      self._schedule_callbacks()  # 执行期物的完成回调方法
    ```
  - `add_done_callback` 方法，4 颗星
    ```
      def add_done_callback(self, fn):
        if self._state != _PENDING:
            self._loop.call_soon(fn, self)  # 如果非 pending 状态，则立即执行完成回调方法
        else:
            self._callbacks.append(fn)  # 如果 pending 状态，则将回调添加到回调列表中，等待给期物设置 result 是调用执行
    ```
  - 其他的方法
    - `__init__` 方法：初始化 loop 对象和 _callbacks 列表
    - `cancel` 方法：将 future 的状态变更为 CANCELLED, 执行完成回调列表
    - `_schedule_callbacks` 方法：执行 _callbacks 列表的回调
    
#### 总结（期物的设计思想）
- 1、通过 _state 标记其当前状态
- 2、将标记 loop._stopping 为 True 的动作或者其他一些标记动作放到 _callbacks 中，在期物设置其结果值`set_result()`时，进行回调，从而达到终止 loop 循环的目的。
- 3、期物因为其`__iter__`方法的实现方式，
  - 在其 _state 为 PENDING 时，通过标记 _asyncio_future_blocking 为 True,且 `yield self` 告知 `_step()`当前期物的状态为 PENDING，需要将 `_step()` 操作添加到期物的 _callbacks 中，等期物的状态为 FINISHED 时，再获取期物的值。
  - 在 _state 为 FINISHED 时，直接获取期物的 result 值。
- 4、上述官方 sample 中：
  - `_run_until_complete_cb` 即负责设置 loop._stopping=True,在 loop.run_until_complete(future) 方法中，会将`_run_until_complete_cb`方法添加到 future 的 _callbacks 中，当 future 有结果值(`set_result()`)时,执行回调，终止 loop 的执行。

#### Note:asyncio.sleep() 解析
- 源码：
  ```
  @coroutine
  def sleep(delay, result=None, *, loop=None):
      """Coroutine that completes after a given time (in seconds)."""
      if delay == 0:
          yield
          return result

      if loop is None:
          loop = events.get_event_loop()
      future = loop.create_future()
      h = future._loop.call_later(delay,
                                  futures._set_result_unless_cancelled,
                                  future, result)
      try:
          return (yield from future)
      finally:
          h.cancel()
  ```
  - 当 delay 为0时，再次使用 send(None) 迭代执行获取 result 值；当 delay 不为0时，将 `_set_result_unless_cancelled`方法添加到延时 delay 后执行，`_set_result_unless_cancelled`是设置 future 的结果值。
  - return (yield from future) 相当于
    ```
    data = None
    for _tmp in future:
        data = yield _tmp
    return data
    ```
    每个future 都会至少一次，至多两次执行 yield 语句来获取其结果值。
    
#### 如何高效处理异步协程
- 关于 sleep(duration)
  - sleep 的设计思想是将回调绑定当前时间+duration实例化 TimeHandle 对象。
  - 将所有的 TimeHandle 对象添加到堆列表 leap 中，TimeHandle 对象使用其时间(time)属性比较大小。
  - loop 在循环执行 leap 列表时，会判定当前时间current_time 是否小于 leap 的第一个值 near_time，如果小于则 time.sleep(near_time-current_time),然后执行第一个回调，否则立即执行第一个回调。
- 关于 I/O
  - 生成 socket 对象绑定 ip 和 port 然后进行监听，并且将 socket 对象及其回调注册到 selector 对象中，当 socket 收到可读或可写消息时(socket注册到selector时，可选择读消息或写消息或读写消息)，执行回调。
  - 回调的应用：一般会在回调中使用 accept 获取连接到当前 socket 的 cli_socket,然后将 cli_socket 注册到 selector 对象，客户端即可以和 cli_socket 进行通信。
 
