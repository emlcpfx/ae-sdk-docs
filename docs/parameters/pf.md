# Pf

> 1 Q&A · source: AE plugin dev community Discord

### Is it safe to use PF_WorldFlag_RESERVED1 to detect 32-bit depth in actual plugins?

Yes, it appears safe to use this approach. The technique was derived from SDK examples and community knowledge on the After Effects SDK mailing list, and has been validated through use in production plugins like AtomKraft for AE, where 32-bit float support was essential.

*Tags: `aegp`, `bit-depth`, `layer`, `pf`, `plugin-development`*

---
