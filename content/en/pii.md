---
title: "PII recognisers"
weight: 28
---

# pii {#pii}

Finds personal data in text, in pure Go with no dependencies (standard library
only, `CGO_ENABLED=0`). It was written for the PII guardrail of the
[gateway]({{< relref "gateway.md" >}}), which masks prompts before they reach the
model and masks completions before they reach the client. The package itself only
reports what it found and where; the caller decides what to do with it.
[Repository →](https://github.com/go-llmflow/pii)

This page is written from the repository's README at **v0.1.0**.

```sh
go get github.com/go-llmflow/pii@v0.1.0
```

## Using it {#api}

```go
s := pii.New()                       // every kind; pii.New(pii.IBAN, pii.Card) for some
ms := s.Find(text)                   // []pii.Match{Kind, Start, End}, ordered, no overlap
clean := pii.Mask(text, ms, pii.Placeholder) // "IBAN [IBAN], mail [EMAIL]"
```

`pii.MaxLen` (254 bytes) bounds the length of any match. A caller masking a stream
can hold back the last `MaxLen` bytes, and will then never release the start of a
match whose end it has not seen yet.

## What a match means {#recognisers}

A recogniser that has a check digit uses it. A match is therefore a value that
could exist, not just a string with the right number of digits.

| Kind | Recognised by | Checked by |
| --- | --- | --- |
| `iban` | 2 letters, 2 digits, then alphanumerics; single spaces allowed | ISO 13616: the exact length for the country, and mod 97 = 1 |
| `nir` | the French social security number: sex 1/2/3/4/7/8, year, month, department (2A/2B for Corsica), commune, order number, 2-digit key | key = 97 - (the 13 digits mod 97), with 2A read as 19 and 2B as 18 (INSEE) |
| `card` | 13 to 19 digits, single spaces or dashes allowed | Luhn (ISO/IEC 7812-1 annex B), plus a known network's prefix and length: Visa, Mastercard, American Express, Discover, JCB, Diners |
| `email` | a practical subset of RFC 5322 addr-spec | local part at most 64 octets and address at most 254 (RFC 5321). A `git@host:path` remote is not treated as a mailbox |
| `phone` | E.164 (`+`, at most 15 digits) and the French plan (`0` + 9 digits, `+33` / `0033`) | format only, since phone numbers have no checksum |
| `ipv4` | dotted decimal, not inside a longer dotted run (version numbers, OIDs) | `net/netip` |
| `ipv6` | the three text forms of RFC 4291, including a trailing IPv4 | `net/netip`; zones are not accepted |

When two candidates overlap, the one that starts first wins, then the longer one,
then the checksum-validated kind.

## False positives, measured {#false-positives}

`testdata/benign.txt` contains no personal data by construction: 45 lines of text
a model reads and writes every day (versions, OIDs, dates, UUIDs, hashes, ISBN,
EAN, SIREN and SIRET numbers, prices, MAC addresses, C++ and Rust `::`, an SSH git
remote, and a card number, a NIR and an IBAN with wrong keys). `TestFalsePositives`
fails if a count rises above what was measured.

- `iban`, `nir`, `card`, `email`, `ipv4`, `ipv6`: **0** false positives.
- `phone`: **1**, `ref 0123456789`, a valid French number in every respect except
  its meaning.

Phone numbers are the one kind that cannot be checked. A guardrail that masks them
will also mask 10-digit references starting with 0.

## Security and performance {#security}

The input is untrusted (prompts and model output). Every regular expression runs
on Go's RE2 engine, in linear time. `FuzzFind` checks that matches are ordered and
within bounds, that every checksum-validated match re-validates on its own, and
that masking twice gives the same result as masking once. The tests also run with
`GOARCH=386`.

`BenchmarkFind` scans 64 KiB of the benign corpus. The comparison against
Microsoft Presidio, the reference in this domain, has **not been made yet**.
