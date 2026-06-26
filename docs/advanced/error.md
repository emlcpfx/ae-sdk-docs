# Error

> 1 Q&A · source: AE plugin dev community Discord

### What causes the 'Not able to acquire AEFX Suite' error in Premiere?

This error can occur during playback in Premiere when code attempts to acquire an AE-specific suite that is not available in the Premiere host. Check if you have a shared function that conditionally calls AEFX suites based on host detection (e.g., if app_id != 'PrMr' then call AEFX suite). The host detection condition might be returning the wrong result, causing Premiere to try acquiring AE-only suites. It could also be a bug with a Premiere suite itself.

*Tags: `aefx-suite`, `error`, `host-detection`, `premiere`, `suite-acquisition`*

---
