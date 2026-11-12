---
name: economico-books
description: Record contracts, invoices, bills, payments and revenue recognition in Economico, then read statements, burn and runway from the same books.
---

# Economico books

Every tool named below comes from Economico's MCP server at
`https://economi.co/mcp`. If it is not connected, add it first (Hermes catalog
plugin `economico`, or any MCP host pointed at that URL with OAuth).

Lead with the finance function: agreements first, then invoices and bills, then
recognition and payments. Ask for the source entries behind every number.

## Rules

- Economico records what you tell it. It does not email invoices, move money,
  sign agreements or file taxes.
- Prefer exact amounts and dates (UTC days as YYYY-MM-DD).
- When estimating burn or runway, state the assumptions.

## Done when

- The books show the new activity and the agent can point at the entries.
- Any invoice or bill document is returned for the founder to send; Economico
  does not send it.
