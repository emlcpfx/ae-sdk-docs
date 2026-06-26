# Compatibility

> 6 Q&As · source: AE plugin dev community Discord

### What causes the 'Invalid Filter 25::3' error on macOS?

This error is typically caused by the Deployment Target being set too high in Xcode. For example, setting it to 12.3 will cause this error on older macOS versions. Try lowering it to support older versions. For Intel builds, supporting back to macOS 10.10 is common.

*Tags: `compatibility`, `deployment-target`, `invalid-filter`, `mac`, `xcode`*

---

### How do you get the AE/Premiere version number programmatically?

in_data->version major/minor are not reliably updated between AE versions. Options: (1) Use ExtendScript via AEGP_ExecuteScript with 'app.version' - but may fail with 'cannot run script while modal dialog waiting'. (2) Use AEGP_GetPluginPaths with AEGP_GetPathTypes_APP to get the AE install folder path, then parse the version from the folder name or binary metadata. (3) For plugins with Premiere GPU entry points, PPixSuite's appinfo provides version data (format like '24.3', add 2000 for year).

*Tags: `aegp`, `compatibility`, `extendscript`, `premiere`, `version-detection`*

---

### Can After Effects CS6 be installed and run on Windows 11?

Yes, AE CS6 runs fine on Windows 11. If you have activation issues with the license key, you may need to run it in compatibility mode. Windows 10 is a fallback option if Windows 11 doesn't cooperate with activation.

*Tags: `compatibility`, `cs6`, `installation`, `legacy`, `windows-11`*

---

### How do you check AE plugin compatibility with older versions of After Effects?

You can get installers for AE 2019 and older from the ProDesignTools website (https://prodesigntools.com/adobe-direct-download-links.html). For all newer versions, contact Adobe support for an offline installer. They typically respond quickly and provide installers within minutes.

*Tags: `compatibility`, `installer`, `older-versions`, `testing`*

---

### Can the same .aex plugin work in both After Effects and Premiere Pro?

Yes, but it requires correct coding. Premiere does many things differently than AE and not all features are supported in both hosts. The AE SDK documentation has a dedicated section for Premiere Pro compatibility (https://ae-plugins.docsforadobe.dev/). Notably, Premiere does not support SmartRender, which is a significant difference. There are also long-standing bugs in Premiere's implementation of AE plugin APIs (present since at least 2006) that are regularly reported to but largely ignored by Adobe.

*Tags: `aex`, `compatibility`, `cross-host`, `premiere-pro`, `smartrender`*

---

### Is it possible to install and activate Adobe After Effects CS6 on Windows 11?

CS6 should work on Windows 11, but you may need to run it in compatibility mode. If Windows 11 causes issues, consider testing on Windows 10 instead. Activation may be problematic due to Adobe's legacy license servers, so compatibility mode is a recommended workaround.

*Tags: `compatibility`, `cross-platform`, `deployment`, `windows`*

---
