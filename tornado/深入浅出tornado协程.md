在研究过几次tornado的核心源码后，一直想写一篇博文作为总结，但一拖再拖，最近想着不能再拖延下去了，挤出时间和大家分享一下tornado协程。

tornado作为一个著名的python异步web框架，在tornado 6.0之前有一个完整的、不依赖python3的asyncio库的协程实现，而且还支持python2，也就是说，即使是在python2环境下也可以实现非常类似于python3的协程。

当然目前的主流的协程实现还是python3的asyncio标准库，tornado 5.1就默认在python3环境使用asyncio的loop，6.0开始则只采用asyncio的loop，所以 6.0并不支持python2。

本文将对tornado 5.1源码进行剖析，以tornado的协程实现来讲讲什么是协程。如果你只对asyncio感兴趣，这篇文章也能对你有启发，因为两者非常相似，而且可以一定程度上兼容。

我理解的tornado协程，有三大基础：生成器、Future类和loop实现。这三者结合之巧妙，一直让我觉得是一个非常天才的设计。下面就先从最简单最常见的生成器讲起吧。

## 生成器
-----------------------

先看一段简单的代码示例（运行环境ipython）
```
In [202]: def test():
     ...:     name = yield 'yao'
     ...:     print('Hello, %s' % name)
     ...:     return 0
     ...:
     ...:

In [203]: g = test() # 一个生成器

In [204]: first = next(g) # 先启动生成器，获取生成器的第一个值

In [205]: first
Out[205]: 'yao'

In [206]: second = g.send('susan')  # 'susan'通过send方法将传给test函数里的name变量。如果生成器还有下一个值将赋值给second，但这里将抛出StopIteration错误
Hello, susan
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-206-04419ce5a913> in <module>()
----> 1 second = g.send('susan')  # 'susan'通过send方法将传给test函数里的name变量。如果生成器还有下一个值将赋值给second，但这里将抛出StopIteration错误

StopIteration: 0
```
对于生成器，我们可以控制生成器的运行，而且通过send方法将外部变量传入生成器的内部变量，如代码中示例将`'susan'`这个值通过send赋予`name`变量。请记住这段示例，接下来这段代码会以一点不同的形式再次出现

## Future
-------------------------
关于Future的定义，我在网上找了一段话说明一下其概念：
> future是concurrent.futures模块和asyncio模块的重要组件，
> 从python3.4开始标准库中有两个名为Future的类：concurrent.futures.Future和asyncio.Future
> 这两个类的作用相同：两个Future类的实例都表示可能完成或者尚未完成的延迟计算。与Twisted中的Deferred类、Tornado框架中的Future类的功能类似

我个人对Future的理解就是Future是对一件发生了的事情的当前状态的描述，并提供通用接口用于

1. 查看future当前的状态，如
    - `def done(self)`查看是否done
    - `def cancelled(self)`查看是否cancelled

2. `def add_done_callback(self, fn)` 添加callback，Future状态变更为done或者已经是done状态时，这些`fn`将执行（执行方式在不同的Future中并不一样）

3. `def set_result(self, result)` 状态设置为done，且保存事件结果

4. `def set_exception(self, exception)` 状态设置为done，且保存事件进行过程中发生的异常对象

5. `def result(self)` 返回future记录的事件结果

Future的代码示例（运行环境ipython）：
示例1：
```
In [3]: from tornado.concurrent import Future

In [4]: from asyncio import Future as AsyncioFuture

In [5]: Future == AsyncioFuture # 在python3.5+下，tornado使用的Future同asyncio.Future
Out[5]: True

In [6]: def event(error=False):
   ...:     if error:
   ...:         raise Exception('error event')
   ...:     return 0
   ...:
   ...:

In [7]: f = Future()

In [8]: f.done()
Out[8]: False

In [9]: f.cancelled()
Out[9]: False

In [10]: result = event()

In [11]: result
Out[11]: 0

In [12]: f.set_result(result)  # 将事件的结果保存至f

In [13]: f
Out[13]: <Future finished result=0>

In [14]: f.done()
Out[14]: True

In [15]: f.result()
Out[15]: 0
```

标例2：如果event进行过程中发生错误，没有返回结果时Future如何记录事件状态呢？
```
In [17]: f = Future()

In [18]: try:
    ...:     event(error=True)
    ...: except Exception as e:
    ...:     f.set_exception(e)  # 如果事件进行过程中发生错误，set_exception保存异常对象
    ...:

In [19]: f.done()  # 事件因异常结束也认为状态是done
Out[19]: True

In [20]: f.result() # 这里将重新抛出保存的异常
```

## 生成器和Future
------------------------
上面简单介绍的生成器和Future，是非常基础的概念，很多python初学者可能都已经掌握了。但就是这样简单的概念，结合起来会产生意想不到的效果。现在将用一个简单的示例，展示如何将两者结合起来。

这里先思考几个问题：

1. Future用于描述事件的状态，生成器可以用于描述事件的过程，那生成器是否可以转化成Future？（即生成器转化为Future）

2. 生成器作为事件过程的描述，是否也可以因其他事件状态的不同而走向不同的结果？（即生成器内部处理Future？）


先举一下非常简单的示例，用于回答上述两个思考问题，这个示例和最开始的生成器的示例会非常相似：
```
In [31]: get_name_event = Future()

In [32]: get_name_event.set_result('susan')

In [33]: get_name_event.done()
Out[33]: True

In [34]: get_name_event.result()
Out[34]: 'susan'

In [35]: def test():
    ...:     name = yield get_name_event  # 对应问题1：生成器内部处理Future
    ...:     print('Hello, %s' % name)
    ...:     return 0
    ...:
    ...:

In [36]: g = test()

In [37]: first = next(g)

In [38]: first
Out[38]: <Future finished result='susan'>

In [39]: test_future = Future() # 对应问题2：接下来的run函数将展示如何将生成器g转化为test_future

In [40]: def run(first, g):  # 这里的代码源于tornado源码的超简化版
    ...:     try:
    ...:         value = first.result()  # first即为get_name_event，get_name_event已是done状态
    ...:         g.send(value)  # 通过send方法将value赋值给test函数里的name变量
    ...:     except StopIteration as e:  # 示例中的生成器只yield一个值，所以g.send(value)会抛 StopIteration
    ...:         result = None
    ...:         try:
    ...:             result = e.value
    ...:         except AttributeError:
    ...:             try:
    ...:                 result = e.args[0]
    ...:             except (AttributeError, IndexError):
    ...:                 pass
    ...:         test_future.set_result(result)  # 生成器运行完毕，将StopIteration里值提取出来，作为test_future的result
    ...:
    ...:

In [41]: run(first, g)  # 将生成器转化为Future
Hello, susan

In [42]: test_future
Out[42]: <Future finished result=0>

In [43]: test_future.done()
Out[43]: True

In [44]: test_future.result()
Out[44]: 0
```
我们可以把生成器看作是对一个事件的过程的描述，Future看作是一个事件的状态的描述，那就可以总结两点：

1. 生成器内部处理Future可以认为是生成器代表的事件过程依赖于其他事件的结果。

2. 生成器作为事件过程的描述，在走向终点的过程中，相应地有事件的状态的变更，即生成器可以转化为Future。

根据上面的示例，可以总结一下示例中生成器如何转化为Future：

1. `next(g)`获取第一个依赖的其他事件的状态`get_name_event`

2. 如果`get_name_event`是done状态（！！！疑问，如果不是done状态呢？），取出`get_name_event`的result，通过`g.send`方法使g内部的`name`变量获得值。

3. 生成器`g`获得`get_name_event`事件返回的`name`变量，执行剩余过程，因生成器只yield一次，抛出 StopIteration 异常

4. `run`方法捕捉生成器`g`的 StopIteration 异常，判断`g`执行完毕，从 StopIteration 异常中获取return值0，作为`test_future`的result

5. `test_future`作为生成器`g`相应的状态对象，最终转为done状态

上述示例中的生成器向Future的转化，是非常特例和简化的一种，可以延伸思考：

1. 如果`get_name_event`不是done状态，如何处理？

2. 如果生成器`g`内部不止处理`get_name_event`这一个其他事件，如何处理？

3. 如果`get_name_event`发生异常，即虽然是done状态，但没有result，只有exception？

等等

这些情形tornado内部有个装饰器`coroutine`都考虑到了，当然，肯定也适用于上述示例中的生成器：
```
In [45]: from tornado.gen import coroutine

In [46]: @coroutine
    ...: def test():
    ...:     name = yield get_name_event
    ...:     print('Hello, %s' % name)
    ...:     return 0
    ...:
    ...:

In [47]: future = test()
Hello, susan

In [49]: future
Out[49]: <Future finished result=0>

In [50]: future.done()
Out[50]: True

In [51]: future.result()
Out[51]: 0
```
是不是简洁多了？仅仅在test方法加上装饰器`coroutine`（coroutine翻译过来就是协程的意思），`test()`不再返回生成器，而是返回Future对象，因为装饰器做了将生成器转化为Future的工作。

关于`coroutine`的用法，还有一些在相对简单易懂的情形的示例，这里作一些展示，可以加深理解
```
In [52]: @coroutine
    ...: def get_name_event_impl():  # 如果 coroutine 装饰的不是生成器，直接将return值作为Future的result
    ...:     return "susan"
    ...:
    ...:

In [53]: get_name_event = get_name_event_impl()

In [54]: get_name_event
Out[54]: <Future finished result='susan'>

In [55]: get_name_event.done()
Out[55]: True

In [56]: get_name_event.result()
Out[56]: 'susan'

In [57]: @coroutine
    ...: def error_event_impl():
    ...:     raise Exception('test')
    ...:
    ...:

In [58]: error_event = error_event_impl()

In [59]: error_event
Out[59]: <Future finished exception=Exception('test',)>

In [60]: error_event.done()
Out[60]: True

In [61]: error_event.result()  # 这里将抛出 Exception('test')，因为error_event_impl发生异常，error_event虽然是done状态，但`result()`无返回值   

In [62]: @coroutine
    ...: def test():
    ...:     name = yield get_name_event_impl()
    ...:     print('Hello, %s' % name)
    ...:     return 0
    ...:
    ...:

In [63]: test_future = test()
Hello, susan

In [64]: test_future
Out[64]: <Future finished result=0>

In [65]: @coroutine
    ...: def handle_error():
    ...:     try:
    ...:         result = yield error_event_impl()
    ...:     except Exception as e:
    ...:         print('error: ', repr(e))
    ...:

In [66]:

In [66]: handle_error()
error:  Exception('test',)
Out[66]: <Future finished result=None>
```
上面的示例里生成器内部处理的Future都无一例外都是done状态，这样的生成器加上`coroutine`装饰器后返回的Future对象也是done状态。那如果生成器内部处理的Future对象不是done状态会如何？
```
In [74]: from tornado import gen
In [75]: pending_future = gen.sleep(1)  # 即便这里等一万年，pending_future也不会变成done状态

In [76]: pending_future
Out[76]: <Future pending>

In [77]: pending_future.done()
Out[77]: False

In [78]: @coroutine
    ...: def handle_unfinished_future():
    ...:     result = yield pending_future
    ...:     print('result is ', result)
    ...:     return True
    ...:
    ...:

In [79]:

In [79]: f = handle_unfinished_future()

In [80]: f
Out[80]: <Future pending cb=[_make_coroutine_wrapper.<locals>.wrapper.<locals>.<lambda>() at /Users/yao/Nustore Files/tornado-py/tornado/gen.py:347]>

In [81]: f.done()
Out[81]: False
```
可以看到生成器内部处理的Future如果不是done状态，生成器就无法继续执行，生成器转化的Future也不会是done状态。
实际上这样的情形才是协程中最常见的情形。而在这些情形下就需要loop的介入了。

## loop
-------------------
这部分将对`coroutine`如何将生成器转化为`Future`对象进行进一步的讲解，那不可避免地需要接触loop的概念了。

首先看一下`coroutine`的源码，这部分源码我去除了`coroutine`里关于`YieldPoint`相关的部分，因为`YieldPoint`是一种类`Future`实现，但已被废弃了；另外新加入了一些注释。

```
def coroutine(func):
    return _make_coroutine_wrapper(func, replace_callback=True)

def _make_coroutine_wrapper(func, replace_callback):
    """The inner workings of ``@gen.coroutine`` and ``@gen.engine``.

    The two decorators differ in their treatment of the ``callback``
    argument, so we cannot simply implement ``@engine`` in terms of
    ``@coroutine``.
    """
    # On Python 3.5, set the coroutine flag on our generator, to allow it
    # to be used with 'await'.
    wrapped = func
    if hasattr(types, 'coroutine'):
        func = types.coroutine(func)

    @functools.wraps(wrapped)
    def wrapper(*args, **kwargs):
        future = _create_future()  # coroutine装饰的方法最终返回对象是一个Future对象

        if replace_callback and 'callback' in kwargs:
            warnings.warn("callback arguments are deprecated, use the returned Future instead",
                          DeprecationWarning, stacklevel=2)
            callback = kwargs.pop('callback')
            IOLoop.current().add_future(
                future, lambda future: callback(future.result()))

        try:
            result = func(*args, **kwargs)  # 如果func是生成器，将返回一个GeneratorType对象
        except (Return, StopIteration) as e:  # 如果抛出StopIteration，说明生成器运行完毕
            result = _value_from_stopiteration(e)  # 从 StopIteration 中提取返回值作为result
        except Exception:
            future_set_exc_info(future, sys.exc_info())  # 如果func运行中发生其他异常，认为func执行完毕，返回future，future保存了异常对象，参看示例中的 handle_error
            try:
                return future
            finally:
                # Avoid circular references
                future = None
        else:
            if isinstance(result, GeneratorType):
                # Inline the first iteration of Runner.run.  This lets us
                # avoid the cost of creating a Runner when the coroutine
                # never actually yields, which in turn allows us to
                # use "optional" coroutines in critical path code without
                # performance penalty for the synchronous case.
                try:
                    orig_stack_contexts = stack_context._state.contexts  # 协程上下文相关代码，跳过
                    yielded = next(result)  # 获得第一个yielded对象，联想一下示例中的 first = next(g)
                    if stack_context._state.contexts is not orig_stack_contexts:  # 协程上下文相关代码，跳过
                        yielded = _create_future()
                        yielded.set_exception(
                            stack_context.StackContextInconsistentError(
                                'stack_context inconsistency (probably caused '
                                'by yield within a "with StackContext" block)'))
                except (StopIteration, Return) as e:
                    # 说明生成器执行完毕
                    future_set_result_unless_cancelled(future, _value_from_stopiteration(e)) 
                except Exception:
                    # 说明生成器运行中发生错误
                    future_set_exc_info(future, sys.exc_info())
                else:
                    # Provide strong references to Runner objects as long
                    # as their result future objects also have strong
                    # references (typically from the parent coroutine's
                    # Runner). This keeps the coroutine's Runner alive.
                    # We do this by exploiting the public API
                    # add_done_callback() instead of putting a private
                    # attribute on the Future.
                    # (Github issues #1769, #2229).

                    # yielded是Future，result是GeneratorType
                    # 如果func是以下示例中的test的情形
                    # In [46]: @coroutine
                    #     ...: def test():
                    #     ...:     name = yield get_name_event
                    #     ...:     print('Hello, %s' % name)
                    #     ...:     return 0
                    # Runner会在__init__方法里执行类似以下代码展示的逻辑，具体请看Runner的代码
                    # value = yielded.result()
                    # try:
                    #     result.send(value)
                    # except StopIteration as e:
                    #     future.set_result(_value_from_stopiteration(e))
                    # 是不是非常似曾相识？
                    runner = Runner(result, future, yielded)
                    future.add_done_callback(lambda _: runner)
                yielded = None
                try:
                    return future
                finally:
                    # Subtle memory optimization: if next() raised an exception,
                    # the future's exc_info contains a traceback which
                    # includes this stack frame.  This creates a cycle,
                    # which will be collected at the next full GC but has
                    # been shown to greatly increase memory usage of
                    # benchmarks (relative to the refcount-based scheme
                    # used in the absence of cycles).  We can avoid the
                    # cycle by clearing the local variable after we return it.
                    future = None
        future_set_result_unless_cancelled(future, result)
        return future

    wrapper.__wrapped__ = wrapped
    wrapper.__tornado_coroutine__ = True
    return wrapper


# 这部分Runner代码采用了tornado 6.1的代码，因5.1的代码还有废弃的`YieldPoint`的相关代码
class Runner(object):
    """Internal implementation of `tornado.gen.coroutine`.

    Maintains information about pending callbacks and their results.

    The results of the generator are stored in ``result_future`` (a
    `.Future`)
    """

    def __init__(
        self,
        gen: "Generator[_Yieldable, Any, _T]",  # next(g) 后的g，已经取过了第一个yielded值的生成器
        result_future: "Future[_T]",  # result_future就是coroutine装饰器装饰的方法内最终返回的Future对象，此时是pending状态
        first_yielded: _Yieldable,  # 第一个yielded的值，一般为Future对象，实际上还可以是某些特定对象，暂不讨论非Future的其他情形
    ) -> None:
        self.gen = gen
        self.result_future = result_future
        self.future = _null_future  # type: Union[None, Future]
        self.running = False
        self.finished = False
        self.io_loop = IOLoop.current()  # loop终于出现了！！！

        # handle_yield判断 first_yielded 是否done状态，如果done状态，执行run方法，可参考示例中 run(first, g)
        # 如果 first_yielded 不是done状态呢？答案在 handle_yield 方法内
        if self.handle_yield(first_yielded):
            gen = result_future = first_yielded = None  # type: ignore
            self.run()

    def run(self) -> None:
        """Starts or resumes the generator, running until it reaches a
        yield point that is not ready.
        """
        if self.running or self.finished:
            return
        try:
            self.running = True
            while True:
                future = self.future  # future是 生成器yield的 Future 对象
                if future is None:
                    raise Exception("No pending future")
                if not future.done():
                    return
                self.future = None
                try:
                    exc_info = None

                    try:
                        value = future.result()  # 获取result
                    except Exception:
                        exc_info = sys.exc_info()  # future代表的事件发生异常，没有返回值
                    future = None

                    if exc_info is not None:
                        try:
                            # 如果future代表的事件发生异常，没有返回值，只能将异常通过throw方法传回生成器
                            yielded = self.gen.throw(*exc_info)  # type: ignore
                        finally:
                            # Break up a reference to itself
                            # for faster GC on CPython.
                            exc_info = None
                    else:
                        # future事件有返回值，通过send方法传入生成器内部处理
                        yielded = self.gen.send(value)
                        # 如果生成器内部处理多个Future对象，类似于
                        # def test():
                        #     result1 = yield f1
                        #     result2 = yield f2
                        # 那 yielded 将会是下一个 Future 对象，比如f2

                except (StopIteration, Return) as e:
                    # 如果生成器运行完毕，self.result_future 保存result
                    self.finished = True
                    self.future = _null_future
                    future_set_result_unless_cancelled(
                        self.result_future, _value_from_stopiteration(e)
                    )
                    self.result_future = None  # type: ignore
                    return
                except Exception:
                    # 如果生成器运行过程发生异常，self.result_future 保存异常
                    self.finished = True
                    self.future = _null_future
                    future_set_exc_info(self.result_future, sys.exc_info())
                    self.result_future = None  # type: ignore
                    return
                if not self.handle_yield(yielded):  # 处理下一个 yielded 对象，逻辑处理同 first_yielded
                    return
                yielded = None
        finally:
            self.running = False

    def handle_yield(self, yielded: _Yieldable) -> bool:
        try:
            # 实际上生成器内部不仅可以处理 Future 对象，只是 Future对象大概是最常见的了
            # 通过 convert_yielded 可以将部分非 Future 对象也转化为 Future
            self.future = convert_yielded(yielded)  # 将 yielded 赋值self.future，这样self.run可以进行处理
        except BadYieldError:
            self.future = Future()
            future_set_exc_info(self.future, sys.exc_info())

        if self.future is moment:  # 可以跳过
            self.io_loop.add_callback(self.run)
            return False
        elif self.future is None:  # 可以跳过
            raise Exception("no pending future")
        elif not self.future.done():

            # 如果生成器内部处理的 Future 对象，这里指 self.future 不是done状态，生成器将无法继续执行
            # 生成器对应的 result_future 也将是pending状态
            # 所以这时就需要loop了！！！
            # loop.add_future(self.future, inner)后，loop运行过程中，一旦 self.future 变更为done状态，loop会执行 inner
            # 这样生成器就能得以继续执行

            def inner(f: Any) -> None:
                # Break a reference cycle to speed GC.
                f = None  # noqa: F841
                self.run()

            self.io_loop.add_future(self.future, inner)
            return False
        return True

    def handle_exception(
        self, typ: Type[Exception], value: Exception, tb: types.TracebackType
    ) -> bool:
        if not self.running and not self.finished:
            self.future = Future()
            future_set_exc_info(self.future, (typ, value, tb))
            self.run()
            return True
        else:
            return False
```

对于上述源码，可以归纳为以下几点：

1. 如果生成器内部处理的`Future`对象都是done状态时
    - loop并未发挥任何作用
    - `coroutine`装饰器即可保证生成器执行完毕，并将执行结果保存在`result_future`中

2. 如果生成器内部处理的`Future`对象有一个不是done状态时
    - 生成器将在执行至该非done状态的`Future`对象时停止，不再运行
    - `Runner`在面对该非done状态的`Future`对象时，会调用`loop.add_future`方法添加一个回调函数callback
    - 这个callback会在该`Future`对象变成done状态时执行
    - callback执行后，会在loop上添加一个新的callback，这里就称为loop_callback吧
    - loop顾名思义，就是循环，loop在无限循环轮询需要处理的事件，比如执行loop_callback
    - loop_callback执行后，生成器会从执行中断的地方继续执行

loop相对生成器、`Future`来说，要复杂一些，这里就不展开论述（太多了……）。实际上，仅仅是最简单的生成器和`Future`这两者的结合就可以实现简单的协程。你可以把loop想象成一个协程的助力器，在协程中断的时候给协程“续命”。

虽然这里并不展开讲loop的实现，但可以通过举一些示例讲一下loop做的事情，聪明的你应该也能从这些事情中知道loop是如何让协程执行完毕的

```
In [3]: from tornado import gen

In [4]: sleep_future = gen.sleep(10)  # gen.sleep(10) 会往loop添加callback

In [5]: sleep_future
Out[5]: <Future pending>

In [6]: sleep_future.done()  # 如果loop不运行，sleep_future将一直是非done状态，loop运行后，会查看当前时间和sleep_future创建的时间，如果间隔超过10s，sleep_future变更为done状态
Out[6]: False
```
loop运行中会将满足一定条件的`Future`对象变更为done状态

```
@gen.coroutine
def handle_connection(sock, address):
    print('start to handle_connection')
    stream = IOStream(sock)  # sock是一个socket对象
    while True:
        message = yield stream.read_bytes(10)  # IOStream.read_bytes(self, num_bytes, partial=False)返回一个Future对象
        print("message from client:", message.decode().strip())
```
`stream.read_bytes(10)`返回一个`Future`对象，并向loop注册了一个handler（见`IOLoop.add_handler`方法），loop在轮询中如果发现注册的socket对象可读，就会执行相应的callback进行读取，读取完成后将该`Future`对象变更为done状态

以下代码来自关于loop的具体实现的部分源码：
```
for i in range(ncallbacks):
    self._run_callback(self._callbacks.popleft())
```
这部分代码显示，loop会执行添加进来的callback，其中就包括使生成器从执行中断的地方继续执行的callback

最后以一段简单的示例（运行环境ipython）结尾吧：
```
In [15]: from tornado import gen
    ...: from tornado.ioloop import IOLoop
    ...:
    ...:
    ...: @gen.coroutine
    ...: def test():
    ...:     print('start to sleep')
    ...:     yield gen.sleep(1)
    ...:     print('end')
    ...:
    ...:
    ...: test()  # 这就完成了一个协程的生成，但因为没有loop运行，协程将无法执行完毕
    ...:
    ...:
start to sleep
Out[15]: <Future pending cb=[coroutine.<locals>.wrapper.<locals>.<lambda>() at /Users/yao/Nustore Files/tornado-py/tornado/gen.py:226]>

In [16]: IOLoop.current().start()  # loop运行中，上面的协程执行完成
end
```
（完）
