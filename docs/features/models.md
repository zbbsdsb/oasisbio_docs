# 3D Character Model System

## Overview

The 3D Character Model system is a key feature of OasisBio that allows users to upload, manage, and display 3D models representing their identities. It provides a visual representation of the identity and enhances the overall presentation of the OasisBio profile.

## Supported Formats

### GLB Format
- **Description**: Binary GLTF file format
- **Advantages**: Single file format, supports animations, efficient loading, widely supported
- **File Extension**: `.glb`

### Texture Files (Embedded in GLB)
- **Description**: Image files used for textures, embedded within the GLB file
- **Supported Formats**: JPEG, PNG, WebP
- **Purpose**: Add details and realism to the 3D model

## Model Management

### Uploading Models
1. **Select File**: Choose a GLB file from your device
2. **Preview**: View the model before uploading
3. **Name Model**: Provide a name for the model
4. **Set Properties**: Configure model settings and associations
5. **Upload**: Submit the model to the system

### Editing Models
- **Update Model**: Replace the GLB file with a new version
- **Change Properties**: Update model name, description, and associations
- **Adjust Preview**: Set a new default view for the model

### Deleting Models
- **Remove Model**: Delete the model and all associated files
- **Confirmation**: Required before permanent deletion

## Model Properties

- **Name**: The name of the model
- **Description**: A brief description of the model
- **File Path**: The storage location of the GLB file
- **Model Format**: Format of the model (default: GLB)
- **Preview Image**: A static image of the model for preview purposes
- **Related World**: World the model is associated with (optional)
- **Related Era**: Era the model is associated with (optional)
- **Is Primary**: Whether this is the primary model for the identity
- **Version**: Version number of the model
- **Created At**: Date and time the model was uploaded
- **Updated At**: Date and time the model was last updated

## 3D Viewer

### Features
- **Rotation**: Rotate the model to view from different angles
- **Zoom**: Zoom in and out to examine details
- **Pan**: Move the model within the viewing area
- **Lighting**: Adjust lighting to highlight features
- **Background**: Toggle between black, white, or transparent background
- **Camera Presets**: Quick access to common viewing angles
- **Model Information**: Display model details and properties

### Implementation
- **Technology**: Three.js with GLTFLoader
- **Performance**: Optimized for web-based viewing
- **Compatibility**: Works across modern browsers
- **Loading**: Progressive loading for large models

## Era and World Binding

### Era Binding
- **Description**: Link models to specific time periods
- **Use Case**: Show how the identity's appearance changes over time
- **Example**: "Young Elara" vs "Elder Elara"

### World Binding
- **Description**: Link models to specific fictional worlds
- **Use Case**: Create world-specific appearances for the identity
- **Example**: "Elara in Cyberpunk World" vs "Elara in Fantasy World"

## Model Storage

### File Structure

```
models/
├── {user_id}/                # User-specific directory
│   ├── {character_id}/        # Character-specific directory
│   │   ├── model.glb          # GLB model file
│   │   ├── preview.webp       # Preview image
│   │   └── history/           # Version history
│   │       ├── {timestamp}/   # Timestamp-based version
│   │           └── model.glb   # Previous version of model
```

### Storage Options
- **Development**: Stored in Cloudflare R2
- **Production**: Stored in Cloudflare R2
- **Access**: Public buckets for images, private bucket for 3D models

## Model Optimization

### Best Practices
1. **File Size**: Keep models under 10MB for faster loading
2. **Polygon Count**: Optimize geometry for web viewing
3. **Textures**: Use compressed textures to reduce file size
4. **LOD (Level of Detail)**: Implement different detail levels for different viewing distances
5. **Caching**: Enable browser caching for frequently accessed models

### Optimization Techniques
- **Mesh Simplification**: Reduce polygon count while maintaining visual quality
- **Texture Compression**: Use appropriate image formats and compression levels
- **Asset Bundling**: GLB format already bundles all assets in one file
- **Lazy Loading**: Load models only when needed

## Preview Generation

### Automatic Preview
- **Process**: System automatically generates a preview image when a model is uploaded
- **Method**: Renders the model from a default angle
- **Format**: WebP format for optimal size and quality
- **Resolution**: Optimized for web display

### Custom Preview
- **Process**: User can set a custom preview by adjusting the model view
- **Method**: Capture the current view in the 3D viewer
- **Options**: Adjust lighting and background before capturing

## Model Display

### Public Profile
- **Location**: Displayed in the 3D Model section of the public OasisBio page
- **Interaction**: Visitors can rotate, zoom, and examine the model
- **Controls**: Intuitive controls for model manipulation
- **Performance**: Optimized for smooth interaction

### Dashboard
- **Location**: Displayed in the Models section of the user dashboard
- **Management**: Options to edit, delete, or set as default
- **Preview**: Thumbnail previews of all models

## Model Templates

### Pre-built Models
- **Description**: Ready-to-use 3D models provided by the platform
- **Categories**: Human, fantasy, sci-fi, abstract
- **Customization**: Users can modify templates to fit their identity
- **Usage**: Ideal for users without 3D modeling experience

### Model Packs
- **Description**: Collections of related models
- **Examples**: "Fantasy Character Pack", "Sci-Fi Explorer Pack"
- **Benefits**: Consistent style across multiple models
- **Availability**: Free and premium options

## Best Practices

1. **File Size**: Keep models small for faster loading
2. **Quality**: Balance detail with performance
3. **Consistency**: Maintain consistent style across related models
4. **Naming**: Use clear, descriptive names for models
5. **Organization**: Group related models together

## Future Enhancements

- **Animation Support**: Add animated models with skeletal rigs
- **Interactive Models**: Allow users to customize model features in real-time
- **Model Sharing**: Enable users to share models with others
- **AI Model Generation**: Generate 3D models from descriptions or images
- **Augmented Reality**: View models in AR using mobile devices
- **Model Marketplace**: Buy and sell custom models