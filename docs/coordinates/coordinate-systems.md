# Coordinate Systems

> 1 Q&A · source: AE plugin dev community Discord

### How do you handle coordinate system differences when drawing UI elements in Quartz on macOS After Effects plugins?

Quartz uses a different coordinate system than QuickDraw on macOS, with the Y-axis inverted and origin at bottom-left instead of top-left. To handle this, use CGContextConvertPointToUserSpace to get the flipped origin point, then adjust your Y coordinates accordingly. The solution involves getting the flip origin once per draw call and incorporating it into your transformation matrix: CGPoint zero = {0,0}; CGPoint flipOrigin = CGContextConvertPointToUserSpace(context, zero); flippedY = flipOrigin.y - (originaY - event_extraP->u.draw.update_rect.top);

```cpp
CGPoint zero = {0,0};
CGPoint flipOrigin = CGContextConvertPointToUserSpace(context, zero);
flippedY = flipOrigin.y - (originaY - event_extraP->u.draw.update_rect.top);
```

*Tags: `coordinate-systems`, `drawing`, `macos`, `quartz`, `ui`*

---
