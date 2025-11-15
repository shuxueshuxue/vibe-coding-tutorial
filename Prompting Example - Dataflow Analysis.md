# Prompting Example: Getting LLM to Analyze and Visualize Dataflow

This document demonstrates how to prompt an LLM to **learn and explain** (not just code), using ASCII diagrams to visualize complex workflows.

## The Context (User's Screenshot)

User shows a mobile app screenshot with:
- "编辑资料" (Edit Profile) header
- Default avatar with "点击更换头像" (Click to change avatar)
- A modal dialog showing "信息更新失败" (Information update failed)
- "保存修改" (Save Changes) button

## The Prompts (Human)

### Prompt 1: Question About Current Implementation

> one problem:
> so right now you are just feeding all the card decks into the context?
>
> could you like, add enabel/disabel mechanism?
>
> **before implementation, analyze the dataflow then tell me how does it work?**

**Key technique:** Request analysis BEFORE implementation

---

### Prompt 2: Clarifying Communication Pattern

> so after pressing toggle, send backend request of update the enable state
>
> then after backend's processing, it sends back the current decks state to frontend through the backend callback? (or polling?)

**Key technique:** Ask about architectural assumptions to force deeper explanation

---

## The Response (LLM)

### Response 1: Dataflow Analysis

⏺ Let me analyze the current dataflow for you:

⏺ Current Dataflow Analysis

Let me trace through how the enable/disable mechanism works (or doesn't work):

**✅ What's Already Working:**

1. **Backend Data Model:**
   - Both Deck and Voice have `enabled: boolean` field
   - Backend API supports updating via `updateDeck()` and `updateVoice()`

2. **Loading Filter (voiceApi.ts:568-614):**
   ```typescript
   export async function loadVoicesFromDecks() {
     const decks = await listDecks();

     for (const deck of decks) {
       if (!deck.enabled) continue;  // ✅ Skips disabled decks

       for (const voice of fullDeck.voices) {
         if (!voice.enabled) continue;  // ✅ Skips disabled voices
         voiceConfigs[voice.name] = {...};
       }
     }
   }
   ```

3. **Visual Indication (DeckManager.tsx:407-413):**
   ```typescript
   <div style={{
     border: `2px solid ${voice.enabled ? voiceColor : '#ddd'}`,
     opacity: voice.enabled ? 1 : 0.6  // Dimmed if disabled
   }}>
   ```

**❌ What's Missing:**

1. **NO Toggle UI:**
   - DeckManager shows enabled/disabled state visually
   - But there's no button/switch to toggle it

2. **NO Reload on Update (App.tsx:2314-2317):**
   ```typescript
   <DeckManager onUpdate={() => {
     console.log('Deck system updated, reloading...');  // ❌ Just logs!
   }} />
   ```

**The Complete Flow Should Be:**

```
┌─────────────────────────────────────────┐
│ DeckManager UI                          │
│                                         │
│ [Deck Name]     [Toggle ●────○]       │ ← User clicks toggle
│   Voice 1       [Toggle ●────○]       │
│   Voice 2       [Toggle ○────●]       │
└─────────────────────────────────────────┘
              ↓
    updateDeck({enabled: false})
    updateVoice({enabled: true})
              ↓
         onUpdate() callback
              ↓
┌─────────────────────────────────────────┐
│ App.tsx                                 │
│                                         │
│ onUpdate = async () => {                │
│   const newVoices =                     │
│     await loadVoicesFromDecks();       │
│   setVoiceConfigs(newVoices);          │
│   engineRef.current.setVoiceConfigs(); │
│ }                                       │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ loadVoicesFromDecks()                   │
│                                         │
│ - Fetches all decks                     │
│ - Skips deck if enabled=false          │
│ - For each enabled deck:               │
│   - Skips voice if enabled=false       │
│   - Adds to voiceConfigs               │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Effects on Writing View:                │
│                                         │
│ - AgentDropdown (@) shows only enabled │
│ - Voice analysis uses only enabled     │
│ - Chat widgets use only enabled        │
└─────────────────────────────────────────┘
```

**Two Levels of Control:**

**Deck-level toggle:**
- Enable/disable entire deck at once
- All voices in disabled deck are excluded

**Voice-level toggle:**
- Fine-grained control within an enabled deck
- User can cherry-pick specific voices

**Proposed UI:**

In DeckManager, add toggle switches:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [○] Philosophy Deck          [Toggle]  ← Deck-level
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  When expanded:

    [●] Logic                  [Toggle]  ← Voice-level
        "Analyzes arguments..."

    [●] Empathy                [Toggle]
        "Feels emotions..."

    [○] Drama                  [Toggle]  ← Disabled
        "Theatrical perspective..."
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Does this make sense? Should I implement:
1. Toggle switches in DeckManager UI?
2. Proper reload logic in App.tsx onUpdate callback?

---

### Response 2: Communication Pattern Explanation

⏺ Good question! Let me clarify the communication pattern used in this codebase:

**Communication Pattern: Request-Response (NOT Callback/Polling)**

The backend is a REST API, not WebSocket. Here's the actual flow:

```
User clicks toggle
       ↓
┌──────────────────────────────────────────┐
│ Frontend (DeckManager)                   │
│                                          │
│ handleToggleDeck(deckId) {               │
│   await updateDeck(deckId,               │
│     { enabled: !deck.enabled });  ←─────┼── 1. Send update request
│                                          │
│   await loadDecks();              ←─────┼── 2. Re-fetch fresh data
│   onUpdate();                     ←─────┼── 3. Notify parent
│ }                                        │
└──────────────────────────────────────────┘
       ↓                    ↑
       ↓                    ↑
    REQUEST              RESPONSE
       ↓                    ↑
       ↓                    ↑
┌──────────────────────────────────────────┐
│ Backend (server.py)                      │
│                                          │
│ PUT /api/decks/:id                       │
│   - Update database                      │
│   - Return updated deck                  │
└──────────────────────────────────────────┘
       ↓
┌──────────────────────────────────────────┐
│ App.tsx onUpdate callback                │
│                                          │
│ onUpdate = async () => {                 │
│   const voices =                         │
│     await loadVoicesFromDecks();  ←─────┼── Re-query backend
│   setVoiceConfigs(voices);               │
│   engineRef.current                      │
│     .setVoiceConfigs(voices);            │
│ }                                        │
└──────────────────────────────────────────┘
```

**Key Points:**

**No Polling:**
- Frontend doesn't continuously check for updates
- Only fetches when user takes action

**No Backend Callbacks:**
- Backend doesn't push updates to frontend
- Backend is stateless REST API

**Double Fetch Pattern:**
```typescript
// In DeckManager:
await updateDeck(id, { enabled: false });  // 1. Update
await loadDecks();                         // 2. Re-fetch to confirm

// In App.tsx onUpdate:
await loadVoicesFromDecks();               // 3. Re-fetch for engine
```

This might seem redundant, but it ensures:
- DeckManager UI shows latest state
- Writing view gets updated voice list
- No stale data

**Optimistic vs Pessimistic:**

**Current plan is pessimistic (wait for backend confirmation):**
```typescript
// Wait for backend, then update UI
await updateDeck(id, { enabled: false });
await loadDecks(); // UI updates after this
```

**We could do optimistic (update UI immediately):**
```typescript
// Update UI first
setDecks(prev => /* toggle locally */);

// Then sync with backend in background
updateDeck(id, { enabled: false })
  .catch(() => loadDecks()); // Rollback if failed
```

Which approach do you prefer?

---

## Why This Prompting Works

1. **Request analysis before action** - Forces LLM to understand the system first
2. **Ask clarifying questions** - Exposes LLM's assumptions (e.g., "callback or polling?")
3. **LLM uses ASCII diagrams naturally** - Visual explanation of complex flows
4. **Structured breakdown** - ✅ What works, ❌ What's missing
5. **Progressive detail** - High-level flow → Specific code → Trade-offs

## LLM's Visualization Techniques

1. **Box diagrams** - Show component boundaries and data flow
2. **Arrows and symbols** - `↓`, `←─────┼──`, `✅`, `❌` for visual cues
3. **Mock UI with ASCII** - `[Toggle ●────○]`, `━━━━━━` borders
4. **Sequence flows** - Top-to-bottom event chains
5. **Annotations** - `← User clicks toggle`, `← Deck-level`

## When to Use This Approach

- Understanding unfamiliar codebases (force LLM to trace through logic)
- Debugging complex state flows (visualize what's happening)
- Architectural discussions (clarify assumptions with questions)
- Teaching/documentation (ASCII diagrams are copy-pasteable to docs)
- Before implementing features (understand existing patterns first)

## Key Prompting Phrases

- "before implementation, **analyze the dataflow** then tell me how does it work?"
- "it sends back through **backend callback? (or polling?)**" (force clarification)
- "explain the **complete flow**" (get end-to-end picture)
- "what's **already working** vs **what's missing**?" (structured analysis)
