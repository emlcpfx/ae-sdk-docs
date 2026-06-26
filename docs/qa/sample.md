# Q&A: sample

**10 entries** tagged with `sample`.

---

## What is a good reference sample project for implementing 3D matrix transformations in After Effects plugins?

The "Artie" sample project included in the After Effects SDK contains macros for getting XY coordinates of a texture using a 4x4 matrix. This sample is useful as a proof of concept for implementing 3D transform functions when antialiasing is not a critical concern.

*Tags: `reference`, `3d`, `matrix`, `sample`, `sdk`, `open-source`*

---

## Where can I find a sample project demonstrating Drawbot usage?

The "CCU" sample project in the After Effects SDK demonstrates the use of Drawbot for custom UI drawing.

*Tags: `ui`, `reference`, `sample`, `drawbot`*

---

## Is there a sample plugin that demonstrates PreRender and SmartRender implementation?

The Shifter sample plugin included in the After Effects SDK demonstrates PreRender and SmartRender usage.

*Tags: `smart-render`, `reference`, `open-source`, `sample`*

---

## How do I access and manipulate individual RGB pixel values in an After Effects plugin?

To access individual pixels and their RGB values, examine the 'shifter' sample project included in the After Effects SDK. This sample demonstrates how to read and modify individual pixel data, which is fundamental for any plugin that needs to perform per-pixel color operations.

*Tags: `sdk`, `reference`, `pixels`, `color`, `sample`*

---

## How do I access pixel data from different frames in an After Effects plugin?

The 'checkout' sample project in the After Effects SDK demonstrates how to retrieve images from points in time other than the currently processed frame. This is useful for plugins that need to reference previous or future frames, such as temporal effects or frame-blending operations.

*Tags: `sdk`, `reference`, `layer-checkout`, `frame-access`, `sample`*

---

## What are good sample projects to study for implementing stream data in After Effects plugins?

The ProjDumper, Streamie, and Mangler sample projects included in the After Effects SDK are recommended as good places to see streams implemented. These projects are included in the SDK download packages available from Adobe's developer portal.

*Tags: `aegp`, `reference`, `open-source`, `sample`*

---

## Is there a sample demonstrating OpenGL usage in After Effects plugins?

Yes, Adobe provides the GLator sample as a reference for using OpenGL in After Effects plugins. This sample can help developers understand how to integrate OpenGL drawing capabilities into their plugins.

*Tags: `opengl`, `gpu`, `reference`, `sample`*

---

## What is the panelator sample plugin used for?

The 'panelator' sample is an AEGP that demonstrates how to create a dockable panel in After Effects, similar to built-in palettes like the Info palette. It serves as a template for AEGP plugins that need to display custom user interfaces and allows you to draw arbitrary content within the panel.

*Tags: `aegp`, `ui`, `reference`, `sample`*

---

## What is the EMP sample and how does it deliver frame data to plugins?

EMP (External Monitor Preview) is a sample plugin template that demonstrates how to receive image data from After Effects. It uses the MyBlit() function as the callback where AE delivers the currently viewed composition buffer to the plugin. This function is registered during the entryPoint() function and is useful for plugins that need real-time access to the composition being viewed.

*Tags: `aegp`, `output-rect`, `reference`, `sample`*

---

## Where can I find sample code for creating and manipulating transformation matrices in After Effects plugins?

The CCU sample included in the After Effects SDK contains helper functions for creating identity matrices and concatenating them with scale and rotation transformations. This sample demonstrates proper matrix manipulation for use with transform_world() and similar functions.

*Tags: `mfr`, `reference`, `sdk`, `sample`, `matrix`*

---
