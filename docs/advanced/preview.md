# Preview

> 1 Q&A · source: AE plugin dev community Discord

### Is it possible to achieve progressive rendering in an After Effects plugin that continuously shows results to the user as more samples are computed?

The conversation acknowledges the desire for progressive rendering where samples accumulate and display over time rather than waiting for all samples to complete before showing results. While the technical feasibility is not explicitly detailed in the response, the user notes they have this working outside of After Effects and finds it more interactive. The challenge with current AE plugin behavior is that it waits for complete render before display, making it feel slow compared to progressive sampling approaches.

*Tags: `preview`, `render-loop`, `smart-render`, `ui`*

---
