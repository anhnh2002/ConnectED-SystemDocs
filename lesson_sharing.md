# Lesson Sharing Module

## Overview

The Lesson Sharing module provides secure, token-based sharing capabilities for lesson plans, enabling teachers to share their work with colleagues, administrators, or the public. It supports both view-only and edit access modes, allowing for collaborative lesson planning while maintaining content control.

## Purpose and Core Functionality

The module enables:
- **Secure Sharing**: Generate unique, cryptographically secure tokens for lesson access
- **Dual Access Modes**: Separate view-only and edit tokens for different permission levels
- **Public Discovery**: Share lessons on a public landing page for broader access
- **Collaborative Editing**: Allow authorized users to modify shared lesson plans
- **Access Control**: Revoke sharing permissions at any time
- **Integration**: Seamlessly works with the complete lesson lifecycle from processing to presentation

## Architecture

### Component Structure

```mermaid
graph TB
    subgraph "Lesson Sharing Module"
        LS[LessonService<br/>Sharing Methods]
        SLR[ShareLinkResponse]
        SLR2[SharedLessonResponse]
    end
    
    subgraph "Dependencies"
        LR[LessonRepository]
        LSDB[LessonInDB<br/>MongoDB Document]
        LStatus[LessonStatus]
    end
    
    subgraph "External Systems"
        Mongo[(MongoDB)]
        API[FastAPI Endpoints]
        Frontend[Frontend<br/>Share UI]
    end
    
    LS --> LR
    LS --> SLR
    LS --> SLR2
    LS --> LStatus
    
    LR --> LSDB
    LSDB --> Mongo
    
    API --> LS
    Frontend --> API
    
    style LS fill:#e1f5ff
    style SLR fill:#e1f5ff
    style SLR2 fill:#e1f5ff
```

### Token-Based Security Model

```mermaid
graph LR
    subgraph "Teacher"
        T[Teacher User]
    end
    
    subgraph "Lesson Sharing"
        L[Lesson Document]
        ST[Share Token<br/>View-Only]
        ET[Edit Token<br/>Editable]
        PS[Public Status]
    end
    
    subgraph "Access Paths"
        V[View Access<br/>/shared/share_token]
        E[Edit Access<br/>/shared/edit_token]
        P[Public List<br/>/shared]
    end
    
    subgraph "Recipients"
        C[Colleagues]
        A[Administrators]
        Pub[Public Users]
    end
    
    T -->|creates| L
    L -->|contains| ST
    L -->|contains| ET
    L -->|has| PS
    
    ST --> V
    ET --> E
    PS -->|enables| P
    
    V --> C
    V --> A
    E --> C
    P --> Pub
```

## Core Components

### LessonService Sharing Methods

The sharing functionality is integrated into the main `LessonService` class ([lesson_management](lesson_management.md) module). Key methods include:

#### Share Link Management

- **`create_share_link()`**: Generates new share and edit tokens for a lesson
- **`get_share_link()`**: Retrieves existing share links
- **`revoke_share_link()`**: Removes sharing permissions
- **`toggle_public()`**: Controls public listing visibility

#### Shared Lesson Access

- **`get_shared_lesson()`**: Public access to shared lessons (no authentication required)
- **`update_shared_lesson()`**: Allows editing via edit token
- **`get_shared_lessons_list()`**: Retrieves public lessons for landing page display

### Data Models

#### ShareLinkResponse

```python
class ShareLinkResponse(BaseModel):
    lesson_id: str
    share_token: str           # View-only access token
    share_url: str             # Full URL for view access
    edit_token: Optional[str]  # Edit access token
    edit_url: Optional[str]    # Full URL for edit access
    created_at: Optional[datetime]
```

**Purpose**: Encapsulates all sharing information for a lesson, providing both view and edit access credentials.

#### SharedLessonResponse

```python
class SharedLessonResponse(BaseModel):
    id: str
    metadata: LessonMetadata
    lesson_plan: Optional[LessonPlan]
    presentation: Optional[LessonPresentation]
    playground: Optional[SharedPlaygroundInfo]
    teacher_name: Optional[str]
    can_edit: bool            # True if accessed via edit_token
    created_at: datetime
```

**Purpose**: Public-facing lesson representation that includes teacher information and playground data while respecting access permissions.

## Process Flows

### Creating a Share Link

```mermaid
sequenceDiagram
    participant Teacher
    participant API
    participant LessonService
    participant LessonRepository
    participant MongoDB
    
    Teacher->>API: POST /lessons/{id}/share
    API->>LessonService: create_share_link(lesson_id, teacher_id, base_url)
    LessonService->>LessonRepository: find_by_id_and_teacher()
    LessonRepository->>MongoDB: Query lesson
    MongoDB-->>LessonRepository: Lesson document
    LessonRepository-->>LessonService: Lesson data
    
    alt Lesson not found or not shareable
        LessonService-->>API: Return None
        API-->>Teacher: 404 Not Found
    else Lesson valid
        LessonService->>LessonService: Generate tokens (secrets.token_urlsafe)
        LessonService->>LessonRepository: set_share_tokens()
        LessonRepository->>MongoDB: Update with tokens
        MongoDB-->>LessonRepository: Acknowledge
        LessonRepository-->>LessonService: Success
        LessonService->>LessonService: Build URLs
        LessonService-->>API: ShareLinkResponse
        API-->>Teacher: 200 OK with share data
    end
```

### Accessing a Shared Lesson (View Mode)

```mermaid
sequenceDiagram
    participant User
    participant API
    participant LessonService
    participant LessonRepository
    participant MongoDB
    
    User->>API: GET /shared/{share_token}
    API->>LessonService: get_shared_lesson(share_token)
    LessonService->>LessonRepository: find_by_share_token()
    LessonRepository->>MongoDB: Query by share_token
    MongoDB-->>LessonRepository: Lesson document
    LessonRepository-->>LessonService: Lesson data
    
    alt Token not found or invalid
        LessonService-->>API: Return None
        API-->>User: 404 Not Found
    else Token valid
        LessonService->>MongoDB: Get teacher name
        MongoDB-->>LessonService: Teacher data
        LessonService->>MongoDB: Get playground info
        MongoDB-->>LessonService: Playground data
        LessonService->>LessonService: Build SharedLessonResponse (can_edit=False)
        LessonService-->>API: SharedLessonResponse
        API-->>User: 200 OK with lesson data
    end
```

### Editing a Shared Lesson

```mermaid
sequenceDiagram
    participant Collaborator
    participant API
    participant LessonService
    participant LessonRepository
    participant MongoDB
    
    Collaborator->>API: PUT /shared/{edit_token}
    API->>LessonService: update_shared_lesson(edit_token, updates)
    LessonService->>LessonRepository: find_by_edit_token()
    LessonRepository->>MongoDB: Query by edit_token
    MongoDB-->>LessonRepository: Lesson document
    LessonRepository-->>LessonService: Lesson data
    
    alt Edit token not found
        LessonService-->>API: Return None
        API-->>Collaborator: 404 Not Found
    else Edit token valid
        LessonService->>LessonRepository: update_one(lesson_plan=updates)
        LessonRepository->>MongoDB: Update lesson
        MongoDB-->>LessonRepository: Acknowledge
        LessonRepository-->>LessonService: Success
        LessonService->>MongoDB: Refresh teacher/playground data
        MongoDB-->>LessonService: Updated data
        LessonService->>LessonService: Build SharedLessonResponse (can_edit=True)
        LessonService-->>API: SharedLessonResponse
        API-->>Collaborator: 200 OK with updated lesson
    end
```

### Public Lesson Listing

```mermaid
sequenceDiagram
    participant PublicUser
    participant API
    participant LessonService
    participant LessonRepository
    participant MongoDB
    
    PublicUser->>API: GET /shared-lessons
    API->>LessonService: get_shared_lessons_list(limit)
    LessonService->>LessonRepository: find_shared_lessons()
    LessonRepository->>MongoDB: Query public shared lessons
    MongoDB-->>LessonRepository: List of lessons
    LessonRepository-->>LessonService: Lessons data
    
    loop For each lesson
        LessonService->>MongoDB: Get teacher name
        MongoDB-->>LessonService: Teacher data
        LessonService->>LessonService: Extract first image URL
    end
    
    LessonService->>LessonService: Build response list
    LessonService-->>API: List of shared lesson summaries
    API-->>PublicUser: 200 OK with public lessons
```

## Data Storage

### Lesson Document Sharing Fields

The sharing functionality extends the `LessonInDB` model with additional fields:

```python
class LessonInDB(MongoBaseModel):
    # ... existing fields ...
    
    # Sharing fields
    share_token: Optional[str] = None      # URL-safe token for view access
    edit_token: Optional[str] = None       # URL-safe token for edit access  
    share_created_at: Optional[datetime] = None
    is_public: bool = False                # Show on public landing page
    
    # Timestamps
    created_at: datetime
    updated_at: datetime
```

### Token Generation

Tokens are generated using Python's `secrets.token_urlsafe(12)`, producing:
- 16-character URL-safe strings
- Cryptographically secure random generation
- No personal data or predictable patterns
- Example: `dGhpcyBpcyBhIHRva2Vu` (base64url encoded)

## Security and Access Control

### Shareable Statuses

Lessons can only be shared when in specific statuses:

```python
shareable_statuses = [
    LessonStatus.REVIEW.value,                    # Lesson plan ready
    LessonStatus.AWAITING_PRESENTATION_REVIEW.value,  # Presentation generated
    LessonStatus.CONFIRMED.value,                 # Teacher confirmed
    LessonStatus.EXPORTED.value,                  # Fully exported
]
```

This ensures only completed or reviewable lessons are shared.

### Access Validation Flow

```mermaid
graph TD
    Request[Incoming Request] --> Token{{Token Type?}}
    
    Token -->|share_token| LookupShare[Lookup by share_token]
    Token -->|edit_token| LookupEdit[Lookup by edit_token]
    Token -->|none| PublicList[Return public list]
    
    LookupShare --> ValidateShare{{Valid & Shareable?}}
    LookupEdit --> ValidateEdit{{Valid & Shareable?}}
    
    ValidateShare -->|Yes| ReturnView[Return with can_edit=False]
    ValidateShare -->|No| Return404[Return 404]
    
    ValidateEdit -->|Yes| ReturnEdit[Return with can_edit=True]
    ValidateEdit -->|No| Return404
    
    PublicList --> FilterPublic[Filter is_public=True]
    FilterPublic --> ReturnList[Return lesson summaries]
```

## Integration with Lesson Lifecycle

### Sharing in the Lesson Pipeline

```mermaid
stateDiagram-v2
    [*] --> DRAFT: Create lesson
    DRAFT --> UPLOADED: Upload images
    UPLOADED --> PROCESSING: Start extraction
    PROCESSING --> AWAITING_CONTENT_REVIEW: Extraction complete
    AWAITING_CONTENT_REVIEW --> AWAITING_CONCEPTS_REVIEW: Approve content
    AWAITING_CONCEPTS_REVIEW --> AWAITING_OBJECTIVES_REVIEW: Approve concepts
    AWAITING_OBJECTIVES_REVIEW --> AWAITING_OUTLINE_REVIEW: Approve objectives
    AWAITING_OUTLINE_REVIEW --> REVIEW: Approve outline
    REVIEW --> AWAITING_PRESENTATION_REVIEW: Generate presentation
    AWAITING_PRESENTATION_REVIEW --> REVIEW: Approve presentation
    REVIEW --> CONFIRMED: Confirm plan
    CONFIRMED --> EXPORTED: Export materials
    
    note right of REVIEW: Can share from here
    note right of AWAITING_PRESENTATION_REVIEW: Can share from here
    note right of CONFIRMED: Can share from here
    note right of EXPORTED: Can share from here
    
    REVIEW --> SHAREABLE: Create share link
    AWAITING_PRESENTATION_REVIEW --> SHAREABLE: Create share link
    CONFIRMED --> SHAREABLE: Create share link
    EXPORTED --> SHAREABLE: Create share link
    
    SHAREABLE --> [*]: Shared lesson
```

### Collaborative Editing Workflow

```mermaid
graph LR
    subgraph "Original Teacher"
        T1[Create Lesson]
        T1 --> T2[Process to REVIEW+]
        T2 --> T3[Generate Share Link]
        T3 --> T4[Send to Colleague]
    end
    
    subgraph "Collaborating Teacher"  
        C1[Receive Edit Link]
        C1 --> C2[Access Shared Lesson]
        C2 --> C3[Edit Lesson Plan]
        C3 --> C4[Save Changes]
        C4 --> C5[Review Updates]
    end
    
    subgraph "System"
        S1[Validate Edit Token]
        S1 --> S2[Allow Modifications]
        S2 --> S3[Update Lesson Document]
        S3 --> S4[Sync to Original]
    end
    
    T4 --> C1
    C2 --> S1
    C4 --> S2
```

## API Endpoints

The sharing functionality is typically exposed through these endpoint patterns:

### Teacher Endpoints (Authenticated)

```
POST   /api/v1/lessons/{lesson_id}/share      # Create share link
GET    /api/v1/lessons/{lesson_id}/share      # Get existing share link
DELETE /api/v1/lessons/{lesson_id}/share      # Revoke share link
PUT    /api/v1/lessons/{lesson_id}/public     # Toggle public status
```

### Public Endpoints (Unauthenticated)

```
GET    /shared/{token}                        # Access shared lesson
PUT    /shared/{edit_token}                   # Edit shared lesson
GET    /shared-lessons                        # List public lessons
```

## Dependencies

### Internal Modules

- **[lesson_management](lesson_management.md)**: Parent module containing `LessonService`
- **[database_layer](database_layer.md)**: Provides `LessonRepository` for data access
- **[database_models](database_models.md)**: Defines `LessonInDB`, `LessonStatus`, and response models
- **[authentication](authentication.md)**: Teacher authorization for share management

### External Services

- **MongoDB**: Stores lesson documents with sharing tokens and status
- **Storage Service**: Provides URLs for lesson materials and images
- **Frontend**: Renders shared lesson views and public listing

## Key Features

### 1. Dual Token System
- **View Token**: Read-only access to lesson content
- **Edit Token**: Full editing capabilities for collaborative work
- Independent revocation and regeneration

### 2. Public Discovery
- Lessons marked `is_public=True` appear on landing page
- Curated showcase of quality lesson plans
- Teacher attribution and metadata display

### 3. Smart Image Selection
- Prioritizes AI-generated materials over textbook scans
- Automatic fallback to uploaded images
- Consistent preview experience

### 4. Playground Integration
- Includes interactive quiz/flashcard data when available
- Shows usage statistics (attempts, average score)
- Enables complete lesson experience sharing

### 5. Status-Aware Sharing
- Prevents sharing of incomplete lessons
- Ensures content quality before distribution
- Maintains professional standards

## Error Handling

### Common Scenarios

```python
# Lesson not found or unauthorized
if not lesson:
    return None  # Results in 404 Not Found

# Lesson not in shareable state
if lesson["status"] not in shareable_statuses:
    service_logger.warning(f"Cannot share lesson {lesson_id} in status {lesson['status']}")
    return None

# Invalid edit token
lesson = await lesson_repository.find_by_edit_token(edit_token)
if not lesson:
    return None  # Edit token invalid or expired
```

### Security Considerations

- Tokens are generated using cryptographically secure random
- No personal information embedded in tokens
- Automatic validation against lesson status
- Separate permissions for view vs edit access
- Teacher retains full control over sharing lifecycle

## Usage Examples

### Creating a Share Link

```python
# Teacher creates share link for a completed lesson
response = await lesson_service.create_share_link(
    lesson_id="507f1f77bcf86cd799439011",
    teacher_id="507f1f77bcf86cd799439012", 
    base_url="https://app.example.com"
)

# Returns:
# {
#   "lesson_id": "507f1f77bcf86cd799439011",
#   "share_token": "dGhpcyBpcyBhIHRva2Vu",
#   "share_url": "https://app.example.com/shared/dGhpcyBpcyBhIHRva2Vu",
#   "edit_token": "YW5vdGhlciB0b2tlbg",
#   "edit_url": "https://app.example.com/shared/YW5vdGhlciB0b2tlbg",
#   "created_at": "2024-01-15T10:30:00Z"
# }
```

### Accessing a Shared Lesson

```python
# Public user accesses shared lesson
lesson = await lesson_service.get_shared_lesson(
    share_token="dGhpcyBpcyBhIHRva2Vu"
)

# Returns SharedLessonResponse with:
# - Lesson metadata and content
# - Teacher name for attribution  
# - Playground info if available
# - can_edit=False (view-only access)
```

### Collaborative Editing

```python
# Collaborator edits shared lesson
updated = await lesson_service.update_shared_lesson(
    edit_token="YW5vdGhlciB0b2tlbg",
    updates={
        "lesson_plan": {
            "periods": [...modified periods...]
        }
    }
)

# Returns updated SharedLessonResponse with:
# - Modified lesson plan
# - can_edit=True (verified edit access)
# - Updated timestamps
```

## Future Enhancements

Potential improvements to the sharing system:

1. **Time-Limited Sharing**: Automatic token expiration after set duration
2. **Access Analytics**: Track views, edits, and user engagement
3. **Version Control**: Maintain edit history for collaborative changes
4. **Comment System**: Allow viewers to add feedback without editing
5. **Template Sharing**: Share lesson structures as reusable templates
6. **Organization Sharing**: Share with school/department groups
7. **Permission Levels**: Granular control (view, comment, edit, admin)
8. **Sharing Requests**: Allow users to request access to private lessons

## Conclusion

The Lesson Sharing module provides a robust, secure foundation for collaborative lesson planning and resource sharing. Its token-based architecture ensures both security and flexibility, while tight integration with the lesson lifecycle maintains content quality and professional standards.