# Frontend Components Module

## Overview

The **Frontend Components** module provides reusable React components that encapsulate complex UI functionality for the educational content creation platform. The primary component in this module is the `MaterialLibraryModal`, which offers a unified interface for browsing, searching, and generating educational materials including animations, interactive experiments, and images.

## Core Functionality

### MaterialLibraryModal Component

The `MaterialLibraryModal` is a sophisticated React component that serves as a comprehensive material management interface with the following capabilities:

**Browse & Search Features:**
- Search materials by keywords with debounced input (300ms delay)
- Filter materials by type (animation, experiment, image)
- Optional filtering by subject and grade level
- Real-time loading indicators and error handling
- Visual preview of materials with thumbnails

**Material Generation Features:**
- Create new animations, experiments, or images from natural language prompts
- Support for optional titles and detailed descriptions
- Real-time generation status polling (3-second intervals)
- Automatic selection of completed materials
- Usage tracking integration

**User Experience:**
- Modal dialog with backdrop blur and body scroll lock
- Tab-based navigation between Browse and Generate modes
- Responsive grid layout for material cards
- Hover effects and visual feedback
- Comprehensive loading and empty states

## Architecture

### Component Structure

```mermaid
graph TB
    subgraph "Frontend Components Module"
        MLM[MaterialLibraryModal]
        MS[MaterialSelection Interface]
        MLMP[MaterialLibraryModalProps Interface]
        MTI[MATERIAL_TYPE_INFO Constant]
    end
    
    subgraph "External Dependencies"
        API[materialsAPI]
        TYPES[SharedMaterial Types]
        BUTTON[Button Component]
        ICONS[Lucide Icons]
    end
    
    subgraph "React Ecosystem"
        REACT[React Hooks]
        PORTAL[React Portal]
        TS[TypeScript]
    end
    
    MLM --> MS
    MLM --> MLMP
    MLM --> MTI
    MLM --> API
    MLM --> TYPES
    MLM --> BUTTON
    MLM --> ICONS
    MLM --> REACT
    MLM --> PORTAL
    MLM --> TS
    
    style MLM fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#fff
    style MS fill:#10b981,stroke:#059669,stroke-width:2px
    style MLMP fill:#10b981,stroke:#059669,stroke-width:2px
```

### Data Models

```mermaid
classDiagram
    class MaterialSelection {
        <<interface>>
        +string id
        +SharedMaterialType type
        +string title
        +string description
        +string prompt
        +string storage_url
        +string html_content
        +boolean generated
    }
    
    class MaterialLibraryModalProps {
        <<interface>>
        +boolean isOpen
        +function onClose()
        +function onSelectMaterial(MaterialSelection)
        +SharedMaterialType filterType
        +string subject
        +number grade
    }
    
    class SharedMaterial {
        <<type>>
        +string id
        +SharedMaterialType type
        +string title
        +string description
        +string prompt
        +string storage_url
        +string thumbnail_url
        +string subject
        +number grade
        +number usage_count
        +string status
    }
    
    class MATERIAL_TYPE_INFO {
        <<constant>>
        +object animation
        +object experiment
        +object image
    }
    
    MaterialSelection --> SharedMaterial : transforms from
    MaterialLibraryModalProps --> MaterialSelection : outputs
```

## Dependencies

### Module Dependencies

```mermaid
graph LR
    subgraph "frontend_components"
        FC[MaterialLibraryModal]
    end
    
    subgraph "frontend_api_client"
        API[materialsAPI]
        CHAT_API[Chat APIs]
    end
    
    subgraph "frontend_types"
        TYPES[SharedMaterial\nSharedMaterialType]
    end
    
    subgraph "frontend_components_internal"
        BUTTON[Button Component]
    end
    
    subgraph "material_management"
        MM[MaterialService]
        MR[MaterialRepository]
    end
    
    subgraph "ai_generation_services"
        AI[AI Generation Services]
        AG[AnimationGenerator]
        EG[ExperimentGenerator]
    end
    
    FC --> API
    FC --> TYPES
    FC --> BUTTON
    API --> MM
    API --> AI
    MM --> MR
    AI --> AG
    AI --> EG
    
    style FC fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#fff
    style API fill:#3b82f6,stroke:#2563eb,stroke-width:2px
    style TYPES fill:#10b981,stroke:#059669,stroke-width:2px
```

### External Libraries

- **React**: Core UI framework with hooks (useState, useEffect, useCallback)
- **ReactDOM**: Portal API for modal rendering
- **TypeScript**: Type safety and interfaces
- **Lucide React**: Icon components (X, Search, Plus, Video, FlaskConical, Image, etc.)
- **Tailwind CSS**: Utility-first styling (implied by class names)

## Data Flow

### Browse Materials Flow

```mermaid
sequenceDiagram
    participant User
    participant Modal as MaterialLibraryModal
    participant API as materialsAPI
    participant Backend as MaterialService
    participant DB as MongoDB
    
    User->>Modal: Open modal
    Modal->>Modal: Initialize state
    Modal->>API: list() / search()
    API->>Backend: HTTP Request
    Backend->>DB: Query materials
    DB-->>Backend: Material documents
    Backend-->>API: SharedMaterial[]
    API-->>Modal: Set materials state
    Modal->>Modal: Render material cards
    User->>Modal: Click material card
    Modal->>API: use(material.id)
    API->>Backend: Track usage
    Modal->>User: onSelectMaterial(selection)
```

### Generate Material Flow

```mermaid
sequenceDiagram
    participant User
    participant Modal as MaterialLibraryModal
    participant API as materialsAPI
    participant Backend as AI Generation
    participant Queue as Generation Queue
    
    User->>Modal: Fill generation form
    User->>Modal: Click "Generate"
    Modal->>Modal: setIsGenerating(true)
    Modal->>API: generateAnimation() / generateExperiment() / generate()
    API->>Backend: POST /materials/generate
    Backend->>Queue: Start async generation
    Queue-->>Backend: Job ID
    Backend-->>API: {id, status}
    API-->>Modal: setGeneratedMaterialId(id)
    loop Every 3 seconds
        Modal->>API: get(id)
        API->>Backend: Check status
        Backend-->>API: {status, material}
        API-->>Modal: Update generationStatus
    end
    Modal->>Modal: handleSelectMaterial()
    Modal->>User: onSelectMaterial(selection)
```

## Component Interactions

### State Management

```mermaid
stateDiagram-v2
    [*] --> Closed : isOpen = false
    Closed --> Open : User opens modal
    
    state Open {
        [*] --> BrowseTab : activeTab = 'browse'
        [*] --> GenerateTab : activeTab = 'generate'
        
        state BrowseTab {
            [*] --> Loading : Initial load
            Loading --> Loaded : Materials fetched
            Loaded --> Searching : User types search
            Searching --> Loaded : Debounced search complete
            Loaded --> MaterialSelected : User clicks material
        }
        
        state GenerateTab {
            [*] --> FormReady : Form initialized
            FormReady --> Generating : User submits
            Generating --> Polling : Generation started
            Polling --> Completed : Status = 'completed'
            Polling --> Failed : Status = 'failed'
            Completed --> MaterialSelected : Auto-select
        }
    }
    
    MaterialSelected --> [*] : onSelectMaterial() & onClose()
```

### Hook Dependencies

```mermaid
graph TD
    subgraph "React Hooks"
        UE1[useEffect: Load materials on open]
        UE2[useEffect: Debounced search]
        UE3[useEffect: Poll generation status]
        UE4[useEffect: Body scroll lock]
        UC[useCallback: loadMaterials]
        US1[useState: materials]
        US2[useState: searchQuery]
        US3[useState: activeTab]
        US4[useState: isGenerating]
        US5[useState: generatedMaterialId]
    end
    
    UE1 --> UC
    UE1 --> US1
    UE2 --> US2
    UE3 --> US5
    UE3 --> US1
    UE4 --> US3
    UC --> US1
    UC --> US2
```

## Process Flows

### Material Selection Process

```mermaid
flowchart TD
    A[User opens MaterialLibraryModal] --> B{Has filterType?}
    B -->|Yes| C[Set selectedType = filterType]
    B -->|No| D[Show all types]
    C --> E[Load materials from API]
    D --> E
    E --> F{Search query?}
    F -->|Yes| G[Call materialsAPI.search]
    F -->|No| H[Call materialsAPI.list]
    G --> I[Display results grid]
    H --> I
    I --> J[User clicks material card]
    J --> K[Call materialsAPI.use]
    K --> L[Transform to MaterialSelection]
    L --> M[Call onSelectMaterial]
    M --> N[Close modal]
```

### Material Generation Process

```mermaid
flowchart TD
    A[User switches to Generate tab] --> B[Select material type]
    B --> C[Enter title optional]
    C --> D[Enter detailed prompt]
    D --> E[Click Generate button]
    E --> F{Material type?}
    F -->|animation| G[Call materialsAPI.generateAnimation]
    F -->|experiment| H[Call materialsAPI.generateExperiment]
    F -->|image| I[Call materialsAPI.generate]
    G --> J[Receive response with ID]
    H --> J
    I --> J
    J --> K{Status = completed?}
    K -->|Yes| L[Fetch full material]
    K -->|No| M[Start polling status]
    M --> N[Check status every 3s]
    N --> O{Status updated?}
    O -->|completed| P[Auto-select material]
    O -->|failed| Q[Show error message]
    O -->|pending/processing| M
    L --> P
    P --> R[Call onSelectMaterial]
    R --> S[Close modal]
```

## API Integration

### materialsAPI Methods Used

The component integrates with the following API endpoints through the `materialsAPI` service:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `list()` | GET /materials | Fetch paginated materials with filters |
| `search()` | GET /materials/search | Search materials by keywords |
| `get(id)` | GET /materials/:id | Get single material by ID |
| `use(id)` | POST /materials/:id/use | Track material usage |
| `generate()` | POST /materials/generate | Generate generic material |
| `generateAnimation()` | POST /materials/animations/generate | Generate animation |
| `generateExperiment()` | POST /materials/experiments/generate | Generate experiment |

### Request/Response Formats

**List Request:**
```typescript
{
  type?: SharedMaterialType;
  subject?: string;
  grade?: number;
  limit?: number;
  sort_by?: string;
  sort_order?: 'asc' | 'desc';
}
```

**Generate Request:**
```typescript
{
  type: SharedMaterialType;
  title: string;
  prompt: string;
  subject?: string;
  grade?: number;
}
```

**Generation Response:**
```typescript
{
  id: string;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  material?: SharedMaterial;
}
```

## UI/UX Features

### Visual Design System

The component implements a sophisticated UI with:

- **Gradient Headers**: Purple-to-lime gradient backgrounds for visual hierarchy
- **Type-based Color Coding**: 
  - Animation: Violet theme (`bg-violet-50`, `text-violet-700`)
  - Experiment: Lime theme (`bg-lime-50`, `text-lime-700`)
  - Image: Pink theme (`bg-pink-50`, `text-pink-700`)
- **Interactive States**: Hover effects, focus rings, and transition animations
- **Loading States**: Spinners and skeleton-like feedback
- **Empty States**: Helpful messages with action prompts

### Accessibility Features

- Semantic HTML structure
- Keyboard navigation support
- Focus management
- ARIA labels (implied by semantic elements)
- High contrast color schemes
- Responsive design for mobile and desktop

## Performance Considerations

### Optimization Strategies

1. **Debounced Search**: 300ms delay prevents excessive API calls
2. **useCallback Hook**: `loadMaterials` function is memoized to prevent unnecessary re-renders
3. **Conditional Polling**: Generation status polling only occurs when `generatedMaterialId` is set
4. **Lazy Loading**: Materials are loaded only when modal opens
5. **Portal Rendering**: Modal renders outside normal DOM hierarchy to avoid CSS conflicts

### Resource Management

- **Body Scroll Lock**: Prevents background scrolling when modal is open
- **Cleanup Effects**: Proper cleanup of timers and event listeners
- **Memory Management**: State reset on modal close

## Error Handling

### Error Scenarios

1. **API Failures**: Try-catch blocks with user-friendly error messages
2. **Generation Failures**: Status polling detects failed jobs
3. **Network Issues**: Graceful degradation with loading states
4. **Invalid Input**: Form validation before submission

### User Feedback

- Loading spinners for async operations
- Status messages for generation progress
- Alert dialogs for critical errors
- Visual feedback for successful actions

## References

This module integrates with and depends on:

- **[frontend_api_client](frontend_api_client.md)**: Provides the `materialsAPI` service for backend communication
- **[frontend_types](frontend_types.md)**: Defines `SharedMaterial`, `SharedMaterialType`, and related type definitions
- **[material_management](material_management.md)**: Backend service handling material CRUD operations
- **[ai_generation_services](ai_generation_services.md)**: Backend AI services for generating animations and experiments
- **[animation_generation](animation_generation.md)**: Specialized service for animation generation pipeline
- **[experiment_generation](experiment_generation.md)**: Specialized service for interactive experiment generation

For information about state management, see [frontend_state_management](frontend_state_management.md).

For context and page-level integration, see [frontend_contexts](frontend_contexts.md).