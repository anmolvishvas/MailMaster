# MailMaster — UI/UX Design Specification

**Document Version:** 1.0.0  
**Last Updated:** 2026-11-25  
**Frontend Stack:** Vue 3 (Composition API), TypeScript, Vite, Tailwind CSS, shadcn-vue, Pinia, TanStack Query  
**Status:** Baseline Design System Specification

---

## 1. Design Philosophy

MailMaster is engineered as a high-craft, keyboard-friendly **AI Productivity Workspace** tailored for high-volume email triage, verifiable knowledge retrieval, and safe human-in-the-loop composition.

Core UX principles:
- **High Information Density Without Clutter:** Show sender, subject, AI summary snippet, priority badge, category tag, and action-item count cleanly without overflowing cards or decorative chrome.
- **Clear Visual Hierarchy:** Distinguish human-authored email content, AI-generated summaries/drafts, and verified RAG citations using distinct surface tones, subtle badges, and crisp typographic contrast.
- **Fast Navigation:** Keyboard shortcuts (`j`/`k` thread traversal, `c` compose, `/` search, `Cmd/Ctrl + K` AI assistant focus, `e` archive/read) and instant optimistic updates via TanStack Query.
- **Instant State Recognition:** Unread state, sync health (`Syncing`, `Synced`, `Error`), LLM provider routing, and draft approval status (`Draft`, `Pending Approval`, `Approved`, `Sent`) must be unmistakable at a glance.
- **Minimal Cognitive Load:** Surface AI intelligence inline next to the conversation rather than forcing constant tab-switching.
- **Accessibility (WCAG 2.1 AA):** Full keyboard focus rings, semantic landmarks, `aria-live` status announcements for sync/stream updates, and `>= 4.5:1` contrast ratios on body text.

---

## 2. Visual Direction

- **Dark-Mode-First Interface:** Engineered around deep charcoal-navy surfaces (`#0B0F17`, `#111827`, `#172033`) to minimize eye strain during extended inbox triage. (A light theme token set may be added later, while MVP ships a cohesive dark-first palette.)
- **Subtle Indigo/Violet AI Accents:** Use `#6366F1` (Primary Indigo) for primary interactive actions and `#8B5CF6` (AI Violet) exclusively to denote AI-generated insights, citations, and Smart Compose surfaces.
- **Minimal Borders & Restrained Elevation:** Rely on `1px` hairline borders (`#1F2937` / `#293548`) and subtle surface luminance steps rather than heavy drop shadows.
- **Clear Semantic Colors:** Unambiguous color coding reserved strictly for priority states, sync status, and destructive/approval gates.

---

## 3. Design Tokens

### 3.1 Color Tokens

| Token | Hex Value | Usage |
|---|---|---|
| `--color-bg-canvas` | `#0B0F17` | Root application background |
| `--color-bg-sidebar` | `#0E1420` | Left navigation rail and secondary panels |
| `--color-surface-primary` | `#111827` | Main cards, thread list rows, email reading pane |
| `--color-surface-elevated` | `#172033` | Selected thread row, dropdowns, modals, Smart Compose drawer |
| `--color-surface-ai` | `#161B33` | AI summary callouts, assistant chat bubbles, citation popovers |
| `--color-surface-hover` | `#1C263B` | Interactive row/button hover state |
| `--color-primary` | `#6366F1` | Primary CTA buttons, active nav indicators, focus rings |
| `--color-primary-hover` | `#4F46E5` | Primary button hover |
| `--color-secondary-ai` | `#8B5CF6` | AI badges, Smart Compose trigger, LangGraph streaming indicator |
| `--color-secondary-ai-soft` | `#241E46` | Subtle fill for AI category/citation pills |
| `--color-text-primary` | `#F3F4F6` | Headings, unread subjects, primary body text |
| `--color-text-secondary` | `#9CA3AF` | Snippets, read subjects, secondary labels |
| `--color-text-muted` | `#6B7280` | Timestamps, token counts, placeholder text |
| `--color-border-subtle` | `#1F2937` | Hairline dividers between panes and list items |
| `--color-border-strong` | `#334155` | Input borders, focused cards, modal outlines |
| `--color-border-ai` | `#3B3475` | Accent border on AI summary & Smart Compose containers |
| `--color-success` | `#10B981` | Sent confirmation, connected Gmail status, positive sentiment |
| `--color-warning` | `#F59E0B` | Pending approval state, token rate-limit warnings, `high` priority |
| `--color-error` | `#EF4444` | Failed sync/send, validation errors, `urgent` priority |
| `--color-info` | `#38BDF8` | Syncing indicator, informational banners, `normal` category tags |

### 3.2 Priority State Tokens

| Priority | Badge Text | Background Fill | Border | Text / Dot Color |
|---|---|---|---|---|
| `urgent` | `#FCA5A5` | `#3B1219` | `#7F1D1D` | `#EF4444` |
| `high` | `#FCD34D` | `#36250B` | `#78350F` | `#F59E0B` |
| `normal` | `#93C5FD` | `#13253E` | `#1E3A8A` | `#3B82F6` |
| `low` | `#9CA3AF` | `#1A2230` | `#374151` | `#6B7280` |

### 3.3 Spacing, Radii & Elevation

- **Spacing Scale:** `4px` base unit (`4`, `8`, `12`, `16`, `20`, `24`, `32`, `48`).
- **Border Radii:** `6px` (badges/inputs), `8px` (buttons/list items), `12px` (cards/modals/AI panels), `9999px` (pills/avatars).
- **Elevation:** Flat by default; `0 8px 24px rgba(0, 0, 0, 0.45)` reserved strictly for modals, command palettes, and popovers.

---

## 4. Typography

- **Primary Sans Font:** `'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`
- **Monospace Font:** `'JetBrains Mono', 'Fira Code', monospace` (used for email headers, RFC Message-IDs, token/cost telemetry, MCP endpoint URLs)

| Scale Role | Size / Line Height | Weight | Tracking | Usage |
|---|---|---|---|---|
| **Display / Auth Title** | `24px / 32px` | `600` (SemiBold) | `-0.02em` | Login/Register page headers |
| **Page Heading (H1)** | `20px / 28px` | `600` | `-0.015em` | Inbox, AI Assistant, Settings headers |
| **Section / Thread Subject (H2)** | `16px / 24px` | `600` | `-0.01em` | Email thread subject in detail header |
| **Card / Panel Title (H3)** | `14px / 20px` | `600` | `0` | AI Summary card, Draft editor header |
| **Body Primary** | `14px / 22px` | `400` / `500` | `0` | Email reading body, AI chat messages |
| **Body Compact / List Row** | `13px / 18px` | `400` / `600` | `0` | Thread list subjects (`600` unread, `400` read) & snippets |
| **Metadata / Badges** | `11px / 16px` | `500` | `0.02em` | Timestamps, priority pills, category tags, citations |

---

## 5. Component Specifications

All 18 components map directly to implemented MVP features using `shadcn-vue` primitives (`Button`, `Badge`, `Card`, `Dialog`, `Sheet`, `Tabs`, `Skeleton`, `Toast`, `Tooltip`, `ScrollArea`).

1. **Application Sidebar (`AppSidebar.vue`):** Collapsible `240px` left rail (`64px` icon-only collapsed mode) containing workspace brand mark, primary **Compose** CTA, route navigation (`Inbox`, `Action Items`, `Drafts`, `AI Assistant`, `Settings`), connected Gmail account switcher with live sync dot, and user profile menu.
2. **Top Navigation (`TopNavBar.vue`):** Fixed `56px` header with global hybrid search input (`/` shortcut, toggling `Hybrid` / `Semantic` / `Keyword` mode), active Gmail sync trigger + last-synced timestamp, active LLM provider indicator badge (`OpenAI` / `Anthropic`), and quick toggle for the AI Assistant side drawer.
3. **Inbox List (`InboxThreadList.vue`):** Virtualized or cursor-paginated thread list (`340px–400px` split pane or full-width compact mode). Each row renders sender avatar/initials, unread blue dot (`#6366F1`), sender name, relative timestamp, bold/regular subject, 1-line AI summary or snippet, `PriorityBadge`, `CategoryTag`, and action-item count pill. Includes filter chips (`All`, `Unread`, `Urgent/High`, `Action Required`).
4. **Email Preview Panel (`EmailPreviewPane.vue`):** Split-pane preview displaying selected thread header, top collapsible **AI Thread Brief** card (summary + extracted action items with owner/due date), chronological message stack, and quick **Draft AI Reply** CTA.
5. **Priority Indicators (`PriorityBadge.vue`):** Compact pill (`11px`) with 6px colored dot + uppercase/title-case label mapped strictly to the priority tokens (`urgent`, `high`, `normal`, `low`).
6. **Category Labels (`CategoryLabel.vue`):** Subtle bordered tag (`work`, `engineering`, `finance`, `scheduling`, `support`, `newsletter`, `personal`) with neutral-cool surface tint to avoid competing visually with priority alerts.
7. **Email Thread View (`EmailThreadDetail.vue`):** Full-width conversation view supporting collapsing earlier messages, expanding headers (`From`, `To`, `Cc`, `Date`), rendering sanitized HTML inside an isolated sandboxed container (`DOMPurify` + safe styling), inline AI analysis per message, and sticky reply/Smart Compose trigger bar.
8. **AI Assistant Interface (`AIAssistantView.vue`):** Two-column layout with `chat_sessions` history sidebar (`260px`) and main conversational canvas. Supports pinning a session to all connected accounts or a specific `account_id`, suggested starter prompts (*"Summarize urgent emails from today"*, *"Find pending action items"*), and live SSE streaming indicator.
9. **Chat Message Component (`ChatMessageBubble.vue`):** User messages align right (`#172033`); assistant messages align left (`#161B33` with `#3B3475` border) rendering Markdown safely, inline interactive citation chips (`[1]`, `[2]`), intent badge (`RAG Search`, `Thread Summary`, `Direct`), and token/model footer tooltip.
10. **RAG Source Citation Component (`CitationCard.vue`):** Expandable citation strip beneath assistant messages. Clicking `[1]` highlights and opens the citation card showing email subject, sender, timestamp, quoted chunk excerpt, and a **View Thread** button navigating directly to `/inbox/threads/:threadId?emailId=:emailId`.
11. **Smart Compose Editor (`SmartComposeEditor.vue`):** Slide-over drawer or inline thread reply box with recipient chips (`To`, `Cc`), subject field, AI instruction bar (e.g., *"Politely decline and propose next Tuesday"*, tone selector `Professional` / `Concise` / `Friendly` / `Firm`, RAG grounding toggle), rich-text/plain-text body editor, and linked `source_citations` pills if RAG was used.
12. **Draft Approval Controls (`DraftApprovalBar.vue`):** Explicit multi-state footer on every draft:
    - In `draft` / `pending_approval`: displays **Save Draft**, **Discard**, and **Review & Approve** (`#6366F1`).
    - Clicking **Review & Approve** opens a concise confirmation summary (recipient emails, subject, account sending from, body preview) requiring explicit confirmation (`Approve & Send` or `Approve Only`).
    - Any edit to an `approved` draft triggers an immediate visual status reset badge (`Edited — Re-approval required`) matching backend state rules.
13. **Gmail Connection Interface (`GmailAccountsPanel.vue`):** Lists linked Gmail accounts (`email_address`, OAuth scope summary, sync status pill, `last_synced_at`, synced message count), **Connect Gmail Account** OAuth button, **Sync Now** button, and **Disconnect Account** confirmation dialog warning that account tokens and synced data will be purged.
14. **AI Provider Settings (`AIProviderSettings.vue`):** Allows choosing default LLM provider (`OpenAI` vs. `Anthropic`), preferred chat model, enabling/disabling automatic provider fallback, and viewing a clean `llm_usage_logs` summary table (total tokens, estimated USD cost, breakdown by task type).
15. **Loading States (`LoadingSkeletons.vue`):** Tailored skeleton placeholders matching exact row heights of `InboxThreadList`, `EmailThreadDetail`, and animated pulsing dots for SSE streaming / Celery background sync progress.
16. **Empty States (`EmptyState.vue`):** Actionable zero-state cards for: (a) No Gmail account connected -> **Connect Gmail** CTA; (b) Empty filter/search results -> **Clear Filters** CTA; (c) No chat history yet -> clickable starter queries.
17. **Error States (`ErrorBanner.vue`):** Inline recoverable error banners with human-readable message, `request_id` copy button for debugging, and **Retry** action (e.g., expired Gmail OAuth reconnect prompt or LLM timeout retry).
18. **Notifications (`ToastContainer.vue`):** Top-right stacked toasts (`4s` auto-dismiss for info/success, persistent dismissible for errors) announcing sync completion, draft approval/send confirmation, or MCP token generation.

---

## 6. Main Screens & Interaction Patterns

1. **Login & Registration (`/login`, `/register`):** Centered `420px` card over `#0B0F17` canvas; email + password validation via Zod/VeeValidate or reactive composables; clear error alert without revealing email enumeration on login; seamless link between sign-in and sign-up.
2. **Inbox Dashboard (`/inbox`):** Three-column workspace (`Sidebar` + `InboxThreadList` + `EmailPreviewPane`). Selecting a thread updates URL query/params cleanly, marks read status optimistically, and shows AI summary + action items at the top of the preview pane.
3. **Email Detail (`/inbox/threads/:threadId`):** Focused full conversation timeline with collapsible AI Thread Summary banner, per-message priority/intent metadata, and expandable **Smart Compose** reply pane anchored at the bottom.
4. **AI Assistant (`/assistant`):** Dedicated RAG chat workspace with session history list, account scope selector, SSE-streamed Markdown answers, and clickable `[n]` source citation cards linking back to exact emails.
5. **Smart Compose & Drafts (`/drafts` + inline drawer):** Dedicated drafts queue highlighting `pending_approval` drafts (including drafts created by external MCP clients!) alongside inline thread reply composition, enforcing the human approval modal before calling Gmail send.
6. **Settings & Gmail Integration (`/settings`):** Tabbed interface (`Gmail Accounts`, `AI Providers & Usage`, `MCP Access & Security`) displaying connected accounts, sync controls, LLM cost metrics, and scoped MCP bearer token copy/revocation controls.

---

## 7. Responsive Behavior

| Breakpoint | Width | Layout Behavior |
|---|---|---|
| **Desktop Wide** | `>= 1280px` | Full 3-pane Inbox (`240px` sidebar + `380px` thread list + fluid preview pane) and optional collapsible `360px` AI Assistant side drawer. |
| **Desktop / Small Laptop** | `1024px – 1279px` | Icon-collapsed `64px` sidebar + `340px` thread list + fluid preview pane; AI Assistant opens as overlay sheet when invoked from inbox. |
| **Tablet** | `768px – 1023px` | Collapsible drawer sidebar (`hamburger` toggle); 2-pane Inbox or list-to-detail navigation where selecting a thread transitions cleanly to detail view with Back button. |
| **Mobile** | `< 768px` | Single-pane stack (`Inbox List` -> `Thread Detail` -> `Smart Compose Sheet`), bottom or slide-out navigation drawer, `44px` minimum touch targets, compact priority/category badges. |

All UI components must correspond directly to implemented backend endpoints—no mock-only widgets, dead toggles, or placeholder dashboards.
