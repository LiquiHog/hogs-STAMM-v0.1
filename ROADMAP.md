# Roadmap

Features and improvements under consideration for STAMM. Items are listed roughly in order of priority but are subject to change.

---

## Under Consideration

### Configurable Fee Parameters
Make protocol fee split ratios (spill percentages, tier-retained/protocol) adjustable per pool or globally through the factory, rather than hardcoded constants.

### Enhanced Oracle
Extended oracle data (volume-weighted metrics, volatility indicators) for richer on-chain and off-chain price feeds.

### Caller-Directed Routed Swaps
Allow callers to provide explicit (tier, amount) pairs computed off-chain, bypassing the on-chain routing table. This enables custom routing strategies and finer-grained control beyond the automatic 3-tier waterfall in `swap_smart`.

---

*This roadmap reflects current thinking and is not a commitment. Items may be added, removed, or reordered based on development progress and community feedback.*
