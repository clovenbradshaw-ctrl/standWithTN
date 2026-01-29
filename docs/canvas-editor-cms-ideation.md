# Canvas Editor & CMS Ideation

## Vision: Squarespace-Style Block Editor for Stand With TN

Transform the current inline editing system into a full canvas-based editor where users can visually manipulate blocks, combined with an intuitive folder-based CMS for content management.

---

## Part 1: Canvas Block Editor

### Core Experience Goals

1. **Direct Manipulation** - Drag blocks anywhere on the canvas
2. **Visual Resizing** - Grab handles to resize blocks
3. **Inline Editing** - Click to edit text without modal popups
4. **Grid Snapping** - Smart alignment guides and grid snap
5. **Layer Control** - Bring forward/send backward
6. **Responsive Breakpoints** - Design for desktop/tablet/mobile views

---

### Recommended Libraries

#### Option A: **GrapesJS** (Best for CMS-style page building)
```
https://grapesjs.com
```

**Why GrapesJS:**
- Purpose-built for page builders and CMS
- Drag-and-drop blocks with resize handles
- Built-in inline text editing
- Component/block system with traits (editable properties)
- Responsive design tools built-in
- Storage adapters (can use localStorage + API)
- Already used by: Mailchimp, Starter Story

**Integration Approach:**
```javascript
const editor = grapesjs.init({
  container: '#editor-canvas',
  storageManager: {
    type: 'local', // or custom for Xano API
    autosave: true,
    stepsBeforeSave: 1,
  },
  blockManager: {
    blocks: [
      { id: 'hero-block', label: 'Hero Section', content: '...' },
      { id: 'org-card', label: 'Organization Card', content: '...' },
      { id: 'campaign-block', label: 'Campaign', content: '...' },
    ]
  },
  panels: { defaults: [...] },
  deviceManager: {
    devices: [
      { name: 'Desktop', width: '' },
      { name: 'Tablet', width: '768px' },
      { name: 'Mobile', width: '375px' },
    ]
  }
});
```

**Pros:**
- Full-featured out of the box
- Rich plugin ecosystem
- Handles responsive design natively
- Clean API for custom blocks

**Cons:**
- 500KB+ bundle size
- Learning curve for customization
- May be overkill for simpler needs

---

#### Option B: **Craft.js** (Best for React integration)
```
https://craft.js.org
```

**Why Craft.js:**
- Lightweight (~50KB)
- React-based (if migrating to React)
- Headless - you control all UI
- Drag, drop, resize built-in
- Excellent for custom component systems

**Integration Approach:**
```jsx
import { Editor, Frame, Element } from '@craftjs/core';

<Editor resolver={{ Hero, OrgCard, Campaign }}>
  <Frame>
    <Element is={Container} canvas>
      <Hero title="Stand With Tennessee" />
      <OrgCard org={selectedOrg} />
    </Element>
  </Frame>
</Editor>
```

**Pros:**
- Very flexible and lightweight
- Great developer experience
- Full control over appearance
- Serializes to JSON (fits event-sourced model)

**Cons:**
- Requires React migration
- More assembly required
- Fewer pre-built features

---

#### Option C: **Editor.js + Custom Blocks** (Best for content-focused)
```
https://editorjs.io
```

**Why Editor.js:**
- Block-based content editor (like Notion)
- Plugin architecture for custom blocks
- Outputs clean JSON
- Lightweight core (~30KB)

**Integration Approach:**
```javascript
const editor = new EditorJS({
  holder: 'editor',
  tools: {
    header: Header,
    paragraph: Paragraph,
    orgCard: OrgCardTool,
    campaign: CampaignTool,
    image: ImageTool,
  },
  data: savedContent
});
```

**Pros:**
- Simple mental model
- Clean JSON output
- Easy to extend
- Good for text-heavy content

**Cons:**
- Vertical stacking only (no free canvas)
- Limited visual manipulation
- Less "page builder", more "document editor"

---

#### Option D: **Custom with interact.js + Sortable** (Most Control)
```
https://interactjs.io
https://sortablejs.github.io/Sortable/
```

**Why Custom:**
- interact.js: drag, resize, multi-touch gestures
- SortableJS: smooth reordering animations
- Full control over behavior
- Minimal footprint (~15KB each)

**Integration Approach:**
```javascript
import interact from 'interactjs';
import Sortable from 'sortablejs';

// Make blocks draggable and resizable
interact('.canvas-block')
  .draggable({
    inertia: true,
    modifiers: [
      interact.modifiers.snap({ targets: [gridSnap] }),
      interact.modifiers.restrict({ restriction: 'parent' })
    ],
    onmove: dragMoveListener
  })
  .resizable({
    edges: { left: true, right: true, bottom: true, top: true },
    modifiers: [
      interact.modifiers.aspectRatio({ ratio: 'preserve' })
    ]
  });

// Sortable for section reordering
Sortable.create(sectionsContainer, {
  animation: 150,
  ghostClass: 'sortable-ghost',
  onEnd: (evt) => recordReorder(evt.oldIndex, evt.newIndex)
});
```

**Pros:**
- Maximum flexibility
- Smallest bundle size
- Works with existing vanilla JS
- No framework dependency

**Cons:**
- More development work
- Need to build UI from scratch
- No built-in undo/redo

---

### Recommended Architecture for Canvas Editor

```
┌─────────────────────────────────────────────────────────────┐
│                        CANVAS EDITOR                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌───────────────────────────┐  ┌───────────┐ │
│  │ BLOCKS  │  │        CANVAS             │  │ INSPECTOR │ │
│  │ PANEL   │  │  ┌─────────────────────┐  │  │   PANEL   │ │
│  │         │  │  │     Hero Block      │  │  │           │ │
│  │ [Hero]  │  │  │  ↔ resize handles   │  │  │ Block     │ │
│  │ [Card]  │  │  └─────────────────────┘  │  │ Settings  │ │
│  │ [Grid]  │  │                           │  │           │ │
│  │ [Image] │  │  ┌──────┐  ┌──────────┐   │  │ - Title   │ │
│  │ [Text]  │  │  │ Card │  │ Campaign │   │  │ - Link    │ │
│  │ [Button]│  │  │      │  │          │   │  │ - Style   │ │
│  │ [Video] │  │  └──────┘  └──────────┘   │  │ - Spacing │ │
│  │         │  │                           │  │           │ │
│  └─────────┘  └───────────────────────────┘  └───────────┘ │
├─────────────────────────────────────────────────────────────┤
│  [Desktop] [Tablet] [Mobile]  |  Zoom: 100%  |  [Preview]  │
└─────────────────────────────────────────────────────────────┘
```

---

### Block Types for Stand With TN

```javascript
const BLOCK_TYPES = {
  // Layout Blocks
  'section': {
    name: 'Section Container',
    resizable: { width: true, height: true },
    accepts: ['*'] // Can contain any block
  },
  'grid': {
    name: 'Grid Layout',
    props: { columns: 3, gap: '1rem' },
    accepts: ['card', 'org-card', 'campaign-card']
  },
  'columns': {
    name: 'Columns',
    props: { layout: '1/2 + 1/2' }, // or '1/3 + 2/3', etc.
  },

  // Content Blocks
  'hero': {
    name: 'Hero Section',
    fields: ['title', 'subtitle', 'cta_text', 'cta_url', 'background_image']
  },
  'org-card': {
    name: 'Organization Card',
    linkedContent: 'organizations', // Pulls from CMS
    fields: ['name', 'description', 'status', 'services']
  },
  'campaign-card': {
    name: 'Campaign Card',
    linkedContent: 'campaigns',
    fields: ['title', 'platform', 'goal', 'url']
  },

  // Basic Blocks
  'text': { name: 'Text Block', inline_edit: true },
  'heading': { name: 'Heading', props: { level: 'h2' } },
  'image': { name: 'Image', props: { src: '', alt: '' } },
  'button': { name: 'Button', props: { text: '', url: '', style: 'primary' } },
  'accordion': { name: 'Accordion', accepts: ['text', 'list'] },
  'divider': { name: 'Divider' },
};
```

---

### Canvas Interaction Features

#### 1. Drag & Drop
```javascript
// Visual feedback during drag
onDragStart: (block) => {
  block.classList.add('dragging');
  showDropZones();
  showAlignmentGuides();
},

onDrag: (block, position) => {
  // Snap to grid
  const snapped = snapToGrid(position, GRID_SIZE);
  // Show alignment guides
  highlightAlignedElements(block, snapped);
},

onDrop: (block, target) => {
  // Record event for undo/redo
  db.insEvent('BLOCK_MOVE', {
    blockId: block.id,
    from: block.originalPosition,
    to: target.position
  });
}
```

#### 2. Resize Handles
```javascript
// 8-point resize (corners + midpoints)
const RESIZE_HANDLES = [
  'nw', 'n', 'ne',  // top row
  'w',       'e',   // middle
  'sw', 's', 'se'   // bottom row
];

// Aspect ratio lock (hold Shift)
// Symmetric resize (hold Alt)
// Grid snap during resize
```

#### 3. Smart Guides & Snapping
```javascript
const SNAP_FEATURES = {
  gridSnap: true,           // Snap to grid
  elementSnap: true,        // Snap to other elements
  marginSnap: true,         // Snap to consistent margins
  centerSnap: true,         // Snap to center lines
  showDistances: true,      // Show pixel distances
};
```

#### 4. Selection & Multi-Select
```javascript
// Click: Select single block
// Shift+Click: Add to selection
// Cmd/Ctrl+A: Select all
// Drag marquee: Select multiple
// Group selection: Ctrl+G
```

---

## Part 2: Folder-Based CMS Structure

### Vision: Content Types as Folders

```
┌─────────────────────────────────────────────────────────────┐
│  CONTENT MANAGER                              [+ New Type]  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  📁 Organizations (12)                          [⚙ Schema]  │
│     ├── 📄 TN Immigrant Rights Coalition                    │
│     ├── 📄 Worker's Dignity Project                         │
│     ├── 📄 TIRRC                                            │
│     └── 📄 + 9 more...                                      │
│                                                             │
│  📁 Campaigns (5)                               [⚙ Schema]  │
│     ├── 📄 Emergency Legal Defense Fund                     │
│     ├── 📄 Rapid Response Network                           │
│     └── 📄 + 3 more...                                      │
│                                                             │
│  📁 Resources (8)                               [⚙ Schema]  │
│     ├── 📄 Know Your Rights                                 │
│     └── 📄 + 7 more...                                      │
│                                                             │
│  📁 Team Members (0)                            [⚙ Schema]  │
│                                                             │
│  📁 Submissions (3 pending)                     [⚙ Schema]  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Schema Builder for Content Types

Each content type has a customizable schema:

```javascript
const ContentTypeSchema = {
  organizations: {
    name: 'Organizations',
    icon: '🏢',
    slug: 'organizations',
    fields: [
      {
        key: 'name',
        type: 'text',
        label: 'Organization Name',
        required: true,
        validation: { minLength: 2, maxLength: 200 }
      },
      {
        key: 'type',
        type: 'select',
        label: 'Type',
        options: ['nonprofit', 'mutual_aid', 'legal', 'religious', 'community'],
        default: 'nonprofit'
      },
      {
        key: 'description',
        type: 'richtext',
        label: 'Description',
        placeholder: 'Describe what this organization does...'
      },
      {
        key: 'services',
        type: 'relation',
        label: 'Services Offered',
        relatedTo: 'services',
        multiple: true
      },
      {
        key: 'region',
        type: 'relation',
        label: 'Region',
        relatedTo: 'regions',
        multiple: false
      },
      {
        key: 'contact_email',
        type: 'email',
        label: 'Contact Email'
      },
      {
        key: 'website',
        type: 'url',
        label: 'Website'
      },
      {
        key: 'logo',
        type: 'image',
        label: 'Logo'
      },
      {
        key: 'status',
        type: 'status',
        label: 'Status',
        states: ['draft', 'pending_review', 'verified', 'archived'],
        default: 'draft'
      },
      {
        key: 'featured',
        type: 'boolean',
        label: 'Featured on Homepage',
        default: false
      }
    ],

    // Display settings
    listView: {
      columns: ['name', 'type', 'status', 'updated_at'],
      sortDefault: '-updated_at',
      searchFields: ['name', 'description']
    },

    // Workflows
    workflows: {
      onSubmit: 'set_pending_review',
      onApprove: 'set_verified',
      onReject: 'set_draft'
    }
  }
};
```

---

### Field Types System

```javascript
const FIELD_TYPES = {
  // Basic Types
  text: {
    component: 'TextInput',
    validation: ['required', 'minLength', 'maxLength', 'pattern']
  },
  richtext: {
    component: 'RichTextEditor',
    toolbar: ['bold', 'italic', 'link', 'list', 'heading']
  },
  number: {
    component: 'NumberInput',
    validation: ['min', 'max', 'integer']
  },
  boolean: {
    component: 'Toggle',
  },

  // Choice Types
  select: {
    component: 'Select',
    options: [] // defined in field config
  },
  multiselect: {
    component: 'MultiSelect',
    options: []
  },
  status: {
    component: 'StatusSelect',
    showBadge: true,
    colors: { draft: 'gray', pending: 'yellow', verified: 'green', archived: 'red' }
  },

  // Media Types
  image: {
    component: 'ImageUpload',
    accept: ['image/jpeg', 'image/png', 'image/webp'],
    maxSize: '5MB',
    transforms: { thumbnail: '200x200', medium: '800x600' }
  },
  file: {
    component: 'FileUpload',
    accept: ['application/pdf', 'application/msword']
  },

  // Structured Types
  url: {
    component: 'UrlInput',
    validation: ['url'],
    showPreview: true
  },
  email: {
    component: 'EmailInput',
    validation: ['email']
  },
  phone: {
    component: 'PhoneInput',
    format: 'US'
  },

  // Relation Types
  relation: {
    component: 'RelationPicker',
    relatedTo: '', // content type slug
    multiple: false,
    displayField: 'name'
  },

  // Location Types
  address: {
    component: 'AddressInput',
    fields: ['street', 'city', 'state', 'zip', 'country']
  },
  geolocation: {
    component: 'MapPicker',
    showMap: true
  },

  // Date Types
  date: {
    component: 'DatePicker',
    format: 'YYYY-MM-DD'
  },
  datetime: {
    component: 'DateTimePicker',
    timezone: true
  },

  // Special Types
  json: {
    component: 'JsonEditor',
    schema: null // optional JSON schema
  },
  markdown: {
    component: 'MarkdownEditor',
    preview: true
  },
  code: {
    component: 'CodeEditor',
    language: 'javascript'
  }
};
```

---

### Content Editor View

When clicking on a content item:

```
┌─────────────────────────────────────────────────────────────┐
│  ← Back to Organizations       [Save Draft] [Publish] [···] │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────┐  ┌───────────────────────┐│
│  │ CONTENT                     │  │ SIDEBAR              ││
│  │                             │  │                       ││
│  │ Organization Name *         │  │ Status               ││
│  │ ┌─────────────────────────┐ │  │ [● Verified     ▾]   ││
│  │ │ TN Immigrant Rights Co..│ │  │                       ││
│  │ └─────────────────────────┘ │  │ Featured             ││
│  │                             │  │ [✓] Show on homepage ││
│  │ Type                        │  │                       ││
│  │ [Nonprofit            ▾]    │  │ ─────────────────────││
│  │                             │  │                       ││
│  │ Description                 │  │ Activity             ││
│  │ ┌─────────────────────────┐ │  │ Created: Jan 15      ││
│  │ │ B I U 🔗 • ─ H1 H2      │ │  │ Updated: Jan 28      ││
│  │ ├─────────────────────────┤ │  │ By: admin            ││
│  │ │ TIRRC advocates for the │ │  │                       ││
│  │ │ rights of immigrants... │ │  │ ─────────────────────││
│  │ │                         │ │  │                       ││
│  │ └─────────────────────────┘ │  │ Linked Content       ││
│  │                             │  │ • Used in 3 pages    ││
│  │ Services Offered            │  │ • 2 campaigns        ││
│  │ [Legal Aid] [Translation]   │  │                       ││
│  │ [Housing] [+ Add]           │  │ ─────────────────────││
│  │                             │  │                       ││
│  │ Region                      │  │ Danger Zone          ││
│  │ [Nashville Metro      ▾]    │  │ [Archive] [Delete]   ││
│  │                             │  │                       ││
│  │ Website                     │  │                       ││
│  │ ┌─────────────────────────┐ │  │                       ││
│  │ │ https://tnimmigrant.org │ │  │                       ││
│  │ └─────────────────────────┘ │  │                       ││
│  │                             │  │                       ││
│  └─────────────────────────────┘  └───────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Schema Editor UI

Allow admins to customize content type schemas:

```
┌─────────────────────────────────────────────────────────────┐
│  Schema: Organizations                    [Save] [Cancel]   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  General                                                    │
│  ───────────────────────────────────────────────────────   │
│  Name: [Organizations        ]  Slug: [organizations]       │
│  Icon: [🏢 ▾]  Description: [Non-profit and mutual aid...]  │
│                                                             │
│  Fields                                           [+ Add]   │
│  ───────────────────────────────────────────────────────   │
│  ⋮⋮ name          Text       Required    [Edit] [Delete]   │
│  ⋮⋮ type          Select     Required    [Edit] [Delete]   │
│  ⋮⋮ description   Rich Text  Optional    [Edit] [Delete]   │
│  ⋮⋮ services      Relation   Optional    [Edit] [Delete]   │
│  ⋮⋮ region        Relation   Optional    [Edit] [Delete]   │
│  ⋮⋮ website       URL        Optional    [Edit] [Delete]   │
│  ⋮⋮ status        Status     Required    [Edit] [Delete]   │
│  ⋮⋮ featured      Boolean    Optional    [Edit] [Delete]   │
│                                                             │
│  [+ Add Field]                                              │
│                                                             │
│  List View Settings                                         │
│  ───────────────────────────────────────────────────────   │
│  Columns: [name] [type] [status] [updated_at] [+ Add]       │
│  Default Sort: [Updated At ▾] [Descending]                  │
│  Search: [name, description]                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 3: Integration Architecture

### How Canvas Editor + CMS Work Together

```
┌─────────────────────────────────────────────────────────────┐
│                    UNIFIED EDITOR                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────┐ │
│   │   CANVAS     │ ←──→ │  CONTENT     │ ←──→ │  SCHEMA  │ │
│   │   EDITOR     │      │  MANAGER     │      │  BUILDER │ │
│   └──────────────┘      └──────────────┘      └──────────┘ │
│          │                     │                     │      │
│          ▼                     ▼                     ▼      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              EVENT-SOURCED DATA LAYER               │  │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │  │
│   │  │ Blocks  │  │ Content │  │ Schemas │             │  │
│   │  │ Events  │  │ Events  │  │ Events  │             │  │
│   │  └─────────┘  └─────────┘  └─────────┘             │  │
│   └─────────────────────────────────────────────────────┘  │
│                            │                                │
│                            ▼                                │
│   ┌─────────────────────────────────────────────────────┐  │
│   │              SYNC LAYER (Xano API)                  │  │
│   └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Linking Content to Canvas Blocks

```javascript
// When dropping an "Organization Card" block onto canvas
const linkedBlock = {
  id: 'block_abc123',
  type: 'org-card',
  contentLink: {
    contentType: 'organizations',
    contentId: 'org_xyz789',  // Specific org, or...
    query: {                   // Dynamic query
      filter: { status: 'verified', featured: true },
      sort: '-updated_at',
      limit: 6
    }
  },
  display: {
    layout: 'grid',
    columns: 3,
    showFields: ['name', 'description', 'services']
  },
  position: { x: 100, y: 400 },
  size: { width: 800, height: 'auto' }
};
```

---

## Part 4: Technical Implementation Plan

### Phase 1: Foundation (Week 1-2)
1. Add GrapesJS or interact.js to the project
2. Create base block components
3. Implement drag/drop on existing sections
4. Add resize handles to blocks

### Phase 2: Canvas Editor (Week 3-4)
1. Build editor layout (blocks panel, canvas, inspector)
2. Implement grid snapping and alignment guides
3. Add undo/redo system (leveraging existing event-sourcing)
4. Create responsive preview modes

### Phase 3: CMS Restructure (Week 5-6)
1. Create schema definition system
2. Build folder navigation UI
3. Implement dynamic form generation from schemas
4. Add relation field type with content linking

### Phase 4: Integration (Week 7-8)
1. Connect canvas blocks to CMS content
2. Implement content queries in blocks
3. Add real-time preview of content changes
4. Build publish/draft workflow

---

## Part 5: Library Comparison Matrix

| Feature | GrapesJS | Craft.js | Editor.js | interact.js |
|---------|----------|----------|-----------|-------------|
| Framework | Vanilla | React | Vanilla | Vanilla |
| Bundle Size | ~500KB | ~50KB | ~30KB | ~15KB |
| Free Canvas | ✅ | ✅ | ❌ | ✅ |
| Drag & Drop | ✅ | ✅ | ✅ | ✅ |
| Resize | ✅ | ✅ | ❌ | ✅ |
| Inline Edit | ✅ | ✅ | ✅ | Manual |
| Responsive | ✅ | Manual | ❌ | Manual |
| Undo/Redo | ✅ | ✅ | ✅ | Manual |
| Learning Curve | Medium | Medium | Low | Low |
| Customization | High | Very High | High | Very High |
| Best For | Page Builders | Custom Apps | Documents | Full Control |

---

## Recommendation

**For Stand With TN, I recommend: GrapesJS**

**Reasons:**
1. **No framework migration needed** - works with vanilla JS
2. **Full-featured** - drag, drop, resize, inline edit all included
3. **CMS-focused** - designed for exactly this use case
4. **Responsive built-in** - device preview already there
5. **Storage adapters** - easy to connect to Xano API
6. **Time to value** - faster to implement than custom solution

**Alternative: interact.js + custom UI** if:
- Bundle size is critical
- You want maximum control
- You prefer building from scratch
- The simpler approach is preferred

---

## Next Steps

1. **Confirm library choice** (GrapesJS vs custom)
2. **Prototype canvas editor** with existing content
3. **Design schema system** for content types
4. **Build folder CMS UI** incrementally
5. **Integrate and test** with real content
