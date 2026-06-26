# Introduction to After Effects Plugin Development

If you have ever applied Sapphire Glow to a layer, stacked a bunch of Red Giant Universe effects on an adjustment layer, or keyed footage with Primatte, you have already used plugins. You just never had to think about how they were made.

This guide is for VFX artists who are curious about what is happening under the hood -- and maybe want to build something of their own. You do not need a computer science degree. You need curiosity, patience, and a willingness to learn a few new tools.

Let's get into it.

---

## 1. What Is a Plugin?

A plugin is a small program that After Effects loads and runs alongside its own code. When you apply "CC Ball Action" or "Sapphire S_Glow" to a layer, AE is handing your footage to that plugin, which processes it and hands it back. The plugin lives as a file on your hard drive (a `.aex` file on both Windows and Mac), and AE picks it up automatically from its Plug-ins folder at startup.

### How Is That Different from Expressions or Scripts?

You have probably written expressions in AE -- maybe a `wiggle()` or a `loopOut()`. And you might have used ExtendScript or the newer ScriptUI panels to automate tasks. Those are interpreted: AE reads your text at runtime and figures out what to do with it, line by line.

Plugins are different. They are written in C++, a compiled programming language. "Compiled" means you write human-readable code, then a tool called a compiler translates it into machine code -- the raw instructions your CPU understands. This makes plugins dramatically faster than scripts or expressions, which matters when you are processing millions of pixels per frame.

Here is a rough comparison:

| | Expressions | Scripts (ExtendScript/CEP) | Plugins (C++ SDK) |
|---|---|---|---|
| **Speed** | Slow (interpreted per-frame) | Moderate | Native CPU speed |
| **Pixel access** | No direct pixel manipulation | Very limited | Full per-pixel control |
| **GPU acceleration** | No | No | Yes (Metal, OpenCL, CUDA, Vulkan) |
| **Custom UI in Effect Controls** | No | Panels only | Full custom parameters |
| **Custom file formats** | No | No | Yes |
| **Difficulty** | Low | Low-Medium | Medium-High |

### Why Do Plugins Exist?

Plugins exist because some things are simply impossible with expressions or scripting:

- **Pixel manipulation**: A blur, a glow, a chromatic aberration -- these all require reading and writing individual pixel values at high speed. Expressions cannot do this.
- **Custom rendering**: Particle systems, fluid simulations, ray-marched volumes -- all of these require a rendering pipeline that only compiled code can deliver at interactive speeds.
- **GPU acceleration**: Offloading work to the graphics card for real-time feedback.
- **Custom file formats**: Making AE understand a proprietary camera format or a studio's internal image sequence format.
- **Deep integration**: Plugins can respond to AE events, access the render queue, manipulate layers and keyframes, and draw custom overlays in the composition viewer.

!!! example "Plugins You Already Know"
    Effects like **Sapphire** (Boris FX), **Red Giant Universe**, **BorisFX Continuum**, **Neat Video**, **RE:Vision Effects** -- those are all plugins built with the same SDK you are about to learn. Every one of them started as an empty C++ file and a developer staring at the Skeleton example.

---

## 2. What You Need to Get Started

You do not need much, but you do need specific tools. Here is the list.

### A Computer

Windows or Mac. Both work. This guide leans toward Windows because the tooling is slightly more straightforward for beginners, but everything applies to Mac as well.

### The After Effects SDK

Free from Adobe. You download it from the [Adobe Developer Console](https://developer.adobe.com/). The SDK is a folder containing:

- **Header files** -- these are like a dictionary that tells your code what functions AE provides. You do not write these; you include them in your project.
- **Example plugins** -- working plugins you can study, modify, and build. The most important one is called **Skeleton**, and it is the "hello world" of AE plugin development.
- **Utility files** -- helper code that makes common tasks easier.
- **PiPLTool** -- a small program used during the build process (more on this later).

!!! tip "Where to Find the SDK"
    Search for "Adobe After Effects SDK" on the Adobe Developer site. You want the "After Effects Plug-in SDK." It is free -- you do not need a paid developer account.

### A Code Editor / IDE

This is where it gets slightly confusing because of naming.

**Visual Studio** (Windows) is what you use to write, organize, and compile your plugin. If AE is the NLE for video, Visual Studio is the NLE for code. It lets you write C++ source files, organize them into a project, and hit "Build" to produce your `.aex` plugin file. You want **Visual Studio 2022 Community Edition** -- it is free.

**Xcode** (Mac) is the equivalent on macOS. Also free from the Mac App Store.

**Visual Studio Code** (VS Code) is a different, lighter program. It is a text editor, not a full build environment. It is great for reading code, searching through files, and quick edits, but it is not what compiles your plugin. Think of it like a script editor versus a full NLE -- useful, but not the main tool here.

!!! warning "Visual Studio vs. Visual Studio Code"
    These are two completely different programs despite the similar names. **Visual Studio** (the full IDE) is what you need to compile plugins on Windows. **VS Code** is a lightweight text editor. You can use VS Code alongside Visual Studio for browsing code, but you cannot build an AE plugin with VS Code alone.

### After Effects

You need a copy of AE installed to test your plugin. When you build your plugin, you copy the resulting `.aex` file into AE's Plug-ins folder, restart AE, and your effect shows up in the Effects menu.

### C++ Basics

Let's be honest: you need to learn some C++. But here is the good news -- you do not need to become an expert. You need to understand:

- **Variables** -- storing values (a number, a color, a pixel)
- **Functions** -- blocks of code that do a specific thing
- **If/else statements** -- making decisions
- **Loops** -- repeating an operation (like processing every pixel in a row)
- **Structs** -- grouping related data together (like an ARGB pixel with four channel values)

That is genuinely enough to start. The AE SDK is heavy on patterns -- once you understand the structure, you will find yourself copying and adapting patterns more than writing code from scratch. It is a lot like learning a new node-based compositor: the first week is confusing, but then the patterns click.

!!! note "Learning C++"
    If you have never written C++ before, spend a few days with an introductory tutorial before diving into the SDK. You do not need to finish an entire course. Get comfortable with variables, functions, loops, and basic data types. Then come back here and the SDK code will make much more sense.

---

## 3. How Plugins Work: The Big Picture

Here is the core concept, and it is simpler than you might expect.

### The Loading Process

1. You build your plugin. The compiler produces a `.aex` file.
2. You copy the `.aex` file into AE's Plug-ins folder.
3. AE starts up and scans the Plug-ins folder.
4. AE finds your `.aex`, reads its metadata (the PiPL -- we will explain this shortly), and registers it.
5. Your plugin now appears in the Effects menu.

### The Conversation Model

Once loaded, AE communicates with your plugin through a simple system: AE sends **commands** (also called **selectors**), and your plugin responds to them. Think of it as a structured conversation:

> **AE:** "Hey, I just found you. Tell me about yourself -- what can you do?"
> **Plugin:** *(handles `PF_Cmd_GLOBAL_SETUP`)* "I support 16-bit and 32-bit color. I am thread-safe. Here is my version number."

> **AE:** "Great. What controls should I show the user?"
> **Plugin:** *(handles `PF_Cmd_PARAMS_SETUP`)* "Give me a slider called 'Amount' from 0 to 100, a color picker called 'Tint Color', and a checkbox called 'Invert'."

> **AE:** "The user applied you to a layer. Here is a frame of video -- process it."
> **Plugin:** *(handles `PF_Cmd_SMART_RENDER`)* "Got it. Here is the processed frame back."

> **AE:** "The user changed the 'Amount' slider."
> **Plugin:** *(handles `PF_Cmd_USER_CHANGED_PARAM`)* "Noted. I will adjust my processing next render."

> **AE:** "We are shutting down. Clean up after yourself."
> **Plugin:** *(handles `PF_Cmd_GLOBAL_SETDOWN`)* "Memory freed. Goodbye."

That is the entire model. Your plugin is a function that receives a command, looks at which command it is, does the appropriate work, and returns.

### The Nuke Analogy

If you come from Nuke, think of an effect plugin as a custom node. An image comes in through the input, you process it (maybe reading parameter values from knobs), and the result goes out. The difference is that in Nuke you wire nodes visually; in AE plugin development, you write the internal logic of that node in C++.

```
Input Image --> [ Your Plugin Code ] --> Output Image
                      ^
                      |
               Parameter Values
          (sliders, colors, checkboxes)
```

The AE SDK handles all the plumbing: getting pixels to you, displaying your parameters in the Effect Controls panel, managing undo, handling previews. You focus on the processing.

---

## 4. Breaking Down the Skeleton Plugin

The SDK ships with an example plugin called **Skeleton**. It is the simplest possible working effect plugin, and it is where every AE plugin developer starts. Let's walk through its key parts conceptually.

### The Entry Point -- The Front Door

Every plugin has a function called `EffectMain`. This is the front door that AE knocks on every time it needs something from your plugin. It receives the command, your parameter values, and pointers to input/output pixel buffers.

```cpp
PF_Err EffectMain(
    PF_Cmd       cmd,        // What AE is asking you to do
    PF_InData    *in_data,   // Info from AE (time, resolution, etc.)
    PF_OutData   *out_data,  // Your response back to AE
    PF_ParamDef  *params[],  // Current parameter values
    PF_LayerDef  *output,    // Where you write your output pixels
    void         *extra)     // Additional data for some commands
```

### The Big Switch -- The Decision Maker

Inside `EffectMain`, there is a `switch` statement. This is where your plugin decides what to do based on which command AE sent. It is like a reception desk: "If the message is about setup, go to room A. If it is about rendering, go to room B."

```cpp
switch (cmd) {
    case PF_Cmd_GLOBAL_SETUP:
        // Tell AE what we can do
        break;
    case PF_Cmd_PARAMS_SETUP:
        // Define our controls
        break;
    case PF_Cmd_RENDER:
        // Process pixels
        break;
    case PF_Cmd_GLOBAL_SETDOWN:
        // Clean up
        break;
}
```

Each `case` calls a separate function that handles that specific command. This keeps the code organized.

### GlobalSetup -- Introducing Yourself

This runs once when AE loads your plugin. You tell AE what your plugin is capable of: "I understand 16-bit color," "I can handle 32-bit float," "I am safe for multi-threaded rendering." You do this by setting flags -- think of them as checkboxes on a capabilities form.

```cpp
static PF_Err GlobalSetup(PF_InData *in_data, PF_OutData *out_data,
                          PF_ParamDef *params[], PF_LayerDef *output)
{
    out_data->my_version = 65536;  // Version 1.0.0
    out_data->out_flags  = PF_OutFlag_DEEP_COLOR_AWARE;  // "I handle 16-bit"
    return PF_Err_NONE;  // No errors
}
```

### ParamsSetup -- Defining Your Controls

This is where you tell AE what knobs and sliders to show in the Effect Controls panel. If you have used expressions to link to effect properties, you have seen these from the user side. Now you are defining them from the developer side.

```cpp
static PF_Err ParamsSetup(PF_InData *in_data, PF_OutData *out_data,
                          PF_ParamDef *params[], PF_LayerDef *output)
{
    PF_ParamDef def;
    AEFX_CLR_STRUCT(def);  // Clear the struct (always do this)

    PF_ADD_FLOAT_SLIDERX("Gain",       // Label shown in Effect Controls
                         0, 100,        // Valid range
                         0, 100,        // Slider range
                         10,            // Default value
                         PF_Precision_HUNDREDTHS,
                         0, 0,
                         GAIN_DISK_ID); // Unique ID for saving

    out_data->num_params = TOTAL_PARAMS; // Tell AE how many params we have
    return PF_Err_NONE;
}
```

!!! note "The Input Layer Is Automatic"
    Parameter index 0 is always the input layer -- AE adds this for you. You do not create it yourself. Your custom parameters (sliders, colors, etc.) start at index 1.

### Render -- The Main Event

This is where the actual pixel processing happens. AE gives you an input buffer (the source pixels) and an output buffer (where you write the result). You loop through every pixel, read it, do something to it, and write the result.

In the Skeleton example, it applies a simple gain (brightness adjustment) to each pixel. In a real plugin, this is where your blur algorithm, your particle simulation, your color science, or your procedural generation lives.

### PiPL -- The Label on the Box

PiPL stands for **Plugin Property List**. It is a small metadata resource embedded in your `.aex` file that tells AE about your plugin *before any of your code runs*. When AE scans its Plug-ins folder at startup, it reads the PiPL from each `.aex` to learn:

- The plugin's name and category (for the Effects menu)
- What the entry point function is called
- What capabilities the plugin claims to have
- A unique identifier called the **Match Name** (used to reconnect effects when you reopen a project)

Think of the PiPL as the nutrition label on a food package. The store (AE) reads the label to know what shelf to put it on, without having to open the package and taste the food.

!!! warning "PiPL Must Match Your Code"
    The capability flags you declare in the PiPL must exactly match the flags you set in your `GlobalSetup` code. If they do not match, AE will refuse to load your plugin and show a "flags mismatch" error. This is one of the most common beginner mistakes.

### The .aex File -- Your Finished Product

When you hit "Build" in Visual Studio or Xcode, the compiler takes all your C++ source code and produces a single `.aex` file. This is your plugin -- the thing you distribute to users, copy into the Plug-ins folder, and that AE loads at startup.

On Windows, a `.aex` file is actually a renamed `.dll` (dynamic link library). On Mac, it is a bundle. But the extension `.aex` is what tells AE "this is a plugin, load me."

### What Does "Building" Actually Mean?

If you come from Python, JavaScript, or ExtendScript, this concept is new. In those languages, you write a `.py` or `.js` file, and the computer reads and executes it directly. There is no separate "build" step -- you just run it.

C++ does not work this way. The code you write is not something the computer can run directly. It needs to be *translated* into machine code first -- the binary instructions that your CPU actually understands. This translation is called **compiling**, and the tool that does it is called a **compiler**.

Here is the full process, step by step:

1. **You write code** in `.cpp` and `.h` files. These are plain text, human-readable.
2. **You hit "Build"** in Visual Studio (or run a build command in the terminal).
3. **The preprocessor** runs first -- it handles `#include` statements (which copy the contents of other files into yours) and `#define` macros.
4. **The compiler** translates each `.cpp` file into an "object file" (`.obj` on Windows). This is machine code, but it is not yet a complete program.
5. **The linker** combines all the object files together, resolves references between them, and produces the final output: your `.aex` file.

The whole thing takes a few seconds for a small plugin, up to a minute for large projects.

!!! tip "The Python Analogy"
    In Python, you write `blur.py` and run `python blur.py`. The Python interpreter reads your code and executes it on the fly.

    In C++, you write `blur.cpp`, then run the compiler, which produces `blur.aex`. After Effects loads `blur.aex`. Your original `.cpp` file is never seen by AE -- only the compiled `.aex` matters.

    This is why plugins are so fast: the heavy translation work happens once at build time, not every time the effect runs.

**Why does this matter to you?** Because when something goes wrong, you need to understand which step failed:

- **Compiler error** -- your code has a syntax mistake or type mismatch. Like a typo in an expression, but stricter. The compiler tells you the file, line number, and what it expected. Fix the code, build again.
- **Linker error** -- your code compiled fine, but the linker cannot find something it needs (a function, a library). Usually means a missing SDK file or wrong project configuration.
- **Runtime crash** -- it built successfully, AE loaded it, but it crashed. This is the hardest to debug, but also where you learn the most.

!!! note "Debug vs. Release"
    Visual Studio has two main build configurations: **Debug** and **Release**. Debug builds are slower but include extra information that helps you find bugs (you can pause the plugin mid-execution and inspect variables). Release builds are optimized for speed -- this is what you ship to users. Always develop in Debug, ship in Release.

---

## 5. Types of Plugins You Can Make

The AE SDK supports several plugin types, each serving a different purpose. Most developers start with Effect plugins, but it helps to know the full landscape.

### Effect Plugins

**What they do:** Process pixels. An image goes in, your code transforms it, a new image comes out.

**Real-world examples:** Sapphire S_Glow (glow/bloom), Red Giant Looks (color grading), Neat Video (denoising), RE:Vision Twixtor (retiming), Trapcode Particular (particles).

**What you can build:** Blurs, sharpens, color corrections, distortions, glitch effects, generators (fractals, noise), compositing tools, keyers, particle systems -- anything that reads or writes pixels.

This is what most people mean when they say "AE plugin," and it is where you should start.

### AEGP (After Effects General Plugins)

**What they do:** Automate and extend AE itself. They do not process pixels directly -- instead, they manipulate the project, timeline, render queue, and UI.

**Real-world examples:** Panel plugins that add custom UI panels to AE, render queue automation tools, project management utilities.

**Think of it as:** ExtendScript on steroids. AEGPs can do things that scripts cannot: add menu items, respond to internal AE events, access low-level project data, and run with full native performance.

### AEIO (After Effects I/O Plugins)

**What they do:** Teach AE how to read and write custom file formats.

**Real-world examples:** If you wanted AE to import a proprietary camera RAW format, or export to a studio's internal image sequence format, you would write an AEIO plugin.

**Think of it as:** A codec plugin, but for AE specifically.

### Artisan Plugins (3D Renderers)

**What they do:** Replace AE's built-in 3D rendering engine with a custom one.

**Real-world examples:** The Cinema 4D Renderer option in AE's composition settings is an Artisan plugin.

**Think of it as:** Swapping out AE's scanline renderer for a ray tracer. This is extremely specialized and rarely needed.

!!! tip "Start with Effects"
    If you are new to plugin development, start with an Effect plugin. It is the most common type, has the most examples and documentation, and the skills transfer directly to the other types later.

---

## 6. The Plugin Ecosystem: AE, Premiere, and OFX

Before you commit to building a plugin, it helps to understand the broader landscape. There are multiple plugin formats in the VFX and post-production world, and they are not all compatible with each other.

### After Effects Plugins (.aex)

These are built with the **Adobe After Effects SDK** -- the SDK this entire documentation site covers.

**Strengths:**

- Full access to every AE feature: SmartFX rendering, AEGP suites, custom parameters, the works.
- Deep integration with AE's timeline, expressions, and compositing engine.
- Also work in **Premiere Pro** (with limitations -- see below).

**Limitations:**

- Only run in Adobe hosts (AE and Premiere). They will not load in Nuke, Resolve, Fusion, or any other application.

#### AE Plugins in Premiere Pro

AE effect plugins (`.aex`) can run inside Premiere Pro. Adobe built a compatibility layer for this. However, there are important differences:

- **No AEGP suites** -- you cannot access project data, compositions, or timeline information from Premiere.
- **No custom effect UI** -- custom parameter UIs built with AE's drawing suites do not work in Premiere.
- **Different pixel formats** -- Premiere uses BGRA and VUYA pixel layouts internally, which differ from AE's ARGB.
- **Different rendering pipeline** -- Premiere's render order and caching behave differently from AE.
- **Sequence data differences** -- if your plugin stores per-instance data, the lifecycle is slightly different.

!!! note "Testing in Both Hosts"
    If you plan to support both AE and Premiere, test in both from day one. Do not assume that "works in AE" means "works in Premiere." The differences are subtle but real.

### OFX Plugins (.ofx)

**OFX** (Open Effects) is an open standard for image processing plugins. It is the cross-platform alternative to proprietary plugin formats.

**Hosts that support OFX:** Nuke, DaVinci Resolve, Fusion, Vegas Pro, Natron, Scratch, Baselight, and many more.

**Hosts that do NOT support OFX:** After Effects. AE does not load `.ofx` files. Period.

OFX plugins work on the same core principle as AE plugins: receive pixels, process them, return the result. But the API is completely different. You cannot take AE SDK code and compile it as an OFX plugin without rewriting the host integration layer.

**Built with:** The [OFX SDK](https://openeffects.org/) (open source, available on GitHub).

!!! info "OFX vs. AE SDK"
    If you want your plugin to work in Nuke, Resolve, and Fusion, you need OFX. If you want it to work in After Effects, you need the AE SDK. If you want it to work everywhere, you need both -- with a shared processing core and separate wrappers for each format.

### Premiere Pro Plugins

Premiere Pro has its own dedicated SDK (separate from the AE SDK, though they share some header files). The Premiere SDK defines its own plugin types:

- **Video filters** -- similar to AE effects but with Premiere's pipeline
- **Transitions** -- wipe, dissolve, and custom transition effects
- **Importers** -- custom file format import
- **Exporters** -- custom file format export
- **Transmitters** -- send video to external devices (preview monitors, capture cards)

AE effect plugins can run in Premiere (with the limitations described above), but Premiere-native plugins offer tighter integration with Premiere's specific features.

### The Cross-Host Reality

In the real world, major plugin companies ship multiple versions of the same effect:

- **Boris FX Sapphire** ships as both AE/Premiere plugins AND OFX plugins (for Nuke, Resolve, Fusion, etc.)
- **Red Giant / Maxon** products target the Adobe ecosystem with AE SDK plugins
- **RE:Vision Effects** ships AE, Premiere, OFX, and sometimes standalone versions

The creative work -- the actual image processing algorithms -- is the same across all versions. What changes is the **wrapper**: the code that talks to each host application. Think of it like shooting a project that needs to deliver in both DCI 4K and UHD. The creative content is the same, but you need different containers and specifications for each deliverable.

Some developers use frameworks like **PluginPort** to write one codebase and build for multiple hosts automatically. But when you are starting out, focus on one target (AE) and worry about cross-host compatibility later.

!!! tip "Analogy: Camera Adapters"
    Think of it like mounting lenses on different camera systems. The glass (your image processing code) is the same, but you need a different adapter (SDK wrapper) for each camera body (host application). A PL-mount lens needs an adapter for EF-mount, another for E-mount. Same idea.

---

## 7. Your First Steps

Here is a concrete path from "I have never built a plugin" to "I just built a plugin."

### Step 1: Download the SDK

Go to the [Adobe Developer site](https://developer.adobe.com/) and download the After Effects Plug-in SDK. Unzip it somewhere sensible -- you will be referencing this folder from your projects.

### Step 2: Install Visual Studio (Windows) or Xcode (Mac)

**Windows:** Download [Visual Studio 2022 Community](https://visualstudio.microsoft.com/). During installation, select the **"Desktop development with C++"** workload. This installs the C++ compiler, debugger, and everything you need.

**Mac:** Install Xcode from the Mac App Store. Open it once to accept the license and install command-line tools.

### Step 3: Open the Skeleton Example

In the SDK folder, navigate to:

```
AfterEffectsSDK/Examples/Effect/Skeleton/
```

On Windows, open the `.sln` (solution) file in Visual Studio. On Mac, open the Xcode project.

### Step 4: Build It

"Building" means turning your human-readable C++ code into a `.aex` file that AE can load. In Visual Studio, go to **Build > Build Solution** (or press `Ctrl+Shift+B`). In Xcode, press `Cmd+B`.

If everything is set up correctly, the compiler will produce a `.aex` file in the output directory.

!!! note "First Build Issues"
    The first build might not work perfectly out of the box, depending on your SDK version and IDE version. If you get errors, check that your include paths point to the SDK's `Headers` directory and that the platform is set to `x64`. The SDK's readme file usually has specific setup instructions for your version.

### Step 5: Copy to AE's Plug-ins Folder

Copy the built `.aex` file to:

**Windows:**
```
C:\Program Files\Adobe\Adobe After Effects <version>\Support Files\Plug-ins\
```

**Mac:**
```
/Applications/Adobe After Effects <version>/Plug-ins/
```

You can also create a subfolder to keep things organized, or set a custom plugin folder in AE's preferences under **Preferences > General > Additional Plug-ins Folder**.

### Step 6: Test It

Open After Effects. Create a new composition, add a solid layer, and go to **Effect > SDK Examples > Skeleton** (or wherever the Skeleton example's category places it). You should see the effect applied with its parameters in the Effect Controls panel.

Congratulations. You just built and loaded an After Effects plugin.

### Step 7: Start Modifying

Now the real learning begins. Try these experiments:

- Change the default value of the Gain slider and rebuild
- Add a second parameter (a color picker or a checkbox)
- Modify the render function to do something different to the pixels
- Break something on purpose, read the error, and fix it

Every modification teaches you something about how the SDK works. Do not be afraid to experiment -- the worst that can happen is AE shows an error or ignores your plugin, and you rebuild.

### Resources

Here are the places to go when you get stuck:

- **This documentation site** -- we are building a comprehensive, practical guide to the AE SDK
- **Community Discord servers** -- there are active communities of plugin developers who help each other. Search around and you will find them.
- **[DocsForAdobe.dev](https://docsforadobe.dev/)** -- community-maintained SDK documentation that fills in gaps the official docs leave
- **The SDK examples** -- every example in the SDK demonstrates a specific feature. Read them. Build them. Modify them.
- **[Adobe Community Forums — After Effects SDK](https://community.adobe.com/t5/after-effects-sdk/ct-p/ct-after-effects-sdk)** -- years of archived questions and answers from Adobe engineers and experienced developers

!!! tip "You Are Not Alone"
    Plugin development can feel isolating because it is a niche skill. But there is a genuine community of developers who have hit every wall you are about to hit and are willing to help. Join the Discord, ask questions, and share your progress. Everyone started exactly where you are now.

---

## What's Next?

Now that you understand what plugins are, how they work at a high level, and what tools you need, the next step is to get your hands dirty. The recommended path:

1. **Build the Skeleton example** and confirm it loads in AE
2. **Read through the Skeleton source code** with this guide as context -- it will make much more sense now
3. **Modify a parameter** and rebuild to see the change
4. **Write a simple pixel operation** (like tinting every pixel toward a color)
5. **Move on to SmartFX** when you are ready for 32-bit float support and the modern rendering pipeline

The learning curve is real, but it is not a cliff. It is more like learning a new compositing application: overwhelming at first, then suddenly the patterns start repeating and things click.

Welcome to plugin development.
