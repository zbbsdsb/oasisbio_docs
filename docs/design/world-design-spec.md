Below is the OasisBio Worldbuilding Standard.
Note: You have previously decided that the website will not include a world exploration system; the world setting is merely a background container for characters. Therefore, this standard is a structural guideline for users to create world settings, not an independent product module.

The goal is for each world setting to be:

- Structurally unified
- Referenceable by characters
- Extensible
- Not complex

OasisBio Worldbuilding Standard

The world setting structure is divided into six hierarchical modules:

World
│
├── Core Identity
├── Time Structure
├── World Rules
├── Civilization
├── Environment
└── Narrative Context

1. Core Identity (World Core)

This is the foundational identity of the world setting.

Fields:

- name
- tagline
- genre
- tone
- summary

name

World name

Example:

Archive City

tagline

One-sentence description

Example:

A civilization where memories are stored as physical artifacts.

genre
- sci-fi
- fantasy
- cyberpunk
- alternate history
- post-human
- mythic

tone
- dark
- hopeful
- mysterious
- utopian
- dystopian
- epic

summary

Brief world introduction.

2. Time Structure

Defines the time state of the world.

Fields:

- era_name
- time_period
- timeline
- major_events

era_name
- Founding Era
- Archive Era
- Collapse Era

time_period
- past
- present
- future
- timeless

timeline

Key historical nodes.

Example:

2030 – Archive technology invented
2050 – Memory trade legalized
2090 – Archive City established

major_events

Major events.

3. World Rules

This is the most important part of the world setting.

Fields:

- physics_rules
- technology_level
- power_system
- limitations

physics_rules

Describes whether physical laws are altered.

Example:

Memory can be materialized.

technology_level
- primitive
- industrial
- modern
- advanced
- post-human

power_system

If a power system exists:

Example:

- Memory manipulation
- Energy channels
- Arcane magic

limitations

World limitations.

Example:

Memories decay after 100 years.

4. Civilization

Describes social structure.

Fields:

- governance
- economy
- factions
- social_structure
- culture

governance

Example:

- Council of Archivists
- Corporate rule
- AI governance

economy

Example:

- memory trade
- energy credits
- data economy

factions

Major factions in the world.

Example:

- The Archivists
- Memory Traders
- Data Erasers

social_structure

Example:

- Archivists
- Citizens
- Nomads

culture

Example:

- Memory rituals
- Archive festivals

5. Environment

Describes world space.

Fields:

- geography
- cities
- landmarks
- environmental_features

geography

Example:

- Floating islands
- Endless ocean
- Desert megacity

cities

Example:

- Archive City
- Neon Harbor

landmarks

Example:

- The Great Archive
- Memory Vault

environmental_features

Example:

- Neon storms
- Data fog

6. Narrative Context

Explains how characters exist in the world.

Fields:

- conflict
- themes
- story_hooks
- character_roles

conflict

Example:

- Memory ownership wars
- AI rebellion
- Archive corruption

themes

Example:

- identity
- memory
- power
- control

story_hooks

Potential stories in the world.

Example:

A lost archive threatens the city's history.

character_roles

Typical character identities in the world.

Example:

- Archivist
- Memory Hunter
- Data Merchant

7. World Creation Process

In the Builder, it is recommended to create in this order:

1. World Name
2. Summary
3. Time Structure
4. World Rules
5. Civilization
6. Environment
7. Narrative Context

Each step corresponds to an editing module.

8. Relationship between World and Characters

The relationship structure in your system is:

World
  └── Characters

Character field:

character.world_id

A world can have multiple characters.

9. Database Structure Suggestion

worlds table:

- id
- name
- tagline
- genre
- tone
- summary
- time_structure
- world_rules
- civilization
- environment
- narrative_context
- visibility
- owner_id
- created_at

Complex fields can be stored as JSON.

10. World Page Structure (Within Character Page)

World module in character page:

World
│
├── Summary
├── Rules
├── Civilization
├── Environment
└── Conflict

Only display key content.

11. Example World
Archive City
Name
Archive City

Genre
Cyberpunk

Summary
A city where memories are stored and traded.

Physics Rules
Memories can be materialized.

Technology
Advanced

Governance
Council of Archivists

Conflict
Memory ownership wars

12. Design Principles

This standard follows three principles:

1. Clear Structure

All world settings have the same framework.

2. Extensible

Future additions could include:

- maps
- organizations
- species

3. Character Priority

The world is just a character background and should not overshadow the character's central role.

13. Final World Structure
World
│
├── Core Identity
├── Time Structure
├── World Rules
├── Civilization
├── Environment
└── Narrative Context