# Iterate

> 1 Q&A · source: AE plugin dev community Discord

### How do you get the correct coordinate in layer coordinate system when using the iterate function?

When a layer is partially off the comp, After Effects may give you a reduced buffer. The effectWorld structure shows you the offset values needed to correctly map pixel coordinates. When iterating, you need to account for this offset by using outPixel=GetColor(x+offsetX,y+offsetY). Additionally, there is an "iterate offset function" available in the SDK that can help handle this automatically. The input and output buffers for non-collapsed layers are in layer coordinates, but AE diminishes the buffer at 20% increments based on what part of the layer is out of sight, not just individual pixels.

```cpp
outPixel=GetColor(x+offsetX,y+offsetY)
```

*Tags: `iterate`, `layer-checkout`, `output-rect`, `sdk`*

---
