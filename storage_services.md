# Storage Services Module

## Overview

The **Storage Services** module provides a unified interface for file storage and management using Cloudflare R2, an S3-compatible object storage service. It handles file uploads, downloads, deletions, and URL generation with support for multiple file types and organizational structures.

## Purpose and Core Functionality

The StorageService class serves as the central component for:
- **File Uploads**: Upload files from local paths or byte content
- **File Management**: Delete, list, and check existence of files
- **URL Generation**: Generate public URLs for accessible files
- **File Organization**: Support for folder-based organization
- **Type Detection**: Automatic detection and categorization of file types
- **Metadata Support**: Attach custom metadata to stored files
- **Configuration Management**: Graceful handling of missing configuration

## Architecture

### Component Structure

```mermaid
graph TB
    subgraph "Storage Services Module"
        SS[StorageService]
        
        SS --> Config[Settings Configuration]
        SS --> Logger[Logging Service]
        SS --> S3Client[S3 Client boto3]
        
        subgraph "Supported File Types"
            ImageTypes[Image Types<br/>jpg, jpeg, png, gif, webp, bmp, svg]
            VideoTypes[Video Types<br/>mp4, avi, mov, wmv, flv, mkv, webm]
            DocTypes[Document Types<br/>pdf, doc, docx, txt, html]
            WebTypes[Web Types<br/>html, htm, css, js, json, xml, txt, md]
        end
        
        SS --> ImageTypes
        SS --> VideoTypes
        SS --> DocTypes
        SS --> WebTypes
    end
    
    subgraph "External Dependencies"
        R2[Cloudflare R2 Storage]
        Settings[App Settings]
    end
    
    S3Client --> R2
    Config --> Settings
```

### Data Flow for File Upload Operations

```mermaid
sequenceDiagram
    participant Client as Client Application
    participant SS as StorageService
    participant S3 as S3 Client (boto3)
    participant R2 as Cloudflare R2
    participant Config as Configuration
    
    Client->>SS: upload_file(file_content, filename)
    SS->>Config: get_settings()
    Config-->>SS: settings object
    
    alt R2 not configured
        SS-->>Client: Return None + warning
    else R2 configured
        SS->>SS: _generate_unique_key(filename)
        SS->>SS: _get_file_type(filename)
        SS->>SS: Detect content_type
        
        SS->>S3: put_object(Bucket, Key, Body, ExtraArgs)
        S3->>R2: Upload file
        R2-->>S3: Upload confirmation
        S3-->>SS: Success response
        
        SS->>SS: get_public_url(key)
        SS-->>Client: Return file info dict
    end
```

### Component Interaction Diagram

```mermaid
graph LR
    subgraph "StorageService Methods"
        upload_path[upload_file_from_path]
        upload_bytes[upload_file bytes]
        delete[delete_file]
        list[list_files]
        exists[file_exists]
        get_url[get_file_url]
    end
    
    subgraph "Internal Helpers"
        gen_key[_generate_unique_key]
        get_type[_get_file_type]
        get_ext[_get_file_extension]
        pub_url[get_public_url]
    end
    
    subgraph "S3 Client Operations"
        s3_upload[upload_file]
        s3_put[put_object]
        s3_delete[delete_object]
        s3_list[list_objects_v2]
        s3_head[head_object]
    end
    
    upload_path --> gen_key
    upload_path --> get_type
    upload_path --> s3_upload
    
    upload_bytes --> gen_key
    upload_bytes --> get_type
    upload_bytes --> s3_put
    
    delete --> s3_delete
    list --> s3_list
    exists --> s3_head
    get_url --> pub_url
    
    gen_key --> get_ext
```

### Process Flow for File Operations

```mermaid
flowchart TD
    Start([Start Operation]) --> ConfigCheck{Is R2<br/>configured?}
    
    ConfigCheck -->|No| Disabled[Set _enabled = False<br/>Log warning]
    Disabled --> ReturnNone[Return None/False]
    
    ConfigCheck -->|Yes| InitS3[Initialize S3 Client]
    
    InitS3 --> Operation{Operation Type?}
    
    Operation -->|Upload| UploadPath{From path<br/>or bytes?}
    UploadPath -->|Path| ValidatePath{File exists?}
    ValidatePath -->|No| LogError[Log error]
    LogError --> ReturnNone
    ValidatePath -->|Yes| GenKey[Generate unique key]
    
    UploadPath -->|Bytes| GenKey
    
    GenKey --> DetectType[Detect content type]
    DetectType --> PrepareArgs[Prepare ExtraArgs<br/>ACL, Metadata]
    PrepareArgs --> S3Upload[Call S3 API]
    S3Upload --> Success{Success?}
    Success -->|Yes| ReturnInfo[Return file info]
    Success -->|No| LogS3Error[Log S3 error]
    LogS3Error --> ReturnNone
    
    Operation -->|Delete| S3Delete[Call S3 delete_object]
    S3Delete --> DeleteSuccess{Success?}
    DeleteSuccess -->|Yes| ReturnTrue[Return True]
    DeleteSuccess -->|No| ReturnFalse[Return False]
    
    Operation -->|List| S3List[Call S3 list_objects_v2]
    S3List --> ListSuccess{Success?}
    ListSuccess -->|Yes| ReturnFiles[Return files list]
    ListSuccess -->|No| ReturnEmpty[Return empty list]
    
    Operation -->|Exists| S3Head[Call S3 head_object]
    S3Head --> HeadSuccess{Success?}
    HeadSuccess -->|Yes| ReturnExistsTrue[Return True]
    HeadSuccess -->|No| ReturnExistsFalse[Return False]
```

## Key Components

### StorageService Class

The `StorageService` class is the main component that provides all storage operations. It's designed as a singleton with the following key attributes:

- **Configuration-based initialization**: Only initializes S3 client if all required settings are present
- **Graceful degradation**: Continues to function (with warnings) if R2 is not configured
- **File type support**: Categorizes files into images, videos, documents, and web files
- **Unique key generation**: Creates organized file paths with timestamps and UUIDs

### Configuration Requirements

The service requires the following configuration settings:
- `r2_account_id`: Cloudflare R2 account identifier
- `r2_access_key_id`: Access key for R2 API
- `r2_secret_access_key`: Secret key for R2 API
- `r2_bucket_name`: Target bucket name
- `r2_public_url_base`: Base URL for public file access

### File Organization

Files are organized using:
- **Folder prefixes**: Optional folder parameters for categorization (e.g., 'images', 'videos')
- **Unique filenames**: Generated as `{timestamp}_{uuid}{extension}`
- **Public URLs**: Constructed from `public_url_base` and file key

## API Reference

### Core Methods

#### `upload_file_from_path(file_path, folder=None, custom_key=None, metadata=None, make_public=True)`
Uploads a file from a local filesystem path to R2 storage.

**Parameters:**
- `file_path`: Path to the file to upload
- `folder`: Optional folder for organization
- `custom_key`: Custom S3 key (auto-generated if None)
- `metadata`: Additional metadata to attach
- `make_public`: Whether to make the file publicly accessible

**Returns:** Dictionary with file_id, url, key, size, file_type, content_type, or None on failure

#### `upload_file(file_content, filename, content_type=None, folder=None, metadata=None, make_public=True)`
Uploads file content from bytes to R2 storage.

**Parameters:**
- `file_content`: File content as bytes
- `filename`: Original filename (for extension and key generation)
- `content_type`: MIME type (auto-detected if None)
- `folder`: Optional folder for organization
- `metadata`: Additional metadata to attach
- `make_public`: Whether to make the file publicly accessible

**Returns:** Dictionary with file_id, url, key, size, file_type, content_type, or None on failure

#### `delete_file(key)`
Deletes a file from R2 storage.

**Parameters:**
- `key`: S3 key of the file to delete

**Returns:** True if successful, False otherwise

#### `list_files(prefix=None, max_keys=1000)`
Lists files in the bucket.

**Parameters:**
- `prefix`: Filter by prefix/folder
- `max_keys`: Maximum number of files to return

**Returns:** List of file information dictionaries

#### `file_exists(key)`
Checks if a file exists in R2 storage.

**Parameters:**
- `key`: S3 key to check

**Returns:** True if file exists, False otherwise

### Helper Methods

#### `_generate_unique_key(original_filename, folder=None)`
Generates a unique S3 key for a file.

**Format:** `{folder}/{timestamp}_{uuid}{extension}`

#### `_get_file_type(filename)`
Determines the file category based on extension.

**Categories:** image, video, document, web, other

#### `get_public_url(key)`
Generates a public URL for a given S3 key.

## Dependencies

### External Dependencies
- **boto3**: AWS SDK for Python, used for S3-compatible API calls
- **botocore**: Low-level AWS service access

### Internal Dependencies
- **app.config**: Application settings management
- **logging**: Python standard logging module

### Configuration Dependencies
The service depends on the following configuration values from the application settings:
- `r2_account_id`
- `r2_access_key_id`
- `r2_secret_access_key`
- `r2_bucket_name`
- `r2_public_url_base`

## Integration Points

### Related Modules

The Storage Services module is used by various other modules in the system:

- **[Lesson Management](lesson_management.md)**: Stores lesson-related files, images, and documents
- **[Material Management](material_management.md)**: Manages educational material files
- **[Export Services](export_services.md)**: Stores generated export files (PDF, DOCX)
- **[Animation Generation](animation_generation.md)**: Stores generated animation files and assets

### Usage Patterns

```python
# Singleton instance
from app.services.storage import storage_service

# Upload a file
result = await storage_service.upload_file_from_path(
    file_path="/path/to/file.pdf",
    folder="documents"
)

if result:
    print(f"File uploaded: {result['url']}")
else:
    print("Upload failed")

# Check if file exists
exists = await storage_service.file_exists("documents/file.pdf")

# Delete a file
success = await storage_service.delete_file("documents/file.pdf")
```

## Error Handling

The service implements comprehensive error handling:

1. **Configuration Errors**: Logs warnings and disables functionality if R2 is not configured
2. **File Not Found**: Validates file existence before upload attempts
3. **S3 API Errors**: Catches and logs `ClientError` exceptions from boto3
4. **General Exceptions**: Catches unexpected errors and logs them appropriately

All error conditions return `None` or `False` to allow graceful failure handling by calling code.

## Security Considerations

- **Access Control**: Uses ACL settings to control public access
- **Credential Management**: Credentials stored in application settings (not hardcoded)
- **Unique Keys**: Prevents filename collisions using UUIDs and timestamps
- **Content Type Validation**: Properly sets Content-Type headers for security

## Performance Characteristics

- **Async Support**: All methods are async-compatible
- **Connection Reuse**: S3 client is initialized once and reused
- **Batch Operations**: Supports listing up to 1000 files per request
- **File Size**: No explicit size limits (subject to R2 constraints)

## Future Enhancements

Potential improvements for the storage service:

1. **CDN Integration**: Direct Cloudflare CDN integration for optimized delivery
2. **Signed URLs**: Support for temporary signed URLs for private files
3. **Multipart Uploads**: Support for large file multipart uploads
4. **File Versioning**: Version control for uploaded files
5. **Cache Control**: Automatic cache header management
6. **Metrics**: Upload/download tracking and analytics