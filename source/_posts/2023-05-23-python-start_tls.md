---
layout: post
title:  "Using asyncio start_tls in Python 3.11"
date:   2023-05-23 09:15:00 +0100
categories:
- Python
tags:
- python
- ssl
- asyncio
comments: true
---

An upgradable stream starts life as a plain old socket connection, but is capable of being “upgraded” to use Transport Layer Security (TLS). This is sometimes known as STARTTLS. Common examples of this are SMTP, LDAP, and HTTP proxy tunneling with CONNECT.

The has been broken in Python, but is fixed in version 3.11!

To make things work you will need an SSL certificate and key, and for that certificate to be trusted by a certificate chain.

You can find a gist for this [here](https://gist.github.com/rob-blackbourn/100dab4ebd7952c41e9bf2f078e6022b).

## Server

The server starts without TLS. When a client connects, the server responds to three messages:

* `PING` — the server responds with `PONG`.
* `STARTLS` — the server upgrades the connection to TLS.
* `QUIT` — the server closes the client connection.

Let’s see the code.

```python
import asyncio
from asyncio import StreamReader, StreamWriter
from functools import partial
from os.path import expanduser
import socket
import ssl

async def handle_client(
        ctx: ssl.SSLContext,
        reader: StreamReader,
        writer: StreamWriter
) -> None:
    print("Client connected")

    while True:
        request = (await reader.readline()).decode('utf8').rstrip()
        print(f"Read '{request}'")

        if request == 'QUIT':
            break

        elif request == 'PING':
            print("Sending pong")
            writer.write(b'PONG\n')
            await writer.drain()

        elif request == 'STARTTLS':
            print("Upgrading connection to TLS")
            await writer.start_tls(ctx)

    print("Closing client")
    writer.close()
    await writer.wait_closed()
    print("Client closed")

async def run_server():
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    ctx.load_verify_locations(cafile="/etc/ssl/certs/ca-certificates.crt")
    ctx.load_cert_chain(
        expanduser("~/.keys/server.crt"),
        expanduser("~/.keys/server.key")
    )

    handler = partial(handle_client, ctx)

    print("Starting server")
    server = await asyncio.start_server(socket.getfqdn(), host, 10001)

    async with server:
        await server.serve_forever()

if __name__ == '__main__':
    asyncio.run(run_server())
```

Looking at the `run_server` function, the first job is to build the SLL context. After creating the context, the certificate authority bundle is loaded, then the certificate and key.

The partial function binds the SSL context as the first argument to the `handle_client` callback.

The server is then started without TLS and set running. The fully qualified domain name (FQDN) is used for the host (the “any” address “0.0.0.0” would be fine, but the FQDN works better on Windows).

When a client connects the `handle_client` function is called. The function begins a loop. For each iteration the loop starts by reading a line from the client. If it reads `PING` it writes `PONG`. If it reads `QUIT` it breaks out of the loop and closes the connection. If it reads `STARTTLS` it calls `await start_tls(ctx)` to upgrade the connection. That’s all there is to it! Very neat.

## Client

This time let’s start with the code.

```python
import asyncio
import socket
import ssl

async def start_client():

    print("Connect to the server with using the fully qualified domain name")
    reader, writer = await asyncio.open_connection(socket.getfqdn(), 10001)

    print(f"The server certificate is {writer.get_extra_info('peercert')}")

    print("Sending PING")
    writer.write(b'PING\n')
    response = (await reader.readline()).decode('utf-8').rstrip()
    print(f"Received: {response}")

    print("Sending STARTTLS")
    writer.write(b'STARTTLS\n')

    print("Upgrade the connection to TLS")
    ctx = ssl.create_default_context(
        purpose=ssl.Purpose.SERVER_AUTH,
        cafile='/etc/ssl/certs/ca-certificates.crt'
    )
    await writer.start_tls(ctx)

    print(f"The server certificate is {writer.get_extra_info('peercert')}")

    print("Sending PING")
    writer.write(b'PING\n')
    response = (await reader.readline()).decode('utf-8').rstrip()
    print(f"Received: {response}")

    print("Sending QUIT")
    writer.write(b'QUIT\n')
    await writer.drain()

    print("Closing client")
    writer.close()
    await writer.wait_closed()
    print("Client disconnected")

if __name__ == '__main__':
    asyncio.run(start_client())
```

The client starts by opening a connection without TLS. As with the server the FQDN is used, but this time it’s important. When the client upgrades to TLS the host must match the certificate, so an IP address won’t work. There is a choice here though, as the `start_tls` call takes the host name as an optional argument. After connecting the client checks to see if there’s an SSL server certificate with `get_extra_info(“peercert”)` call. This should return `None`.

Next the client writes a `PING` to the server over the unencrypted stream and reads the result (which should be `PONG`).

The next step is to upgrade the connection. The client writes `STARTTLS` to instruct the server to start the handshake. An SSL context is then made, and the `start_tls(ctx)` function is called on the writer. The client then checks for an SSL server certificate with `get_extra_info("peercert”)` which should now exist.

The client then writes `PING` over the now encrypted stream and reads the result (which should be `PONG`).

Finally the client writes `QUIT` and closes the connection.

## Thoughts

It’s been a long time coming, but the result is so simple!

Good luck with your coding.