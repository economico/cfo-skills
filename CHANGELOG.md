# Changelog

## 1.0.0 (2026-09-30)

A new skill set for Economico's current tool surface, where every write is a command from one
catalog.

- `economico`: one skill that connects Economico, models your business from your code, Stripe
  and email, records it, and proves the books with reports. It asks you its questions as
  choices with a recommendation, and replaces `setup-economico`, `company-setup`, `pricing`,
  `creating-contracts`, `invoicing` and `expense-tracking`.
- Slash commands for the requests you make most: `/add-customer`, `/send-invoice`,
  `/record-payment`, `/record-vendor-invoice`, `/record-receipts` and `/investor-update`.
- Removed: `payment-rails`, `financial-analyst`, `investor-reporting`, `cpa` and `forecasting`.
  They called tools the current Economico does not have. `v0.1.0` and the `go-era` branch keep
  them.
- One Claude Code plugin, `economico`, installs every skill.
