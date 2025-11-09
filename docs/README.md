# egui Documentation

Welcome to the comprehensive egui documentation! This directory contains in-depth guides, architecture documentation, tutorials, and API references for egui.

## 📚 Documentation Structure

### [Architecture](/docs/architecture/)
Deep dives into how egui works internally:
- **[Codebase Analysis](architecture/CODEBASE_ANALYSIS.md)** - Comprehensive technical analysis of egui's architecture, execution flow, and novel patterns

### [Guides](/docs/guides/)
Step-by-step guides for building applications:
- **[Building a Text Editor](guides/building-a-text-editor.md)** - Complete guide from basic input to syntax-highlighted code editors

### [Tutorials](/docs/tutorials/) *(Coming Soon)*
Hands-on tutorials for common tasks:
- Getting Started with egui
- Creating Custom Widgets
- State Management Patterns
- Performance Optimization

### [Reference](/docs/reference/) *(Coming Soon)*
API references and quick lookups:
- Widget Gallery
- Style System Reference
- Event Handling Reference
- Integration Guide

## 🚀 Quick Start

New to egui? Start here:

1. Read the main [README.md](../README.md) for project overview
2. Check out [examples/hello_world](../examples/hello_world/) for a minimal example
3. Explore the [Architecture docs](architecture/CODEBASE_ANALYSIS.md) to understand how egui works
4. Follow the [Text Editor Guide](guides/building-a-text-editor.md) to build something real

## 📖 What is egui?

egui (pronounced "e-gooey") is an **immediate mode GUI library** for Rust. Key characteristics:

- **Immediate Mode**: UI code runs every frame (~60fps)
- **Pure Rust**: No unsafe code, minimal dependencies
- **Cross-Platform**: Web (WASM), native (Windows/Mac/Linux), and game engines
- **Simple API**: No callbacks, no retained widget trees, just straightforward code

### Simple Example

```rust
if ui.button("Click me").clicked() {
    println!("Button was clicked!");
}
```

That's it! No widget objects, no callbacks, no event handlers.

## 📊 Repository Structure

```
egui/
├── crates/
│   ├── emath/           # 2D math primitives
│   ├── ecolor/          # Color types
│   ├── epaint/          # Shape tessellation (shapes → triangles)
│   ├── egui/            # Core immediate mode GUI library
│   ├── egui-winit/      # Winit integration
│   ├── egui_glow/       # OpenGL renderer
│   ├── egui-wgpu/       # WebGPU renderer
│   ├── eframe/          # Official framework (web + native)
│   ├── egui_extras/     # Additional features (images, syntax highlighting)
│   └── egui_demo_lib/   # Demo applications
├── examples/            # Example applications
├── docs/                # This documentation
└── tests/               # Integration tests
```

## 🎯 Core Concepts

### Immediate Mode
Your UI code runs every frame. There are no widget objects to create, manage, or destroy:

```rust
impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            // This code runs every frame
            ui.label("Hello, world!");
            if ui.button("Increment").clicked() {
                self.counter += 1;
            }
            ui.label(format!("Counter: {}", self.counter));
        });
    }
}
```

### Selective State Retention
Despite being immediate mode, egui **does** store some state:
- Window positions and sizes
- Scroll positions
- Text editor cursor positions
- Whether collapsing headers are open

This state is stored in `Memory` and keyed by widget IDs.

### Three-Phase Execution
Every frame:
1. **Input**: Collect mouse/keyboard/touch input
2. **UI Logic**: Run your application code, generating `Shape` primitives
3. **Tessellation**: Convert shapes to triangles and render

## 🔧 Key Components

### Context
The `Context` is egui's brain:
- Stores persistent `Memory`
- Manages multiple viewports (OS windows)
- Handles repaint logic
- Manages fonts and textures

### Ui
The `Ui` represents a region of screen with a layout direction:
- Where you add widgets
- Handles layout incrementally
- Tracks available space

### Response
What every widget returns:
```rust
let response = ui.button("Click");
if response.clicked() { /* ... */ }
if response.hovered() { /* ... */ }
if response.dragged() { /* ... */ }
```

### Painter
Converts high-level shapes to renderable primitives:
- Circles, rectangles, text → triangle meshes
- Anti-aliasing via feathering
- Precomputed vertex arrays for performance

## 📝 Documentation Philosophy

This documentation follows these principles:

1. **Truth First**: Document what exists, not what should exist
2. **Code References**: Link to actual source code (e.g., `crates/egui/src/ui.rs:123`)
3. **From First Principles**: Explain *why*, not just *how*
4. **Runnable Examples**: All examples should be copy-paste ready
5. **Performance Aware**: Call out performance implications

## 🛠️ Common Tasks

### Adding a Widget
```rust
ui.label("I am a label");
ui.button("Click me");
ui.checkbox(&mut self.my_bool, "Toggle me");
ui.add(egui::Slider::new(&mut self.value, 0..=100));
```

### Layout
```rust
// Horizontal layout
ui.horizontal(|ui| {
    ui.label("Left");
    ui.label("Right");
});

// Vertical layout (default)
ui.vertical(|ui| {
    ui.label("Top");
    ui.label("Bottom");
});
```

### Windows and Panels
```rust
// Side panel
egui::SidePanel::left("my_panel").show(ctx, |ui| {
    ui.label("Panel content");
});

// Window
egui::Window::new("My Window").show(ctx, |ui| {
    ui.label("Window content");
});

// Central panel (takes remaining space)
egui::CentralPanel::default().show(ctx, |ui| {
    ui.label("Main content");
});
```

### Handling Input
```rust
let response = ui.button("Click me");

if response.clicked() {
    println!("Clicked!");
}

if ui.input(|i| i.key_pressed(egui::Key::Escape)) {
    println!("Escape was pressed");
}
```

## 🎨 Styling

```rust
// Access the style
let style = ui.style();

// Modify the style
ui.style_mut().spacing.item_spacing = egui::vec2(10.0, 5.0);

// Change colors
ui.visuals_mut().override_text_color = Some(egui::Color32::RED);
```

## 🐛 Debugging

### Enable Debug Rendering
```rust
ctx.set_debug_on_hover(true);  // Show widget rectangles on hover
```

### Inspect State
```rust
// View what's stored in Memory
egui::Window::new("Memory Debug").show(ctx, |ui| {
    ctx.memory(|mem| {
        ui.label(format!("Areas: {:?}", mem.areas()));
    });
});
```

### Profiling
```rust
// Enable the puffin profiler
ctx.set_embed_viewports(false);

// View detailed frame timings
if ui.button("Open Profiler").clicked() {
    puffin::set_scopes_on(true);
}
```

## 📚 Additional Resources

### Official Resources
- **Main README**: [../README.md](../README.md)
- **API Docs**: <https://docs.rs/egui>
- **Web Demo**: <https://www.egui.rs/#demo>
- **GitHub**: <https://github.com/emilk/egui>
- **Discord**: <https://discord.gg/JFcEma9bJq>

### Community Resources
- [3rd Party Integrations](https://github.com/emilk/egui/wiki/3rd-party-integrations)
- [3rd Party Crates](https://github.com/emilk/egui/wiki/3rd-party-egui-crates)
- [GitHub Discussions](https://github.com/emilk/egui/discussions)

## 🤝 Contributing to Documentation

Found an error? Want to add documentation?

1. Documentation source is in `docs/`
2. Keep the style: truth-first, with code references
3. Test all code examples
4. Follow the existing structure
5. Submit a PR!

## 📜 License

This documentation is part of the egui project and follows the same license (MIT OR Apache-2.0).

---

**Version**: egui 0.33.0
**Last Updated**: 2025
**Maintained by**: The egui community
