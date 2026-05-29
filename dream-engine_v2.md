# DreamEngine — Design Document
**Version 1.0 | Personal Worldbuilding & Creative Pipeline App**

---

## Table of Contents

1. [Vision & Goals](#1-vision--goals)
2. [Core Concepts & Mental Model](#2-core-concepts--mental-model)
3. [Data Architecture](#3-data-architecture)
4. [Node Types & Templates](#4-node-types--templates)
5. [Relationship System](#5-relationship-system)
6. [Timeline & State System](#6-timeline--state-system)
7. [Specialized Views](#7-specialized-views)
8. [LLM Generation System](#8-llm-generation-system)
9. [Image System & Visual Catalogue](#9-image-system--visual-catalogue)
10. [Character Crafting Pipeline](#10-character-crafting-pipeline)
11. [UI/UX Design Principles](#11-uiux-design-principles)
12. [Tech Stack](#12-tech-stack)
13. [Development Stages](#13-development-stages)
14. [Future Considerations](#14-future-considerations)

---

## 1. Vision & Goals

DreamEngine is a personal creative workspace for designing, organizing, and generating worldbuilding elements — characters, places, factions, events, and more. It serves as:

- A **living knowledge base** for fictional worlds, where everything is interconnected and time-aware
- A **generation assistant** that uses LLMs to help expand and iterate on creative ideas
- A **visual design pipeline** for creating and cataloguing image generation parameters, enabling fast iteration on character and location art
- A **long-term asset library** whose content can feed directly into SillyTavern roleplay setups and future game development projects

### Non-Goals (for now)
- Real-time collaboration with other users
- Cloud sync or multi-device support (personal-use, local-first)
- Integrated image generation (API calls to image gen models are out of scope; the app produces *prompts*, not images)
- Mobile support

---

## 2. Core Concepts & Mental Model

### Everything is a Node

All worldbuilding elements — Characters, Places, Factions, Events, etc. — are **Nodes** in a graph. Every Node has:

- A **type** that determines its template (which fields it has)
- A **rich text body** with typed, structured fields
- **Typed edges** to other Nodes (relationships)
- A **history** of state changes over time

### The World Graph

Nodes and their relationships form a directed property graph. This graph is the single source of truth for the entire world. Every view in the app — family trees, faction hierarchies, timelines, influence maps — is simply a *filtered, rendered lens* over this graph.

### Time as a First-Class Citizen

The world is not static. Characters age, factions rise and fall, borders shift. Rather than storing a single snapshot of each node, the system stores **state diffs tied to timeline events**. At any given point on the timeline, the full state of any node can be reconstructed by replaying all prior diffs.

This means:
- The timeline is not just a cosmetic feature — it is structurally integrated into how all data is stored and queried
- Every specialized view (family trees, influence maps, etc.) is time-responsive
- Every node has a personal timeline sub-page as a first-class feature

---

## 3. Data Architecture

### Technology Choice: Local-First with SQLite Backend

DreamEngine runs as a **local web app** — a lightweight backend (Node.js + Express or Fastify) serving a React frontend, both running on your machine. Data is stored in a local **SQLite** database. This approach provides:

- Full filesystem access for storing images locally
- Safe, local API key management (keys never leave your machine)
- Fast, reliable queries via SQLite
- No subscriptions, no cloud dependencies
- Easy backup (just copy the `.db` file and the image directory)

### Core Database Schema

#### `nodes` table
```
id          UUID (primary key)
type        ENUM (character, place, faction, event, item, species, faith, technology, cultural_belief, misc)
name        TEXT
slug        TEXT (url-safe identifier)
created_at  TIMESTAMP
updated_at  TIMESTAMP
```

#### `node_fields` table
Stores typed field values per node. Supports versioning via timeline events.
```
id          UUID
node_id     UUID (FK → nodes)
field_key   TEXT          (e.g. "biography", "population", "founding_date")
field_value TEXT (JSON)   (rich text stored as TipTap JSON, primitives as JSON values)
is_current  BOOLEAN       (true = current state, false = historical)
```

#### `edges` table
Stores all relationships between nodes.
```
id              UUID
source_node_id  UUID (FK → nodes)
target_node_id  UUID (FK → nodes)
edge_type       TEXT   (e.g. "member_of", "born_in", "at_war_with", "holds_faith", "controls")
metadata        TEXT (JSON)   (optional: e.g. { "role": "High Priest", "since_event_id": "..." })
is_current      BOOLEAN
```

#### `timeline_events` table
The master timeline. Each event can carry state diffs for any number of nodes.
```
id           UUID
name         TEXT
description  TEXT (rich text JSON)
date_value   INTEGER   (in-world date, an abstract integer — see Timeline section)
date_label   TEXT      (human-readable: "Year 312, Age of Ash")
tags         TEXT (JSON array of node IDs this event involves)
created_at   TIMESTAMP
```

#### `state_diffs` table
Records what changed at each timeline event, for each affected node.
```
id              UUID
event_id        UUID (FK → timeline_events)
node_id         UUID (FK → nodes)
field_key       TEXT
old_value       TEXT (JSON)
new_value       TEXT (JSON)
edge_change     TEXT (JSON)   (optional: { action: "add"|"remove"|"modify", edge_id: "..." })
```

#### `images` table
```
id              UUID
node_id         UUID (FK → nodes)
filename        TEXT
filepath        TEXT
label           TEXT          (e.g. "Default Outfit", "Battle Armor", "Young Years")
image_type      ENUM (character_illustration, place_illustration, item_illustration, concept, reference)
metadata        TEXT (JSON)   (full image generation metadata — see Image System section)
created_at      TIMESTAMP
```

#### `catalogue_components` table
The reusable visual prompt component library.
```
id              UUID
category        TEXT   (e.g. "outfit_top", "hairstyle", "background", "art_style", "eye_color")
subcategory     TEXT   (optional finer grouping)
label           TEXT   (display name, e.g. "Victorian High Collar Blouse")
prompt_text     TEXT   (the actual words to insert into a prompt)
thumbnail_id    UUID (FK → images, nullable)
tags            TEXT (JSON array)
notes           TEXT
created_at      TIMESTAMP
```

#### `generation_templates` table
LLM prompt templates are stored as data, not hardcoded.
```
id           UUID
node_type    TEXT
name         TEXT      (e.g. "Full Character", "Simplified NPC", "Faction Roster Member")
system       TEXT      (system prompt)
user_template TEXT     (handlebars-style template with {{field}} placeholders)
output_schema TEXT (JSON) (expected JSON structure of the LLM response)
created_at   TIMESTAMP
updated_at   TIMESTAMP
```

---

## 4. Node Types & Templates

Each node type has a predefined set of fields rendered as a structured form/page. Fields support: short text, long rich text, numbers, dates (in-world), selects, multi-selects, and node references (linked nodes).

### Character
| Field | Type |
|---|---|
| Full Name | Text |
| Aliases / Nicknames | Text |
| Species | Node Reference (Species) |
| Gender | Select |
| Age (at current timeline position) | Computed from birth event |
| Birth Date | In-world date |
| Death Date | In-world date (optional) |
| Birthplace | Node Reference (Place) |
| Factions (membership) | Node References (Faction) + role metadata |
| Faith | Node Reference (Faith) |
| Relationships | Edge list (see Relationship System) |
| Biography | Rich text |
| Personality | Rich text |
| Physical Description | Rich text |
| Dialogue Examples | Structured list (label + text) |
| Illustrations | Image gallery |
| Notes / GM Notes | Rich text |

### Place
| Field | Type |
|---|---|
| Name | Text |
| Place Type | Select (planet, solar system, plane, continent, region, country, city, settlement, landmark, dungeon, …) |
| Parent Location | Node Reference (Place) |
| Controlling Faction | Node Reference (Faction) |
| Population | Number |
| Founded | In-world date |
| Description | Rich text |
| Culture & Customs | Rich text |
| Notable Residents | Node References (Character) |
| Illustrations | Image gallery |

### Faction
| Field | Type |
|---|---|
| Name | Text |
| Faction Type | Select (kingdom, empire, guild, religion, cult, company, clan, …) |
| Founded | In-world date |
| Dissolved | In-world date (optional) |
| Leader | Node Reference (Character) |
| Members | Edge list (Character → Faction, with role) |
| Faith | Node Reference (Faith) |
| Allied With | Node References (Faction) |
| At War With | Node References (Faction) |
| Controls (territory) | Node References (Place) |
| Hierarchy | Structured tree (see Faction Hierarchy view) |
| Description | Rich text |
| Goals & Ideology | Rich text |
| History | Rich text |

### Event
| Field | Type |
|---|---|
| Name | Text |
| Date | In-world date |
| Duration | Text |
| Involved Nodes | Node References (any type) |
| Description | Rich text |
| Consequences | Rich text (what changed; also auto-linked to state diffs) |
| Illustrations | Image gallery |

### Item
| Field | Type |
|---|---|
| Name | Text |
| Item Type | Select (weapon, artifact, relic, mundane, …) |
| Creator | Node Reference (Character or Faction) |
| Current Owner | Node Reference (Character or Faction) |
| Created | In-world date |
| Description | Rich text |
| Powers / Properties | Rich text |
| History / Provenance | Rich text |
| Illustrations | Image gallery |

### Species
| Field | Type |
|---|---|
| Name | Text |
| Parent Species (for taxonomy) | Node Reference (Species) |
| Lifespan | Text |
| Origin | Node Reference (Place) |
| Notable Traits | Rich text |
| Biology | Rich text |
| Culture Overview | Rich text |
| Illustrations | Image gallery |

### Faith
| Field | Type |
|---|---|
| Name | Text |
| Deity / Deities | Node References (Character or custom) |
| Primary Faction | Node Reference (Faction) |
| Holy Sites | Node References (Place) |
| Description & Tenets | Rich text |
| Rituals | Rich text |
| History | Rich text |
| Illustrations | Image gallery |

### Technology
| Field | Type |
|---|---|
| Name | Text |
| Prerequisite Technologies | Node References (Technology) |
| Invented By | Node Reference (Character or Faction) |
| Discovered | In-world date |
| Description | Rich text |
| Applications | Rich text |
| Illustrations | Image gallery |

### Cultural Belief / Concept
| Field | Type |
|---|---|
| Name | Text |
| Associated Factions / Species | Node References |
| Description | Rich text |
| Origins | Rich text |

### Miscellaneous
A catch-all type with a minimal template: Name, Type Label (free text), Description, Notes, Illustrations.

---

## 5. Relationship System

### Edge Types

Edges are typed and directional. The system ships with a predefined set of edge types, but users can define custom types.

**Character ↔ Character**
- `parent_of` / `child_of`
- `sibling_of`
- `married_to`
- `romantically_involved_with`
- `allied_with`
- `rivals_with`
- `mentors`
- `enemies_with`
- `knows` (acquaintance-level)

**Character ↔ Faction**
- `member_of` (+ role metadata)
- `leads`
- `formerly_member_of`

**Character ↔ Place**
- `born_in`
- `resides_in`
- `rules`

**Faction ↔ Faction**
- `allied_with`
- `at_war_with`
- `vassal_of`
- `trade_partner_of`

**Faction ↔ Place**
- `controls`
- `has_presence_in`
- `founded`

**Place ↔ Place**
- `located_in` (for spatial hierarchy)

**Species ↔ Species**
- `descended_from` (for taxonomy)

**Technology ↔ Technology**
- `enables` (prerequisite relationship, for tech tree)

### Relationship Display on Node Pages

Each node's page renders a **Relationships panel** — automatically grouped by edge type and rendered as linked node cards. Examples:

- A Character page shows: Family, Romantic Relationships, Faction Memberships, Allies, Enemies, Residences
- A Faction page shows: Members (grouped by role), Territories, Allies, Enemies, Associated Faiths
- A Place page shows: Parent Location, Sub-locations, Controlling Faction, Notable Residents

---

## 6. Timeline & State System

### In-World Date System

The app uses an **abstract integer** as the canonical date representation. Users define their own calendar system separately (e.g. "Age of Ash, Year 312" = integer 10312). A calendar configuration object maps integer ranges to human-readable labels.

This means the timeline works for any world — sci-fi, fantasy, historical alternate — without assumptions about Earth-like calendars.

### The Master Timeline

A scrollable, zoomable horizontal timeline showing all events. Features:
- Zoom levels: broad era view → century → decade → year → month
- Events displayed as cards on the timeline, color-coded by type and involved node category
- Clicking an event opens the event node page
- A **timeline cursor** (a vertical scrubber) can be moved to any date; all time-responsive views and node field values update to reflect the world at that moment
- Events can be tagged with involved nodes; those nodes' cards appear below/above the timeline track in swim-lane style

### Per-Node Timeline Sub-Page

Every node has a `/timeline` sub-page showing only the events that involve that node, in chronological order. This is the "individual timeline" — a filtered view of the master timeline, auto-populated from state diffs and event tags.

For a Character, this reads like a biography of life events. For a Faction, it reads like a political history. For a Place, it reads like a settlement record.

### State Diff Model

When a state change occurs (recorded via an Event), the system stores a `state_diff` row:
- `event_id` — which event triggered the change
- `node_id` — which node changed
- `field_key` — which field changed
- `old_value` / `new_value` — the before/after values

At any timeline position T, a node's state is computed by:
1. Starting with the node's "origin state" (the fields as defined at node creation)
2. Replaying all `state_diffs` with `event.date_value ≤ T`, in chronological order
3. Returning the resulting field values

This is done in real time when the timeline cursor moves. For performance, snapshots are cached at major event boundaries.

### Time-Responsive Fields

Some fields are special — they are automatically computed from state diffs rather than stored statically:
- **Character age**: computed from `birth_date` event vs. current timeline cursor position
- **Faction membership**: drawn from current active `member_of` edges (edges can have start/end events)
- **Territory control**: drawn from current active `controls` edges

---

## 7. Specialized Views

All views are time-responsive — they render the world at the current timeline cursor position.

### Family Tree View
- **Trigger**: Character node → "Family Tree" tab, or a dedicated global view with a Character as root
- **Layout**: Top-down generational tree
- **Nodes**: Character cards showing name, portrait thumbnail, lifespan
- **Edges**: `parent_of`, `married_to`, `sibling_of`
- **Interactions**: Click to open character page; drag to reposition; button to add a new character as a relation

### Relationship Graph
- **Trigger**: Global view, filterable by node type(s)
- **Layout**: Force-directed graph (D3 or React Flow)
- **Nodes**: Any node type, colored by type
- **Edges**: All typed relationships, with edge labels
- **Filter panel**: Show/hide edge types, show only nodes within N degrees of a selected node
- **Use case**: Visualize the web of political alliances, personal relationships, power structures

### Faction Hierarchy Tree
- **Trigger**: Faction node → "Hierarchy" tab
- **Layout**: Top-down org-chart tree
- **Nodes**: Character cards (members) arranged by role tier
- **Interactions**: Drag to reorder; right-click to generate new member via LLM

### Spatial Hierarchy Navigator
- **Trigger**: Place node → "Spatial View" tab, or global "World Map" view
- **Layout**: Collapsible nested tree (Plane → Solar System → Planet → Continent → Region → Country → City → Landmark)
- **Interactions**: Click to drill down; breadcrumb trail shows current position in hierarchy
- **Note**: This is a *logical* spatial hierarchy, not a visual map (no lat/lng). A visual map editor is a future consideration.

### Spheres of Influence View
- **Trigger**: Global view, or Faction node → "Territories" tab
- **Layout**: Faction cards on the left, Place hierarchy on the right, lines indicating `controls` edges
- **Filter**: Show/hide factions, show only contested territories
- **Time-responsive**: As timeline cursor moves, control lines appear/disappear based on when factions gained/lost territory

### Species Taxonomy Graph
- **Trigger**: Species node → "Taxonomy" tab, or global Species view
- **Layout**: Collapsible tree, root at top (most ancestral species)
- **Nodes**: Species cards with thumbnail
- **Edges**: `descended_from`

### Tech Tree View
- **Trigger**: Global Technology view
- **Layout**: Left-to-right DAG (directed acyclic graph)
- **Nodes**: Technology cards, color-coded by category (e.g. military, arcane, industrial)
- **Edges**: `enables` (prerequisite arrows)
- **Time-responsive**: Technologies available at the current timeline position are highlighted; future/lost technologies are grayed

---

## 8. LLM Generation System

### API Key Management

Users configure API keys in a Settings page. Keys are stored in a local `.env` file or encrypted local config — never in the database. 

OpenRouter is supported to allow for using multiple models and swapping between them for easy iteration.

### Generation Templates

Templates are stored in the `generation_templates` table and are fully editable by the user. Each template defines:

- A **system prompt** (sets the LLM's role and constraints)
- A **user template** (handlebars-style, with `{{field}}` placeholders for linked node descriptions, user-supplied constraints, world context)
- An **output schema** (the JSON structure the LLM is expected to return, which maps to node fields)

This makes the generation system a data-driven pipeline: swapping or tuning templates doesn't require code changes.

### Single Node Generation Flow

1. User opens a new blank node page (e.g. new Character)
2. Fills in desired constraints: species, gender, age, faction membership, family ties, free-text notes
3. Clicks **"Generate"** → a generation panel opens showing the template being used and a preview of the assembled prompt
4. On confirm, the LLM call is made; the response is parsed against the output schema
5. The result is presented in a **Preview Panel**: all generated fields shown, editable before committing
6. If the LLM suggested new related nodes (e.g. a sibling, a hometown), those appear as **Proposed Nodes** — user can accept/reject/edit each one
7. On final confirm, the node(s) are committed to the database

### Batch Generation Flows

**Faction Roster Population**
- User opens Faction Hierarchy view
- Selects "Generate Members" → specifies: count, role tier(s), level of detail (full character vs. simplified NPC)
- LLM generates N characters as a JSON array
- Preview shows all proposed characters; user can edit, delete, or re-roll individual entries
- On confirm, all characters are created and linked to the faction

**Territory Seeding**
- On a Place node (e.g. a Continent), user selects "Seed Sub-locations"
- Specifies: type (kingdoms/cities/settlements), count, and optional constraints
- LLM generates a structured list of places with basic descriptions
- Preview, edit, confirm flow as above

**Family Tree Extension**
- On a Character node → Family Tree view, select "Extend Tree"
- Specifies: direction (ancestors/descendants), generations (1 or 2)
- LLM generates characters with family relationships
- Preview shows the proposed tree extension overlaid on the existing tree

### Context Assembly

When the LLM is called, the system assembles context from linked nodes automatically:
- If generating a Character linked to a Species, the species description is injected
- If the Character is a member of a Faction, the faction's description and ideology are injected
- If parent characters are specified, their descriptions are injected
- World-level "lore context" (a global free-text field in Settings) is always prepended

The context assembly is transparent — the generation panel shows exactly what context is being sent, and users can collapse/expand sections or manually override.

---

## 9. Image System & Visual Catalogue

### Per-Image Metadata Schema

Every image stored in DreamEngine — regardless of node type — carries a structured metadata object:

```json
{
  "generation_model": "string (e.g. Pony Diffusion XL v6)",
  "loras": [
    { "name": "string", "weight": 0.8, "trigger_words": "string" }
  ],
  "prompt": {
    "style": "string",
    "subject": {
      "body_type": "string",
      "hair_style": "string",
      "hair_color": "string",
      "eye_color": "string",
      "skin": "string",
      "body_markings": "string",
      "outfit": "string",
      "accessories": "string",
      "supernatural_elements": "string",
      "misc": "string"
    },
    "scenery_background": "string",
    "artist_references": "string",
    "other": "string"
  },
  "negative_prompt": "string",
  "generation_params": {
    "sampler": "string",
    "steps": 30,
    "cfg_scale": 7,
    "seed": 12345678
  },
  "notes": "string (free-form notes about this specific image)"
}
```

For Place/Location images, the `subject` section is replaced with:
```json
"subject": {
  "architectural_style": "string",
  "atmosphere": "string",
  "time_of_day": "string",
  "weather": "string",
  "landscape_elements": "string",
  "foreground_elements": "string",
  "misc": "string"
}
```

### Image Gallery on Node Pages

Each node's image gallery shows:
- Thumbnail grid with labels
- Click to open full-size with metadata panel
- "Add Image" button: upload image + fill metadata form
- "Promote to Default" to set the node's primary portrait/thumbnail
- Images are stored in a local `/dreamengine-data/images/` directory; the DB stores only the filename

### The Visual Component Catalogue

A dedicated section of the app (accessible from the sidebar) for managing reusable prompt components. Organized as:

**For Character Images:**
- Outfits (top, bottom, full outfit, footwear, layering)
- Hairstyles
- Hair Colors
- Eye Colors
- Skin Tones & Textures
- Body Markings (tattoos, scars, birthmarks, fur patterns)
- Accessories (jewelry, weapons, bags, etc.)
- Supernatural Elements (wings, tails, horns, auras, etc.)
- Art Styles
- Artist References
- Background / Scenery
- LoRAs library (name, trigger words, typical weight range, example thumbnail)

**For Place Images:**
- Architectural Styles
- Atmosphere & Mood
- Lighting Conditions
- Landscape Elements
- Art Styles
- Artist References
- Background Presets

Each catalogue entry has:
- Label
- Prompt text (the exact words to insert)
- Thumbnail (an image example)
- Tags
- Notes

Catalogue entries can be created manually or extracted from an existing image's metadata ("Save this to catalogue" button on any image).

---

## 10. Character Crafting Pipeline

The Character Crafting system is a dedicated workflow for rapidly brewing and visually designing characters. It is separate from the standard node generation flow — it is faster, more iterative, and more opinionated.

### Phase 1: Concept Brewing

**Entry point**: Sidebar → "Character Crafting"

**Interface**: A brewing panel with:
- Optional seed inputs: genre/setting context, archetype notes (free text), existing nodes to draw from (node reference pickers for species, factions, relationships)
- Generate button → produces **3 concept cards** simultaneously

**Each concept card shows:**
- Name + one-line archetype summary
- 3-4 bullet points: core personality traits, role/background, a distinctive hook
- "Regenerate this one" button
- "Lock In" button

The LLM generates these as minimal sketches, not full character sheets. The goal is to quickly find a direction worth pursuing.

### Phase 2: Visual Design Iteration

Once a concept is locked in, the user enters the **Visual Design Screen**:

**Left panel**: The locked concept brief (read-only summary)

**Center panel**: A prompt assembly workspace
- Sections matching the image metadata schema (style, subject fields, background, artist references, negative prompt)
- Each section shows current text + a "Suggest" button
- A **"Build from Catalogue"** button opens a catalogue browser filtered by category; clicking a catalogue entry inserts its prompt text into the relevant field

**Right panel**: Suggestions from the LLM
- On load (or on "Refresh Suggestions"), the LLM reads the character concept and all currently filled prompt sections, then suggests:
  - 2-3 style options (with catalogue references where applicable)
  - 2-3 outfit combinations (pulling from catalogue)
  - 2-3 background/scenery options
  - Any additional prompt elements it thinks fit the character concept
- Clicking a suggestion inserts it into the relevant field

**Bottom bar**: 
- "Copy Full Prompt" — assembles all fields into a formatted, copy-ready prompt string
- "Copy Negative Prompt"
- "Save as Image Record" — creates a new image entry on the (optionally selected) character node with the current metadata pre-filled; does not require an actual image to be uploaded yet

### Phase 3: Character Commit (Optional)

After the concept is visually designed, the user can optionally hit "Create Character Page" to run the standard single-node generation flow pre-seeded with the concept brief. This bridges the Crafting pipeline into the main knowledge graph.

---

## 11. UI/UX Design Principles

### Layout

- **Three-panel layout**: Left sidebar (navigation + search), Center (main content area), Right (context panel — relationships, timeline mini-view, generation panel)
- The right panel is collapsible to give more space to the main content area
- Sidebar sections: Dashboard, All Nodes (filterable list), Timeline, Catalogue, Character Crafting, Settings

### Navigation

- Each node has a stable URL: `/nodes/{type}/{slug}`
- Sub-pages: `/nodes/{type}/{slug}/timeline`, `/nodes/{type}/{slug}/gallery`, `/nodes/{type}/{slug}/hierarchy`
- Browser history works normally (back/forward)
- A **global search** (Cmd/Ctrl+K) opens a command palette: fuzzy search nodes, jump to views, run quick actions

### Node Pages

- The top of every node page shows: name, type badge, current timeline position indicator, default illustration (if any)
- Tabs for: Overview (main fields), Relationships, Timeline, Gallery, (type-specific tabs like Hierarchy, Family Tree, etc.)
- Fields are **inline-editable** — click any field to edit it in place; changes auto-save
- A **"Generate / Assist"** button is always visible in the top action bar

### Timeline UI

- The global timeline renders as a horizontal scrollable track at the bottom of the screen (toggleable)
- The cursor is a draggable vertical line; current in-world date shown in a floating label
- Zoom in/out with scroll wheel or pinch
- Events render as color-coded dots; hover shows event name; click opens event detail

### Graph Views

- Pan and zoom on all graph views
- Click a node to highlight it and show its direct relationships; click again to open its page
- "Focus" button on any node zooms to center it and dims non-adjacent nodes
- Legend always visible showing edge type colors

### Generation UI

- Generation is always non-destructive until confirmed
- A persistent **"Proposals" tray** holds accepted-but-not-yet-committed generated nodes; user can review and bulk commit
- Generation progress shows a live streaming preview of the LLM output where supported

### Dark Theme

Default and only theme — appropriate for long creative sessions and consistent with creative tool aesthetics (think Obsidian, Notion dark mode, Fantasia Archive).

---

## 12. Tech Stack

### Frontend
| Concern | Choice | Reason |
|---|---|---|
| Framework | React + TypeScript | Ecosystem, component libraries, personal familiarity |
| Build tool | Vite | Fast HMR, simple config |
| Routing | React Router v6 | Standard, supports nested routes |
| Rich text editor | TipTap | JSON document model, extensible, good table/image support |
| Graph views | React Flow | Handles trees, DAGs, force graphs; good interaction model |
| Timeline | vis-timeline or custom | vis-timeline is mature for horizontal timelines |
| Styling | Tailwind CSS | Utility-first, fast iteration |
| State management | Zustand | Lightweight, no boilerplate |
| API client | TanStack Query | Caching, async state, background refetch |

### Backend
| Concern      | Choice                     | Reason                                                     |
| ------------ | -------------------------- | ---------------------------------------------------------- |
| Runtime      | Node.js                    | Same language as frontend, fast to iterate                 |
| Framework    | Fastify                    | Lightweight, fast, good TypeScript support                 |
| Database     | SQLite via better-sqlite3  | Local, zero config, excellent performance for personal use |
| ORM / Query  | Drizzle ORM                | TypeScript-native, close to SQL, SQLite support            |
| File storage | Local filesystem           | Simple; images live at `/dreamengine-data/images/`         |
| LLM clients  | Anthropic SDK + OpenAI SDK | Direct SDKs for each provider                              |

### Development
| Concern | Choice |
|---|---|
| Monorepo structure | `/frontend` + `/backend` + `/shared` (shared TypeScript types) |
| Package manager | pnpm |
| Dev launcher | `concurrently` to run both frontend and backend with one command |
| Linting | ESLint + Prettier |

---

## 13. Development Stages

Each stage is designed to be useful on its own before the next stage begins.

---

### Stage 1 — The Core Knowledge Graph
**Goal**: A fully functional structured wiki with time-aware relationships.
**Duration estimate**: 6–10 weeks

#### Milestones

**1.1 — Project Setup**
- Monorepo scaffold (frontend + backend + shared types)
- SQLite database setup with Drizzle ORM
- Core schema: `nodes`, `node_fields`, `edges`, `timeline_events`, `state_diffs`
- Basic Fastify API routes: CRUD for nodes, edges, events
- React app scaffold with routing and sidebar layout

**1.2 — Node Pages**
- Node creation flow: type selection → blank page with template fields
- Inline field editing with TipTap for rich text fields
- Auto-save on edit
- Node page URL routing and navigation
- All 9 node types with their field templates

**1.3 — Relationship System**
- Relationship panel on node pages (grouped by edge type)
- Add/remove relationship UI (node picker modal)
- Bidirectional edge display (if Character A is `parent_of` B, B shows `child_of` A)
- Edge metadata (role, notes)

**1.4 — Master Timeline**
- Timeline event CRUD
- Scrollable horizontal timeline view (vis-timeline)
- Tag events with involved nodes
- Timeline cursor / date scrubber (functional but not yet affecting node field values)

**1.5 — State Diff System**
- `state_diffs` linked to timeline events
- UI for recording a field change: "At this event, this field changed from X to Y"
- Timeline cursor now drives time-resolved field values across all node pages
- Per-node Timeline sub-page (filtered event list)

**1.6 — Image Attachments**
- Image upload per node
- Image metadata form (full schema)
- Image gallery tab on node pages
- Default portrait display at top of node page

**1.7 — Global Search**
- Cmd+K command palette: fuzzy search nodes by name, jump to views
- Filter by node type

**Stage 1 Deliverable**: A fully usable personal wiki for worldbuilding — structured, interconnected, time-aware. Comparable to Fantasia Archive or Campfire Write but with a working timeline state system.

---

### Stage 2 — Specialized Graph Views
**Goal**: Make the knowledge graph *visible* through domain-specific visual lenses.
**Duration estimate**: 5–8 weeks

**2.1 — Family Tree View**
- Hierarchical tree layout via React Flow
- Pulls `parent_of`, `married_to`, `sibling_of` edges
- Character cards with portrait thumbnails and lifespan
- Add-relation button from within tree

**2.2 — Faction Hierarchy View**
- Org-chart tree layout via React Flow
- Pulls members from `member_of` edges, grouped by role metadata
- Time-responsive (shows members active at current cursor position)

**2.3 — Spatial Hierarchy Navigator**
- Collapsible nested tree (custom component)
- Pulls `located_in` edges
- Breadcrumb navigation; drill-down to child places

**2.4 — Relationship Graph**
- Force-directed graph via React Flow or D3
- All node types as nodes, all edge types as edges
- Filter panel: toggle node types, edge types, degree of separation
- Time-responsive

**2.5 — Species Taxonomy Tree**
- Same layout as Faction Hierarchy
- Pulls `descended_from` edges
- Collapsible branches

**2.6 — Tech Tree View**
- Left-to-right DAG layout via React Flow
- Pulls `enables` edges
- Technology cards grouped by category; color-coded
- Time-responsive: visually distinguish available/unavailable techs at cursor position

**2.7 — Spheres of Influence View**
- Two-column layout: Factions left, Place hierarchy right
- Connection lines showing `controls` edges
- Hover a faction to highlight all its territories; hover a place to show its controller
- Time-responsive

**Stage 2 Deliverable**: All major structural views of the world available as interactive, time-responsive graph visualizations.

---

### Stage 3 — LLM Generation System
**Goal**: AI-assisted creation of nodes, with batch generation and cascading proposals.
**Duration estimate**: 6–10 weeks

**3.1 — API Key Configuration**
- Settings page: enter/manage API keys for Anthropic, OpenAI, custom endpoints
- Keys stored in local `.env` or encrypted local config
- Test connection button

**3.2 — Generation Template System**
- `generation_templates` table and CRUD UI in Settings
- Default templates for each node type (Character, Place, Faction, etc.)
- Template editor: system prompt, user template with `{{field}}` syntax, output schema definition
- Context assembly engine: pulls linked node descriptions into template slots

**3.3 — Single Node Generation**
- "Generate" button on blank/partial node pages
- Constraint form: fills what the user wants to pre-set
- Generation panel: shows assembled prompt (expandable, editable)
- Streaming LLM response preview
- Preview panel: editable generated fields + proposed linked nodes
- Confirm flow: commits node and optionally creates proposed linked nodes

**3.4 — Batch Generation Flows**
- Faction Roster: generate N members from faction page
- Territory Seeding: generate sub-locations from a place node
- Family Tree Extension: generate ancestors/descendants from character node
- All batch flows use the same preview → confirm pattern
- Individual entries can be re-rolled, edited, or removed before commit

**3.5 — Proposals Tray**
- Persistent tray (bottom of screen or right panel) for accepted-but-uncommitted generated nodes
- Bulk commit, bulk discard, or per-item review

**Stage 3 Deliverable**: LLM generation deeply integrated into the app, usable from node pages and all batch contexts. The world can be populated rapidly from seeds and constraints.

---

### Stage 4 — Visual Catalogue & Character Crafting Pipeline
**Goal**: A complete image prompt management system and rapid character ideation workflow.
**Duration estimate**: 6–9 weeks

**4.1 — Visual Component Catalogue UI**
- Catalogue section in sidebar
- Category/subcategory browser
- Catalogue entry CRUD: label, prompt text, thumbnail, tags, notes
- "Save to Catalogue" from any image's metadata panel

**4.2 — Enhanced Image Metadata UI**
- Full structured form for image metadata (all schema fields)
- Auto-complete for model names, LoRA names (from previously used values)
- "Copy Prompt" button assembles all fields into a formatted prompt string
- Link image metadata fields to catalogue entries (click a field value to browse catalogue)

**4.3 — Character Crafting: Concept Brewing**
- Character Crafting entry point in sidebar
- Seed input form (archetype notes, linked nodes)
- Generate 3 concept cards simultaneously
- Per-card: regenerate, lock in
- LLM template for minimal concept sketches

**4.4 — Character Crafting: Visual Design Screen**
- Three-panel layout (concept brief / prompt workspace / LLM suggestions)
- Prompt sections with individual "Suggest" buttons
- Catalogue browser integration per field
- LLM generates style, outfit, and background suggestions referencing the catalogue
- "Copy Full Prompt" and "Copy Negative Prompt" bottom bar

**4.5 — Character Commit Bridge**
- "Create Character Page" from Visual Design Screen
- Pre-seeds the standard generation flow with the concept brief
- Created image record pre-populated with the visual design metadata

**Stage 4 Deliverable**: A complete character ideation and visual prompt design pipeline, tightly integrated with the knowledge graph and the catalogue system.

---

### Stage 5 — Polish, Export & SillyTavern Integration
**Goal**: Quality-of-life improvements and export features for downstream use.
**Duration estimate**: 4–6 weeks

**5.1 — SillyTavern Character Card Export**
- Export a Character node as a SillyTavern-compatible character card (V2 format JSON/PNG)
- Field mapping: biography → description, personality → personality, dialogue examples → mes_example, etc.
- Option to include relationship context in the character card's world info

**5.2 — World Info / Lorebook Export**
- Export selected nodes (or the full world) as a SillyTavern world info JSON
- Each node becomes a lorebook entry with its description as content and its name + aliases as keys

**5.3 — Bulk Export / Backup**
- Export the full database as a structured JSON file
- Export images as a zip archive
- Import from backup

**5.4 — Dashboard & Statistics**
- Homepage dashboard: recently edited nodes, timeline at a glance, generation activity log, world statistics (node counts by type, most-connected nodes)

**5.5 — Node Duplication & Templating**
- Duplicate any node as a starting point for a similar one
- Save a node's field configuration as a reusable personal template

**Stage 5 Deliverable**: A polished, complete app ready for regular daily use, with direct integration into SillyTavern and a solid backup/export story.

---

## 14. Future Considerations

These are intentionally out of scope for the initial build but are worth keeping in mind architecturally:

- **Visual map editor**: An actual spatial map (draw continents, place city pins) rather than just the logical spatial hierarchy. Would require a canvas-based editor (Konva, Fabric.js, or Leaflet with custom tiles).
- **Multiple worlds**: Currently the app is designed for one world. A "worlds" abstraction would let users switch between separate knowledge graphs.
- **Plugin system**: Custom node types, custom edge types, and custom views defined by the user without code changes.
- **Game engine integration**: A read-only API that a Unity or Godot game can query to pull worldbuilding data at runtime (e.g. NPC descriptions, faction relationships) — a natural evolution for when the game dev work matures.
- **Local image generation**: Direct integration with a local ComfyUI or Automatic1111 API to go from assembled prompt to image without leaving the app.
- **Obsidian plugin**: Since Obsidian is popular for worldbuilding, a plugin that reads DreamEngine's SQLite database and renders basic node info as Obsidian notes.

---

*Document maintained by Alex. Last updated: 2026-05-29.*