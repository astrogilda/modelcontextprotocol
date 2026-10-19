# SEP-0000: Execution Evidence Binding in Result `_meta`

- **Status**: Draft
- **Type**: Standards Track
- **Created**: 2026-10-09
- **Updated**: 2026-10-18 (adds the call binding)
- **Author(s)**: Sankalp Gilda (@astrogilda)
- **Sponsor**: None (seeking a sponsor)
- **PR**: https://github.com/modelcontextprotocol/modelcontextprotocol/pull/{NUMBER}

## Abstract

This SEP defines one optional `_meta` entry that a server MAY attach to a
`tools/call` result, and that an intermediary MAY attach on the server's behalf,
to bind the result to a signed record of what the call did. The entry carries
two values: the SHA-256 digest of an in-toto Statement, and that Statement's
`predicateType`. It carries no evidence itself. A client that holds the
Statement, from the server, an intermediary or a log, can check that it is the
one the result names, and can then verify it under the rules its
`predicateType` defines.

The SEP also defines a registry of the `predicateType` values that MCP
recognises in this entry. A type is listed only with a conformance corpus: a
published set of accepting and rejecting Statements at a released version, with
a digest, that any verifier can be run against. The first entry is the Agent
Audit Record predicate.

## Motivation

SEP-2817 carries client-asserted audit context on the request and states that
its fields "are not authorization evidence". It leaves the decision record, the
evidence of what the call actually did, to follow-up SEPs. SEP-3004 proposed a
record format and was closed without merging.

Today a server that produces a signed record of a call has no standard place to
say so in the protocol. Each implementation invents its own key, so a client or
a gateway cannot find the record, cannot tell which rules verify it, and cannot
detect that a record was swapped for another. A digest in the result binds the
two: an intermediary that substitutes a different record changes the digest,
and a record that does not hash to the digest the result carries is refused.

Naming the `predicateType` beside the digest tells a verifier which rules apply
before it fetches anything. Requiring a conformance corpus for each registered
type means two independent verifiers of one type reach the same verdict on the
same bytes, which is the property an audit consumer needs and the property a
type with only prose cannot show.

## Specification

### The `_meta` entry

A `tools/call` result MAY include, in `result._meta`:

```typescript
_meta["io.modelcontextprotocol/evidence"]?: {
  /** Lowercase hex SHA-256 of the in-toto Statement's bytes, as signed. */
  statementDigest: { sha256: string };
  /** The Statement's predicateType. MUST be a registered value. */
  predicateType: string;
  /** Optional location from which the Statement or its envelope can be fetched. */
  uri?: string;
}
```

Rules:

1. `statementDigest.sha256` MUST be 64 lowercase hexadecimal characters, and it
   MUST equal the SHA-256 of the Statement bytes that the DSSE envelope signs
   (the envelope's decoded `payload`).
2. `predicateType` MUST equal the `predicateType` member of that Statement, and
   MUST be listed in the registry below. A client MUST NOT treat an unlisted
   value as verifiable under this SEP.
3. A client that verifies the entry MUST refuse a Statement whose digest or
   `predicateType` differs from the entry, and MUST then apply the verification
   rules of the registered type. Agreement of the digest alone establishes only
   that the Statement is the one named; it establishes nothing the Statement
   claims.
4. Intermediaries that forward results SHOULD preserve the entry unmodified. An
   intermediary that attaches its own record MUST NOT replace a server's entry;
   it MAY add its own under a vendor prefix.
5. The entry is optional. Its absence MUST NOT change how a client handles a
   result.

### The call binding

A record about a call (a proxy's decision, a server's audit entry, the Statement
named above) needs a value that every hop agrees on. The JSON-RPC `id` is not
one: each hop may renumber it. `params._meta` is not part of it either: hops add
and rewrite it (`progressToken`, trace context, the SEP-2817 `aiInvocation`
entry). This SEP defines:

```
callBinding = "sha256:" + lowercase-hex(SHA-256(JCS({
  "method": <request method>,
  "params": <request params with the member "_meta" removed>
})))
```

where JCS is the JSON Canonicalization Scheme of RFC 8785.

Rules:

1. `id`, `jsonrpc` and `params._meta` are excluded. Every other member of
   `params` is included, at every depth, so a changed argument value, a changed
   array order or a changed tool name gives a different binding, and reordered
   object members give the same one.
2. A request whose `params._meta` is present and is not a JSON object is
   malformed, and no binding is computed for it.
3. A record about a call SHOULD carry its binding, in the record's own field or
   under `_meta["io.modelcontextprotocol/callBinding"]`. A verifier recomputes
   the binding from the request it holds and MUST refuse a record whose binding
   differs.
4. An intermediary MUST NOT move a member of `params` into `params._meta`. The
   binding detects it, because the moved member leaves the bound value.
5. The binding is computed over `params` as received, with no normalization
   beyond JCS. A request that omits `arguments` and one that carries
   `"arguments": {}` are different requests and bind differently, so an
   intermediary that fills in a default changes the binding.

The binding identifies the call, not its outcome: two records with equal
bindings are about the same request, and nothing more is established by it.

### Key name

Under the `_meta` key-name rules, a prefix whose second label is
`modelcontextprotocol` or `mcp` is reserved for MCP. This SEP requests that, if
accepted, MCP define `io.modelcontextprotocol/evidence`. Until then,
implementations SHOULD use the vendor key `io.github.probityai/evidence` with
the same value shape, and a client that recognises one SHOULD recognise both.

### The registry

The registry is a table in the specification. Each entry lists:

| Field              | Meaning                                           |
| ------------------ | ------------------------------------------------- |
| `predicateType`    | The type URI, exactly as it appears in Statements |
| Specification      | The normative text of the type                    |
| Conformance corpus | Repository, released version and corpus digest    |
| Counts             | Accepting and rejecting members in that release   |

A type is added by pull request. The pull request MUST name a conformance corpus
that contains at least one accepting and one rejecting Statement for every rule
the type's specification states, in which each rejecting member names the
accepting member it differs from, so that a verifier rejecting everything does
not pass. The corpus MUST be published at a released version with its digest,
and the specification's own CI MUST replay it against at least one verifier.

The first entry:

| Field              | Value                                                                                                                                                     |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `predicateType`    | `https://probityai.github.io/agent-evidence-vectors/predicate/v1/agent-audit-record`                                                                      |
| Specification      | Internet-Draft `draft-gilda-wimse-agent-audit-record-02`, which fixes the URI                                                                             |
| Conformance corpus | probityai/agent-evidence-vectors v0.20.0, `vectors-agent-audit-record/`, corpus digest `fec27bdf2660063872967d6abde35d7c4b86a4c5038237e952edfb0d92d884cf` |
| Counts             | 71 members (17 accept, 52 reject, 2 indeterminate)                                                                                                        |

The corpus ships in the PyPI package `agent-evidence-vectors==0.20.0`, and the
GitHub Action `probityai/agent-evidence-vectors@v0.20.0` with
`corpus: vectors-agent-audit-record` replays it against any verifier in CI.

## Conformance for the call binding

The reference cases are the `invocation_axes` corpus of
probityai/agent-evidence-vectors at commit
`b7d50a8e`, which pins the binding of its base request as
`sha256:16944d7c4e9028cd7923febf8e5c35d8b1be54b6d08ccfe2a35e1fb22e579761`.
Same binding: `argument_members_reordered`, `hop_metadata_added`. Different
binding: `argument_value_changed`, `array_order_changed`, `tool_name_changed`,
`argument_moved_into_meta`. The corpus ships with the reader
`agent-evidence-vectors-invocation-axes` in the PyPI package
`agent-evidence-vectors`; release 0.20.0 predates the `_meta` exclusion, so the
pinned release for this SEP is the first one after it.

## Rationale

A digest rather than the record: records can be large, can carry data a client
is not entitled to, and are often stored apart from the call. A digest keeps the
result small and still makes substitution detectable.

The `predicateType` beside the digest: without it a verifier must fetch and
parse the record to learn which rules apply, and a record of an unknown type
looks the same as a missing one.

A corpus per registered type rather than prose: prose specifications of signed
records have repeatedly admitted two readings of the same bytes. A shared corpus
is the cheapest way to make two verifiers agree, and it is checkable by anyone.

## Backward Compatibility

The entry is optional and lives in `_meta`, so clients and servers that do not
recognise it are unaffected.

## Security Implications

The entry is a pointer with a digest. It is not evidence by itself, and rule 3
forbids treating a digest match as verification of the record's content. A
server that lies in its record is caught only by the record type's own rules,
which is why each registered type must carry a corpus.

## Reference Implementation

The conformance corpus and verifier for the first registry entry are published
in probityai/agent-evidence-vectors. A corpus of `_meta` entries, accepting and
rejecting, for the rules in this SEP is to be published there before this SEP
leaves Draft.
