# Technology Stack

## Architecture

OpenList follows a **layered monolith** pattern with clear separation:
- **HTTP Layer**: Gin framework handling REST/WebDAV APIs
- **Driver Abstraction**: Pluggable storage drivers implementing common interface
- **Operations**: CRUD operations through `internal/op` with task queuing
- **Storage**: GORM-abstracted database with SQLite/MySQL/PostgreSQL support

## Core Technologies

| Category | Technology |
|----------|-----------|
| **Language** | Go 1.23+ |
| **Web Framework** | [Gin](https://github.com/gin-gonic/gin) |
| **CLI Framework** | [Cobra](https://github.com/spf13/cobra) |
| **ORM** | [GORM](https://gorm.io/) |
| **Frontend** | [SolidJS](https://www.solidjs.com/) (embedded at build time) |
| **WebDAV** | Custom implementation based on Go standard library |

## Key Libraries

- **Authentication**: JWT (`golang-jwt/jwt`), WebAuthn (`go-webauthn/webauthn`), OIDC (`coreos/go-oidc`)
- **Search**: Bleve (`blevesearch/bleve`) for full-text file indexing
- **Archive**: Sevenzip (`bodgit/sevenzip`), MHolt archives (`mholt/archives`)
- **Task Queue**: Custom task scheduler (`OpenListTeam/tache`)
- **Caching**: Typed cache layer (`OpenListTeam/go-cache`)
- **Cloud SDKs**: AWS SDK, Azure SDK, Google Drive SDK, and 80+ storage-specific clients

## Development Standards

### Type Safety
- No `interface{}` in new code; use generics where applicable
- Driver interfaces must be fully implemented (compile-time enforcement)
- Configuration structs use struct tags for JSON/XML/TOML marshaling

### Code Quality
- `go fmt` enforced in CI
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/)
- PR requires at least 1 review from maintainer with write access
- Signed commits recommended (GPG)

### Testing
- Unit tests in `*_test.go` files alongside source
- Integration tests for critical paths (auth, storage operations)
- Test coverage focus: driver implementations, API handlers, auth flows

## Development Environment

### Required Tools
| Tool | Version |
|------|---------|
| Go | 1.23+ |
| GCC | Any (for CGO with SQLite) |
| Node.js | 18+ (for frontend development) |
| pnpm | 8+ (frontend package manager) |

### Common Commands

```bash
# Backend development
go run main.go                    # Run server locally
go build -o openlist main.go      # Build binary

# Frontend development (separate repository)
cd OpenList-Frontend
pnpm dev                          # Start dev server
pnpm build                      # Build for production

# Add a new storage driver
cp -r drivers/template drivers/your_driver
# Edit drivers/your_driver/meta.go and implement the Driver interface

# Run tests
go test ./...
```

## Key Technical Decisions

1. **Monolith over Microservices**: Single binary deployment reduces ops complexity; plugin-style driver system provides extensibility without service boundaries

2. **Gin over stdlib**: Fast, well-documented HTTP framework with middleware ecosystem; balances performance with developer velocity

3. **Driver Interface Pattern**: All storage backends implement identical Go interface (`List`, `Get`, `MakeDir`, etc.) enabling transparent storage switching

4. **Embedded Frontend**: SolidJS SPA compiled and embedded via Go embed; single binary deployment without separate web server

5. **Task Queue (tache)**: Custom scheduler for long-running operations (offline downloads, batch operations) instead of external message queue

6. **GORM with SQLite default**: Zero-config SQLite for personal use; production can switch to MySQL/PostgreSQL via config change

7. **AGPL-3.0 License**: Copyleft ensures derivative works remain open; aligns with project ethos of transparency and community ownership

---
_Document patterns and decisions, not every dependency. New dependencies following existing patterns don't require updates._
