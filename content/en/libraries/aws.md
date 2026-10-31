---
title: "aws: SigV4 and event streams"
weight: 10
---

# aws {#aws}

[`github.com/go-llmflow/aws`](https://github.com/go-llmflow/aws) **v0.1.1** is AWS Signature
Version 4 request signing and AWS event-stream decoding, in pure Go, **standard library only**:
no aws-sdk-go, no cgo, no network, no files. It has two generic packages and no service-specific
code. This page is written from the repository's README and the v0.1.0 and v0.1.1 release notes (v0.1.1 has no API
change); none of it was run here.

```sh
go get github.com/go-llmflow/aws@v0.1.1
```

## What it is for

The [gateway]({{< relref "gateway.md" >}}) uses it for its
[Amazon Bedrock provider]({{< relref "bedrock.md" >}}) **since gateway v0.23.0**: SigV4 signing of
the InvokeModel requests and decoding of the response event stream. The library's CI has an
`sdk-witness` job that compares its signatures with aws-sdk-go-v2's on the gateway's Bedrock
paths. (The README of the v0.1.0 tag still says "pre-release (no tag yet)"; it predates the tag.)

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

Benchmarks against aws-sdk-go-v2 are in a separate `_bench/` module, so the library has no
dependency; `TestSameSignature` and `TestSameWire` check that both sides do the same work.

**These figures are indicative only.** The rule is to measure when the machine's one-minute load
average is at most 4; these runs (2026-11-01, Apple M4 Max, one core, `-count 6` or `8`) started at
load 3.6 to 4.3 and **ended at 4.4 to 6.8**, so they will be replaced by an idle run. They are the
README's, not re-run here.

| Benchmark | go-llmflow/aws | aws-sdk-go-v2 | |
| --- | --- | --- | --- |
| decode 1000 messages of 120 B | 283 us, 6.0 k allocs | 815 us, 40 k allocs | 2.9x faster |
| decode 1000 messages of 8 KiB | 1.27 ms | 2.87 ms | 2.3x faster |
| decode 1000 messages of 256 KiB | 42.0 ms | 47.6 ms | 1.13x faster |
| encode one message | 108 ns, 1 alloc | 395 ns, 19 allocs | 3.7x faster |
| sign a Bedrock request | 2.99 us, 49 allocs | 2.97 us, 51 allocs | equal within +/-2 % |

Signing was 22 % slower than the SDK in the first run (3.57 us against 2.94 us), because the SDK
derives the signing key once per day and region/service and v0.1.0 derived it on every request.
**v0.1.1 caches derived keys** and closes the gap. The cache is stricter than the SDK's: an entry
is keyed by the day, the region, the service **and a SHA-256 of the secret** (two credentials that
share an access key ID never share a key), it is bounded to 256 entries, and it holds no secret.
