# Plan: Webchat Unified Message Architecture

**Branch:** `feat/webchat-streaming-tools`  
**Date:** 2026-01-24  
**Status:** In Progress

## Problem Statement

The webchat UI currently has two separate states:
- `chatMessages` — loaded from history (the "past")
- Streaming state — built from real-time events (the "present")

This causes:
1. Two different code paths for rendering
2. Two different data structures
3. Visual changes/jumps during transitions
4. Error-prone reconciliation logic

## User Requirement

> "When I view the UI it should only have the concept of 'message'. There should never be a reason to reload the page. While streaming, it should update in real time to match exactly what it will be if I reloaded the page. There should be ONE 'message' type."

## Actual Session Format (from .jsonl files)

### User Message
```json
{"role": "user", "content": [{"type": "text", "text": "..."}], "timestamp": ...}
```

### Assistant Message with Tool Call
```json
{
  "role": "assistant",
  "content": [
    {"type": "thinking", "thinking": "...", "thinkingSignature": "..."},
    {"type": "text", "text": "Let me check..."},
    {"type": "toolCall", "id": "toolu_...", "name": "exec", "arguments": {...}}
  ],
  "api": "...", "provider": "...", "model": "...", "usage": {...}
}
```

### Tool Result (SEPARATE message, not embedded)
```json
{
  "role": "toolResult",
  "toolCallId": "toolu_...",
  "toolName": "exec",
  "content": [{"type": "text", "text": "output here"}],
  "details": {...},
  "isError": false,
  "timestamp": ...
}
```

## Current Streaming Events

- `delta` — text chunk
- `tool-start` — name, args, toolCallId
- `tool-end` — name, result

## Implementation Plan

### Phase 1: Streaming → Build Exact History Format

1. **On `delta` event:**
   - Find or create assistant message in `chatMessages`
   - Ensure content array has a `{ type: "text", text: "" }` entry
   - Append delta text to that entry

2. **On `tool-start` event:**
   - Add `{ type: "toolCall", id, name, arguments }` to current assistant message's content array

3. **On `tool-end` event:**
   - Create a NEW message with:
     - `role: "toolResult"`
     - `toolCallId: "..."`
     - `toolName: "..."`
     - `content: [{ type: "text", text: result }]`
   - Push this message to `chatMessages` (just like history would have it)

4. **On `final` event:**
   - Do nothing! The streaming already built the correct structure
   - No history reload needed

### Phase 2: Verify `extractToolCards` Compatibility

The renderer uses `extractToolCards()` to turn messages into tool cards. Need to verify it handles:
- `{ type: "toolCall" }` entries (it checks for `toolcall`, `tool_call`, `tooluse`, `tool_use`)
- `role: "toolResult"` messages (via `isToolResultMessage`)

May need to add `toolcall` to the list of recognized types.

### Phase 3: Remove Dual-State Logic

- Remove any concept of "streaming state" vs "history state"
- Remove history reload on final
- `chatMessages` is the single source of truth
- Streaming events modify it in place

## Files to Modify

1. `ui/src/ui/chat.ts` — rendering, `extractToolCards`
2. `ui/src/ui/app-gateway.ts` — event handlers (delta, tool-start, tool-end, final)
3. `ui/src/ui/chat-stream.ts` (if exists) — streaming state management

## Success Criteria

- [ ] Send a message with tool calls
- [ ] While streaming, UI shows tool cards with loading state
- [ ] After final, UI looks identical (no jump/transition)
- [ ] Refresh the page → history loads → UI looks identical
- [ ] No "streaming" vs "history" distinction visible to user

## Implementation Progress

### Completed (2026-01-24)

1. **Backend (server-chat.ts)**
   - Added `toolCallId` to tool event payload
   - Events now include `tool.id` for pairing

2. **Frontend (controllers/chat.ts)**
   - `tool-start`: Adds `{ type: "toolCall", id, name, arguments }` to assistant content
   - `tool-end`: Adds SEPARATE `{ role: "toolResult", toolCallId, toolName, content }` message
   - Matches exact history format

3. **View (views/chat.ts)**
   - Removed skip for toolResult messages
   - Removed toolMessages rendering (duplicate data path)
   - All messages flow through ONE array

4. **Tool cards (tool-cards.ts)**
   - Added recognition for `toolCall` type (history format)
   - Added `id` field to ToolCard type

### Remaining

- [ ] Manual testing with real tool calls
- [ ] Verify no visual differences between streaming and refresh
- [ ] Consider removing `chatToolMessages` state entirely (currently unused)

## Notes

- Keep tool results as separate messages (not embedded in assistant message)
- This matches the actual JSONL format Pi writes
- The `extractToolCards` function already knows how to pair them
