---
title: WebSocket Channel<T> Queues
tags: c# .net asp.net asp.net-core websocket
header:
  image: "/assets/2019/08-17/header1280.png"
  teaser: "/assets/2019/08-17/header1280.png"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2020/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2020/mm-dd/pic.jpg)
---

High-performance Channel queues for threadsafe WebSocket management.

<!--more-->

Last year I wrote a few articles about WebSockets (the important topics being [closing WebSockets]({{ site.baseurl }}{% post_url 2019-08-17-how-to-close-websocket-correctly %}) and [Kestrel hosting]({{ site.baseurl }}{% post_url 2019-08-18-minimal-full-feature-kestrel-websocket-server %})). Those articles used the .NET `BlockingCollection` object to share data between threads -- basically a publish/subscribe model. With just a few minor changes, the modern, high-performance, `Span`-based `Channel` class simplifies and improves those examples.

Like the earlier articles, I have added new projects to the WebSocketExample solution found in the same GitHub repository used by my previous posts about WebSockets, namely [WebSocketExample](https://github.com/MV10/WebSocketExample). Those projects now target .NET Core 3.1. The repository still contains the projects from the older articles based on the obsolete `HttpListener` class, but I have removed them from the solution.

## The Client Application

The new project `ChannelWSClient` is almost identical to the `BlockingCollection` project from last year, `WebSocketClient`. Instead of declaring a `BlockingCollection<string>` variable, we declare a `Channel<string>` variable:

```csharp
// private static BlockingCollection<string> KeystrokeQueue = new BlockingCollection<string>();
private static Channel<string> KeystrokeQueue;
```

This time I'm creating the instance in the startup method, since the new-up process is a bit longer and a bit more complex. Channels have "bounded" and "unbounded" versions. A bounded channel is of a limited size -- channel-writers can call a method that either fails if the channel is full, or another method that waits until the channel is available for writing again, and the channel definition includes options about how to handle write-requests when the channel is full. However, we're using an unbounded channel, meaning it can hold an unlimited amount of data. Most articles warn you that if channel-readers are too slow, you risk memory exhaustion -- it is truly unbounded.

We also specify two options in the call to the channel factory -- `SingleReader` and `SingleWriter` are both true. Various optimizations / assumptions can be applied inside the channel object when it knows concurrent reads and/or writes will never occur.

```csharp
public static async Task StartAsync(Uri wsUri)
{
    Console.WriteLine($"Connecting to server {wsUri.ToString()}");

    KeystrokeQueue = Channel.CreateUnbounded<string>(
        new UnboundedChannelOptions 
        { 
            SingleReader = true, 
            SingleWriter = true 
        });

    SocketLoopTokenSource = new CancellationTokenSource();
    // ...etc.
}
```

The next change adds a user's keystroke to the queue. The old code is shown commented out, with the replacement immediately following. Even though the method is named `TryWrite`, it is guaranteed to succeed when writing to an unbounded channel. 

```csharp
// public static void QueueKeystroke(string message)
//     => KeystrokeQueue.Add(message);

public static void QueueKeystroke(string message)
    => KeystrokeQueue.Writer.TryWrite(message);
```

The final change is in the `KeystrokeTransmitLoopAsync` method. Originally the main processing loop looked like this (excluding error handling):

```csharp
while(!cancellationToken.IsCancellationRequested)
{
    await Task.Delay(Program.KEYSTROKE_TRANSMIT_INTERVAL_MS, cancellationToken);
    if (!cancellationToken.IsCancellationRequested && KeystrokeQueue.TryTake(out var message))
    {
        var msgbuf = new ArraySegment<byte>(Encoding.UTF8.GetBytes(message));
        await Socket.SendAsync(msgbuf, WebSocketMessageType.Text, endOfMessage: true, CancellationToken.None);
    }
}
```

The channel-based implementation eliminates the delay. Instead we asynchronously wait for data to show up in the channel with a call to `WaitToReadAsync`. When that returns `true` we then call `ReadAsync` to retrieve the data. The new loop (again sans error handling, which hasn't changed) looks like this:

```csharp
while(!cancellationToken.IsCancellationRequested)
{
    while(await KeystrokeQueue.Reader.WaitToReadAsync(cancellationToken))
    {
        string message = await KeystrokeQueue.Reader.ReadAsync(cancellationToken);
        var msgbuf = new ArraySegment<byte>(Encoding.UTF8.GetBytes(message));
        await Socket.SendAsync(msgbuf, WebSocketMessageType.Text, endOfMessage: true, CancellationToken.None);
    }
}
```

That's all it takes. And believe it or not, even though that `Task.Delay` was only 100ms, this already feels more responsive. The changes in the Kestrel web server project are similarly simple.

## The Kestrel Application

Once again, the new `ChannelWSServer` project is almost identical to the older `BlockingCollection`-based project, `KestrelWebSocketServer`. The `ConnectedClient` class is used to track individual clients. As we did earlier, we declare a `Channel<string>` variable to queue our data:

```csharp
// public BlockingCollection<string> BroadcastQueue { get; } = new BlockingCollection<string>();
public Channel<string> BroadcastQueue { get; private set; }
```

This time we create the channel in the class constructor. Notice in the server we're setting `SingleWriter` to false. The server has two threads running -- one which echoes user keystrokes, but also a timer-based service which periodically emits a timestamp to all connected clients. This means two separate threads _could_ try to write to the channel simultaneously, so we must disable the single-writer optimizations.

```csharp
public ConnectedClient(int socketId, WebSocket socket, TaskCompletionSource<object> taskCompletion)
{
    SocketId = socketId;
    Socket = socket;
    TaskCompletion = taskCompletion;
    BroadcastQueue = Channel.CreateUnbounded<string>(
        new UnboundedChannelOptions 
        { 
            SingleReader = true, 
            SingleWriter = false 
        });
}
```

We then have the exact same processing loop in `BroadcastLoopAsync` that we saw in the client code (again, with error handling omitted for brevity):

```csharp
while (!cancellationToken.IsCancellationRequested)
{
    while(await BroadcastQueue.Reader.WaitToReadAsync(cancellationToken))
    {
        string message = await BroadcastQueue.Reader.ReadAsync();
        Console.WriteLine($"Socket {SocketId}: Sending from queue.");
        var msgbuf = new ArraySegment<byte>(Encoding.UTF8.GetBytes(message));
        await Socket.SendAsync(msgbuf, WebSocketMessageType.Text, endOfMessage: true, CancellationToken.None);
    }
}
```

Over in the `WebSocketMiddleware` class, we have to change how the `Broadcast` method puts data into the queue. The commented-out lines are the old loop, followed by the channel-based version. Once again we use `TryWrite` because unbounded queues are guaranteed to be available for writes:

```csharp
// foreach (var kvp in Clients)
//     kvp.Value.BroadcastQueue.Add(message);

foreach (var kvp in Clients)
    kvp.Value.BroadcastQueue.Writer.TryWrite(message);
```

The final change is to alter the echo feature in `SocketProcessingLoopAsync`. The old code is commented out and the channel version follows it:

```csharp
if (client.Socket.State == WebSocketState.Open)
{
    string message = Encoding.UTF8.GetString(buffer.Array, 0, receiveResult.Count);
    //client.BroadcastQueue.Add(message);
    client.BroadcastQueue.Writer.TryWrite(message);
}
```

It's that easy.

## Channel Completion

One `Channel` feature I'm not using here is "completion" -- it is possible for the channel writer to essentially close the channel, which signals to the reader that no messages will ever be written again. Because of the way WebSocket lifecycles work, it's not useful here, but in other pub-sub scenarios it may be useful to allow the reader processes to release resources and end processing. The completion is an awaitable task on the `Reader.Completion` property, and the writer would call `Write.TryComplete`.

## Conclusion

I actually started playing around with channels while trying to improve image-processing throughput on a Raspberry Pi-based security camera. It dawned on me that these would be ideal for managing the queues in a WebSocket application (which I'll also need for my Pi project). Even though I'd already used the `Channel` class, I was still pretty amazed at how easy it was to migrate these projects. 

There are other interesting usage-patterns with the `Channel` class that are worth reading about. I found [this](https://ndportmann.com/system-threading-channels/) article by Nicholas Portmann to be an especially easy read with a lot of interesting tips and commentary.
