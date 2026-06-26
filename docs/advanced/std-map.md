# Std Map

> 1 Q&A · source: AE plugin dev community Discord

### Why does my std::map of install keys and matchnames return wrong effect keys?

The map was declared as std::map<A_char, A_long> which stores only a single A_char character, not the full string. The map key type should be a string type (like std::string) or at least an array of A_char to store the full matchname. Using just A_char means all matchnames starting with the same character would overwrite each other.

*Tags: `aegp`, `install-key`, `matchname`, `std-map`, `string-handling`*

---
