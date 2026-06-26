# Copy Func

> 1 Q&A · source: AE plugin dev community Discord

### How does PF_Arbitrary_NEW_FUNC work, and when is it called?

PF_Arbitrary_NEW_FUNC is never called in post-CS6 AE. The way arb data works now: you allocate a default arb handle during ParamSetup. For every new instance, AE sends PF_Arbitrary_COPY_FUNC with your default arb handle. Your arb data won't be zero length because it copies from the default you set up. Follow the ColorGrid SDK example which uses both NEW_FUNC and ParamSetup for this.

*Tags: `arb-data`, `arbitrary-data`, `copy-func`, `new-func`, `param-setup`*

---
