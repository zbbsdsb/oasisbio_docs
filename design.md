# OasisBio Design Documentation

## Design Philosophy

OasisBio follows a minimalist, modern design approach inspired by Swiss design principles. The design emphasizes clarity, precision, and functionality while maintaining a strong visual identity. The focus is on creating a clean, intuitive interface that allows users to focus on building their digital identities without unnecessary distractions.

## Core Design Principles

### 1. Swiss Grid System
- **12-column grid** for consistent layout structure
- **Precise alignment** of elements
- **Strong negative space** to enhance readability
- **Modular design** for consistent component sizing

### 2. Black/White Minimalism
- **Monochromatic color scheme** with black, white, and shades of gray
- **High contrast** for improved readability
- **Clean typography** with clear hierarchy
- **Subtle animations** for enhanced user experience

### 3. Editorial Typography
- **Clear hierarchy** with distinct title and body text styles
- **Generous line spacing** for improved readability
- **Consistent font usage** across the application
- **Responsive typography** that adapts to different screen sizes

### 4. Motion Restraint
- **Subtle hover effects** for interactive elements
- **Smooth transitions** between states
- **Purposeful animations** that enhance functionality
- **Performance-optimized** motion effects

## Color Palette

### Primary Colors
- **Black**: #000000
- **White**: #FFFFFF

### Secondary Colors (Grayscale)
- **Dark Gray**: #111111
- **Medium Gray**: #666666
- **Light Gray**: #D9D9D9
- **Very Light Gray**: #F5F5F5

### Functional Colors
- **Success**: #10B981 (Emerald)
- **Warning**: #F59E0B (Amber)
- **Error**: #EF4444 (Red)
- **Info**: #3B82F6 (Blue)

## Typography

### Font Families
- **Headings**: Inter Tight / Helvetica Now / Satoshi
- **Body Text**: Inter / Suisse Intl
- **Monospace**: JetBrains Mono / IBM Plex Mono

### Font Sizes
- **H1**: 4rem - 6rem (64px - 96px)
- **H2**: 3rem - 4rem (48px - 64px)
- **H3**: 2rem - 2.5rem (32px - 40px)
- **H4**: 1.5rem - 1.75rem (24px - 28px)
- **Body**: 1rem - 1.25rem (16px - 20px)
- **Small**: 0.875rem (14px)
- **Caption**: 0.75rem (12px)

### Line Heights
- **Headings**: 1.1 - 1.2
- **Body Text**: 1.5 - 1.6
- **Small Text**: 1.4

## Layout Structure

### Grid System
```
.container {
  max-width: 1280px;
  margin: 0 auto;
  padding: 0 1rem;
}

.grid {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  gap: 1rem;
}
```

### Spacing System
- **xs**: 0.25rem (4px)
- **sm**: 0.5rem (8px)
- **md**: 1rem (16px)
- **lg**: 1.5rem (24px)
- **xl**: 2rem (32px)
- **2xl**: 3rem (48px)
- **3xl**: 4rem (64px)
- **4xl**: 6rem (96px)

## Components

### Button
- **Variants**: primary, secondary, outline, ghost
- **Sizes**: sm, md, lg, xl
- **States**: default, hover, active, disabled
- **Styles**:
  ```css
  .btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    padding: 0.5rem 1rem;
    border-radius: 0.25rem;
    font-weight: 500;
    transition: all 0.2s ease;
  }
  
  .btn-primary {
    background-color: #000000;
    color: #FFFFFF;
  }
  
  .btn-primary:hover {
    background-color: #111111;
  }
  ```

### Card
- **Structure**: header, content, footer
- **Variants**: default, elevated, outlined
- **Styles**:
  ```css
  .card {
    background-color: #FFFFFF;
    border: 1px solid #D9D9D9;
    border-radius: 0.5rem;
    overflow: hidden;
  }
  
  .card-header {
    padding: 1rem;
    border-bottom: 1px solid #D9D9D9;
  }
  
  .card-content {
    padding: 1rem;
  }
  ```

### Input
- **Types**: text, password, email, number, date, select
- **States**: default, focus, error, disabled
- **Styles**:
  ```css
  .input {
    width: 100%;
    padding: 0.5rem 0.75rem;
    border: 1px solid #D9D9D9;
    border-radius: 0.25rem;
    font-size: 1rem;
    transition: all 0.2s ease;
  }
  
  .input:focus {
    outline: none;
    border-color: #000000;
    box-shadow: 0 0 0 3px rgba(0, 0, 0, 0.1);
  }
  
  .input-error {
    border-color: #EF4444;
  }
  
  .input-error:focus {
    border-color: #EF4444;
    box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.1);
  }
  
  .error-message {
    color: #EF4444;
    font-size: 0.875rem;
    margin-top: 0.25rem;
  }
  ```

### Error Handling and User Feedback
- **Error States**: Clear visual indicators for form errors
- **Success Messages**: Green background with checkmark icon
- **Loading States**: Subtle loading indicators during form submission
- **Toast Notifications**: Non-intrusive feedback for user actions
- **Animation**: Smooth transitions for error/success states
- **Accessibility**: ARIA attributes for screen reader support

### Navbar
- **Structure**: logo, navigation links, user menu
- **Responsive**: collapsible for mobile devices
- **Styles**:
  ```css
  .navbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 1rem;
    background-color: #FFFFFF;
    border-bottom: 1px solid #D9D9D9;
  }
  
  .navbar-links {
    display: flex;
    gap: 1rem;
  }
  ```

## Page Layouts

### Homepage
- **Hero Section**: Large headline, subtitle, CTA buttons
- **Features Section**: Grid of feature cards
- **CTA Section**: Bold call-to-action
- **Footer**: Links and copyright information

### Dashboard
- **Sidebar**: Navigation menu
- **Main Content**: Cards for OasisBios, worlds, models
- **Header**: User information and actions

### OasisBio Builder
- **Step-by-Step**: Linear progression through creation steps
- **Form Sections**: Organized input fields
- **Navigation**: Previous/Next buttons

### Public OasisBio Page
- **Hero**: Identity name, tagline, era information
- **Basic Profile**: Core identity information
- **Ability Pool**: Visual representation of skills
- **Repositories**: Tabs for DCOS, References, Worlds
- **3D Model Viewer**: Interactive model display
- **Timeline**: Era versions navigation

## Responsive Design

### Breakpoints
- **Mobile**: < 640px
- **Tablet**: 640px - 1024px
- **Desktop**: > 1024px

### Responsive Strategies
- **Fluid typography** that scales with viewport
- **Grid system** that adjusts column count
- **Component stacking** for smaller screens
- **Touch-friendly** interactive elements

## Animations

### Micro-interactions
- **Button hover**: Subtle scale and color change
- **Card hover**: Slight lift and shadow increase
- **Input focus**: Border and shadow animation
- **Page transitions**: Fade-in effects

### Keyframe Animations
- **Hero text reveal**: Typewriter effect
- **Feature card entrance**: Staggered fade-in
- **Timeline navigation**: Smooth sliding transitions
- **3D model rotation**: Subtle continuous rotation

## Accessibility

### Color Contrast
- **Text vs Background**: Minimum 4.5:1 contrast ratio
- **Interactive Elements**: Minimum 3:1 contrast ratio
- **WCAG 2.1 AA compliance**

### Keyboard Navigation
- **Tab order** that follows logical flow
- **Focus indicators** for all interactive elements
- **Skip links** for screen reader users

### Screen Reader Support
- **Semantic HTML** for proper element identification
- **ARIA labels** for non-semantic elements
- **Landmark regions** for page structure

## Performance Optimization

### CSS Optimization
- **Tailwind CSS** for utility-first styling
- **Purge unused CSS** in production
- **Minify CSS** for reduced file size
- **Critical CSS** for above-the-fold content

### Image Optimization
- **Responsive images** with srcset
- **Lazy loading** for off-screen images
- **Optimal file formats** (WebP for modern browsers)
- **Image compression** without quality loss

### Font Optimization
- **Font subsetting** for reduced file size
- **Font display** strategy for better UX
- **Preload critical fonts** for faster rendering

## Implementation Guidelines

### CSS Best Practices
- **Utility-first** approach with Tailwind CSS
- **Component-based** styling for reusability
- **Consistent naming conventions**
- **Avoid !important** declarations
- **Organize styles** logically

### Design System
- **Consistent component usage** across the application
- **Shared design tokens** for colors, spacing, and typography
- **Documentation** for design patterns and components
- **Regular design reviews** to maintain consistency

## Future Design Considerations

### Dark Mode
- **Toggleable dark mode** for user preference
- **Optimized color palette** for dark backgrounds
- **Accessibility** considerations for dark mode

### Customization
- **Theme options** for personalization
- **Custom CSS** support for advanced users
- **Brand integration** for organizational users

### Internationalization
- **RTL support** for right-to-left languages
- **Cultural considerations** for design elements
- **Localized date and time formats**

## Conclusion

The OasisBio design system is built on the principles of clarity, precision, and functionality. By following these guidelines, we ensure a consistent, accessible, and visually appealing user experience across all devices and screen sizes. The design system is flexible enough to accommodate future enhancements while maintaining the core aesthetic that makes OasisBio unique.
