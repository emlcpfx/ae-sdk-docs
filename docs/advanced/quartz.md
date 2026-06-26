# Quartz

> 2 Q&As · source: AE plugin dev community Discord

### How do you handle coordinate system differences when drawing UI elements in Quartz on macOS After Effects plugins?

Quartz uses a different coordinate system than QuickDraw on macOS, with the Y-axis inverted and origin at bottom-left instead of top-left. To handle this, use CGContextConvertPointToUserSpace to get the flipped origin point, then adjust your Y coordinates accordingly. The solution involves getting the flip origin once per draw call and incorporating it into your transformation matrix: CGPoint zero = {0,0}; CGPoint flipOrigin = CGContextConvertPointToUserSpace(context, zero); flippedY = flipOrigin.y - (originaY - event_extraP->u.draw.update_rect.top);

```cpp
CGPoint zero = {0,0};
CGPoint flipOrigin = CGContextConvertPointToUserSpace(context, zero);
flippedY = flipOrigin.y - (originaY - event_extraP->u.draw.update_rect.top);
```

*Tags: `coordinate-systems`, `drawing`, `macos`, `quartz`, `ui`*

---

### How do you draw transformed circular handles in macOS After Effects plugins accounting for layer scale, rotation, and position?

Use Quartz context transformation matrices to apply layer-to-frame transformations. First get the layer2frame transformation matrix using get_layer2comp_xform and source_to_frame callbacks. Then create a CGAffineTransform that includes the Y-axis inversion correction, and apply it to the context using CGContextConcatCTM before drawing. The Y-inversion adjustment should negate the b and d components and adjust the ty translation by the flipped origin offset.

```cpp
float a = xform.mat[0][0];
float b = xform.mat[0][1];
float c = xform.mat[1][0];
float d = xform.mat[1][1];
float tx = xform.mat[2][0];
float ty = xform.mat[2][1];
CGPoint origin = CGPointMake(0,0);
CGPoint originFlipped = CGContextConvertPointToUserSpace(context, origin);
CGAffineTransform trans = CGAffineTransformMake (a, -b, c, -d, tx, originFlipped.y - (ty - event_extraP->u.draw.update_rect.top));
CGContextConcatCTM (context, trans);
CGRect rect = CGRectMake ( center.x - cr, center.y - cr, 2*cr, 2*cr);
CGContextAddEllipseInRect(context, rect);
CGContextStrokePath(context);
```

*Tags: `drawing`, `macos`, `matrix`, `quartz`, `transformation`, `ui`*

---
