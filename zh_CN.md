# muxado - Go流复用

## 什么是流复用

想象你有一个像一个TCP连接的单一的流（双向的字节流）。流复用
是使多个同时流在一个基础传输流上传输的方法。

## 什么是muxado?

muxado是一个用go语言基于net.Conn之上实现的流复用库。muxado的协议在本文档中不是很明确，但跟在框架层移除了所有http明细字节的HTTP2协议实现非常相似。深度借鉴HTTP2, SPDY, 与WebMUX。

## muxado是怎么工作的?
Simplifying, muxado chunks data sent over each multiplexed stream and transmits each piece
as a "frame" over the transport stream. It then sends these frames,
often interleaving data for multiple streams, to the remote side.
The remote endpoint then reassembles the frames into distinct streams
of data which are presented to the application layer.

## muxado有什么好处?
A stream multiplexing library is a powerful tool for an application developer's toolbox which solves a number of problems:

- It allows developers to implement asynchronous/pipelined protocols with ease. Instead of matching requests with responses in your protocols, just open a new stream for each request and communicate over that.
- muxado can do application-level keep-alives and dead-session detection so that you don't have to write heartbeat code ever again.
- You never need to build connection pools for services running your protocol. You can open as many independent, concurrent streams as you need without incurring any round-trip latency costs.
- muxado allows the server to initiate new streams to clients which is normally very difficult without NAT-busting trickery.

## 演示代码!
As much as possible, the muxado library strives to look and feel just like the standard library's net package. Here's how you initiate a new client session:

    sess, err := muxado.DialTLS("tcp", "example.com:1234", tlsConfig)
    
And a server:

    l, err := muxado.ListenTLS("tcp", ":1234", tlsConfig))
    for {
        sess, err := l.Accept()
        go handleSession(sess)
    }

Once you have a session, you can open new streams on it:

    stream, err := sess.Open()

And accept streams opened by the remote side:

    stream, err := sess.Accept()

Streams satisfy the net.Conn interface, so they're very familiar to work with:
    
    n, err := stream.Write(buf)
    n, err = stream.Read(buf)
    
muxado sessions and streams implement the net.Listener and net.Conn interfaces (with a small shim), so you can use them with existing golang libraries!

    sess, err := muxado.DialTLS("tcp", "example.com:1234", tlsConfig)
    http.Serve(sess.NetListener(), handler)

## 一个更完整的muxado客户端

    // open a new session to a remote endpoint
    sess, err := muxado.Dial("tcp", "example.com:1234")
    if err != nil {
	    panic(err)
    }

    // handle streams initiated by the server
    go func() {
	    for {
		    stream, err := sess.Accept()
		    if err != nil {
			    panic(err)
		    }

		    go handleStream(stream)
	    }
    }()

    // open new streams for application requests
    for req := range requests {
	    str, err := sess.Open()
	    if err != nil {
		    panic(err)
	    }

	    go func(stream muxado.Stream) {
		    defer stream.Close()

		    // send request
		    if _, err = stream.Write(req.serialize()); err != nil {
			    panic(err)
		    }

		    // read response
		    if buf, err := ioutil.ReadAll(stream); err != nil {
			    panic(err)
		    }

		    handleResponse(buf)
	    }(str)
    }

## 怎么编译?
muxado is a modified implementation of the HTTP2 framing protocol with all of the HTTP-specific bits removed. It aims
for simplicity in the protocol by removing everything that is not core to multiplexing streams. The muxado code
is also built with the intention that its performance should be moderately good within the bounds of working in Go. As a result,
muxado does contain some unidiomatic code.

## API 文档
API documentation is available on godoc.org:

[muxado API documentation](https://godoc.org/github.com/inconshreveable/muxado)

## 最大的缺点?
Any stream-multiplexing library over TCP will suffer from head-of-line blocking if the next packet to service gets dropped.
muxado is also a poor choice when sending large payloads and speed is a priority.
It shines best when the application workload needs to quickly open a large number of small-payload streams.

## 状态
Most of muxado's features are implemented (and tested!), but there are many that are still rough or could be improved. See the TODO file for suggestions on what needs to improve.

## 协议
Apache
