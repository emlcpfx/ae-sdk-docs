# After Effects

> 2 Q&As · source: AE plugin dev community Discord

### How can I restrict an After Effects plugin to specific app versions while keeping it in a single installation folder?

Installing a plugin in the Adobe/Common/Plug-ins/7.0/MediaCore folder makes it available in all versions of After Effects and Premiere Pro. Unfortunately, there is no way to restrict it to specific app versions or applications from the MediaCore folder. To restrict a plugin to only specific versions (e.g., After Effects CC 2014-2019), you must install it in each application version's individual plug-ins folder instead of using the shared MediaCore location.

*Tags: `after effects`, `deployment`, `installation`, `plugin`, `premiere`*

---

### Can I disable an After Effects plugin when it loads in Premiere Pro by checking the application ID?

Previously, it was possible to disable a plugin in Premiere Pro by checking in_data->appl_id for 'PrMr' and returning an error code (-1 or similar) at the beginning of the entrypoint function. However, this method no longer works with Premiere Pro CC2018 and later versions. The recommended approach is to install the plugin in version-specific folders rather than relying on application ID checking.

*Tags: `aegp`, `after effects`, `deployment`, `plugin`, `premiere`*

---
