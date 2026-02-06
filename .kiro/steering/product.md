# Product Overview

**OpenList** is a file management gateway that aggregates multiple storage providers behind a unified interface. It serves as a bridge between diverse cloud storage services (object storage, personal clouds, network drives) and end users, providing a consistent API and web interface.

## Core Capabilities

1. **Multi-Storage Aggregation**: Normalize access to 80+ storage providers (S3, WebDAV, Google Drive, OneDrive, and many more) through a single interface.

2. **Unified File Operations**: Provide consistent CRUD operations (list, upload, download, rename, move, copy, delete) across all supported storage types.

3. **Driver Architecture**: Pluggable driver system that allows adding new storage backends by implementing a standard Go interface.

4. **WebDAV Gateway**: Serve any configured storage as a WebDAV-compatible endpoint.

## Target Use Cases

- **Personal Media Servers**: Organize and serve media from multiple cloud accounts
- **Data Migration**: Transfer files between incompatible storage systems
- **Backup Aggregation**: Centralize access to distributed backup locations
- **Team File Gateway**: Provide consistent API access to heterogeneous storage infrastructure

## Value Proposition

OpenList is the **community-maintained fork of AList**, specifically created to preserve open-source values against trust-based attacks. It prioritizes:

- **Code Transparency**: Complete AGPL-3.0 licensed source code, no proprietary components
- **Driver Abstraction**: New storage backends require only implementing the `Driver` interface
- **Deployment Flexibility**: Single binary with embedded frontend, Docker support, or development mode

---
_This document focuses on architectural patterns and design philosophy, not exhaustive feature lists_
