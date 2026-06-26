# Version Detection

> 2 Q&As · source: AE plugin dev community Discord

### How do you get the AE/Premiere version number programmatically?

in_data->version major/minor are not reliably updated between AE versions. Options: (1) Use ExtendScript via AEGP_ExecuteScript with 'app.version' - but may fail with 'cannot run script while modal dialog waiting'. (2) Use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP to get the AE install folder path, then parse the version from the folder name or binary metadata. (3) For plugins with Premiere GPU entry points, PPixSuite's appinfo provides version data (format like '24.3', add 2000 for year).

*Tags: `aegp`, `compatibility`, `extendscript`, `premiere`, `version-detection`*

---

### How can you detect the After Effects version from within a plugin?

You can use ExtendScript with 'app.version' via the AEGP execute script. Alternatively, if your plugin supports both After Effects and Premiere Pro with GPU acceleration, you have access to a Premiere Pro PICA Suite AppInfo that provides the version. The version data is in format like 24.3, and you need to add 2000 to get the actual version number (e.g., 24.3 becomes 2024.3). Some versions may include additional components like 24.3.x which require parsing.

*Tags: `aegp`, `gpu`, `premiere`, `scripting`, `version-detection`*

---
