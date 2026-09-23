# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository holds the Protocol Buffer definitions for the Flipcash API: the contract between Flipcash clients and servers. The `.proto` files under `proto/` are the source of truth. Go and TypeScript code is generated from them inside Docker and committed to `generated/` so consumers can import the repo directly (Go consumers import the module `github.com/code-payments/flipcash2-protobuf-api`).

There is no CI in this repo. Regeneration is done locally and the output is committed alongside the proto change in the same PR.

## Code Generation

Docker is required. Nothing runs `protoc` on the host.

```bash
make                 # clean + build both Docker images + generate Go and TypeScript
make go              # build Go image + regenerate Go only
make protobuf-es     # build TS image + regenerate TypeScript only
make docker-build    # build both images without generating
make clean           # rm -rf generated/
```

Notes on the Makefile:
- `make` runs `clean` first, so `generated/` is deleted and fully rebuilt. `make go` / `make protobuf-es` only wipe their own subdirectory.
- The first run builds images from scratch (installs protoc, Go/Node plugins, clones googleapis and protoc-gen-validate). Subsequent runs use the Docker cache and are much faster.
- Targets are `.NOTPARALLEL`; don't try `make -j`.
- Generated code is always committed. After any proto change, run `make` and include the `generated/` diff in the commit.

### What the generators do

**Go** (`build/go/`): `golang:1.25` image, `protoc-gen-go` v1.35.1, `protoc-gen-go-grpc` v1.5.1, `protoc-gen-validate` v1.1.0, protoc v3.21.12 from apt. `generate.sh` runs protoc per file with `--experimental_allow_proto3_optional` and emits `*.pb.go`, `*_grpc.pb.go`, and `*.pb.validate.go`. The Makefile then moves output out of the nested `github.com/...` path into `generated/go/{domain}/v1/`.

**TypeScript** (`build/protobuf-es/`): `node:16-bookworm` image with `@bufbuild/protoc-gen-es` (v1.10.1) and `@bufbuild/protoc-gen-connect-es` (the legacy `@bufbuild/connect` packages, not `@connectrpc`). Emits `*_pb.ts` and `*_connect.ts` with `target=ts`, then post-processes:
1. Strips `.js` from `import ... _pb.js` lines.
2. Appends `export * from './<file>'` to an `index.ts` in each generated directory.
3. Appends `export * as {Domain} from './{domain}/v1'` to the root `generated/protobuf-es/index.ts`.

Known quirk: step 3 runs once per proto file, so domains with both a service and a model file get a duplicate `export * as X` line in the root `index.ts`. This is how the generator behaves today; don't hand-edit `generated/` to fix it.

### Viewing generated diffs

`.gitattributes` marks `generated/**` as `linguist-generated` and `-diff`, so `git diff` and `git show` report those files as binary. Use `git diff --text` (or `git show --text`) if you actually need to read a generated change.

## Repository Layout

- `proto/{domain}/v1/` – one directory per domain. Services live in `{name}_service.proto`; shared message types live in `model.proto`. Some domains only have one or the other. `e2ee` has two service files plus `content.proto`, the client-to-client plaintext schema that is encrypted before it reaches the server.
- `generated/go/{domain}/v1/` – Go package `{domain}pb`.
- `generated/protobuf-es/{domain}/v1/` – TypeScript, plus barrel `index.ts` files.
- `build/go/`, `build/protobuf-es/` – Dockerfile + `generate.sh` for each generator.
- `go.mod` – module root so generated Go is importable; pins runtime deps (`google.golang.org/protobuf`, `google.golang.org/grpc`, `protoc-gen-validate`).

## Service Domains

| Domain | Package | Service | Purpose |
|---|---|---|---|
| account | `flipcash.account.v1` | `Account` | Register, Login, user flags (authenticated and unauthenticated) |
| activity | `flipcash.activity.v1` | `ActivityFeed` | Notification feed: latest, paged, batch |
| blob | `flipcash.blob.v1` | `BlobStorage` | Direct-to-storage uploads via presigned targets; blob status and fresh download URLs. Server derives renditions and metadata; clients only upload ORIGINALs |
| blocklist | `flipcash.blocklist.v1` | `Blocklist` | Block/unblock users, IsBlocked, paged blocklist |
| chat | `flipcash.chat.v1` | `Chat` | Chat metadata, DM and group chat feeds, Start/Join/Leave group chats, chat rules |
| common | `flipcash.common.v1` | none | Shared types: Auth, PublicKey, Signature, UserId, Username, ChatId, IntentId, PhoneNumber, EmailAddress, payment amounts, PagingToken, QueryOptions, Locale, Region, Color, Substitution |
| contact | `flipcash.contact.v1` | `ContactList` | Contact sync (CheckSync, DeltaUpload, streaming FullUpload) and Flipcash contact discovery |
| e2ee | `flipcash.e2ee.v1` | `KeyDistribution`, `Mailbox` | Signal-protocol end-to-end encrypted messaging: the key distribution and ciphertext relay layer for E2EE chats, which share the `chat` domain's chat records, feeds and rosters with plaintext chats. `KeyDistribution`: per-device identity key certified by the account key, signed/one-time/KEM prekeys (PQXDH), prekey bundles, device list (Sesame). `Mailbox`: SendEnvelopes (ciphertext fan-out with device-mismatch enforcement) plus per-device mailbox drain and ack; live delivery is meant to ride `event.v1.StreamEvents`. `content.proto` is the client-to-client plaintext schema the server never sees, with its own content kinds (text and replies today) |
| email | `flipcash.email.v1` | `EmailVerification` | Send/check verification codes, unlink |
| event | `flipcash.event.v1` | `EventStreaming` | Bidirectional `StreamEvents` for real-time updates (ChatUpdate, BlobUpdate); internal `ForwardEvents` for server-to-server fan-out |
| iap | `flipcash.iap.v1` | `Iap` | In-app purchase completion |
| intent | `flipcash.intent.v1` | none | Models only: app metadata attached to payment intents (e.g. chat payments), embedded in other services' requests |
| messaging | `flipcash.messaging.v1` | `Messaging` | Messages (get/send/edit/delete), reactions, `GetDelta` event log stream, pointer advancement (read state), typing notifications |
| moderation | `flipcash.moderation.v1` | `Moderation` | Text and image moderation with attestations; `FlaggedCategory` is reused by other services |
| phone | `flipcash.phone.v1` | `PhoneVerification` | Send/check verification codes, unlink, link for payment |
| profile | `flipcash.profile.v1` | `Profile` | Display name, username, profile picture, tip card, min DM init fee, social account linking |
| push | `flipcash.push.v1` | `Push` | Push token registration; `Payload` models for push content |
| reporting | `flipcash.reporting.v1` | `Reporting` | Report a user, chat, message, or blob for review, with optional free-form description |
| resolver | `flipcash.resolver.v1` | `Resolver` | Resolves real-world identifiers (phone number, username) to payment destinations |
| settings | `flipcash.settings.v1` | `Settings` | UpdateSettings |
| thirdparty | `flipcash.thirdparty.v1` | `ThirdParty` | Third-party integrations (GetJwt) |

Cross-domain model dependencies form these layers (each imports the ones before it):

```
common → moderation → blob → { profile, messaging } → { chat, reporting } → { event, push, e2ee }
```

Concretely: `blob` imports `moderation`; `profile` and `messaging` import `blob`; `chat/v1/model.proto` imports `blob`, `profile`, and `messaging`; `reporting` imports `blob` and `messaging`; `event` and `push` import `chat` and `messaging`; `e2ee` imports `chat` (for `IdempotencyKey`) and `messaging` (for `Emoji` only; content kinds are defined in `e2ee`), but not `blob` or `event`. A lower layer must not import a higher one, or Go gets an import cycle. `common/v1` is imported by everything and must never import another Flipcash domain (see below).

## Proto Conventions

These are the patterns the existing files follow. Match the file you're editing.

### File header

Every file sets these three options with the domain substituted:

```proto
option go_package = "github.com/code-payments/flipcash2-protobuf-api/generated/go/{domain}/v1;{domain}pb";
option java_package = "com.codeinc.flipcash.gen.{domain}.v1";
option objc_class_prefix = "FPB{Domain}V1";
```

Known inconsistencies that exist in shipped code and must be left alone (changing them breaks consumers): `account` uses Go package name `acountpb` (typo), `event` uses `java_package ... .events.v1`, and `activity/v1/model.proto` uses `objc_class_prefix = "FCPBActivityV1"`.

### `common/v1` is a leaf package

`common/v1/common.proto` is imported by every domain. It must never import another `flipcash.*` package, or Go gets an import cycle. If a shared identifier is needed across domains, move it into `common` (this is how `ChatId`, `IntentId`, etc. ended up there) rather than importing a domain from `common`.

### Authentication

Authenticated requests carry `common.v1.Auth auth` with `[(validate.rules).message.required = true]`. The signature is over the serialized request with the `auth` field unset. By convention, the chat, blocklist, messaging, profile, reporting, and settings services put `auth` at **field number 10**, leaving 1–9 for payload fields. Other domains (blob, contact, event, resolver, activity, account, and the rest) place it wherever it fell, usually field 1 or the next free number. Follow whichever convention the file already uses; for a new domain, use 10.

Streaming RPCs authenticate on the first message: `StreamEventsRequest.Params` carries `auth` plus a `ts` timestamp used as a nonce (server rejects timestamps too far from now).

### Result enums

Every response message has a nested `Result` enum as field 1, with `OK = 0` and usually `DENIED = 1`, followed by RPC-specific failures (`NOT_FOUND`, `RULES_NOT_SATISFIED`, `TITLE_MODERATED`, ...). Payload fields are documented as "Set only when result == OK" (or when a specific non-OK result applies). Add new result values at the end; never renumber.

Enum values are aligned with spaces inside the enum block, e.g.:

```proto
enum Result {
    OK        = 0;
    DENIED    = 1;
    NOT_FOUND = 2;
}
```

### Pagination

Paged RPCs take `common.v1.QueryOptions query_options` (only `page_size` and `paging_token` are honored; ordering is fixed server-side and not client-selectable) and return `repeated ... [max_items: 100]`, `common.v1.PagingToken paging_token`, and `bool has_more`. The token is opaque and server-minted: unset on the first request, echoed back verbatim afterward.

Feeds ordered by a mutable key (the DM and group chat feeds) are served against a snapshot pinned by the paging token, and the contract requires clients to open the event stream **before** the first page and merge stream updates onto the paged set. That contract is documented at length in `chat/v1/chat_service.proto`; mirror that documentation style if adding a similar RPC.

### Validation

All constrained fields carry `validate/validate.proto` rules (`protoc-gen-validate`). Typical patterns: `message.required = true` for required sub-messages, `bytes min_len/max_len` for fixed-size ids, `string.pattern` for formats, `repeated max_items`, `enum in: [...]` to restrict accepted values, and `option (validate.required) = true` inside a `oneof`. Sizes marked `// todo: arbitrary` are placeholders that were never revisited.

### Style

- Packages `flipcash.{domain}.v1`; `snake_case` fields; `PascalCase` messages/enums; `SCREAMING_SNAKE_CASE` enum values.
- 4-space indentation in most files. Seven files use tabs (account, email, messaging, phone, profile, settings). Match the file you're in and don't reformat.
- Every RPC and non-obvious field gets a `//` doc comment. Server-side semantics (idempotency, "advisory" vs authoritative, what's set on which result) are documented in the proto because this file is the only contract clients see.
- Nested request/response sub-messages are declared inside the message that uses them (e.g. `StartChatRequest.GroupChatParameters`).
- Sub-message types used by more than one service go in `model.proto`; request/response messages stay in the service file.

## Making Changes

### Adding a field or RPC

1. Edit the proto. Only add: new fields with fresh field numbers, new enum values at the end, new RPCs. Never change a field's type or number, remove a field, or rename in a way that changes the wire/JSON name. To retire a field, the repo uses one of two patterns: keep serving it and mark it `[deprecated = true]` (see `bought_crypto` / `sold_crypto` in `activity/v1/model.proto`), or, when it is safe to stop serving, delete it and `reserved` its number with a comment naming what replaced it (see `ChatUpdate` in `event/v1/model.proto`, where `reserved 2` was `new_messages`, superseded by `events`).
2. Run `make` (or the single-language target) and confirm `generated/` changed for each proto file you edited. Only the edited files' output changes; importers of that file are not regenerated differently.
3. Commit the proto and generated files together.

### Adding a new domain

1. Create `proto/{domain}/v1/{name}_service.proto` and, if there are shared types, `proto/{domain}/v1/model.proto`.
2. Set the three file options above and `package flipcash.{domain}.v1`.
3. Import `common/v1/common.proto` and `validate/validate.proto` as needed.
4. Run `make`. The generators discover files with `find`, so no Makefile or script change is needed.
5. Add the service to the list in `README.md`.

### Breaking changes

Create a new version directory (`proto/{domain}/v2/`) rather than editing `v1`. No domain has needed this yet.

## Git and Release Conventions

- PRs are squash-merged to `main` with the PR number appended to the title, e.g. `Add GetGroupChatFeed RPC (#99)`. Commit titles are imperative and describe the API change.
- Releases are git tags `vMAJOR.MINOR.PATCH` (currently in the `v1.2x` range). Not every PR is tagged; a tag is cut when consumers need the change. Additive API changes bump MINOR. The one PATCH release so far (`v1.20.1`) was a fix that moved identifiers into `common` to break a Go import cycle.
- Consumers pin the Go module to a tag, so anything merged to `main` is effectively published once tagged.
