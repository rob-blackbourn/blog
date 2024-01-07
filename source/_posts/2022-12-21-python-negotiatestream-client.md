---
layout: post
title:  "A Python client for .Net NegotiateStream"
date:   2022-12-21 11:52:00 +0000
categories:
- Python
tags:
- python
- NegotiateStream
- asyncio
comments: true
---

I find the .Net `NegotiateStream` class useful for internal services as it provides single sign on, authentication and basic encryption. However, as a python programmer, I couldn’t find a client library.

## TL;DR

For the impatient the repo can be found on [GitHub](https://github.com/rob-blackbourn/jetblack-negotiate-stream).

# The Protocol

I won’t go into detail regarding the protocol, but it has two key features: a handshake (during which credentials are exchanged), and then the encryption/decryption of data sent/received.

The handshake stage is widely used by Windows intranet servers to avoid the need for entering credentials, and involves the generation and consumption of tokens which are base64 encoded and passed as HTTP headers. I’ve done this before using the
[pyspnego](https://github.com/jborean93/pyspnego) package
for the [SSPI authentication](https://github.com/rob-blackbourn/bareasgi-sspi) layer
of a Python web server. The SPNEGO client generates a token that is passed to the server, which responds with a new token, and so on until authentication is complete.

The wire protocol uses the same tokens, but has different "headers". This was revealed by inspecting the code from the
[net.tcp-proxy](https://github.com/ernw/net.tcp-proxy) project.
During the handshake the header takes the form:

```python
struct.pack(">BBBH", state, major, minor, payload_size)
```

The state can be one of "done" (0x14), "error" (0x15) and "in progress" (0x16). The major.minor version is 1.0, and the payload size is the length of the generated token. When the handshake is complete the header changes to simply the payload size, but as an unsigned int.

Let’s look at some code! First I needed to handle the handshake header record.

```python
from __future__ import annotations

import enum
import struct

class HandshakeState(enum.IntEnum):
    DONE = 0x14
    ERROR = 0x15
    IN_PROGRESS = 0x16


class HandshakeRecord:

    FORMAT = ">BBBH"

    def __init__(
            self,
            state: HandshakeState,
            major: int,
            minor: int,
            payload_size: int
    ) -> None:
        self.state = state
        self.major = major
        self.minor = minor
        self.payload_size = payload_size

    def pack(self) -> bytes:
        return struct.pack(
            self.FORMAT,
            self.state,
            self.major,
            self.minor,
            self.payload_size
        )

    @classmethod
    def unpack(cls, buf: bytes) -> HandshakeRecord:
        (state, major, minor, payload_size) = struct.unpack(cls.FORMAT, buf)
        return HandshakeRecord(state, major, minor, payload_size)
With that, and the help of pyspnego I can implement reading and writing.

class NegotiateStream(Stream):

    def __init__(self, hostname: str, socket_: socket.socket) -> None:
        super().__init__(socket_)
        self._handshake_state = HandshakeState.IN_PROGRESS
        self._client = spnego.client(hostname=hostname)

    def write(self, data: bytes) -> None:
        if self._handshake_state == HandshakeState.IN_PROGRESS:
            handshake = HandshakeRecord(self._handshake_state, 1, 0, len(data))
            header = handshake.pack()
            self.send(header + data)
        else:
            while data:
                chunk = self._client.wrap(data[:0xFC30])
                header = struct.pack('<I', len(chunk.data))
                self.send(header + chunk.data)
                data = data[0xFC30:]

    def read(self) -> bytes:
        if self._handshake_state == HandshakeState.DONE:

            payload_size = struct.unpack('<I', self.recv(4))[0]
            payload = self.recv(payload_size)
            unencrypted = self._client.unwrap(payload)
            return unencrypted.data

        buf = self.recv(struct.calcsize(HandshakeRecord.FORMAT))
        handshake = HandshakeRecord.unpack(buf)

        self._handshake_state = handshake.state

        if self._handshake_state != HandshakeState.ERROR:
            return self.recv(handshake.payload_size)

        if handshake.payload_size == 0:
            raise IOError("Negotiate error")

        payload = self.recv(handshake.payload_size)
        _, error = struct.unpack('>II', payload)
        raise IOError(f"Negotiate error: {error}")

    ...
```

Note the different headers. When the handshake is "in progress" the handshake style of header is used. After authentication the shorter form is used, and the client has to encrypt and decrypt the data.

Finally the handshake itself.

```python
class NegotiateStream:

    ...

    def authenticate_as_client(self) -> None:
        in_token: Optional[bytes] = None
        while not self._client.complete:
            out_token = self._client.step(in_token)
            if not self._client.complete:
                self.write(out_token)
                in_token = self.read()

    ...
```

This turned out to be pretty simple!

I coded up a simple client:

```python
import socket

from jetblack_negotiate_stream import NegotiateStream

def main():
    hostname = socket.gethostname()
    port = 8181

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((hostname, port))

        stream = NegotiateStream(hostname, sock)

        stream.authenticate_as_client()
        for data in (b'first line', b'second line', b'third line'):
            stream.write(data)
            response = stream.read()
            print("Received: ", response)

    print("Done")


if __name__ == '__main__':
    main()
```

And a simple C# echo server:

```csharp
using System;
using System.Net;
using System.Net.Security;
using System.Net.Sockets;
using System.Text;

namespace NegotiateStreamServer
{
    internal class Program
    {
        static void Main(string[] args)
        {
            var listener = new TcpListener(IPAddress.Any, 8181);
            listener.Start();

            while (true)
            {
                Console.WriteLine("Listening ...");
                var client = listener.AcceptTcpClient();

                try
                {
                    Console.WriteLine("... Client connected.");

                    Console.WriteLine("Authenticating...");
                    var stream = new NegotiateStream(client.GetStream(), false);
                    stream.AuthenticateAsServer();

                    Console.WriteLine(
                        "... {0} authenticated using {1}",
                        stream.RemoteIdentity.Name,
                        stream.RemoteIdentity.AuthenticationType);

                    var buf = new byte[4096];
                    for (var i = 0; i < 4; ++i)
                    {
                        var bytesRead = stream.Read(buf, 0, buf.Length);
                        var message = Encoding.UTF8.GetString(buf, 0, bytesRead);
                        Console.WriteLine(message);
                        stream.Write(buf, 0, bytesRead);
                    }
                    stream.Close();
                }
                catch (Exception ex)
                {
                    Console.WriteLine(ex.ToString());
                }
            }
        }
    }
}
```

What a lot of code the server is. If you run the server, then the client, you should see the handshake, and the messages passing to and fro, encrypted.

As most of my Python servers are async, I made a couple of async versions. One is a straight async version of the synchronous socket client. The second is more"asyncio" style. I won’t show the code here, but here is a demo program using it:

```python
import asyncio
import socket

from jetblack_negotiate_stream import open_negotiate_stream

async def main():
    hostname = socket.gethostname()
    port = 8181

    reader, writer = await open_negotiate_stream(hostname, port)

    for data in (b'first line', b'second line', b'third line'):
        writer.write(data)
        await writer.drain()
        response = await reader.read()
        print("Received: ", response)

    writer.close()
    await writer.wait_closed()

if __name__ == '__main__':
    asyncio.run(main())
```

By using the same pattern as `asyncio.open_connection` the negotiation gets tidied away inside `open_negotiate_stream` providing a clean consistent interface.

If you find any bugs, or wish to add features, please post issues and pull requests to the repo.

