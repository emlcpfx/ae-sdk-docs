# Pixel Access

> 1 Q&A · source: AE plugin dev community Discord

### Why do rowbytes sometimes differ from width * bytes_per_pixel?

Rowbytes can be larger than expected because AE uses it as an optimization for layer cropping. When a layer is cropped (e.g., dragged partially off the comp), AE updates width/height and moves the data pointer to the new top-left pixel, but leaves rowbytes unchanged. This avoids memory reallocation. After a cache purge, rowbytes may return to the expected value. Always use rowbytes (not width * pixel_size) when iterating rows.

*Tags: `buffer-stride`, `cropping`, `memory-layout`, `pixel-access`, `rowbytes`*

---
