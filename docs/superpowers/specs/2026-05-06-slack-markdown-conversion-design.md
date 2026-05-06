# Design: Slack-Specific Markdown Conversion

**Date:** 2026-05-06 **Issue:** CA-8 **Status:** Approved

## Problem

Agent responses are written in standard Markdown. The Slack bot sends them directly into Slack's
`mrkdwn` block fields without any conversion. Because Slack uses a different dialect (`mrkdwn`), raw
Markdown syntax renders as literal characters — e.g., `## Some Header` appears verbatim instead of
as bold text, and `**bold**` appears as two asterisks on each side instead of bold.

## Approach

Add `slackify-markdown` (an npm library) to the slack-bot package. Insert a single conversion call
in `buildCompletionBlocks()` before the agent's text content is truncated and placed into the
response block. All other message fields are left unchanged.

## Architecture

### New module: `src/utils/markdown.ts`

Exports one function:

```ts
export function markdownToMrkdwn(text: string): string;
```

This is a thin wrapper around `slackify-markdown`. Having a wrapper keeps the library an
implementation detail — if we ever need to swap or extend it, only this file changes.

### Change to `src/completion/blocks.ts`

In `buildCompletionBlocks()`, convert `response.textContent` before passing it to
`truncateForSlack()`:

```ts
// Before (current):
const text = truncateForSlack(response.textContent, TRUNCATE_LIMIT);

// After:
const text = truncateForSlack(markdownToMrkdwn(response.textContent), TRUNCATE_LIMIT);
```

Conversion happens before truncation so character counts are accurate for the final mrkdwn output.

### `getFallbackText()` — no change

This function feeds Slack's `text` field, which is used for notifications and accessibility
fallback. Slack expects plain text here, not mrkdwn. It is left as-is.

## Conversion Scope

`slackify-markdown` handles the full standard Markdown → mrkdwn translation:

| Markdown input       | mrkdwn output                    |
| -------------------- | -------------------------------- |
| `**bold**`           | `*bold*`                         |
| `# Heading`          | `*Heading*`                      |
| `[text](url)`        | `<url\|text>`                    |
| `` `inline code` ``  | `` `inline code` `` (unchanged)  |
| ` ```code block``` ` | ` ```code block``` ` (unchanged) |
| `- list item`        | `• list item`                    |
| `> blockquote`       | `> blockquote` (unchanged)       |

## Testing

### `src/utils/markdown.test.ts` (new)

Unit tests for `markdownToMrkdwn()` covering each conversion case: bold, headings, links, inline
code, code blocks, lists. This is the definitive test for conversion correctness.

### `src/completion/blocks.test.ts` (addition)

One smoke test confirming the conversion is wired into `buildCompletionBlocks()`. Passes Markdown
input (e.g., `**bold**`) in `response.textContent` and asserts the resulting `mrkdwn` block text
contains the converted form (`*bold*`), not the raw Markdown. Does not re-test all conversion cases.

## Files Changed

| File                                               | Change                                     |
| -------------------------------------------------- | ------------------------------------------ |
| `packages/slack-bot/package.json`                  | Add `slackify-markdown` dependency         |
| `packages/slack-bot/src/utils/markdown.ts`         | New — exports `markdownToMrkdwn()`         |
| `packages/slack-bot/src/utils/markdown.test.ts`    | New — unit tests for conversion cases      |
| `packages/slack-bot/src/completion/blocks.ts`      | Call `markdownToMrkdwn()` on `textContent` |
| `packages/slack-bot/src/completion/blocks.test.ts` | Add one wiring smoke test                  |

## Out of Scope

- Conversion in any package other than `slack-bot`
- Changes to how messages are sent to the control plane
- Modifying `getFallbackText()` or any non-agent-response message fields
