# Roadmap

Features and improvements under consideration for STAMM. Items are listed roughly in order of priority but are subject to change.

---

## Under Consideration

### Routed Swaps
Split a single trade across multiple tiers to reduce slippage on large swaps. The caller provides (tier, amount) pairs computed off-chain; the contract validates and executes each sub-swap atomically.

### Configurable Fee Parameters
Make protocol fee split ratios (spill percentages, tier-retained/protocol) adjustable per pool or globally through the factory, rather than hardcoded constants.

### Enhanced Oracle
Extended oracle data (volume-weighted metrics, volatility indicators) for richer on-chain and off-chain price feeds.

### Hybrid Burn
Burn LP tokens and receive both assets in a user-specified ratio, controlled by a swap percentage parameter. At low percentages, behaves close to a proportional burn; at high percentages, behaves close to a single-sided burn.

---

*This roadmap reflects current thinking and is not a commitment. Items may be added, removed, or reordered based on development progress and community feedback.*
