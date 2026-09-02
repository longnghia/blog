---
date: "2026-08-05T10:55:35+07:00"
draft: false
title: "gRPC Playground"
summary: "Testing public gRPC endpoints on grpcb.in with grpcurl"
categories:
  - Code
tags:
  - grpc
  - grpcurl
  - testing
---

`grpcb.in` exposes both plaintext and TLS gRPC endpoints and has server reflection enabled, so it's ideal for testing with `grpcurl`.

### List available services

**TLS (port 9001):**

```bash
grpcurl grpcb.in:9001 list
```

**Plaintext (port 9000):**

```bash
grpcurl -plaintext grpcb.in:9000 list
```

### Describe a service

```bash
grpcurl grpcb.in:9001 describe hello.HelloService
```

Or list methods in a service:

```bash
grpcurl grpcb.in:9001 list hello.HelloService
```

### Unary RPC example

```bash
grpcurl \
  -d '{"greeting":"ChatGPT"}' \
  grpcb.in:9001 \
  hello.HelloService/SayHello
```

Expected response:

```json
{
  "message": "Hello ChatGPT"
}
```

### Addition service example

```bash
grpcurl \
  -d '{"a":10,"b":20}' \
  grpcb.in:9001 \
  addsvc.Addition/Sum
```

### Send custom headers

```bash
grpcurl \
  -H "Authorization: Bearer my-token" \
  -d '{"greeting":"Alice"}' \
  grpcb.in:9001 \
  hello.HelloService/SayHello
```

### Verbose output

To inspect the HTTP/2 exchange and metadata:

```bash
grpcurl -v \
  -d '{"greeting":"Alice"}' \
  grpcb.in:9001 \
  hello.HelloService/SayHello
```

If you're not sure what services are available, start with:

```bash
grpcurl grpcb.in:9001 list
```

and then:

```bash
grpcurl grpcb.in:9001 describe <service-name>
```

to discover the available methods and message schemas.
