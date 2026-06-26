# Use the Minimum AEGP Suite Version That Provides the Functions You Need

> AE SDK suites are versioned and each version is "frozen" at the AE release where
> it was last changed. Requesting a version newer than the running AE ships causes
> `AcquireSuite` to fail and leave the pointer null -- a crash if you don't
> null-check, or a silent resource leak if the pointer was needed for cleanup.

## Symptom

Code built against a recent suite version is deployed to an older AE. On that
older host, `AcquireSuite` returns an error and the output suite pointer stays
null. If the suite was needed to *dispose* something acquired earlier in the same
path (for example an `AEGP_EffectRefH`), that object is never released -- a memory
leak on every render. If the code calls through the null pointer without checking,
it crashes instead.

## Root cause

Each AEGP suite has a version number, and a given version is frozen at the AE
release where it was last modified -- meaning that version (and only that
version) ships in that release and all later ones. Requesting a version newer
than the running AE provides fails the acquire and nulls the pointer.

Requesting an *older* version on a *newer* AE is always safe: suite families are
backward compatible.

## The fix

Downgrade the request to the lowest version that actually contains the function
you call. For example, if you only need `AEGP_DisposeEffect` (present since the
v1 Effect suite), do not request v5 just because it was the newest at build time:

```cpp
// BEFORE -- requires a recent AE; fails on older hosts
AEGP_EffectSuite5* effect_suite = nullptr;
pica_basicP->AcquireSuite(kAEGPEffectSuite, kAEGPEffectSuiteVersion5, ...);

// AFTER -- works on far older AE, because Dispose exists in v4
AEGP_EffectSuite4* effect_suite = nullptr;
pica_basicP->AcquireSuite(kAEGPEffectSuite, kAEGPEffectSuiteVersion4, ...);
```

## How to choose the version

**Rule: request the minimum suite version that contains the function(s) you
need.**

1. Before calling `AcquireSuite`, find the function in `AE_GeneralPlug.h` (or
   `AE_GeneralPlugOld.h` for older suites). Use the **lowest-numbered** suite
   version whose struct contains that function.

2. The `/* frozen AE X.Y */` comment on each `#define kAEGP...Version` tells you
   the minimum AE release that ships it. Roughly: AE 23.0 = 2023, 24.0 = 2024,
   25.0 = 2025, 26.0 = 2026. Suite versions frozen *after* the oldest AE you want
   to support will break on that AE.

3. **Always null-check the suite pointer after `AcquireSuite`**, even for suites
   you believe exist on the host -- it guards against unusual builds and future
   version changes.

4. When an acquire fails for a *cleanup* operation (dispose, release), log the
   leak in debug builds but do not crash. A missed dispose is a minor leak;
   dereferencing a null pointer is far worse.

5. To find the minimum AE version for any suite, grep the headers:

   ```sh
   grep -n "frozen\|since AE" <AE_SDK>/Headers/AE_GeneralPlug.h
   grep -n "frozen\|since AE" <AE_SDK>/Headers/AE_GeneralPlugOld.h
   ```

*Tags: `aegp`, `suite-version`, `compatibility`, `acquire-suite`, `cross-version`*
