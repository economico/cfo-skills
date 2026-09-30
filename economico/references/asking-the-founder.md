# Asking the founder

A founder answers a question fastest when it arrives as a choice: what is being decided, what
you recommend and why, and the options with yours first. Ask only for **decisions**. Facts are
yours to find: a price, a customer or a vendor that the code, Stripe or the mailbox can tell you
is never a question.

## Use the host's question tool

When the host offers a structured question tool, ask through it. In Claude Code it is
`AskUserQuestion`; some hosts expose a variant named `mcp__<server>__AskUserQuestion`, which works
the same way. One call carries up to four questions, and each question has:

| Field | What goes in it |
|---|---|
| `header` | A label of a word or two: `Business`, `Access`, `History`, `Recording`, `Pro plan` |
| `question` | One or two plain sentences: what is being decided, what changes with the answer, and your recommendation with its reason ("I recommend recording from today: your Stripe history starts in March and nothing earlier is billed") |
| `options` | Two to four choices. Yours first, its label ending in `(recommended)`; each option's `description` says what happens if they pick it |
| `multiSelect` | `true` only when several answers can hold at once (which sources you may read) |

The host adds an "Other" answer for free text, so do not add one. Group related questions into
one call rather than asking them one at a time; a round of questions is one interruption.

Options come from what you can see. Offer the businesses `businesses {action: "list"}` returned,
the integrations actually connected, the dates the sources actually cover. An option you cannot
act on is not an option.

## Without the tool

If the host has no question tool, or the call is refused or errors, write the same questions as
one message: per question, the decision in a sentence, `Recommendation: <option>, because
<reason>`, then the options as a short lettered list with yours first. Then stop and wait for the
reply.

If the founder already told you to proceed without review (or the prompt already answers the
question), do not stop: take the recommended option, note it under "Assumptions and gaps" in
`business-model.md`, and list every decision you took on their behalf in the final message, so
they can reverse any of them. Never take a recommendation on their behalf for something that
cannot be undone by a later command, such as writing to a real business instead of a sandbox.

## Where the first session asks

| When | Questions | Reference |
|---|---|---|
| Before discovery | Which business, what you may read, how far back, whether to record after review without asking again | [first session](first-session.md#0-agree-the-scope-one-round-of-questions) |
| At the review | The three to five judgments that change the numbers, including every disagreement between sources you could not settle ([reconcile](first-session.md#2-reconcile)) | [first session](first-session.md#3-propose-write-business-modelmd) |

Everything else runs without interruption between those two rounds.
