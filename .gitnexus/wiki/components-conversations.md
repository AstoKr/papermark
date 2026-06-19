# components — conversations

# Conversations Components

A thin re-export layer that provides access to enterprise-grade conversation UI components.

## Overview

This module acts as a facade, exposing two conversation-related components that are implemented in the Enterprise Edition (EE) package. The re-export pattern allows the public API to remain stable while the underlying implementation lives in the EE module, enabling feature gating and code splitting.

## Exports

### `ConversationListItem`

Renders a single conversation item in a list view.

**Source:** `@/ee/features/conversations/components/dashboard/conversation-list-item`

**Props:** Accepts all props defined by the EE implementation. See the EE component for detailed prop documentation.

### `ConversationMessage`

Displays a single message within a conversation thread.

**Source:** `@/ee/features/conversations/components/shared/conversation-message`

**Props:** Accepts all props defined by the EE implementation. See the EE component for detailed prop documentation.

## Architecture

```
┌─────────────────────────────────────────┐
│     components/conversations/index.tsx  │
│              (this module)              │
├───────────────┬─────────────────────────┤
│ Conversation  │  Conversation           │
│ ListItem      │  Message                │
└───────┬───────┴────────────┬────────────┘
        │                    │
        ▼                    ▼
┌───────────────────┐ ┌─────────────────────┐
│ @/ee/features/    │ │ @/ee/features/      │
│ conversations/    │ │ conversations/      │
│ components/      │ │ components/shared/  │
│ dashboard/       │ │ conversation-       │
│ conversation-    │ │ message.tsx         │
│ list-item.tsx    │ │                     │
└───────────────────┘ └─────────────────────┘
```

## Usage

Import components from the public path:

```tsx
import { ConversationListItem, ConversationMessage } from "@/components/conversations";

// In a conversation list
<ConversationListItem
  conversation={conversationData}
  onSelect={handleSelect}
/>

// In a message thread
<ConversationMessage
  message={messageData}
  isOwnMessage={true}
/>
```

## Relationship to EE Features

This module is part of the feature layering strategy:

- **Public API** — Components here form the stable contract consumed by non-EE code
- **EE Implementation** — The actual logic and UI live in `@/ee/features/conversations/`
- **Future migration** — If features become generally available, implementations can be moved here without changing consumers