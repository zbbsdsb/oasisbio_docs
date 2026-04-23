# Repository System

## Overview

The Repository system is a core feature of OasisBio that allows users to store and organize content related to their identities. It consists of three interconnected repositories: DCOS (Dynamic Core Operating Script), References, and Worlds. These repositories provide a structured way to manage identity-related content and enhance the depth of each OasisBio.

## DCOS Repository

### Overview

**DCOS (Dynamic Core Operating Script)** is a narrative layer where users define their identity's core logic, principles, and internal scripts. It serves as the foundational narrative framework for the identity.

### Purpose

- Define the identity's core principles and values
- Establish the identity's voice and tone
- Create narrative fragments that shape the identity
- Set rules and guidelines for the identity's behavior
- Document the identity's evolution over time

### Implementation

#### API Endpoints

- **GET /api/oasisbios/[id]/dcos**: Retrieves all DCOS files for a specific OasisBio
- **POST /api/oasisbios/[id]/dcos**: Creates a new DCOS file
- **PUT /api/dcos/[id]**: Updates an existing DCOS file
- **DELETE /api/dcos/[id]**: Deletes a DCOS file

#### Data Model

```typescript
interface DcosFile {
  id: string;
  oasisBioId: string;
  title: string;
  slug: string;
  content: string;
  folderPath: string;
  status: string; // draft, published
  version: number;
  eraId: string | null;
  createdAt: Date;
  updatedAt: Date;
}
```

#### Frontend Features

- **File List**: Displays all DCOS files with status and version information
- **File Editor**: Markdown-enabled editor for creating and editing DCOS content
- **Versioning**: Automatically increments version number on updates
- **Status Management**: Supports draft and published states
- **Loading States**: Visual feedback during API requests
- **Error Handling**: Clear error messages for user actions

## References Repository

### Overview

The References repository is a structured collection of external sources that relate to the identity. It allows users to curate and organize links to articles, videos, music, images, and other resources that inform or inspire the identity.

### Purpose

- Curate resources that influence the identity
- Provide context for the identity's interests and influences
- Create a personal knowledge base for the identity
- Document external references that shape the identity

### Reference Types

- **Article**: Written content from websites, blogs, or publications
- **Video**: Video content from platforms like YouTube, Vimeo, etc.
- **Song**: Audio content including songs and music
- **Image**: Visual content including photos, artwork, and graphics
- **Research**: Academic or research-based content
- **Moodboard**: Collections of visual references that inspire the identity
- **Social Account**: Links to social media profiles or online presence
- **Archive Page**: Links to archived content or historical resources

### Implementation

#### API Endpoints

- **GET /api/oasisbios/[id]/references**: Retrieves all reference items for a specific OasisBio
- **POST /api/oasisbios/[id]/references**: Creates a new reference item
- **PUT /api/references/[id]**: Updates an existing reference item
- **DELETE /api/references/[id]**: Deletes a reference item

#### Data Model

```typescript
interface ReferenceItem {
  id: string;
  oasisBioId: string;
  url: string;
  title: string;
  description: string | null;
  sourceType: string;
  provider: string | null;
  coverImage: string | null;
  metadata: string | null;
  eraId: string | null;
  worldId: string | null;
  tags: string;
}
```

#### Frontend Features

- **Reference Grid**: Visual display of references as cards
- **Filtering**: Filter references by type
- **Search**: Search references by title, description, or tags
- **Add New References**: Form for adding new reference items
- **Delete References**: Remove unwanted reference items
- **Loading States**: Visual feedback during API requests
- **Error Handling**: Clear error messages for user actions

## Worlds Repository

### Overview

The Worlds repository allows users to create and manage fictional or conceptual worlds that their identities can inhabit. It provides a structured way to build detailed worlds with their own rules, histories, and characteristics.

### Purpose

- Create fictional settings for identities
- Establish consistent rules and lore for worlds
- Build immersive environments for fictional identities
- Document world-building details for reference

### Implementation

#### API Endpoints

- **GET /api/oasisbios/[id]/worlds**: Retrieves all worlds for a specific OasisBio
- **POST /api/oasisbios/[id]/worlds**: Creates a new world
- **PUT /api/worlds/[id]**: Updates an existing world
- **DELETE /api/worlds/[id]**: Deletes a world

#### Data Model

```typescript
interface WorldItem {
  id: string;
  oasisBioId: string;
  name: string;
  summary: string;
  timeSetting: string | null;
  geography: string | null;
  physicsRules: string | null;
  socialStructure: string | null;
  aestheticKeywords: string | null;
  majorConflict: string | null;
  visibility: string; // private, public
  timeline: string | null;
  rules: string | null;
  factions: string | null;
}
```

#### Frontend Features

- **World List**: Displays all worlds with basic information
- **World Editor**: Comprehensive form for creating and editing worlds
- **Detailed Fields**: Supports all world-building fields including geography, physics rules, social structure, etc.
- **Loading States**: Visual feedback during API requests
- **Error Handling**: Clear error messages for user actions

## World Documents

### Overview

World Documents are individual documents associated with a specific world. They allow users to create detailed content about different aspects of a world.

### Implementation

#### API Endpoints

- **GET /api/worlds/[id]/documents**: Retrieves all documents for a specific world
- **POST /api/worlds/[id]/documents**: Creates a new world document
- **PUT /api/worlddocuments/[id]**: Updates an existing world document
- **DELETE /api/worlddocuments/[id]**: Deletes a world document

#### Data Model

```typescript
interface WorldDocument {
  id: string;
  worldId: string;
  title: string;
  docType: string;
  slug: string;
  content: string;
  folderPath: string;
  sortOrder: number;
  createdAt: Date;
  updatedAt: Date;
}
```

## Authentication and Authorization

All repository endpoints implement:
- **Authentication**: Uses Supabase Auth to verify user is logged in
- **Authorization**: Verifies that the requested resource belongs to the authenticated user
- **Error Handling**: Proper error responses for unauthorized access, not found resources, and server errors

## Best Practices

1. **Organization**: Use clear and descriptive titles for all repository items
2. **Documentation**: Provide detailed descriptions for references and worlds
3. **Cross-Referencing**: Link related content across repositories using eraId and worldId fields
4. **Version Control**: Take advantage of automatic versioning for DCOS files
5. **Accessibility**: Ensure content is well-organized and easy to navigate

## Future Enhancements

- **Collaborative Editing**: Allow multiple users to edit repository content
- **AI-Assisted Content Generation**: Generate content suggestions based on existing material
- **Content Import/Export**: Import and export repository content between OasisBios
- **Interactive World Maps**: Create interactive maps for world visualization
- **Repository Templates**: Pre-built templates for common repository structures
- **Advanced Search**: More powerful search capabilities across all repositories
- **Content Tagging**: Enhanced tagging system for better organization
- **Media Integration**: Direct integration with media platforms for references
