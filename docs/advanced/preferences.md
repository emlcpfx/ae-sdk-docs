# Preferences

> 1 Q&A · source: AE plugin dev community Discord

### How should you diagnose a plugin crash that only occurs in After Effects CC 2015+ but not CC 2014, particularly when it involves custom UI?

Ask the user to send over their old After Effects preferences folder and then delete the pref folder before relaunching AE. If resetting the preferences solves the issue, try reproducing the issue on your machine using the user's old preferences. This approach has a high probability of resolving the issue. If successful, the old prefs can help identify what setting or cached data was causing the crash.

*Tags: `debugging`, `macos`, `preferences`, `ui`*

---
