---
title: "aws: SigV4 and event streams"
weight: 10
---

# aws {#aws}

[`github.com/go-llmflow/aws`](https://github.com/go-llmflow/aws) **v0.1.0** is AWS Signature
Version 4 request signing and AWS event-stream decoding, in pure Go, **standard library only**:
no aws-sdk-go, no cgo, no network, no files. It has two generic packages and no service-specific
code. This page is written from the repository's README and the v0.1.0 release notes; none of it
was run here.

```sh
go get github.com/go-llmflow/aws@v0.1.0
```

## What it is for

The [gateway]({{< relref "gateway.md" >}}) is planned to use it for an **Amazon Bedrock**
provider (gateway issue #89). **That work is in progress and not released: the gateway does not
support Bedrock today**, and nothing on this page says otherwise. (The library's README says the
gateway "uses them for its Amazon Bedrock provider" and "pre-release (no tag yet)"; the first is
the plan, the second predates the v0.1.0 tag.)

| Package | What |
| --- | --- |
| `sigv4` | sign an `*http.Request` with SigV4 (an `Authorization` header) |
| `eventstream` | decode the `application/vnd.amazon.eventstream` framing; an encoder for tests and tools |

## The API in brief

**`sigv4`.**

```go
s := sigv4.Signer{
    Credentials: sigv4.Credentials{AccessKeyID: id, SecretAccessKey: secret, SessionToken: tok},
    Region:      "us-east-1",
    Service:     "bedrock",
}
req, _ := http.NewRequest("POST", url, bytes.NewReader(body))
req.Header.Set("Content-Type", "application/json")
err := s.Sign(req, body, time.Now()) // sets X-Amz-Date, X-Amz-Security-Token (if any), Authorization
```

- `Sign(req, body, now)` only hashes the body; `req.Body` is not read. On error the request is
  left as it was.
- Every header on the request is signed except hop-by-hop ones, `User-Agent`, `X-Amzn-Trace-Id`,
  `Expect` and `Authorization`; `host` is always signed.
- For services other than S3 the path is encoded once more and normalised; service `s3` uses it as
  is.
- **Header-based signing only: presigned URLs and SigV4a are not implemented.**

**`eventstream`.**

```go
d := eventstream.NewDecoder(resp.Body, eventstream.Limits{}) // zero fields take the defaults
for {
    m, err := d.Next()           // *Message{Headers []Header; Payload []byte}
    if err == io.EOF { break }   // clean end, at a message boundary
    if err != nil { return err } // io.ErrUnexpectedEOF mid-message; errors.Is(err, eventstream.ErrLength) ...
    v, _ := m.Header(":event-type")
    fmt.Println(v.Str(), len(m.Payload))
}
```

Header values are typed (`Bool Int8 Int16 Int32 Int64 Bytes String Timestamp UUID`).
`Marshal` and `Encode` exist for tests and tools and reject what the decoder rejects. After an
error other than `io.EOF` the stream is desynchronised and `Next` keeps returning that error.

## Limits

The decoder reads untrusted input, so it is bounded (`Limits`; zero fields take these defaults):

| Field | Default |
| --- | --- |
| `MaxMessage` | 16 MiB |
| `MaxHeadersLen` | 128 KiB |
| `MaxHeaders` | 256 |
| `MaxValueLen` | 32,767 bytes (also the maximum accepted) |

The checks run in this order: the prelude bounds, then the **prelude CRC before anything is read
or allocated**, then the header block (no duplicate, empty or non-UTF-8 name, no unknown type,
valid UTF-8 strings, nothing running past `headers_len`), then the message CRC. The body buffer
grows with the bytes actually received, so a peer announcing 16 MiB and sending nothing costs
nothing.

## Test vectors

All **38 cases** of the aws-c-auth SigV4 test suite pass, at the canonical-request, string-to-sign
and signature levels. They are vendored with their Apache-2.0 licence and a `NOTICE`. Not covered
yet: SigV4a and the presigned-query variants. The decoder is also tested against the examples of
Amazon's Transcribe "Event stream encoding" page, whose first example is corrupted as printed:
the test suite pins that it is rejected as printed and decodes once four bytes are repaired.

## Security notes

- **Secrets are redacted.** `Credentials` redacts the secret key and the session token for every
  `fmt` verb, also inside a `Signer` or any struct, and no error contains a secret. A
  planted-secret test checks errors, prints, logs and the signed request.
- **A defect of the reference is not repeated.** In aws-sdk-go-v2, the payload length is computed
  as `Length - HeadersLen - 16`, which underflows a `uint32` when `Length < HeadersLen + 16`. This
  decoder **rejects that case** (`ErrLength`), with a planted test.
- Header names must be HTTP tokens and values are folded, so a value cannot forge another
  canonical header line; the access key, region and service cannot carry `/ , =` or control
  characters into `Credential=`.
- All length arithmetic is in `uint64`, limits are clamped to what a 32-bit `int` holds, and the
  decoder is fuzzed (`FuzzDecode`, `FuzzSign`, with committed seeds).

## Performance

**Not measured yet.** Benchmarks against aws-sdk-go-v2 are in a separate `_bench/` module, so
the library has no dependency, but the machine's load was above the measurement gate when v0.1.0
was released. No figure is given here.
