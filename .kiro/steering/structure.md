# Project Structure

## Organization Philosophy

OpenList uses a **domain-driven layered architecture** with clear separation between API, business logic, and storage concerns:

- **`cmd/`** - CLI entry points using Cobra
- **`internal/`** - Business logic that shouldn't be imported by external packages
- **`server/`** - HTTP handlers, middleware, and WebDAV implementation
- **`drivers/`** - Pluggable storage driver implementations
- **`pkg/`** - Reusable packages that may be imported externally
- **`public/`** - Static assets and frontend embedding
- **`wrapper/`** - Cross-compilation helpers for different architectures

## Directory Patterns

### CLI Commands
**Location**: `/cmd/`  
**Purpose**: Cobra-based CLI commands and subcommands  
**Example**: `cmd/root.go` defines the main command; `cmd/admin.go`, `cmd/user.go` implement subcommands

### Driver System
**Location**: `/drivers/`  
**Purpose**: Storage driver implementations following the `driver.Driver` interface  
**Example**: `drivers/aliyundrive/`, `drivers/s3/`, `drivers/webdav/` - each implements `List()`, `Get()`, `MakeDir()`, etc.

### HTTP Layer
**Location**: `/server/`  
**Purpose**: HTTP handlers, middleware, and protocol implementations  
**Example**: `server/handles/` contains Gin handlers; `server/middlewares/` has auth, rate limiting, etc.

### Business Logic
**Location**: `/internal/`  
**Purpose**: Core business operations not exposed to external packages  
**Example**: `internal/op/` contains file operations; `internal/driver/` has driver abstraction

### Reusable Libraries
**Location**: `/pkg/`  
**Purpose**: Packages that can be imported by external projects  
**Example**: `pkg/utils/` has general utilities

### Static Assets
**Location**: `/public/`  
**Purpose**: Static files and frontend SPA embedding  
**Example**: `public/public.go` uses Go embed to include built frontend

### Cross-Compilation Wrappers
**Location**: `/wrapper/`  
**Purpose**: Platform-specific compilation helpers for cross-compilation  
**Example**: `zcc-arm64`, `zcxx-win7` scripts for different target architectures

## Naming Conventions

- **Files**: Use `snake_case.go` for most files; `type.go` for type definitions, `util.go` for utilities
- **Drivers**: Directory named after storage provider (e.g., `aliyundrive/`, `google_drive/`)
- **Handler files**: Named by resource (e.g., `user.go`, `storage.go`)
- **Functions**: MixedCase (Go convention); handlers start with resource name (e.g., `UserLogin`, `StorageCreate`)

## Import Organization

Standard Go import organization with three groups:

```go
import (
    // Standard library
    "context"
    "fmt"
    "net/http"

    // Third-party packages
    "github.com/gin-gonic/gin"
    "github.com/spf13/cobra"

    // Project packages
    "github.com/OpenListTeam/OpenList/v4/internal/driver"
    "github.com/OpenListTeam/OpenList/v4/internal/op"
    "github.com/OpenListTeam/OpenList/v4/pkg/utils"
)
```

**Path Aliases**:
- No custom path aliases used; imports use full module path `github.com/OpenListTeam/OpenList/v4/...`

## Code Organization Principles

### Driver Interface Pattern
All storage drivers implement the `driver.Driver` interface defined in `internal/driver/`:
```go
type Driver interface {
    Config() Config
    GetAddition() Additional
    Init(ctx context.Context) error
    Drop(ctx context.Context) error
    List(ctx context.Context, dir model.Obj, args model.ListArgs) ([]model.Obj, error)
    // ... additional methods
}
```
New drivers only need to implement this interface; no changes to core code required.

### Operation Layer (`internal/op/`)
Business logic for file operations is abstracted in `internal/op/`:
- `fs.go` - Filesystem operations (list, get, mkdir, move, etc.)
- `path.go` - Path utilities
- Operations coordinate between drivers and database, handle caching

### Specialized Subsystems (`internal/`)
OpenList includes specialized subsystems for advanced functionality:
- **`archive/`** - Archive file handling with support for multiple formats (ZIP, RAR, 7z, ISO9660)
- **`fuse/`** - FUSE filesystem integration for mounting storage as local filesystems
- **`offline_download/`** - Offline download management and queuing

### HTTP Handler Pattern
Handlers in `server/handles/` receive Gin context, validate input, call operation layer, return JSON:
- **File naming**: Resource-based (`user.go`, `storage.go`) not functional
- **Error handling**: Use `common.ErrorResp()` for consistent error responses

```go
func ListUsers(c *gin.Context) {
    var req model.PageReq
    if err := c.ShouldBind(&req); err != nil {
        common.ErrorResp(c, err, 400)
        return
    }
    req.Validate()
    users, total, err := op.GetUsers(req.Page, req.PerPage)
    if err != nil {
        common.ErrorResp(c, err, 500, true)
        return
    }
    common.SuccessResp(c, common.PageResp{
        Content: users,
        Total:   total,
    })
}
```

### Configuration Pattern
Configuration managed in `internal/conf/`:
- Structs tagged for JSON/TOML/YAML unmarshaling
- Environment variable support with `env` prefix
- Sensible defaults defined in code

### Error Handling
- Operations return errors to be handled by HTTP layer
- Common error responses via `common.ErrorResp()` in `server/common/`
- Driver implementations wrap provider-specific errors

---
_Document patterns and principles, not exhaustive file listings. New code following these patterns shouldn't require updates to this document._
