# Custom Param

> 1 Q&A · source: AE plugin dev community Discord

### How can I offset an Angle parameter by 90 degrees so that 0° points to 3 o'clock instead of 12 o'clock?

You need to implement a custom parameter to achieve this offset. While you may want to reuse drawing code from the stock angle parameter, you'll likely need to create a custom parameter implementation from scratch rather than simply modifying the existing one.

*Tags: `custom-param`, `params`, `ui`*

---
