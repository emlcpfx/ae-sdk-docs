# Crash Reports

> 1 Q&A · source: AE plugin dev community Discord

### Why might Adobe report high numbers of After Effects force-quits during launch?

A significant portion of force-quit reports during AE launch may actually be caused by plugin developers stopping AE from their debuggers (e.g., hitting 'Stop Debugging' in Visual Studio or Xcode while AE is attached). This registers as a force quit in Adobe's telemetry but is not an actual crash or bug.

*Tags: `crash-reports`, `debugging`, `force-quit`, `telemetry`, `visual-studio`*

---
