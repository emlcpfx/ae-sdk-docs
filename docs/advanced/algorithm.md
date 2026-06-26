# Algorithm

> 2 Q&As · source: AE plugin dev community Discord

### What is an efficient algorithm for creating edge outlines or choker mattes based on alpha channels?

Instead of checking neighbors for each transparent pixel (which results in O(n^2) complexity), use a distance field approach. Create an intermediate 2D buffer of signed distance values (e.g., int16_t) to exchange memory for speed. The Jump Flooding Algorithm (JFA) is a fast method for generating distance fields and is well-suited for this exact use case.

*Tags: `algorithm`, `gpu`, `matte`, `memory`, `performance`*

---

### What is a good resource for understanding the Jump Flooding Algorithm for distance field generation?

The blog post at https://itscai.us/blog/post/jfa/ provides an intuitive description of the Jump Flooding Algorithm and demonstrates how it applies directly to creating outlines and choker effects from alpha channels.

*Tags: `algorithm`, `distance-field`, `reference`, `tool`*

---
