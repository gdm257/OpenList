# Cross-Storage Driver Method Usage

OpenList supports transferring files between different storage backends (e.g., from S3 to OneDrive). This document clarifies which driver methods are invoked during cross-storage operations.

## Core Principle

Cross-storage transfers are implemented as **download-then-upload** operations using the Link/Put pattern. The operation layer handles the coordination; drivers only provide read and write capabilities.

## Driver Methods Called During Cross-Storage

### Always Called

**For Source Storage:**
- `List(ctx, dir, args)` - Lists directory contents when transferring folders
  - Called in `internal/fs/copy_move.go:188` for recursive directory transfers

- `Get(ctx, path)` - Retrieves file metadata
  - Called in `internal/fs/copy_move.go:181` to verify source exists

- `Link(ctx, file, args)` - Provides download link or stream
  - Called in `internal/fs/copy_move.go:248` to access file content
  - Returns a `model.Link` that can be read as a stream

**For Destination Storage:**
- `Put(ctx, dstDir, file, up)` - Uploads file content
  - Called in `internal/fs/copy_move.go:263` to write data to destination
  - Accepts a `FileStreamer` interface that wraps the source link

### Conditionally Called

- `MakeDir(ctx, parentDir, dirName)` - Creates destination directory during Put
  - Called in `internal/op/fs.go:642` before uploading files

- `Rename(ctx, srcObj, newName)` - Temporarily renames existing files during upload
  - Called in `internal/op/fs.go:634,700` when `Config.NoOverwriteUpload` is enabled
  - Used to backup existing files before overwrite, then restore on failure

- `Remove(ctx, obj)` - Deletes zero-sized files before upload
  - Called in `internal/op/fs.go:628` to clean up corrupted files

## Driver Methods NOT Called During Cross-Storage

### Same-Storage Only Operations

These methods are only invoked when source and destination are in the same storage:

- `Move(ctx, srcObj, dstDir)` - See `internal/fs/copy_move.go:113-127`
- `Copy(ctx, srcObj, dstDir)` - See `internal/fs/copy_move.go:113-127`

If the driver doesn't implement these, the operation layer falls back to the cross-storage Link/Put pattern even for same-storage transfers.

### Metadata and Lifecycle

Not called during active transfers (only during initialization):

- `Config()` - Configuration getter
- `GetAddition()` - Additional driver properties
- `Init(ctx)` - Driver initialization
- `Drop(ctx)` - Driver cleanup
- `GetDetails(ctx)` - Storage capacity information

### Archive Operations

Not used during cross-storage transfers (archive handling is separate):

- `GetArchiveMeta(ctx, obj, args)`
- `ListArchive(ctx, obj, args)`
- `Extract(ctx, obj, args)`
- `ArchiveDecompress(ctx, srcObj, dstDir, args)`

## Implementation Pattern

Cross-storage transfer flow (from `internal/fs/copy_move.go`):

```go
// 1. Get source download link
link, srcObj, err := op.Link(ctx, srcStorage, srcPath, model.LinkArgs{})

// 2. Convert link to seekable stream
ss, err := stream.NewSeekableStream(&stream.FileStream{...}, link)

// 3. Upload to destination (calls driver.Put)
return op.Put(ctx, dstStorage, dstDirPath, ss, progressCallback)
```

## Hash Handling in Put()

Hash retrieval is **optional** - only required for drivers supporting rapid upload.

### Retrieval Flow

For cross-storage transfers, `Put()` should:

```go
// 1. Get hash from stream (if source storage provided it)
hash := stream.GetHash().GetHash(utils.SHA1)

// 2. If hash exists and valid, use it directly (optional)
if len(hash) == utils.SHA1.Width {
    return d.uploadWithHash(hash)  // Rapid upload
}

// 3. Otherwise, calculate full hash from stream (optional)
// Skip this if driver doesn't support rapid upload
_, hash, err := streamPkg.CacheFullAndHash(stream, &up, utils.SHA1)
if err != nil {
    return err
}
return d.uploadWithHash(hash)
```

**Key point**: Hash retrieval is optional. Drivers can:
- Skip hash entirely and do normal upload
- Only calculate hash if rapid upload is enabled (`d.RapidUpload == true`)

### HashInfo Structure

```go
type HashInfo struct {
    h map[*HashType]string  // Supports MD5, SHA1, SHA256, etc.
}

// Access specific hash type (returns "" if not present)
hash := stream.GetHash().GetHash(utils.SHA1)
```

**Important**: Hash may be empty - always check `len(hash) > 0` before using.

### Hash Retention

Hash is preserved through `SeekableStream` inheritance:

```
srcObj (contains HashInfo from source storage)
  ↓ Link(ctx, file)
  ↓ NewSeekableStream({Obj: srcObj}, link)
  ↓ Put(seekableStream)
  ↓ stream.GetHash() → srcObj.GetHash() (preserved)
```

### Hash Calculation Helper

`CacheFullAndHash` reads entire stream and returns hash:

```go
func CacheFullAndHash(stream model.FileStreamer, up *model.UpdateProgress, hashType *utils.HashType) (model.File, string, error) {
    h := hashType.NewFunc(hashParams...)
    tmpF, err := stream.CacheFullAndWriter(up, h)  // Reads full stream
    if err != nil {
        return nil, "", err
    }
    return tmpF, hex.EncodeToString(h.Sum(nil)), nil  // Returns hash string
}
```

**Note**: Optional - only call if driver supports rapid upload and needs hash. Caches entire file to disk/memory. For large files, this is slow.

## Driver Implementation Requirements

For cross-storage transfers to work correctly, drivers must:

1. **Implement `Link()`**: Return a readable stream that can be consumed asynchronously
   - Link must support seeking for resumable uploads
   - Must implement `io.ReadCloser` interface

2. **Implement `Put()`**: Accept a `FileStreamer` and write to storage
   - **Optional**: Get hash via `stream.GetHash().GetHash()` or calculate with `CacheFullAndHash`
   - Hash is only needed if driver supports rapid upload
   - Handle stream reading and writing
   - Support progress reporting via `UpdateProgress` callback

3. **Optional optimizations**: Implement `Move()`/`Copy()` for efficient same-storage transfers
   - Returns `errs.NotImplement` to fall back to Link/Put pattern
   - Allows storage backends to optimize when both paths are within the same backend

## Rationale

The Link/Put pattern was chosen because:

1. **Uniformity**: All storage types support basic read/write operations
2. **Task Queue Integration**: Long-running transfers can be queued and resumed
3. **Error Handling**: Streaming approach allows partial recovery on network failures
4. **Simplicity**: Drivers only need to implement read and write, not complex transfer logic

---
_Document patterns, not every method. Follow existing driver implementations for context._
