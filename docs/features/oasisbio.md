# OasisBio Identity Container

## Overview

The OasisBio identity container is the core building block of the OasisBio system. It acts as a modular identity container that users can create and manage to represent different aspects of their identity across time and worlds.

## Identity Modes

OasisBio supports multiple identity modes to accommodate different types of identities:

### 1. Real Identity
- **Description**: Represents the user's actual self in the real world
- **Use Case**: Personal profiles, professional identities
- **Example**: "John Doe - Software Engineer"

### 2. Fictional Identity
- **Description**: Completely fictional characters created by the user
- **Use Case**: Role-playing characters, story characters, game avatars
- **Example**: "Elara Stormrider - Space Explorer"

### 3. Hybrid Identity
- **Description**: A mix of real and fictional elements
- **Use Case**: Personal brands with creative elements, semi-fictional personas
- **Example**: "Tech Wizard - Real developer with fictional magical elements"

### 4. Future Identity
- **Description**: Representation of the user's future self
- **Use Case**: Goal setting, future planning, career development
- **Example**: "Dr. Jane Smith - 2030 Nobel Prize Winner"

### 5. Alternate Identity
- **Description**: Parallel universe versions of the user
- **Use Case**: What-if scenarios, alternative life paths
- **Example**: "Jane Smith - Parallel Universe Rockstar"

## Basic Profile Information

Each OasisBio includes the following basic profile information:

### Core Attributes
- **Name**: The primary name of the identity
- **Slug**: Unique URL-friendly identifier
- **Tagline**: Short description of the identity
- **Identity Mode**: Real, fictional, hybrid, future, or alternate
- **Birth Date**: Date of birth (optional)
- **Gender**: Gender identity (optional)
- **Pronouns**: Preferred pronouns (optional)
- **Place of Origin**: Birthplace or origin location (optional)
- **Current Era**: Time period the identity belongs to (optional)
- **Species**: Species or identity class (optional, for fictional characters)
- **Status**: Status of the identity (Alive, Archived, Mythic, Unknown)
- **Description**: Detailed description of the identity

## Era-Specific Identities

OasisBio allows users to create multiple identity versions across different time periods:

### Era Types
- **Past Self**: Historical versions of the identity
- **Present Self**: Current version of the identity
- **Future Self**: Future versions of the identity
- **Alternate Self**: Parallel universe versions
- **Fictional Self**: Fictional character versions
- **Worldbound Self**: Versions bound to specific worlds

### Era Management
- Create multiple era identities for a single OasisBio
- Set start and end dates for each era
- Link abilities, models, and repositories to specific eras
- Navigate between era versions in the public profile

## Identity Management

### Creating an OasisBio
1. **Choose Identity Mode**: Select the type of identity you want to create
2. **Basic Information**: Enter core profile details
3. **Ability Pool**: Add skills and abilities
4. **Repositories**: Set up DCOS, References, and Worlds
5. **3D Model**: Upload a visual representation
6. **Publish**: Make the OasisBio public or keep it private

### Editing an OasisBio
- Update basic information
- Manage era identities
- Modify ability pool
- Edit repository content
- Update 3D model
- Change visibility settings

### Deleting an OasisBio
- Permanently remove an OasisBio and all associated data
- Confirmation required before deletion

## Visibility Settings

OasisBio identities can have different visibility settings:

- **Private**: Only visible to the creator
- **Public**: Visible to anyone
- **Unlisted**: Visible only via direct URL

## Example OasisBio Structure

```
OasisBio: "Elara Stormrider"
├── Basic Info
│   ├── Name: "Elara Stormrider"
│   ├── Tagline: "Intergalactic Explorer"
│   ├── Identity Mode: "Fictional"
│   ├── Species: "Human-Alien Hybrid"
│   ├── Place of Origin: "Planet Vaelor"
│   └── Description: "A fearless explorer traversing the galaxy..."
├── Era Identities
│   ├── "Elara - Cadet"
│   ├── "Elara - Captain"
│   └── "Elara - Legend"
├── Ability Pool
├── Repositories
└── 3D Model
```

## Best Practices

1. **Consistency**: Maintain consistent information across era identities
2. **Organization**: Use clear naming conventions for era identities
3. **Detail**: Provide sufficient detail to make each identity compelling
4. **Visuals**: Use 3D models to enhance identity representation
5. **Worldbuilding**: Link identities to appropriate fictional worlds

## Future Enhancements

- **Cross-Identity Relationships**: Connect multiple OasisBios
- **Identity Evolution Tracking**: Visualize identity changes over time
- **AI-Assisted Identity Creation**: Generate identity suggestions based on user input
- **Identity Templates**: Pre-built identity templates for different use cases
