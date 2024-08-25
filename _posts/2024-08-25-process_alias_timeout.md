---
layout: post
title: Erlang OTP：Process 如何使用 Alias 改进消息通信
date: 2024-08-25 11:32:00 +0800
categories: ["技术工具"]
---

(以下内容由内容和 claude ai 协作完成)
## 引言

在Erlang OTP的世界中，高效可靠的进程间通信至关重要。传统上，`Ref`（引用）一直是维护进程通信对应关系的首选机制。然而，随着OTP 21中`Alias`的引入，开发者们现在拥有了一个更强大的工具。本文将探讨`Ref`的局限性，`Alias`的优势，并详细比较它们在超时处理场景中的使用。

otp20 源码
```erl
do_call(Process, Label, Request, Timeout) when is_atom(Process) =:= false ->
    Mref = erlang:monitor(process, Process),

    %% OTP-21:
    %% Auto-connect is asynchronous. But we still use 'noconnect' to make sure
    %% we send on the monitored connection, and not trigger a new auto-connect.
    %%
    erlang:send(Process, {Label, {self(), Mref}, Request}, [noconnect]),

    receive
        {Mref, Reply} ->
            erlang:demonitor(Mref, [flush]),
            {ok, Reply};
        {'DOWN', Mref, _, _, noconnection} ->
            Node = get_node(Process),
            exit({nodedown, Node});
        {'DOWN', Mref, _, _, Reason} ->
            exit(Reason)
    after Timeout ->
            erlang:demonitor(Mref, [flush]),
            % 假如在此处超时的 message 回来，将会一直保存在信箱中
            exit(timeout)
    end.
```

otp24 源码
```erl
do_call(Process, Label, Request, Timeout) when is_atom(Process) =:= false ->
    Mref = erlang:monitor(process, Process, [{alias,demonitor}]),

    Tag = [alias | Mref],

    %% OTP-24:
    %% Using alias to prevent responses after 'noconnection' and timeouts.
    %% We however still may call nodes responding via process identifier, so
    %% we still use 'noconnect' on send in order to try to send on the
    %% monitored connection, and not trigger a new auto-connect.
    %%
    erlang:send(Process, {Label, {self(), Tag}, Request}, [noconnect]),

    receive
        {[alias | Mref], Reply} ->
            erlang:demonitor(Mref, [flush]),
            {ok, Reply};
        {'DOWN', Mref, _, _, noconnection} ->
            Node = get_node(Process),
            exit({nodedown, Node});
        {'DOWN', Mref, _, _, Reason} ->
            exit(Reason)
    after Timeout ->
            erlang:demonitor(Mref, [flush]),
            % demonitor 后，alias 自动失效，alias 的 message 会被自动丢弃
            % 检查 mailbox 中可能存在的 message，作清理处理
            receive
                {[alias | Mref], Reply} ->
                    {ok, Reply}
            after 0 ->
                    exit(timeout)
            end
    end.

```

https://hexdocs.pm/elixir/1.13.4/GenServer.html#call/3

>`timeout` is an integer greater than zero which specifies how many milliseconds to wait for a reply, or the atom `:infinity` to wait indefinitely. The default value is `5000`. If no reply is received within the specified time, the function call fails and the caller exits. If the caller catches the failure and continues running, and the server is just late with the reply, it may arrive at any time later into the caller's message queue. The caller must in this case be prepared for this and discard any such garbage messages that are two-element tuples with a reference as the first element.

从 Elixir 的 GenServer.call 文档中，也有提到上面场景。


## 理解Ref及其局限性

`Ref`是Erlang的一种内置数据类型，用于创建全局唯一的标识符。它通常在进程通信中用于匹配请求和相应的响应。然而，`Ref`有几个局限性：

1. **消息顺序和匹配**：在高并发环境中，使用`Ref`可能导致消息顺序问题，使得难以匹配响应和请求。

2. **超时处理**：使用`Ref`处理超时可能比较复杂，常常导致引用在内存中长时间存在。

3. **分布式系统错误处理**：在分布式系统中，`Ref`可能无法及时提供有关远程节点故障的信息。

4. **性能开销**：在高并发场景下，创建和管理大量的`Ref`可能带来性能开销。

5. **生命周期管理**：`Ref`的生命周期由垃圾收集器管理，这可能不够灵活以满足所有使用场景。


## 引入Alias：

OTP 21中引入的`Alias`解决了`Ref`的许多局限性。以下是一些主要优势：

1. **改进的消息控制**：Alias可以设置为只接收一条消息后自动失效。

2. **增强的超时处理**：Alias可以与定时器结合使用，在超时后自动取消。

3. **更好的错误检测**：在分布式系统中，Alias提供了更快速、更可靠的错误检测机制。

4. **性能优化**：使用Alias可能比创建大量Ref更高效，尤其是在高并发场景下。

5. **灵活的生命周期管理**：Alias允许显式控制其生命周期，在资源管理方面提供了更大的灵活性。

官方 OTP Alias 的文档：https://www.erlang.org/doc/system/ref_man_processes#process-aliases

## 比较示例：超时处理

让我们比较一下使用`Ref`和`Alias`实现超时处理的方法：

### 使用Ref（传统方法）

```erlang
traditional_timeout() ->
    Ref = make_ref(),
    Server = spawn(fun() -> server_loop() end),
    Server ! {self(), Ref, request},
    receive
        {Ref, Response} ->
            io:format("收到响应: ~p~n", [Response])
    after 5000 ->
        io:format("请求超时~n")
    end.

server_loop() ->
    receive
        {From, Ref, request} ->
            timer:sleep(random:uniform(10000)),  % 模拟随机处理时间
            From ! {Ref, response}
    end.
```

### 使用Alias（改进方法）

```erlang
improved_timeout() ->
    Server = spawn(fun() -> server_loop() end),
    {Alias, Ref} = alias([{expiry, 5000}]),  % 创建一个5秒后过期的Alias
    Server ! {Alias, request},
    receive
        {Ref, Response} ->
            io:format("收到响应: ~p~n", [Response])
    after 5000 ->
        unalias(Alias),
        io:format("请求超时~n")
    end.

server_loop() ->
    receive
        {Alias, request} when is_reference(Alias) ->
            timer:sleep(random:uniform(10000)),  % 模拟随机处理时间
            Alias ! {response}
    end.
```

## 主要区别和改进

1. **创建和使用**：
   - Ref：使用`make_ref()`创建。
   - Alias：使用`alias([{expiry, 5000}])`创建，包含过期时间。

2. **消息发送**：
   - Ref：发送`{self(), Ref, request}`。
   - Alias：直接发送`{Alias, request}`。Alias本身包含了发送者信息。

3. **超时处理**：
   - Ref：使用`receive`的`after`子句。服务器可能在超时后仍继续处理。
   - Alias：5秒后自动过期，确保服务器不会处理已超时的请求。

4. **资源清理**：
   - Ref：没有显式的清理机制。
   - Alias：使用`unalias(Alias)`进行显式清理。即使不显式解除别名，Alias也会自动过期。

5. **服务器端处理**：
   - Ref：服务器需要记住发送者和Ref。
   - Alias：服务器只需知道Alias，简化了消息处理逻辑。

6. **错误处理**：
   - Ref：如果服务器崩溃，客户端可能会一直等到超时。
   - Alias：过期机制确保在服务器崩溃时更快地检测到错误。

## 结论

虽然`Ref`在简单场景中仍然有用，但`Alias`在复杂、分布式和高并发环境中提供了显著的改进。它提供了更好的消息生命周期控制、改进的错误处理和潜在的更好性能。

`Alias`在Erlang OTP中的引入展示了该语言为满足现代分布式系统需求而持续演进。通过理解和利用`Alias`，Erlang开发者可以创建更加健壮、高效和可维护的系统。

记住，`Ref`和`Alias`的选择应该基于您的具体使用场景。对于简单的本地通信，`Ref`可能仍然足够。然而，对于具有严格时间要求的复杂分布式系统，`Alias`可以提供实质性的好处。
