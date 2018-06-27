---
title: Elixir.process.介绍
date: 2018-06-28 00:21:51
tags:
---
这两天学习了一下 Elixir，第一次了解了 Elixir 中 process 的概念，现在以记录的方式让自己加深一下理解，如果错误的地方请指出。

在许多语言中，并发编程一般都是会牵涉到很复杂的情况，但是在 Elixir 中会相对来说比较容易，Elixir 作为建立在 Erlang 上的语言可以说本身就是为了并发而生的。像别的函数式编程语言一样，Elixir 中没有变量这一说，一切皆是不可变的。另外，Elixir 中的 process 之间是不可以进行资源的共享的，所以也不存在出现语言中存在的多线程下资源竞争的问题，虽然不能共享资源，但是 Elixir 中的 process 是可以相互之间进行消息的发送与接受。
<!-- more -->

### 创建一个 process

Elixir 提供了 `Kernel.spawn/1` 和 `Kernel.spawn/3` 方法来进行 process 的创建，两者区别在于前者接收一个匿名函数作为参数，后者接收 module 名、函数名和 `atom` 作为参数，使用 `spawn` 创建 process 后会返回一个 PID 用来获取到创建的 process.

``` elixir
defmodule HelloWorld do
	def hello do
		IO.puts "Hello World"	
	end
end

# iex
spawn(HelloWorld, :hello, [])
spawn(fn -> HelloWord.hello end)
```

### process 之间相互通信

前面说了，process 不能共享资源，但可以通过 `receive` 和 `send` 来进行通信。我们先将上面的 HelloWorld module 改写一下，用来接收来自 process 的信息。

``` elixir
defmodule HelloWorld do
	def hello do
		receive do
			{from, name} -> send(from, {:ok, "Hello, #{name}"})
		end
	end
end
```

在这里，`HelloWorld` 中的 `hello` 函数中，会按照模式匹配接收格式为 `{from, name}` 的信息，这里 `from` 代表发送来的 process 的 PID 信息，接收到信息之后，再使用 `send` 方法向发过来的 process 返回一个信息。定义好了上述 module 之后，我们可以在 iex 中进行消息发送：

``` elixir
pid = spawn(HelloWorld, :hello, [])
send pid, {self(), "first"}
receive do
	{:ok, msg} -> msg
end
```

在这里用到了 `self()`，这里是获取到当前调用 `self()` 所在 process 的 PID，然后将该 PID 和一个字符串信息发送到了 HelloWorld module 中，HelloWorld module 的 process 收到信息后会返回一个消息，返回的消息会放入一个消息池中，当用 `receive` 的时候，就会去池子中取出，达到了接收的效果。

在上述代码之后，我们如果继续向 pid 发送消息并进行接收，那么将接收不到任何消息，这是因为第一次发送消息后，`HelloWorld.hello` 接收到了然后就运行结束了，比如下面的代码将会超时打印出 “time out”：

``` elixir
send pid, {self(), "second"}
receive do
	{:ok, msg} -> msg
	after 500 -> "time out"
end
```

如果 `HelloWorld.hello` 仍然想继续保持监听，可以在 `send` 完信息之后再次调用 `hello` 方法，这样虽然是递归调用，但是 Elixir 做了尾递归优化，不会造成 stack overflow 的问题。

### 多个 process 之间的链接

在前面的代码中，process 之间是完全相互独立的，所以当一个 process 出现问题，比如出现了异常，别的 process 也是完全不知道的。如果需要知晓这种状况，可以用 `spawn_link` 方法进行 process 的创建，这样当出现了异常的时候，向异常 process 发送消息时会得到异常的提示，同时出现异常的 process 的 PID 也不再可用。通过 `spawn_link` 可以实现 Erlang 中经典的 OTP 模型中的 supervisor.

### 持有状态

虽然说 Elixir 中都是不可变得，但是仍然可以通过对函数的调用来让 process 持有一个状态。比如我们 spawn 了一个 Stack module 的 process，当对它进行 push 操作的时候可以添加一个值，而 pop 的时候可以剔除掉最新添加进去的值，peek 的话可以获取到当前 process 中存在的值，为了实现这样的要求，我们可以通过下列这样来做到：

``` elixir
defmodule Stack do
  def start do
    {:ok, spawn(__MODULE__, :loop, [[]])}
  end

  def loop(stack) do
    receive do
      {:push, from, val} ->
        send from, {:ok}
        loop([val | stack])
      {:pop, from} ->
        [first | rest] = stack
        send from, {:ok, first}
        loop(rest)
      {:peek, from} ->
        send from, {:ok, stack}
        loop(stack)
    end
  end

  def push(pid, val) do
    send pid, {:push, self(), val}
    receive do
      {:ok} ->
        IO.puts "push #{val} success"
    end
  end

  def pop(pid) do
    send pid, {:pop, self()}
    receive do
      {:ok, first} ->
        IO.puts "#{first} poped"
    end
  end

  def peek(pid) do
    send pid, {:peek, self()}
    receive do
      {:ok, stack} ->
        IO.puts "#{inspect stack}"
    end
  end
end
```

上面的代码比较多，但其实很好理解，简单解释一下。上述代码中 process 本身其实并没有持有任何值，而是通过 `loop` 方法的递归调用以及调用函数时传递的参数，并将参数通过 process 通信，再结合 Elixir 自身模式匹配的方式来达到 process 和状态值绑定的目的。

理解了本文的代码后，应该就能对 Elixir 的 process 有一定的了解了，process 是 Elixir 中的核心概念，理解它之后，之后会再记录一点对 OTP 介绍的文章，敬请期待。
