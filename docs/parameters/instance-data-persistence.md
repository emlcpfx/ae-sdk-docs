# Per-Instance Runtime Data Is Not Persisted -- Durable State Belongs in a Param

> Anything that must survive a project save and reopen has to live in a
> *parameter*. Per-instance runtime state -- a live session object, a cached
> handle, anything you rebuild on demand -- is in-memory only. Storing the durable
> truth there silently loses it on reopen, and the plugin comes back up with its
> settings gone.

## Symptom

A plugin needs to remember a user-chosen value across save -> reopen -- for
example a selected file path (a model file, a LUT, an external asset). The
"obvious" home is the per-instance runtime object that already holds the live
resource. After a save and reopen, the value is gone: the plugin comes up with no
selection and renders passthrough, looking like a "forgot my settings" bug.

## Root cause

Per-instance runtime data is **runtime-only**. Whether you keep it in AE's
sequence-data handle, a process-global map keyed by effect ref, or an instance
member, none of that is serialized into the project file. It is rebuilt from
scratch when the project reopens. So any state that lives *only* in instance data
is lost on save/load. (Instance-data teardown runs at sequence setdown; nothing
flattens it to disk unless you explicitly serialize it.)

The trap is assuming instance data is durable because it survives across renders
within a session. It does -- but a session is not a project lifetime.

## The fix

Split runtime resources from durable truth:

- **Keep instance data for runtime-only resources** -- the live session object,
  decoded caches, GPU handles. Correct, because they're rebuilt on demand from
  the persisted value.

- **Persist the durable value in a param.** Standard params already persist for
  scalars. For blobs, strings, and paths, store the bytes in an **arbitrary-data
  param** (`PF_Param_ARBITRARY_DATA`): AE serializes ARB params via your
  flatten/unflatten callbacks, so they survive save/load. Write from the
  user-changed-param handler with the standard
  `PF_CHECKOUT_PARAM` + `PF_LOCK_HANDLE` + `memcpy` +
  `PF_ChangeFlag_CHANGED_VALUE` pattern, and read it back in render.

> AE has **no native string param.** A file-path or string value on AE must be
> stored in an arbitrary-data param. (Some adapters add a disabled placeholder
> control for "string"/"filepath" purely to preserve param indexing -- it cannot
> store text.) On OFX, a native file-path param persists and gives a file browser
> for free.

## How to avoid it

- Treat per-instance data as a per-instance **runtime cache**, never as
  persistent storage.
- Anything that must survive save/load goes in a **param**: standard params for
  scalars; an arbitrary-data param for blobs/strings/paths on AE.
- The live resource is rebuilt from the persisted param on the next render -- so
  it is fine, and correct, for it to live in instance data.

*Tags: `instance-data`, `sequence-data`, `arb-data`, `persistence`, `save-load`,
`param`*
