# Dependency

> 1 Q&A · source: AE plugin dev community Discord

### How can a renderer effect communicate its dependencies on other layers and effects to AE's caching system?

Dependencies must be explicitly registered through the AEGP API when your renderer effect enumerates and checks out layers. When you call layer checkout functions to read pixel data, masks, shapes, and transforms from dependent layers, and query properties of effects on those layers, AE tracks these accesses as dependencies. To ensure proper cache invalidation, you should checkout all relevant layer properties at render time—if any of those checked-out properties change, AE will invalidate the cached result and trigger a re-render. This allows the disk cache to function correctly by only using cached data when all dependencies are identical.

*Tags: `aegp`, `caching`, `compute-cache`, `dependency`, `layer-checkout`*

---
