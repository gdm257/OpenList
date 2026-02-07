# Logger Specification

This document defines the logging standards and conventions for the OpenList project.

## Overview

OpenList uses [logrus](https://github.com/sirupsen/logrus) as the primary logging library. logrus provides structured logging with multiple output formats, log levels, and hooks for extensibility.

## Import Convention

All files requiring logging should import logrus with the following alias:

```go
import (
    log "github.com/sirupsen/logrus"
)
```

## Logger Instances

### Primary Logger: `logrus.StandardLogger()`

Use the standard logrus logger for all application logging:

```go
log.Infof("operation completed: %s", identifier)
log.Errorf("failed to process request: %v", err)
```

### Secondary Logger: `utils.Log`

The `utils.Log` instance is a wrapper around `logrus.Logger` used for package-level utilities. Initialize it during application startup:

```go
// In internal/bootstrap/log.go
utils.Log = logrus.StandardLogger()
```

Use `utils.Log` when you need a logger instance that outlives the standard logger initialization.

## Log Levels

Use appropriate log levels based on the severity and context of the message:

| Level | Usage | Example |
|-------|-------|---------|
| **Debug** | Detailed information for troubleshooting | `log.Debugf("processing item: %s", key)` |
| **Info** | General operational events | `log.Infof("server started on port: %d", port)` |
| **Warn** | Unexpected but recoverable issues | `log.Warnf("cache miss for key: %s", key)` |
| **Error** | Errors that don't terminate the application | `log.Errorf("failed to connect: %v", err)` |
| **Fatal** | Critical errors followed by application exit | `log.Fatalf("database connection failed: %v", err)` |
| **Panic** | Errors followed by panic | `log.Panicf("unrecoverable state: %v", err)` |

### Level Guidelines

- **Debug**: Only output when debugging is enabled (`--debug` or `--dev` flags). Include detailed variable states and flow information.
- **Info**: Default level in production. Log all significant operational events (startup, shutdown, configuration changes).
- **Warn**: Log conditions that may lead to issues but don't prevent operation.
- **Error**: Log failures that affect a specific operation but not the entire application.
- **Fatal/Panic**: Use sparingly. These terminate the application. Reserve for truly unrecoverable states.

## Message Formatting

### Structured Messages

Use structured, descriptive messages with relevant context:

```go
// Good
log.Errorf("failed to get storage details: storage_id=%s error=%v", storageID, err)

// Avoid
log.Errorf("failed")
log.Errorf("error getting details: %+v", err)  // Too generic
```

### Format String Conventions

- Use `%s` for strings
- Use `%d` for integers
- Use `%v` for general values (objects, errors)
- Use `%+v` for detailed struct inspection
- Use `%f` for floating-point numbers
- Use `%t` for booleans

### Key-Value Pairs

For structured logging with key-value pairs:

```go
log.WithField("storage_id", storageID).
    WithField("operation", "list").
    Error("storage operation failed")
```

Or using `WithFields` for multiple fields:

```go
log.WithFields(log.Fields{
    "storage_id": storageID,
    "path":       path,
    "action":     "upload",
}).Error("storage operation failed")
```

## Error Logging

### Error Context

Always include sufficient context when logging errors:

```go
// Good - includes operation, resource, and error
log.Errorf("failed to upload file: storage=%s path=%s error=%v", storageID, path, err)

// Good - using structured fields
log.WithField("storage_id", storageID).
    WithField("file_path", path).
    Error("upload failed")
```

### Error Wrapping

Use error wrapping for stack traces when appropriate:

```go
log.Errorf("failed to process request: %+v", err)

// For wrapped errors with context
log.Errorf("database connection failed: %w", err)  // If using Go 1.20+ error wrapping
```

### Avoid Redundant Error Logging

Don't log errors at multiple levels in the same call stack:

```go
// Avoid - double logging
func A() error {
    if err := B(); err != nil {
        log.Error(err)  // Log here
        return err      // Then return
    }
    return nil
}

func B() error {
    if err := C(); err != nil {
        log.Error(err)  // Don't log here too
        return err
    }
    return nil
}
```

Return errors up the stack and log at the appropriate level (typically at the handler or boundary).

## Performance Considerations

### Avoid Expensive Operations in Debug Logs

```go
// Avoid - %+v on large structures in debug
log.Debugf("state: %+v", hugeStruct)

// Better - log specific fields
log.Debugf("items_count=%d memory=%d", len(items), memory)

// Better - use structured fields
log.WithField("items_count", len(items)).
    WithField("memory", memory).
    Debug("state snapshot")
```

### Conditional Debug Logging

Use `IsDebugEnabled()` check for expensive debug operations:

```go
if log.IsLevelEnabled(log.DebugLevel) {
    result := expensiveDebugOperation()
    log.Debugf("result: %v", result)
}
```

## Configuration

Logging configuration is managed in `internal/bootstrap/log.go`:

```go
// Log level based on flags
if flags.Debug || flags.Dev {
    l.SetLevel(logrus.DebugLevel)
    l.SetReportCaller(true)
} else {
    l.SetLevel(logrus.InfoLevel)
    l.SetReportCaller(false)
}

// File rotation with lumberjack
if logConfig.Enable {
    w = &lumberjack.Logger{
        Filename:   logConfig.Name,
        MaxSize:    logConfig.MaxSize,
        MaxBackups: logConfig.MaxBackups,
        MaxAge:     logConfig.MaxAge,
        Compress:   logConfig.Compress,
    }
}
```

### Standard Configuration Values

- **Format**: Text with timestamp (`2006-01-02 15:04:05`)
- **Default Level**: `Info` (production), `Debug` (development)
- **Output**: stdout by default, file with rotation when configured
- **Colors**: Enabled in development, configurable in production

## Request Logging

For HTTP request logging, use the middleware pattern:

```go
func RequestLogger() gin.HandlerFunc {
    return gin.LoggerWithConfig(gin.LoggerConfig{
        Output: log.StandardLogger().Out,
        Skip:   skipFunc,
    })
}
```

Include request IDs for tracing:

```go
log.WithField("request_id", requestID).
    Infof("request completed: method=%s path=%s status=%d", 
        method, path, status)
```

## Driver Logging

Storage drivers should use consistent logging patterns:

```go
// Driver initialization
log.Infof("initializing driver: type=%s", driverType)

// API operations
log.Debugf("API request: endpoint=%s params=%v", endpoint, params)
log.Warnf("rate limit warning: api=%s remaining=%d", api, remaining)

// Errors with context
log.Errorf("API error: endpoint=%s code=%d error=%v", 
    endpoint, code, err)
```

## Log Message Patterns

### Success Messages

```go
log.Infof("storage loaded: id=%s mount_path=%s", 
    storage.ID, storage.MountPath)
log.Infof("task completed: name=%s duration=%v", 
    task.Name, duration)
```

### Progress Messages

```go
log.Debugf("processing item: index=%d total=%d", 
    index, total)
log.Infof("progress: percent=%d%% items=%d/%d", 
    percent, processed, total)
```

### Warning Messages

```go
log.Warnf("cache miss: key=%s", key)
log.Warnf("rate limit approaching: api=%s qps=%d", 
    api, qps)
log.Warnf("configuration deprecated: key=%s use=%s", 
    key, replacement)
```

### Error Messages

```go
log.Errorf("operation failed: type=%s id=%s error=%v", 
    operationType, id, err)
log.Errorf("validation error: field=%s value=%s reason=%s", 
    field, value, reason)
log.Errorf("storage error: id=%s error=%v", storageID, err)
```

## Anti-Patterns to Avoid

### 1. Sensitive Data in Logs

```go
// Never log passwords, tokens, or API keys
log.Infof("user login: password=%s", password)  // BAD

// Instead, log success/failure without sensitive data
log.Infof("login attempt: username=%s success=%t", 
    username, success)
```

### 2. Inconsistent Error Messages

```go
// Bad - inconsistent error messages
log.Error("error")  // Too vague
log.Error("failed") // What failed?

// Good - consistent format
log.Errorf("operation failed: reason=%s", reason)
```

### 3. Logging Without Context

```go
// Bad
log.Error(err)

// Good
log.Errorf("failed to process request: path=%s error=%v", 
    path, err)
```

### 4. Using Printf for Errors

```go
// Bad - loses log level semantics
fmt.Printf("error: %v\n", err)

// Good - uses proper log level
log.Error(err)
```

## Configuration File Integration

Log settings in `config.json`:

```json
{
  "log": {
    "enable": true,
    "name": "logs/openlist.log",
    "max_size": 50,
    "max_backups": 3,
    "max_age": 30,
    "compress": false,
    "filter": {
      "filters": [
        {"cidr": "192.168.1.0/24"},
        {"path": "/health"},
        {"method": "GET"}
      ]
    }
  }
}
```

## Summary

- Use `logrus` with alias `log`
- Choose appropriate log levels (Debug/Info/Warn/Error/Fatal/Panic)
- Include context in log messages with key-value pairs
- Avoid logging sensitive data
- Don't double-log errors
- Use structured fields for complex logging needs
- Configure log rotation for production deployments
