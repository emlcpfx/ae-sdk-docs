# Drawing

> 6 Q&As · source: AE plugin dev community Discord

### How does the AddArc function work in the Drawbot Path Suite, and why do two consecutive arcs produce unexpected shapes?

When drawing multiple arcs/circles in the same path, you need to call close() on the path after each circle. Without closing, the entire sequence is treated as one continuous path, and the renderer draws a connecting line from the end point of one arc back to the beginning of the next, creating unexpected shapes. You can have multiple closed shapes in one path, or alternatively draw each shape in separate paths with separate drawing calls.

*Tags: `addarc`, `custom-ui`, `drawbot`, `drawing`, `path-suite`*

---

### How should DrawImage be called in drawbotSuites to ensure correct opacity and alpha rendering?

When using drawbotSuites.surface_suiteP->DrawImage, you must use an opacity of 1.0. If you don't use 1.0 opacity, the drawing will appear darker than intended and the alpha will still show as 1.0 since After Effects' UI will be behind it with solid alpha. This is important for correct compositing behavior.

```cpp
drawbotSuites.surface_suiteP->DrawImage(/* ... */, 1.0 /* opacity */, /* ... */)
```

*Tags: `aegp`, `debugging`, `drawing`, `ui`*

---

### How can you implement live drawing and UI interactions like rotoscope in After Effects plugins?

Live drawing and UI interactions such as rotoscope functionality in After Effects plugins are implemented through the PF_CMD_EVENT command, which allows plugins to handle real-time user input and interactive drawing on the canvas.

*Tags: `aegp`, `drawing`, `events`, `interactive`, `ui`*

---

### How can I draw custom curves outside the composition bounds in After Effects?

To draw custom UI elements like curves outside the composition, you need to use PF_Event and Drawbot APIs. Look up "PF_Event" and "Drawbot" in the After Effects SDK guide for implementation details.

*Tags: `aegp`, `custom-ui`, `drawing`, `ui`*

---

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
