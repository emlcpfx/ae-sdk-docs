# AE SDK Docs

Community-maintained documentation for After Effects C++ plugin development.
This repository is the **single source of truth** for the prose docs — it contains
*only* Markdown (no website code, no deploy scripts, no credentials).

Published as a site at https://emlcpfx.github.io/ae-sdk-docs/ and consumed by:

- the `/ae-plugin` Claude Code skill (as a submodule),
- the AE SDK Discord helper bot,
- the `ae-sdk-site` build that publishes the styled docs site.

## Layout

```
docs/        all documentation, organized by topic
             (rendering, parameters, memory-threading, gpu, aegp, layers, platform, ...)
mkdocs.yml   site config (MkDocs + Material)
```

## Contributing

Documentation is the product here — corrections and new lessons are welcome.

- **Invited collaborators** push directly to `master`; changes fan out to the site,
  the skill, and the bot automatically.
- **Everyone else** can open an issue or a pull request.

Keep entries focused on the **After Effects (and cross-host) SDK itself** — symptom,
root cause, and the SDK-level fix, with minimal example code. Avoid anything specific
to a particular plugin or build framework.
