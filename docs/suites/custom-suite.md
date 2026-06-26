# Custom Suite

> 1 Q&A · source: AE plugin dev community Discord

### How can two AEGPs communicate with each other?

Custom suites are the way to enable inter-AEGP communication. The key is to ensure all AEGPs have loaded before attempting to use the suite, since AE loads AEGPs before effects. The first idle call is a good time to load and use the suite. Refer to the 'sweetie' sample in the SDK to see how to create a custom suite, and the 'checkout' sample (globalSetup() function) to see how to use a custom suite.

*Tags: `aegp`, `custom-suite`, `reference`*

---
