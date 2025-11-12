# Prompting Example: How to Get Deep Analysis from LLMs

This document demonstrates effective prompting technique for getting an LLM to do serious analytical work.

## The Prompt (Human)

> OK.
> Now let's design the data structure. (the relationship between deck and card, the reference, mutable and immutable)
> and imagine what a new voice management menu should look like
> notice we don't use localStorage, all in backend
> **and this is a serious math problem**
> **you do this by first proposing several potential arguable design choices (as QA), and classical references for such systems**

## Key Prompting Techniques

1. **Frame as "serious math problem"** - Signals need for rigor, not quick answers
2. **Request "arguable design choices"** - Forces exploration of trade-offs
3. **Ask for "classical references"** - Grounds analysis in proven systems
4. **Specify QA approach** - Structured exploration before conclusions

---

## The Response (LLM)

⏺ Excellent framing. Let me approach this as a data structure design problem with proper analysis.

🎯 Problem Statement

Design a system where:
- Decks are collections of voices (cards)
- Users can create/edit/delete decks and voices
- System provides default decks
- Backend is source of truth (not localStorage)
- Support for future features: sharing, versioning, i18n

---
📚 Classical References

1. **Magic: The Gathering / Hearthstone**
   - Model: Decks reference cards (cards exist independently)
   - Key insight: Players build decks from a shared card pool
   - Trade-off: Same card in multiple decks, but deck doesn't "own" the card

2. **Obsidian/Notion Databases**
   - Model: Many-to-many relationships via tags/relations
   - Key insight: Entities can belong to multiple collections
   - Trade-off: Flexible but complex to maintain consistency

3. **Unix Filesystem (inode model)**
   - Model: Directories contain references to inodes (files)
   - Key insight: Hard links allow same file in multiple directories
   - Trade-off: Deletion semantics (reference counting)

4. **Git Object Model**
   - Model: Immutable objects, mutable references (branches/tags)
   - Key insight: Separation of content from organization
   - Trade-off: Complexity in mutable layer, simplicity in immutable layer

5. **WordPress Taxonomies**
   - Model: Posts have many-to-many relationship with categories/tags
   - Key insight: Hierarchical categories + flat tags
   - Trade-off: Complex queries, but flexible organization

6. **Database Normalization (Codd's Rules)**
   - Model: Separate tables with foreign keys
   - Key insight: Minimize redundancy, maximize integrity
   - Trade-off: Joins required for queries

---
🤔 Design Questions & Trade-offs

### Q1: One-to-Many vs Many-to-Many?

**Option A: One-to-Many (Strict Hierarchy)**
```
Deck 1:1→* Voice
```
- Each voice belongs to exactly ONE deck
- Deleting deck deletes all its voices

Pros:
- Simple ownership semantics
- Easy deletion (cascade)
- Clear mental model

Cons:
- Can't have "Linguist" in both "Scholar" and "Creative" decks
- Duplication needed for multi-category voices

---
**Option B: Many-to-Many (Flexible Association)**
```
Deck *→* Voice (via junction table)
```
- Each voice can belong to multiple decks
- Voice exists independently of deck

Pros:
- Maximum flexibility
- No duplication
- Like tags/categories

Cons:
- Complex deletion semantics (what if voice is in 5 decks?)
- Ambiguous ownership (who can edit this voice?)
- Harder to reason about

---
**Option C: Hybrid (Primary + Secondary)**
```
Voice has ONE primary deck, MANY secondary decks
```
- Primary deck "owns" the voice
- Secondary decks "reference" the voice

Pros:
- Clear ownership (primary deck)
- Still allows sharing
- Deletion semantics clear

Cons:
- More complex model
- UX confusion (what's primary vs secondary?)

---
🎯 **Recommendation: Option A (One-to-Many) for MVP**

Rationale:
- Simplicity beats flexibility for v1
- Can always migrate to Many-to-Many later
- Users can duplicate voices if needed
- Clear mental model: "Deck contains voices"

---
### Q2: Reference vs Embedding?

**Option A: Reference (Store IDs)**
```json
// Deck document
{
  "id": "scholar_deck",
  "name": "学者卡组",
  "voiceIds": ["linguist", "painter", "physicist"]  // Just IDs
}

// Voice document (separate)
{
  "id": "linguist",
  "name": "语言学家",
  "deckId": "scholar_deck",
  "systemPrompt": "..."
}
```

Pros:
- Normalized (no duplication)
- Easy to update voice (change once, affects all references)
- Smaller deck documents

Cons:
- Requires joins to fetch full deck
- More database queries

---
**Option B: Embedding (Store Full Objects)**
```json
// Deck document (contains everything)
{
  "id": "scholar_deck",
  "name": "学者卡组",
  "voices": [
    {
      "id": "linguist",
      "name": "语言学家",
      "systemPrompt": "..."  // Full voice object embedded
    }
  ]
}
```

Pros:
- Single query to fetch deck with all voices
- Atomic updates (one transaction)
- Simpler read path

Cons:
- Duplication (if voice is conceptually shared)
- Larger documents
- Update propagation (if voice changes, update all decks)

---
**Option C: Hybrid (Reference + Cache)**
```json
// Deck document
{
  "id": "scholar_deck",
  "voiceIds": ["linguist"],
  "voicesCache": [  // Denormalized cache for fast reads
    { "id": "linguist", "name": "语言学家" }  // Subset of fields
  ]
}
```

Pros:
- Fast reads (no joins)
- Canonical data still normalized
- Cache can be rebuilt

Cons:
- Cache invalidation complexity
- More storage

---
🎯 **Recommendation: Option A (Reference) for backend, Option C (Hybrid) for frontend**

Rationale:
- Backend: Normalized for data integrity
- Frontend: Cache full objects after fetching
- Best of both worlds

---
### Q3: Mutable vs Immutable?

**Option A: Fully Mutable**
Users can edit/delete any deck or voice (including system defaults)

Pros:
- Maximum flexibility
- No special cases

Cons:
- User can break system defaults
- No way to "reset to default"
- Update propagation issues

---
**Option B: System Immutable, User Mutable**
System decks/voices are read-only
Users can only edit their custom creations

Pros:
- Safe defaults
- Can provide "reset" function
- Clear separation

Cons:
- User frustration ("why can't I edit this?")
- Still need copy mechanism ("duplicate and edit")

---
**Option C: Copy-on-Write**
When user edits system deck/voice, create a user copy
System originals remain unchanged

Pros:
- Never lose defaults
- User gets full control over their copy
- Like Git branches

Cons:
- Complexity in tracking "derived from"
- Storage duplication

---
🎯 **Recommendation: Option C (Copy-on-Write)**

Rationale:
- Git model is well-understood
- Users get flexibility without breaking system
- Can track lineage for updates

---
### Q4: Ownership & Permissions?

**Option A: Global Namespace (No Ownership)**
All decks/voices are shared across all users

Pros:
- Simple
- Encourages sharing

Cons:
- No privacy
- Naming conflicts

---
**Option B: User-Scoped (Per-User)**
Each user has their own decks/voices
System provides defaults on first login

Pros:
- Privacy
- No conflicts
- Users can experiment

Cons:
- No sharing
- Duplication across users

---
**Option C: Hybrid (System + User)**
System provides read-only defaults
Users can create private decks/voices
Optional: Users can share (public flag)

Pros:
- Best of both worlds
- Future-proof for social features

Cons:
- More complex query logic

---
🎯 **Recommendation: Option C (System + User scoped)**

Rationale:
- Most apps work this way (Notion, Obsidian)
- Scales to multi-user
- Enables future sharing features

---
💾 Proposed Data Model

### Database Schema

```sql
-- Decks table
CREATE TABLE decks (
  id TEXT PRIMARY KEY,           -- "scholar_deck"
  name TEXT NOT NULL,            -- "学者卡组"
  name_en TEXT,                  -- "Scholar Deck"
  description TEXT,
  icon TEXT,
  color TEXT,
  is_system BOOLEAN DEFAULT FALSE,  -- System deck (read-only)
  parent_id TEXT,                   -- If copied from system deck (copy-on-write)
  owner_id TEXT,                    -- NULL for system, user_id for user decks
  enabled BOOLEAN DEFAULT TRUE,
  order_index INTEGER,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,

  FOREIGN KEY (parent_id) REFERENCES decks(id),
  FOREIGN KEY (owner_id) REFERENCES users(id)
);

-- Voices table
CREATE TABLE voices (
  id TEXT PRIMARY KEY,           -- "linguist"
  deck_id TEXT NOT NULL,         -- "scholar_deck"
  name TEXT NOT NULL,            -- "语言学家"
  name_en TEXT,                  -- "Linguist"
  system_prompt TEXT NOT NULL,
  icon TEXT,
  color TEXT,
  is_system BOOLEAN DEFAULT FALSE,
  parent_id TEXT,                -- If copied from system voice
  owner_id TEXT,                 -- NULL for system, user_id for user voices
  enabled BOOLEAN DEFAULT TRUE,
  order_index INTEGER,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,

  FOREIGN KEY (deck_id) REFERENCES decks(id) ON DELETE CASCADE,
  FOREIGN KEY (parent_id) REFERENCES voices(id),
  FOREIGN KEY (owner_id) REFERENCES users(id)
);

-- Index for common queries
CREATE INDEX idx_voices_deck ON voices(deck_id);
CREATE INDEX idx_voices_owner ON voices(owner_id);
CREATE INDEX idx_decks_owner ON decks(owner_id);
```

---
### Backend API Design

```python
# GET /api/decks
# Returns all decks visible to user (system + user's own)
[
  {
    "id": "scholar_deck",
    "name": "学者卡组",
    "isSystem": True,
    "enabled": True,
    "voiceCount": 6
  }
]

# GET /api/decks/{deck_id}
# Returns full deck with all voices
{
  "id": "scholar_deck",
  "name": "学者卡组",
  "isSystem": True,
  "voices": [
    {
      "id": "linguist",
      "name": "语言学家",
      "systemPrompt": "...",
      "enabled": True
    }
  ]
}

# POST /api/decks/{deck_id}/fork
# Copy-on-write: Create user copy of system deck
{
  "id": "scholar_deck_user_copy_123",
  "parentId": "scholar_deck",
  "name": "学者卡组 (My Copy)",
  "isSystem": False,
  "ownerId": "user_123"
}

# PUT /api/decks/{deck_id}
# Update deck (only if user owns it)
{
  "name": "My Custom Deck",
  "enabled": False
}

# DELETE /api/decks/{deck_id}
# Delete deck (only if user owns it, cascades to voices)

# POST /api/voices
# Create new voice
{
  "deckId": "my_custom_deck",
  "name": "新角色",
  "systemPrompt": "..."
}
```

---
🎨 Voice Management UI Design

```
┌──────────────────────────────────────────────────────────────┐
│ Voice Management                                    [+ New Deck]│
├──────────────────────────────────────────────────────────────┤
│                                                                │
│ ┌────────────────────────────────────────────────────────┐   │
│ │ 📚 学者卡组 (Scholar Deck)               [System] [✓]    │   │
│ │ 6 voices · From academic perspectives                   │   │
│ │                                          [🍴 Fork] [▼]    │   │
│ └────────────────────────────────────────────────────────┘   │
│   ┌──────────────────────────────────────────────┐            │
│   │ 📖 语言学家 (Linguist)              [✓] [⋮]  │            │
│   │ Analyzes from linguistic perspective...      │            │
│   └──────────────────────────────────────────────┘            │
│   ┌──────────────────────────────────────────────┐            │
│   │ 🎨 画家 (Painter)                   [✓] [⋮]  │            │
│   │ Analyzes aesthetics and visual elements...   │            │
│   └──────────────────────────────────────────────┘            │
│   ... 4 more voices                                            │
│                                                                │
│ ┌────────────────────────────────────────────────────────┐   │
│ │ 🧘 内省卡组 (Introspection Deck)     [System] [✓]    │   │
│ │ 6 voices · Inner dialogue archetypes                   │   │
│ │                                          [🍴 Fork] [▼]    │   │
│ └────────────────────────────────────────────────────────┘   │
│   ... (collapsed)                                              │
│                                                                │
│ ┌────────────────────────────────────────────────────────┐   │
│ │ 🎯 我的测试卡组 (My Test Deck)       [Custom] [✓]    │   │
│ │ 2 voices · My custom voices                  (forked from │
│ │ 3 voices · Personal experiments              Scholar Deck) │
│ │                                      [Edit] [Delete] [▼]   │   │
│ └────────────────────────────────────────────────────────┘   │
│   ┌──────────────────────────────────────────────┐            │
│   │ 📖 语言学家 (复刻)                  [✓] [⋮]  │  ← Copy   │
│   │ [Modified] Analyzes from linguistic...      │            │
│   └──────────────────────────────────────────────┘            │
│   ┌──────────────────────────────────────────────┐            │
│   │ 🤖 我的机器人                       [✓] [⋮]  │  ← Custom │
│   │ My custom voice...                           │            │
│   └──────────────────────────────────────────────┘            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### UI Features

1. **Deck Card:**
   - Badge: [System] or [Custom]
   - Fork button: Copy system deck to create editable version
   - Expand/collapse to show voices
   - Enable/disable toggle
   - Drag to reorder

2. **Voice Card:**
   - Badge: [Modified] if forked and changed
   - Three-dot menu: Edit / Duplicate / Delete / View History
   - Enable/disable toggle
   - Inline preview of system prompt

3. **Actions:**
   - Fork: Copy system deck → user deck (with lineage tracking)
   - Edit: Only available for user-owned items
   - Delete: Only available for user-owned items (with confirmation)
   - Reset: For forked items, revert to system version

4. **Visual Hierarchy:**
   - System decks have subtle badge
   - Custom decks have edit controls
   - Forked voices show lineage (e.g., "forked from Scholar Deck")

---
🏗️ Migration Strategy

**Phase 1: Backend Schema**
- Add decks and voices tables
- Seed with system defaults
- Add API endpoints

**Phase 2: Frontend Integration**
- Replace localStorage with API calls
- Update VoiceSettings component
- Add deck management UI

**Phase 3: Copy-on-Write**
- Add fork functionality
- Track lineage
- Add reset/update features

---
📊 Final Recommendation

| Aspect         | Choice                 | Rationale                         |
|----------------|------------------------|-----------------------------------|
| Relationship   | One-to-Many            | Simpler for MVP, can extend later   |
| Storage        | Reference (normalized) | Data integrity, easier updates      |
| Mutability     | Copy-on-Write          | Git-like, safe defaults             |
| Ownership      | System + User          | Scalable, privacy-friendly          |
| Frontend Cache | Denormalized           | Fast reads, rebuild on invalidation |

---

## Why This Prompting Works

1. **Forces systematic thinking** - LLM can't give quick/shallow answer
2. **Leverages pattern matching** - References to known systems ground the analysis
3. **Explores trade-offs** - QA format ensures pros/cons are weighed
4. **Provides decision framework** - Multiple options lead to justified recommendations
5. **Signals seriousness** - "Math problem" framing activates analytical mode

## When to Use This Approach

- Architectural decisions with multiple valid approaches
- Design problems requiring deep domain knowledge
- Situations where you want to see the reasoning, not just the answer
- Teaching the LLM to think through a problem space before concluding
