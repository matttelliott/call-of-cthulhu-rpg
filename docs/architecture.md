# Call of Cthulhu Character Sheet - System Architecture

## 1. Component Structure

### 1.1 HTML Structure
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <!-- Meta, title, inline styles -->
</head>
<body>
  <div id="app">
    <header class="app-header">
      <h1>Call of Cthulhu Character Sheet</h1>
      <nav class="tab-navigation">
        <button data-tab="basic">Basic Info</button>
        <button data-tab="intermediate">Skills & Combat</button>
        <button data-tab="advanced">Backstory & Notes</button>
      </nav>
    </header>

    <main class="app-main">
      <div id="basic-tab" class="tab-content">
        <!-- Basic Info Components -->
      </div>
      <div id="intermediate-tab" class="tab-content hidden">
        <!-- Skills & Combat Components -->
      </div>
      <div id="advanced-tab" class="tab-content hidden">
        <!-- Backstory & Notes Components -->
      </div>
    </main>

    <footer class="app-footer">
      <div class="auto-save-indicator">
        <span id="save-status">Saved</span>
      </div>
    </footer>
  </div>
</body>
</html>
```

### 1.2 JavaScript Module Structure
```
App
├── Core
│   ├── CharacterModel (data structure)
│   ├── StorageManager (persistence layer)
│   └── EventBus (state management)
├── UI
│   ├── TabController (tab switching)
│   ├── FormController (auto-save handling)
│   └── ValidationController (input validation)
└── Components
    ├── BasicInfoComponent
    ├── SkillsComponent
    └── BackstoryComponent
```

## 2. Data Models

### 2.1 Character Interface
```javascript
/**
 * Complete character data model
 * @typedef {Object} Character
 */
const CharacterModel = {
  // Metadata
  id: 'uuid-v4-string',
  created: 'ISO-8601-timestamp',
  modified: 'ISO-8601-timestamp',
  version: '1.0.0',

  // Basic Info (Tab 1)
  basic: {
    name: '',
    player: '',
    occupation: '',
    age: null,
    sex: '',
    residence: '',
    birthplace: '',

    // Characteristics (0-100)
    characteristics: {
      str: null,  // Strength
      con: null,  // Constitution
      siz: null,  // Size
      dex: null,  // Dexterity
      app: null,  // Appearance
      int: null,  // Intelligence
      pow: null,  // Power
      edu: null,  // Education
      luck: null  // Luck
    },

    // Derived Attributes (auto-calculated)
    derived: {
      hp: null,           // (CON + SIZ) / 10
      mp: null,           // POW / 5
      san: null,          // POW (starting)
      sanCurrent: null,   // Current sanity
      mov: null,          // Movement rate
      build: null,        // Build (from STR+SIZ)
      db: ''             // Damage bonus (from build)
    }
  },

  // Intermediate Info (Tab 2)
  intermediate: {
    // Skills (array of skill objects)
    skills: [
      {
        name: '',
        baseValue: 0,
        points: 0,
        total: 0,      // baseValue + points
        checked: false  // improvement checkbox
      }
    ],

    // Combat
    combat: {
      weapons: [
        {
          name: '',
          skill: '',
          damage: '',
          range: '',
          attacks: 1,
          ammo: '',
          malfunction: ''
        }
      ],
      armor: {
        type: '',
        value: 0
      }
    },

    // Assets & Gear
    possessions: {
      cash: 0,
      spendingLevel: '',
      assets: [],
      gear: []
    }
  },

  // Advanced Info (Tab 3)
  advanced: {
    // Backstory
    backstory: {
      personalDescription: '',
      ideology: '',
      significantPeople: '',
      meaningfulLocations: '',
      treasuredPossessions: '',
      traits: '',
      injuriesScars: '',
      phobiasManias: '',
      arcaneTomes: '',
      encountersStrange: ''
    },

    // Fellows Investigators
    fellows: [
      {
        name: '',
        playerName: '',
        occupation: ''
      }
    ],

    // Notes
    notes: ''
  }
};
```

### 2.2 Validation Rules
```javascript
const ValidationRules = {
  characteristics: {
    min: 0,
    max: 100,
    type: 'number'
  },
  age: {
    min: 15,
    max: 99,
    type: 'number'
  },
  sanity: {
    min: 0,
    max: 99,
    type: 'number'
  }
};
```

## 3. Storage Abstraction Layer

### 3.1 Storage Interface
```javascript
/**
 * Abstract storage interface for future flexibility
 * Current: LocalStorage
 * Future: IndexedDB, Cloud, etc.
 */
class ICharacterStorage {
  /**
   * Save character data
   * @param {Character} character - Character object
   * @returns {Promise<string>} - Character ID
   */
  async save(character) {
    throw new Error('Method not implemented');
  }

  /**
   * Load character by ID
   * @param {string} id - Character ID
   * @returns {Promise<Character|null>}
   */
  async load(id) {
    throw new Error('Method not implemented');
  }

  /**
   * List all character IDs and names
   * @returns {Promise<Array<{id: string, name: string, modified: string}>>}
   */
  async list() {
    throw new Error('Method not implemented');
  }

  /**
   * Delete character by ID
   * @param {string} id - Character ID
   * @returns {Promise<boolean>}
   */
  async delete(id) {
    throw new Error('Method not implemented');
  }

  /**
   * Export character to JSON
   * @param {string} id - Character ID
   * @returns {Promise<string>} - JSON string
   */
  async export(id) {
    throw new Error('Method not implemented');
  }

  /**
   * Import character from JSON
   * @param {string} json - JSON string
   * @returns {Promise<string>} - New character ID
   */
  async import(json) {
    throw new Error('Method not implemented');
  }
}
```

### 3.2 LocalStorage Implementation
```javascript
class LocalStorageAdapter extends ICharacterStorage {
  constructor() {
    super();
    this.storageKey = 'coc_characters';
    this.activeKey = 'coc_active_character';
  }

  async save(character) {
    // Ensure ID exists
    if (!character.id) {
      character.id = this._generateUUID();
      character.created = new Date().toISOString();
    }

    // Update modified timestamp
    character.modified = new Date().toISOString();

    // Get all characters
    const characters = await this._getAllCharacters();

    // Update or add character
    characters[character.id] = character;

    // Save to localStorage
    localStorage.setItem(this.storageKey, JSON.stringify(characters));

    // Set as active character
    localStorage.setItem(this.activeKey, character.id);

    return character.id;
  }

  async load(id) {
    const characters = await this._getAllCharacters();
    return characters[id] || null;
  }

  async list() {
    const characters = await this._getAllCharacters();
    return Object.values(characters).map(char => ({
      id: char.id,
      name: char.basic.name || 'Unnamed Character',
      modified: char.modified
    }));
  }

  async delete(id) {
    const characters = await this._getAllCharacters();
    if (characters[id]) {
      delete characters[id];
      localStorage.setItem(this.storageKey, JSON.stringify(characters));
      return true;
    }
    return false;
  }

  async export(id) {
    const character = await this.load(id);
    return JSON.stringify(character, null, 2);
  }

  async import(json) {
    const character = JSON.parse(json);
    character.id = this._generateUUID(); // New ID for imported character
    return await this.save(character);
  }

  async _getAllCharacters() {
    const data = localStorage.getItem(this.storageKey);
    return data ? JSON.parse(data) : {};
  }

  _generateUUID() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
      const r = Math.random() * 16 | 0;
      const v = c === 'x' ? r : (r & 0x3 | 0x8);
      return v.toString(16);
    });
  }
}
```

### 3.3 Storage Migration Path
```javascript
/**
 * Future storage adapters can be implemented:
 * - IndexedDBAdapter (for larger data, offline support)
 * - FirebaseAdapter (cloud sync)
 * - RESTAPIAdapter (server-side storage)
 *
 * Usage:
 * const storage = new LocalStorageAdapter();
 * // Later: const storage = new IndexedDBAdapter();
 */
```

## 4. UI Component Breakdown

### 4.1 Tab 1: Basic Info
```
BasicInfoComponent
├── PersonalInfo
│   ├── Name (text input)
│   ├── Player (text input)
│   ├── Occupation (text input)
│   ├── Age (number input, 15-99)
│   ├── Sex (text input)
│   ├── Residence (text input)
│   └── Birthplace (text input)
│
├── CharacteristicsGrid (3-column responsive)
│   ├── STR (0-100, number input)
│   ├── CON (0-100, number input)
│   ├── SIZ (0-100, number input)
│   ├── DEX (0-100, number input)
│   ├── APP (0-100, number input)
│   ├── INT (0-100, number input)
│   ├── POW (0-100, number input)
│   ├── EDU (0-100, number input)
│   └── LUCK (0-100, number input)
│
└── DerivedAttributes (auto-calculated, read-only)
    ├── Hit Points (HP)
    ├── Magic Points (MP)
    ├── Sanity (SAN)
    ├── Movement (MOV)
    ├── Build
    └── Damage Bonus (DB)
```

**Responsive Behavior:**
- 320px: 1-column stack
- 768px: 2-column characteristics
- 1024px: 3-column characteristics

### 4.2 Tab 2: Skills & Combat
```
SkillsComponent
├── SkillsList (scrollable, searchable)
│   ├── Skill Row (repeatable)
│   │   ├── Name (text)
│   │   ├── Base Value (number, read-only)
│   │   ├── Points Allocated (number input)
│   │   ├── Total (calculated, read-only)
│   │   └── Improvement Checkbox
│   └── Add Skill Button
│
├── CombatSection
│   ├── Weapons Table
│   │   ├── Weapon Row (repeatable)
│   │   │   ├── Name
│   │   │   ├── Skill %
│   │   │   ├── Damage
│   │   │   ├── Range
│   │   │   ├── Attacks/Round
│   │   │   ├── Ammo
│   │   │   └── Malfunction
│   │   └── Add Weapon Button
│   └── Armor
│       ├── Type (text)
│       └── Value (number)
│
└── PossessionsSection
    ├── Cash (number)
    ├── Spending Level (select)
    ├── Assets (textarea)
    └── Gear (textarea)
```

**Responsive Behavior:**
- 320px: Stacked cards, horizontal scroll for tables
- 768px: Side-by-side sections
- 1024px: Full table layout

### 4.3 Tab 3: Advanced/Backstory
```
BackstoryComponent
├── BackstoryFields (expandable textareas)
│   ├── Personal Description
│   ├── Ideology/Beliefs
│   ├── Significant People
│   ├── Meaningful Locations
│   ├── Treasured Possessions
│   ├── Traits
│   ├── Injuries/Scars
│   ├── Phobias/Manias
│   ├── Arcane Tomes
│   └── Encounters with Strange
│
├── FellowInvestigators
│   ├── Fellow Row (repeatable)
│   │   ├── Name
│   │   ├── Player Name
│   │   └── Occupation
│   └── Add Fellow Button
│
└── GeneralNotes (rich textarea)
```

**Responsive Behavior:**
- 320px: Full-width textareas, min-height 80px
- 768px: Two-column layout for shorter fields
- 1024px: Optimal reading width (max 800px)

## 5. State Management

### 5.1 Event-Driven Architecture
```javascript
/**
 * Event Bus for component communication
 */
class EventBus {
  constructor() {
    this.events = {};
  }

  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);
  }

  emit(event, data) {
    if (this.events[event]) {
      this.events[event].forEach(callback => callback(data));
    }
  }

  off(event, callback) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(cb => cb !== callback);
    }
  }
}
```

### 5.2 Auto-Save Strategy
```javascript
/**
 * Auto-save on blur with debouncing
 */
class AutoSaveController {
  constructor(storage, eventBus) {
    this.storage = storage;
    this.eventBus = eventBus;
    this.saveTimeout = null;
    this.saveDelay = 500; // 500ms debounce
  }

  /**
   * Schedule save after input blur
   */
  scheduleSave(character) {
    clearTimeout(this.saveTimeout);
    this.saveTimeout = setTimeout(() => {
      this.performSave(character);
    }, this.saveDelay);
  }

  /**
   * Immediate save (for critical operations)
   */
  async performSave(character) {
    this.eventBus.emit('save:start');
    try {
      await this.storage.save(character);
      this.eventBus.emit('save:success');
    } catch (error) {
      this.eventBus.emit('save:error', error);
    }
  }

  /**
   * Attach blur handlers to form inputs
   */
  attachHandlers(formElement, getCharacterDataFn) {
    formElement.addEventListener('blur', (e) => {
      if (e.target.matches('input, textarea, select')) {
        const character = getCharacterDataFn();
        this.scheduleSave(character);
      }
    }, true); // Use capture phase
  }
}
```

### 5.3 State Flow Diagram
```
User Input
    ↓
Input Validation
    ↓
Update Character Model (in-memory)
    ↓
Blur Event
    ↓
Debounced Auto-Save
    ↓
Storage Layer
    ↓
Update Save Indicator
    ↓
Emit Success/Error Event
```

## 6. Responsive Breakpoints

### 6.1 Breakpoint Definitions
```css
/* Mobile First - Base Styles (320px+) */
:root {
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;

  --font-size-sm: 14px;
  --font-size-base: 16px;
  --font-size-lg: 18px;
  --font-size-xl: 24px;

  --input-height: 44px; /* Touch-friendly minimum */
  --border-radius: 4px;
}

/* Tablet (768px+) */
@media (min-width: 768px) {
  :root {
    --spacing-lg: 32px;
    --spacing-xl: 48px;
    --font-size-base: 16px;
    --input-height: 40px;
  }
}

/* Desktop (1024px+) */
@media (min-width: 1024px) {
  :root {
    --spacing-lg: 40px;
    --spacing-xl: 64px;
    --font-size-base: 16px;
  }
}
```

### 6.2 Component-Specific Breakpoints
```css
/* Tab Navigation */
.tab-navigation {
  /* Mobile: Stacked */
  display: flex;
  flex-direction: column;
}

@media (min-width: 768px) {
  .tab-navigation {
    /* Tablet+: Horizontal */
    flex-direction: row;
  }
}

/* Characteristics Grid */
.characteristics-grid {
  /* Mobile: 1 column */
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--spacing-md);
}

@media (min-width: 768px) {
  .characteristics-grid {
    /* Tablet: 2 columns */
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1024px) {
  .characteristics-grid {
    /* Desktop: 3 columns */
    grid-template-columns: repeat(3, 1fr);
  }
}

/* Skills Table */
.skills-table {
  /* Mobile: Card layout */
  display: block;
}

@media (min-width: 768px) {
  .skills-table {
    /* Tablet+: Table layout */
    display: table;
    width: 100%;
  }
}
```

### 6.3 Typography Scale
```css
/* Mobile */
h1 { font-size: 24px; }
h2 { font-size: 20px; }
h3 { font-size: 18px; }
body { font-size: 16px; }

/* Tablet (768px+) */
@media (min-width: 768px) {
  h1 { font-size: 28px; }
  h2 { font-size: 22px; }
  h3 { font-size: 18px; }
}

/* Desktop (1024px+) */
@media (min-width: 1024px) {
  h1 { font-size: 32px; }
  h2 { font-size: 24px; }
  h3 { font-size: 20px; }
}
```

## 7. Touch Interaction Patterns

### 7.1 Touch-Friendly Input Sizing
```css
/* Minimum touch target: 44x44px (iOS HIG) */
input, button, select, textarea {
  min-height: 44px;
  min-width: 44px;
  padding: 12px 16px;
  font-size: 16px; /* Prevents zoom on iOS */
}

/* Number inputs with +/- buttons */
input[type="number"] {
  -webkit-appearance: none;
  -moz-appearance: textfield;
}
```

### 7.2 Touch Gestures
```javascript
/**
 * Touch gesture handling
 */
class TouchController {
  constructor() {
    this.startX = 0;
    this.startY = 0;
  }

  /**
   * Swipe between tabs
   */
  enableSwipeNavigation(tabContainer) {
    tabContainer.addEventListener('touchstart', (e) => {
      this.startX = e.touches[0].clientX;
      this.startY = e.touches[0].clientY;
    });

    tabContainer.addEventListener('touchend', (e) => {
      const endX = e.changedTouches[0].clientX;
      const endY = e.changedTouches[0].clientY;

      const deltaX = endX - this.startX;
      const deltaY = endY - this.startY;

      // Horizontal swipe (deltaX > deltaY)
      if (Math.abs(deltaX) > Math.abs(deltaY) && Math.abs(deltaX) > 50) {
        if (deltaX > 0) {
          this.swipeRight();
        } else {
          this.swipeLeft();
        }
      }
    });
  }

  swipeLeft() {
    // Next tab
    window.eventBus.emit('tab:next');
  }

  swipeRight() {
    // Previous tab
    window.eventBus.emit('tab:previous');
  }
}
```

### 7.3 Keyboard Optimization
```javascript
/**
 * Optimized keyboard types for mobile
 */
const InputTypes = {
  name: {
    type: 'text',
    autocomplete: 'name',
    autocapitalize: 'words'
  },
  number: {
    type: 'number',
    inputmode: 'numeric',
    pattern: '[0-9]*'
  },
  age: {
    type: 'number',
    inputmode: 'numeric',
    pattern: '[0-9]*',
    min: 15,
    max: 99
  },
  text: {
    type: 'text',
    autocorrect: 'on',
    spellcheck: 'true'
  },
  notes: {
    type: 'textarea',
    autocorrect: 'on',
    spellcheck: 'true',
    rows: 5
  }
};
```

### 7.4 Scroll Optimization
```css
/* Smooth scrolling */
html {
  scroll-behavior: smooth;
}

/* Momentum scrolling on iOS */
.scrollable {
  -webkit-overflow-scrolling: touch;
  overflow-y: auto;
}

/* Prevent overscroll bounce on body */
body {
  overscroll-behavior-y: none;
}

/* Tab content scroll */
.tab-content {
  height: calc(100vh - 120px); /* Header + footer */
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}
```

## 8. Architecture Decision Records (ADRs)

### ADR-001: Single-Page Application (SPA)
**Decision:** Build as single HTML file with inline CSS/JS
**Rationale:**
- Simplicity: No build process, easy deployment
- Performance: Single HTTP request
- Offline-capable: Can be saved locally
**Consequences:**
- Limited code organization
- Mitigated by: Clear module structure in JS

### ADR-002: LocalStorage for Initial Storage
**Decision:** Use LocalStorage with abstraction layer
**Rationale:**
- Zero setup required
- Works offline immediately
- Sufficient for character sheet data (~50KB/character)
**Migration Path:** ICharacterStorage interface enables future migration

### ADR-003: Auto-Save on Blur
**Decision:** Save on input blur with 500ms debounce
**Rationale:**
- Better UX than manual save
- Prevents data loss
- Debouncing reduces storage writes
**Consequences:** Requires clear save indicators

### ADR-004: Mobile-First Responsive Design
**Decision:** Design for 320px first, enhance upward
**Rationale:**
- Majority of RPG players use tablets/phones at table
- Touch interactions require larger targets
- Easier to enhance than to constrain
**Breakpoints:** 320px (mobile), 768px (tablet), 1024px (desktop)

### ADR-005: Vanilla JavaScript
**Decision:** No frameworks, pure JavaScript
**Rationale:**
- Single-file constraint
- Minimal dependencies
- Full control over behavior
- Smaller file size
**Consequences:** More manual DOM manipulation

## 9. Performance Considerations

### 9.1 Initial Load
- Target: < 100KB total file size
- Inline CSS/JS to eliminate additional requests
- Minify in production (manual or build step)

### 9.2 Runtime Performance
- Use event delegation for dynamic content
- Debounce auto-save operations
- Lazy-load skill lists if > 50 items
- Use CSS transforms for animations (GPU-accelerated)

### 9.3 Storage Limits
- LocalStorage: 5-10MB per domain (browser-dependent)
- Estimate: ~50KB per character = ~100 characters storage
- Monitor: Implement storage quota checking

## 10. Security Considerations

### 10.1 Data Validation
- Sanitize all text inputs (XSS prevention)
- Validate number ranges (characteristics 0-100)
- Type checking on deserialization

### 10.2 Storage Security
- LocalStorage is domain-scoped (isolated)
- No sensitive data (character sheets only)
- Export functionality uses client-side JSON generation

### 10.3 Future Considerations
- If adding cloud sync: HTTPS only, authentication required
- If adding images: Client-side image compression, size limits

## 11. Testing Strategy

### 11.1 Manual Testing Checklist
- [ ] Auto-save triggers on blur
- [ ] Data persists across page reloads
- [ ] Derived attributes calculate correctly
- [ ] Tabs switch properly
- [ ] Responsive layouts work at all breakpoints
- [ ] Touch gestures work on mobile
- [ ] Input validation prevents invalid data

### 11.2 Browser Compatibility
- Target: Modern evergreen browsers (Chrome, Firefox, Safari, Edge)
- Test on: iOS Safari, Android Chrome
- Fallback: LocalStorage unavailable warning

## 12. Deployment Strategy

### 12.1 Single File Deployment
```html
<!-- Complete app in single HTML file -->
index.html (contains CSS, JS, HTML)
```

### 12.2 Hosting Options
- Static hosting: GitHub Pages, Netlify, Vercel
- Local file: Can run as `file://` protocol
- Web server: Any HTTP server

### 12.3 Versioning
- Embed version in Character model
- Migration logic for future schema changes
- Backward compatibility for older saves

---

## Summary

This architecture provides:
✅ Mobile-first responsive design (320px → 1024px)
✅ Clear storage abstraction for future migration
✅ Auto-save with debouncing on blur events
✅ Vanilla JavaScript modular structure
✅ Touch-optimized interactions
✅ Three-tab organization (Basic/Intermediate/Advanced)
✅ Complete Call of Cthulhu character data model
✅ Performance and security considerations
✅ Clear migration paths for future enhancements

**Next Steps:**
1. Review and approve architecture
2. Create HTML structure (/src/index.html)
3. Implement storage layer (/src/storage.js)
4. Build UI components (/src/components.js)
5. Add styling (/src/styles.css)
6. Integration and testing
