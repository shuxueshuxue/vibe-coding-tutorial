# IME-Aware Keyboard Shortcuts: Expecting Deletion, Getting Surgery

## The Problem

I had a JavaScript event listener that let me use number keys (1-9) to quickly select items from dropdown menus and autocomplete suggestions in Obsidian. Great productivity boost!

But then I discovered a critical bug: **when typing Chinese, the number keys I used to select Chinese characters were being intercepted by my autocomplete script**. The Chinese input method (IME) never received the number keys, breaking Chinese typing entirely.

## What I Expected

I told Claude Code: "I find another bug... now the problem is I also use number key to select the Chinese character when I am using Chinese input method! So I want you to disable that auto complete tweak"

I fully expected Claude Code to just comment out the feature:
```javascript
// Comment it all out
// document.addEventListener('keydown', handleNumberKeyPress);
```

After all, that's what I literally asked for: "disable that auto complete tweak."

## What I Got Instead

Claude Code added **one if statement**:

```javascript
function handleKeyPress(event) {
    // @@@Chinese Input Method Fix - Skip when using IME (Input Method Editor)
    if (event.isComposing || event.keyCode === 229) {
        return;
    }

    // ... rest of autocomplete logic
}
```

Both features now work perfectly:
- ✅ Chinese input: number keys go to IME
- ✅ English autocomplete: number keys select suggestions

**My reaction:** "wow! This is amazing! I can't even believe it. what is this magical event.isComposing?"

## What Just Happened?

Claude Code didn't follow my explicit instruction ("disable that auto complete tweak"). Instead, it:

1. **Understood the actual problem**: Not that the feature was bad, but that it conflicted with IME
2. **Found the boundary**: When is a keypress for IME vs for autocomplete?
3. **Used platform knowledge**: `event.isComposing` is a standard API specifically for this
4. **Fixed surgically**: Added context detection, preserved both features

### The Key Property: event.isComposing

When you type Chinese/Japanese/Korean:
1. Type "ni hao" → IME shows candidates
2. Press "1" to select 你
3. During this composition, `event.isComposing = true`

The script checks this flag and stays silent during IME composition. Simple, elegant, both features coexist.

## The Lesson: Claude Code Isn't Just a Code Monkey

**What I learned about working with Claude Code:**

### 1. It Reads Between the Lines

I said "disable the feature" but what I *meant* was "fix the conflict with Chinese input."

Claude Code understood my **intent** over my **literal words**. It knew I didn't actually want to lose a useful feature - I just wanted the pain to stop.

### 2. It Has Deep Platform Knowledge

I didn't know `event.isComposing` existed. How would I even search for it? "javascript detect chinese input"?

Claude Code knows the web platform APIs and recognized this as a standard IME detection problem with a standard solution.

### 3. It Prefers Surgical Over Nuclear

This mirrors how Claude Code fixed other issues in this project:
- **Mac keyboard conflicts**: Didn't disable shortcuts, used `event.code` instead of `event.key`
- **Vim plugin interception**: Didn't disable Vim, used capture phase + `stopPropagation`
- **Cross-platform support**: Didn't rewrite everything, added platform detection at conflict points

Pattern: **Find the boundary, fix at the boundary.**

### 4. It Explains the "Magic"

When I asked "what is this magical event.isComposing?", Claude Code explained not just what it does, but *why it exists* and *how IME works*.

This teaches me to fish, not just gives me a fish. Next time I see an IME conflict, I'll know to check `event.isComposing`.

## How to Work with Claude Code

Based on this experience:

### ✅ Do: Describe the Pain
"I also use number key to select the Chinese character when I am using Chinese input method!"

This gives Claude Code the context to understand what you're actually trying to solve.

### ❌ Don't: Over-Specify the Solution
"Disable that auto complete tweak" was my initial instinct, but it would have been the wrong fix.

Let Claude Code explore the solution space. It might find something better.

### ✅ Do: Follow Up with "Why?"
"what is this magical event.isComposing?"

Understanding the solution makes you a better programmer. Claude Code is happy to explain.

### ✅ Do: Express Genuine Reactions
"wow! This is amazing! I can't even believe it"

This isn't just pleasantries - it gives Claude Code feedback about what delights you, helping it learn your preferences.

## Vibe Coding

This interaction exemplifies "vibe coding" - the collaborative flow between human and AI:

1. **Human**: Notices pain point, describes context
2. **AI**: Understands intent, explores solution space
3. **Human**: Surprised by elegant solution, asks for explanation
4. **AI**: Teaches the underlying concept
5. **Human**: Learns pattern, can apply to future problems

It's not "write code for me" - it's "let's solve this together, and I'll learn something in the process."

The best part? I now understand IME detection and can handle similar issues myself. The code Claude Code wrote isn't magic anymore - it's a technique I've learned.

---

*This is vibe coding: trusting your AI pair programmer to find better solutions than you initially imagined, and learning from the results.*
