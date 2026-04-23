# OasisBio Documentation

Welcome to the OasisBio Documentation! This documentation provides comprehensive information about the OasisBio platform, including technical details, design guidelines, feature documentation, and strategic plans.

## What is OasisBio?

OasisBio is a digital identity builder and character creator platform for building expandable fictional character identities across eras. Users can create rich character profiles with ability pools, worldbuilding, DCOS documents, references, and 3D model support.

## Tech Stack

- **Framework**: Next.js 16 (App Router)
- **Language**: TypeScript 5.4
- **Styling**: Tailwind CSS 3.4
- **Database**: PostgreSQL via Supabase (Prisma 6 ORM)
- **Auth**: Supabase Auth (OTP / passwordless)
- **Storage**: Supabase Storage (images) + Cloudflare R2 (3D models, exports)
- **Deployment**: Cloudflare Pages

## Key Features

- **OasisBio Builder** — step-by-step character creation with era system
- **Era Timeline** — manage past/present/future/alternate/worldbound eras per character
- **World Builder** — 6-module guided worldbuilding wizard
- **Ability Pool** — categorized skills with era/world binding
- **DCOS Repository** — character narrative documents
- **References Library** — external links and resources
- **3D Model Viewer** — GLB format with Three.js
- **Import/Export** — ZIP-based character data portability
- **Publish System** — atomic publish/unpublish via database RPC
- **OAuth Provider** — "Continue with Oasis" for third-party apps (Authorization Code + PKCE)

## Documentation Structure

- **Technical Documentation**: Architecture, API reference, database schema, and deployment details
- **Design Documentation**: Design system, color palette, typography, and components
- **Feature Documentation**: Detailed documentation for each feature module
- **Strategic Documentation**: Strategic plans and roadmap for the project

## Getting Started

To get started with OasisBio, please refer to the [Technical Reference](technical/technical.md) for installation and setup instructions.

## License

MIT — see [LICENSE](LICENSE)