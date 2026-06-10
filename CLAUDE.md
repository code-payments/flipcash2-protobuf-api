# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains the Protocol Buffer definitions for the Flipcash API, defining the contract between Flipcash clients and servers. It uses Docker-based code generation to produce client/server code for multiple languages.

## Code Generation

### Generate All Languages
```bash
make
```

### Generate Specific Languages
```bash
make go              # Generate Go code only
make protobuf-es     # Generate TypeScript/JavaScript code only
```

### Build Docker Images (without generation)
```bash
make docker-build          # Build both Docker images
make docker-build-go       # Build Go builder image only
make docker-build-protobuf-es  # Build Protobuf-ES builder image only
```

### Clean Generated Code
```bash
make clean  # Removes all generated code from generated/ directory
```

## Architecture

### Directory Structure

- `proto/` - Protocol Buffer definitions organized by domain
  - `proto/{domain}/v1/` - Each domain has its own versioned directory
  - Services and models are split into separate files where appropriate
- `generated/` - Auto-generated code (never edit directly)
  - `generated/go/` - Generated Go code with gRPC stubs and validation
  - `generated/protobuf-es/` - Generated TypeScript code with Connect-ES
- `build/` - Docker build contexts for code generation
  - `build/go/` - Go code generator (protoc + protoc-gen-go + protoc-gen-validate)
  - `build/protobuf-es/` - TypeScript code generator (protoc + Protobuf-ES plugins)

### Service Domains

The API is organized into logical service domains:

- **Account** - User registration, login, and account flags
- **Activity** - Activity feed for user actions
- **Chat** - Chat metadata and DM chat feed (paged, ordered by last activity; must be combined with the event stream for a complete view)
- **Common** - Shared message types used across domains (Auth, UserId, PublicKey, ChatId, IntentId, payment amounts, QueryOptions, etc.)
- **Contact** - Contact list syncing (full/delta uploads) and Flipcash contact discovery
- **Email** - Email verification and management
- **Event** - Bidirectional event streaming for real-time updates, plus server-to-server event forwarding
- **IAP** - In-app purchase processing
- **Intent** - Models only (no service): app metadata attached to payment intents (e.g. chat payments)
- **Messaging** - Sending/fetching chat messages, pointer advancement (read state), and typing notifications
- **Moderation** - Text and image content moderation with attestations
- **Phone** - Phone number verification
- **Profile** - User profile management
- **Push** - Push notification services
- **Resolver** - Maps real-world identifiers (e.g. phone numbers) to payment destination addresses
- **Settings** - User settings management
- **Third Party** - Third-party integrations

Some domains (e.g. Intent) only define models and have no service; their messages are embedded in other services' requests.

### Common Patterns

**Authentication**: Most services use the `common.v1.Auth` message with keypair-based signatures. The signature should be computed over the request message without the Auth field set.

**Validation**: All proto files use `protoc-gen-validate` annotations to enforce constraints on fields (min/max lengths, patterns, required fields, etc.).

**Versioning**: All packages use `/v1/` namespacing. When breaking changes are needed, create a new version directory.

**Pagination**: Services that return lists use `common.v1.QueryOptions` with `page_size`, `paging_token`, and `order` fields.

**Result Enums**: RPC responses typically include a `Result` enum indicating success or specific error conditions.

## Code Generation Details

### Go Generation

The Go code generator:
1. Builds a Docker image with Go 1.25, protoc, and required plugins
2. Installs `protoc-gen-go`, `protoc-gen-go-grpc`, and `protoc-gen-validate`
3. Clones common proto dependencies (googleapis, validate)
4. Generates `.pb.go`, `.pb.validate.go`, and `_grpc.pb.go` files
5. Go package path: `github.com/code-payments/flipcash2-protobuf-api/generated/go/{domain}/v1`

### TypeScript/JavaScript Generation

The Protobuf-ES code generator:
1. Builds a Docker image with Node.js 16 and Bufbuild's Protobuf-ES toolchain
2. Generates `*_pb.ts` (message types) and `*_connect.ts` (service clients) files
3. Post-processes imports to remove `.js` extensions
4. Creates `index.ts` barrel exports in each directory
5. Creates a root `index.ts` with namespaced exports

## Working with Proto Files

### Adding a New Service

1. Create `proto/{domain}/v1/{service}_service.proto` with service definition
2. Create `proto/{domain}/v1/model.proto` if you need domain-specific message types
3. Set the standard file options used by every proto in this repo:
   - `go_package = "github.com/code-payments/flipcash2-protobuf-api/generated/go/{domain}/v1;{domain}pb"`
   - `java_package = "com.codeinc.flipcash.gen.{domain}.v1"`
   - `objc_class_prefix = "FPB{Domain}V1"`
4. Use imports from `common/v1/common.proto` for shared types (Auth, UserId, PublicKey, etc.)
5. Add validation rules using `validate/validate.proto` annotations
6. Run `make` to generate code for all languages

### Modifying Existing Protos

- Never make breaking changes to existing fields (changing types, removing fields)
- You can add new optional fields or new RPC methods
- After changes, run `make` to regenerate code
- The generated code is committed to the repository for easy consumption

### Proto Style Guidelines

- Package names follow `flipcash.{domain}.v1`
- Use `snake_case` for field names
- Use `PascalCase` for message and enum names
- Use `SCREAMING_SNAKE_CASE` for enum values
- Include validation rules on all fields where constraints are needed
- Document complex fields with comments
- Keep models separate from services when they're reused across multiple services
