# Understanding Container Image and Container Filesystems

## Introduction

Container technology relies on a sophisticated approach to filesystem management that separates immutable application code from runtime changes. This document explains the distinct concepts of container image filesystems and container runtime filesystems, their relationship, and how they enable the core benefits of containerization.

## Container Image Filesystem

The container image filesystem is the foundation that defines what files a container will have access to when it starts.

### Characteristics of Image Filesystems

1. **Immutable and Read-Only**
   - Once built, an image filesystem cannot be modified
   - Any changes require building a new image version
   - Ensures consistency and reproducibility

2. **Layered Architecture**
   - Composed of multiple read-only layers
   - Each layer represents incremental filesystem changes
   - Layers are stacked to form a complete filesystem view
   - Implemented using union filesystem technology (overlay2, aufs, etc.)

3. **Content-Addressable Storage**
   - Each layer is identified by a cryptographic hash of its contents
   - Ensures integrity and enables deduplication
   - Simplifies caching and distribution

4. **Efficient Storage**
   - Common layers are stored only once on a host
   - Multiple images can share base layers
   - Reduces overall disk usage

5. **Distributable Units**
   - Images can be pushed to and pulled from registries
   - Only missing layers need to be transferred
   - Enables efficient image distribution

### Contents of the Image Filesystem

A typical image filesystem contains:
- Base operating system files
- Runtime dependencies
- Application binaries
- Default configuration
- Application code
- Static content

```
Image Layers Example:
┌─────────────────────────┐
│   Application Code      │  Layer 4: 5MB
├─────────────────────────┤
│   Dependencies          │  Layer 3: 50MB
├─────────────────────────┤
│   Runtime Environment   │  Layer 2: 70MB
├─────────────────────────┤
│   Base OS               │  Layer 1: 80MB
└─────────────────────────┘
Total Image Size: 205MB
```

## Container Runtime Filesystem

When a container starts, it gets its own filesystem view based on the image but with additional characteristics.

### Characteristics of Container Filesystems

1. **Writable Layer**
   - A thin, container-specific writable layer added on top of image layers
   - Captures all runtime modifications
   - Implemented using copy-on-write technology
   - Isolated from other containers running from the same image

2. **Union Mounting**
   - Merges read-only image layers with the writable layer
   - Presents a unified view of the filesystem to processes in the container
   - Transparently handles the complexity of multiple layers

3. **Ephemeral Nature**
   - By default, all data in the writable layer exists only for the container's lifetime
   - When the container is removed, its writable layer is discarded
   - Reinforces the stateless design principle

4. **Runtime-Specific Files**
   - Process IDs and lock files
   - Log data
   - Temporary files
   - Runtime configurations
   - Generated content

5. **Isolation**
   - Changes in one container's filesystem are not visible to other containers
   - Provides process and filesystem isolation without VM overhead

### Container Filesystem Layers

```
Container Filesystem:
┌─────────────────────────┐
│   Writable Layer        │  Container-specific, ephemeral
├─────────────────────────┤
│                         │
│                         │
│   Image Layers          │  Read-only, shared across containers
│   (multiple layers)     │
│                         │
│                         │
└─────────────────────────┘
```

## Relationship Between Image and Container Filesystems

The key relationship can be summarized as:
- Image filesystem: Defines the initial state (read-only)
- Container filesystem: Adds a writable layer for runtime changes

### File Operations in a Container

When a container process:

1. **Reads a file**:
   - The file is served from the topmost layer where it exists
   - If not modified, it comes from the image layers

2. **Modifies an existing file**:
   - The file is copied from the image layer to the writable layer (copy-on-write)
   - Modifications are made to the copy in the writable layer
   - Subsequent reads see the modified version

3. **Creates a new file**:
   - The file is written directly to the writable layer
   - Original image layers remain unchanged

4. **Deletes a file**:
   - A "whiteout" file is created in the writable layer
   - This masks the file from lower layers without actually removing it from the image

## Practical Implications

### Performance Considerations

1. **Copy-on-Write Overhead**
   - Modifying large files has overhead (the entire file must be copied first)
   - Databases and high I/O applications may experience performance impacts
   - Volume mounts often recommended for write-heavy workloads

2. **Layer Depth Impact**
   - Too many layers can impact filesystem performance
   - Best practice is to minimize unnecessary layers

### Storage Efficiency

1. **Space Usage**
   - Container writable layers consume additional space
   - Multiple containers from the same image share the image layers
   - `docker system df` shows space usage by images, containers, and volumes

2. **Cleanup Considerations**
   - Removing containers reclaims writable layer space
   - Removing unused images reclaims layer space
   - Regular cleanup may be necessary in production environments

### Exploring Filesystem Contents

1. **Viewing Image Contents**
   ```bash
   # Create a container without starting it
   CONT_ID=$(docker create image:tag)
   
   # Export the filesystem
   docker export ${CONT_ID} -o image.tar.gz
   
   # Clean up
   docker rm ${CONT_ID}
   
   # Extract and explore
   mkdir image-rootfs
   tar -xf image.tar.gz -C image-rootfs
   ```

2. **Viewing Container Filesystem**
   ```bash
   # For a running container
   docker exec -it container_name ls -la /some/path
   
   # Or get an interactive shell
   docker exec -it container_name sh
   ```

3. **Inspecting Layers**
   ```bash
   # View image history and layers
   docker history image:tag
   ```

## Conclusion

The distinction between container image filesystems and container runtime filesystems is fundamental to how containers work. This architecture provides the ideal balance between immutability, efficiency, and flexibility that makes containers so powerful for modern application deployment.

By understanding these concepts, developers and operators can better design, optimize, and troubleshoot containerized applications while taking full advantage of the containerization model's benefits.
