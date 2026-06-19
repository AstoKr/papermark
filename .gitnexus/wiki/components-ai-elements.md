# components — ai-elements

# AI Elements Components

The `ai-elements` module provides a composable, headless-style component library for building AI chat interfaces. It handles the full lifecycle of a conversation: message display, branching response navigation, user input with file attachments, and streaming text rendering.

## Overview

This module is organized into four component groups:

| File | Purpose |
|------|---------|
| `conversation.tsx` | Scrollable conversation container with auto-scroll and empty state |
| `message.tsx` | Message display, streaming responses, branching navigation, attachments |
| `prompt-input.tsx` | User input form with textarea, attachments, voice, menus |
| `shimmer.tsx` | Animated shimmer effect for loading states |

## Conversation Components

The conversation layer provides a sticky-bottom scrolling container that automatically follows new content, similar to a terminal or chat log.

### Conversation

```tsx
<Conversation className="h-[500px]">
  <ConversationContent>
    {/* messages */}
  </ConversationContent>
  <ConversationScrollButton />
</Conversation>
```

Uses `use-stick-to-bottom` under the hood. The container:
- Sticks to the bottom on initial load (`initial="smooth"`)
- Resizes smoothly as content changes
- Exposes `isAtBottom` and `scrollToBottom` via React Context

### ConversationScrollButton

A floating button that appears when the user scrolls away from the bottom. Clicking it smoothly scrolls back to the latest message.

## Message Components

### Message

```tsx
<Message from="user">
  <MessageContent>Hello</MessageContent>
  <MessageActions>
    <MessageAction tooltip="Copy">...</MessageAction>
  </MessageActions>
</Message>
```

The `from` prop determines alignment and styling:
- `user` — Right-aligned, secondary background
- `assistant` (or any other role) — Left-aligned

### MessageResponse

Wraps the `streamdown` library to render streaming AI responses. It's memoized to avoid re-renders as tokens arrive:

```tsx
<MessageResponse>
  {streamText}
</MessageResponse>
```

### Message Branching

When an AI response has multiple interpretation branches (e.g., different ways to continue a response), the `MessageBranch*` components handle navigation:

```tsx
<MessageBranch defaultBranch={0} onBranchChange={(i) => console.log(i)}>
  <MessageBranchContent>
    <div>Branch A content</div>
    <div>Branch B content</div>
  </MessageBranchContent>
  <MessageToolbar>
    <MessageBranchSelector from="assistant">
      <MessageBranchPrevious />
      <MessageBranchPage />
      <MessageBranchNext />
    </MessageBranchSelector>
  </MessageToolbar>
</MessageBranch>
```

The context provider tracks `currentBranch`, `totalBranches`, and navigation functions. Branch navigation wraps around (last → first, first → last).

### MessageAttachments

```tsx
<MessageAttachments>
  <MessageAttachment data={filePart} onRemove={() => {}} />
</MessageAttachments>
```

Renders file attachments. Images display as thumbnails; other files show a paperclip icon. The `onRemove` callback is optional.

## Prompt Input Components

### Architecture

The prompt input supports two modes:

1. **Self-managed mode** — `PromptInput` owns all state locally
2. **Provider mode** — `PromptInputProvider` lifts state upward for cross-component access

```tsx
// Self-managed
<PromptInput onSubmit={handleSubmit}>
  <PromptInputHeader>
    <PromptInputAttachments>
      {(file) => <PromptInputAttachment data={file} />}
    </PromptInputAttachments>
  </PromptInputHeader>
  <PromptInputBody>
    <PromptInputTextarea />
  </PromptInputBody>
  <PromptInputFooter>
    <PromptInputTools>...</PromptInputTools>
    <PromptInputSubmit />
  </PromptInputFooter>
</PromptInput>
```

```tsx
// Provider mode
<PromptInputProvider initialInput="...">
  <PromptInput onSubmit={handleSubmit}>...</PromptInput>
  {/* Other components can access via hooks */}
</PromptInputProvider>
```

### PromptInputProvider

Lifts `textInput` and `attachments` state to a context. Useful when child components (like action menus) need to manipulate the input without prop drilling.

### File Handling

Files are stored as `FileUIPart` objects with generated blob URLs. On submit, blob URLs are converted to data URLs before being passed to `onSubmit`:

```tsx
onSubmit({ text, files })  // files contain data URLs, not blob URLs
```

The `PromptInputAttachments` render prop receives each file and must render `PromptInputAttachment` components.

### Submit Behavior

- **Enter** submits (unless Shift+Enter or IME composition is active)
- **Backspace** on an empty textarea removes the last attachment
- **Paste** detects image files and adds them as attachments
- **Drag-and-drop** works on the form (or document if `globalDrop` is true)
- Files are validated against `accept`, `maxFiles`, and `maxFileSize` props
- Error callbacks receive structured error objects with `code` and `message`

### Speech Input

`PromptInputSpeechButton` wraps the Web Speech API. When activated:
- Listens continuously for speech
- Appends final transcripts to the textarea
- Calls `onTranscriptionChange` for external sync

```tsx
<PromptInputSpeechButton textareaRef={textareaRef} />
```

## Shimmer

```tsx
<Shimmer duration={2} spread={2}>
  Generating response...
</Shimmer>
```

Creates an animated shimmer effect over text using CSS `background-position` animation and `motion/react`. The `spread` prop scales with text length for consistent visual effect.

## Integration with `ai` SDK

These components work with the `ai` SDK's `UIMessage` type:

```tsx
import type { UIMessage } from "ai";

<Message from={message.role as UIMessage["role"]}>
```

The `FileUIPart` type from `ai` is used throughout for attachment data structures.

## Component Hierarchy

```
Conversation
├── ConversationContent
│   └── [messages]
│       ├── Message
│       │   ├── MessageContent
│       │   │   └── MessageResponse (streaming)
│       │   ├── MessageAttachments
│       │   │   └── MessageAttachment
│       │   ├── MessageActions
│       │   │   └── MessageAction (with optional Tooltip)
│       │   └── MessageToolbar
│       │       ├── MessageBranch
│       │       │   ├── MessageBranchContent
│       │       │   └── MessageBranchSelector
│       │       │       ├── MessageBranchPrevious
│       │       │       ├── MessageBranchPage
│       │       │       └── MessageBranchNext
│       │       └── [other toolbar items]
│       └── ConversationEmptyState
└── ConversationScrollButton

PromptInput (or PromptInputProvider > PromptInput)
├── PromptInputHeader
│   └── PromptInputAttachments (render prop)
│       └── PromptInputAttachment
│           └── PromptInputHoverCard
├── PromptInputBody
│   └── PromptInputTextarea
├── PromptInputFooter
│   ├── PromptInputTools
│   │   ├── PromptInputButton
│   │   ├── PromptInputActionMenu
│   │   └── PromptInputSpeechButton
│   └── PromptInputSubmit
└── PromptInputSelect (model selector)
```

## Key Patterns

**Context duality**: `usePromptInputAttachments` and `usePromptInputController` work in both provider and non-provider modes by checking provider context first, then falling back to local context.

**Branch navigation**: Wraps around cyclically. The `MessageBranchContext` tracks all branches and exposes navigation callbacks to child components.

**Blob URL lifecycle**: Files are created as blob URLs for preview. On submit, they're converted to data URLs. On removal or unmount, `URL.revokeObjectURL()` is called to prevent memory leaks.