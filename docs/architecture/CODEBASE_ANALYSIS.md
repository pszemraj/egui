# egui: Immediate Mode GUI in Rust - A Deep Dive

## What Problem Does This Solve?

egui solves the fundamental tension in GUI programming: **making UI code simple to write while maintaining high performance**. Traditional retained-mode GUI frameworks require you to:
- Create widget objects
- Wire up callbacks and event handlers
- Manually sync application state with UI state
- Manage widget lifecycles

egui collapses all of this into direct, procedural code that runs every frame:

```rust
if ui.button("Click me").clicked() {
    do_something();
}
```

That's it. No callbacks, no state synchronization, no widget lifetime management.

## The Core Insight: Immediate Mode with Retained State

Despite being "immediate mode," egui is not purely stateless. The genius lies in **what** state it retains and **how** it manages it.

### The Three-Phase Architecture

Every frame follows this precise flow (see `crates/egui/src/lib.rs:122-144`):

1. **Input Collection**: Gather raw input (mouse position, clicks, key presses) into a `RawInput` struct
2. **UI Logic Execution**: Run your application code via `Context::run()`, which generates `Shape` primitives
3. **Tessellation & Rendering**: Convert shapes to textured triangles and paint them

```rust
// The actual game loop pattern egui expects
loop {
    let raw_input: egui::RawInput = gather_input();

    let full_output = ctx.run(raw_input, |ctx| {
        egui::CentralPanel::default().show(&ctx, |ui| {
            ui.label("Hello world!");
            if ui.button("Click me").clicked() {
                // take action
            }
        });
    });

    handle_platform_output(full_output.platform_output);
    let clipped_primitives = ctx.tessellate(full_output.shapes, full_output.pixels_per_point);
    paint(full_output.textures_delta, clipped_primitives);
}
```

## The Crate Layering: Separation of Concerns

The repository is a **monorepo** with 10+ crates that form a strict dependency hierarchy:

```
emath (2D math primitives)
  ↓
ecolor (Color representations)
  ↓
epaint (Shape → Triangle tessellation)
  ↓
egui (The UI library itself)
  ↓
egui-winit, egui_glow, egui-wgpu (Platform integrations)
  ↓
eframe (The official framework)
```

### emath: Minimal 2D Math (`crates/emath/`)
Pure mathematical primitives with zero dependencies:
- `Vec2`, `Pos2`: 2D vectors and positions
- `Rect`: Axis-aligned rectangles
- Utilities: `lerp`, `remap`, `clamp`

### epaint: The Graphics Layer (`crates/epaint/`)
Converts high-level shapes into renderable triangles:

**Input**: `Shape` enum (`crates/epaint/src/shapes.rs`)
- `Shape::Circle`
- `Shape::Rect`
- `Shape::Text`
- `Shape::Path`
- `Shape::Mesh` (pre-tessellated triangles)

**Output**: `Mesh` (`crates/epaint/src/mesh.rs`)
- Array of `Vertex` (position, UV, color)
- Array of triangle indices
- `TextureId` reference

**The Tessellator** (`crates/epaint/src/tessellator.rs`):
- Converts circles to triangles using precomputed vertex arrays (CIRCLE_8, CIRCLE_16, CIRCLE_32, CIRCLE_64)
- Handles anti-aliasing via feathering (adding semi-transparent edge vertices)
- Supports path stroking and filling using lyon-inspired algorithms
- Flattens Bézier curves into line segments

### egui: The Immediate Mode UI (`crates/egui/`)

This is where the magic happens. Key files:

#### `context.rs` (159KB, ~5000 lines)
The `Context` is egui's singleton brain. It holds:
- **Memory**: Persistent state across frames (`crates/egui/src/memory/`)
- **ViewportState**: Per-window state (one `Context` can manage multiple OS windows)
- **RepaintLogic**: Determines when to redraw (on-demand repainting)
- **TextureManager**: Font atlas and user textures

**The RwLock Pattern**:
```rust
ctx.input(|i| i.key_down(Key::A))
```
Every access to `Context` internals uses closures to minimize lock duration. This prevents deadlocks and allows parallel UI execution.

#### `ui.rs` (118KB, ~3000 lines)
The `Ui` struct represents a region of screen with a layout direction. Creating a button looks like this:

```rust
// crates/egui/src/widgets/button.rs:253-346
pub fn atom_ui(self, ui: &mut Ui) -> AtomLayoutResponse {
    // 1. Calculate size and padding
    let mut prepared = layout
        .frame(Frame::new().inner_margin(button_padding))
        .min_size(min_size)
        .allocate(ui);  // Reserves space in the Ui

    // 2. Check interaction state
    let response = if ui.is_rect_visible(prepared.response.rect) {
        let visuals = ui.style().interact_selectable(&prepared.response, selected);

        // 3. Generate shapes (if visible)
        if visible_frame {
            prepared.frame = prepared.frame
                .fill(fill)
                .stroke(stroke)
                .corner_radius(corner_radius);
        }

        prepared.paint(ui)  // Adds shapes to the layer
    } else {
        AtomLayoutResponse::empty(prepared.response)
    };

    response
}
```

**The Placer** (`crates/egui/src/placer.rs`):
Manages layout incrementally. As widgets are added, it:
- Allocates space based on layout direction (horizontal/vertical)
- Tracks the cursor position for the next widget
- Updates the `min_rect` (the bounding box of all content)

#### `response.rs` (40KB)
The `Response` is what every widget returns. It tells you:
- `clicked()`: Was this widget clicked this frame?
- `hovered()`: Is the mouse over it?
- `dragged()`: Is it being dragged?
- `changed()`: Did the value change? (for inputs)

**Bitflags for State** (`response.rs:81-141`):
```rust
bitflags! {
    impl Flags: u16 {
        const ENABLED = 1<<0;
        const CONTAINS_POINTER = 1<<1;
        const HOVERED = 1<<2;
        const CLICKED = 1<<4;
        const DRAGGED = 1<<8;
        const CHANGED = 1<<11;
        // ... etc
    }
}
```
Compact representation of all interaction states in a single u16.

#### `memory/` - The Persistence Layer
The immediate mode paradox: **how do you handle stateful widgets without stored widget objects?**

Answer: **IDs and a global hashmap** (`crates/egui/src/memory/mod.rs:34-48`):

```rust
pub struct Memory {
    /// Widget state storage keyed by Id
    pub data: IdTypeMap,

    /// Frame-to-frame animation cache
    pub caches: CacheStorage,

    /// Window positions and sizes
    areas: ViewportIdMap<Areas>,

    /// Which widget has focus?
    pub(crate) focus: ViewportIdMap<Focus>,

    /// Open popup state
    popups: ViewportIdMap<OpenPopup>,
}
```

Widgets generate stable IDs (see `crates/egui/src/id.rs`) based on their parent's ID and their position. Collapsing headers, scroll positions, text edit cursor positions—all stored in `Memory::data`.

**Example**: ScrollArea stores its scroll offset:
```rust
// Inside ScrollArea::show():
let scroll_offset: Vec2 = ctx.memory(|mem| {
    mem.data.get_temp(scroll_id).unwrap_or_default()
});

// After scrolling:
ctx.memory_mut(|mem| {
    mem.data.insert_temp(scroll_id, new_scroll_offset);
});
```

### egui-winit, egui_glow, egui-wgpu: The Integration Layer

These crates bridge egui to actual windowing systems and graphics APIs:

- **egui-winit** (`crates/egui-winit/`): Translates `winit` events into `RawInput`
- **egui_glow** (`crates/egui_glow/`): Renders `Mesh`es using OpenGL/glow
- **egui-wgpu** (`crates/egui-wgpu/`): WebGPU renderer

### eframe: The Official Framework

`eframe` (egui framework) provides the batteries-included experience:
- Single codebase for web and native
- Window management
- Persistent storage
- Event loop management

Users implement a single trait:

```rust
impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            // Your UI code here
        });
    }
}
```

## Novel Patterns and Clever Bits

### 1. The Sense System
Widgets don't just "exist"—they declare what interactions they care about (`crates/egui/src/sense.rs`):

```rust
pub struct Sense {
    pub click: bool,
    pub drag: bool,
}
```

This lets egui optimize: if no widget under the cursor senses clicks, it doesn't bother checking for clicks.

### 2. Smart Repainting
egui doesn't repaint every frame unconditionally. It tracks:
- Animations in progress
- Mouse movement over interactive regions
- Pending events

(`crates/egui/src/context.rs:127-171` - the repaint logic)

When idle, CPU usage drops to near-zero.

### 3. The Widget Trait is Trivial
```rust
pub trait Widget {
    fn ui(self, ui: &mut Ui) -> Response;
}
```

That's it! Consume self, take a `&mut Ui`, return interaction info. The simplicity enables the builder pattern:

```rust
ui.add(Button::new("Click me")
    .fill(Color32::RED)
    .frame(false)
    .min_size(vec2(100.0, 40.0)))
```

### 4. Premultiplied Alpha Everywhere
All colors use premultiplied alpha (`epaint/src/color.rs`). This isn't just a choice—it's fundamental to correct rendering:
- Correct blending: `(ONE, ONE_MINUS_SRC_ALPHA)`
- No color fringing
- Proper compositing

### 5. The "Frame Delay" Problem and Solutions
The fundamental immediate mode issue: you need to know window size before you lay out contents, but you need to lay out contents to know the size.

**egui's solutions**:
1. **Previous frame data**: Windows use the size from last frame (causes a 1-frame flicker)
2. **Multi-pass**: `Context::request_discard()` can request a second layout pass the same frame
3. **For atomic widgets**: Button sizes are known before layout (text is measured first)

See `crates/egui/src/lib.rs:218-227` for discussion.

### 6. The TextureAtlas
All font glyphs and small images pack into a single texture (`epaint/src/texture_atlas.rs`). This minimizes draw calls—critical for performance.

### 7. Automatic ID Generation
Every widget gets a unique ID without you thinking about it. The `Ui` maintains an auto-incrementing counter:

```rust
// crates/egui/src/ui.rs:72-78
next_auto_id_salt: u64,
```

For widgets that need stable IDs (like windows), you provide an explicit ID.

## What's Incomplete or Experimental?

### Work in Progress
1. **Advanced Layout**: No flexbox equivalent (yet). Grid and manual layout only.
2. **Styling System**: Less powerful than CSS. `Style` struct has fixed fields (`crates/egui/src/style.rs`).
3. **Accessibility**: AccessKit integration exists but is still maturing.
4. **Mobile Touch**: Works, but not as polished as desktop mouse interaction.

### Experimental Features
- **Viewports**: Multi-window support (popup OS windows)
- **Loaders**: Pluggable image loading system (`crates/egui/src/load/`)

### Known Dragons
- **No built-in async**: You must use channels or `Arc<Mutex<>>` to communicate with background tasks (see README.md FAQ)
- **Large scroll areas**: Layout runs every frame, so 10,000-item lists are slow (requires virtualization pattern)
- **Breaking changes**: API is unstable (currently v0.33.0)

## Performance Characteristics

From observation and code analysis:

- **Per-frame overhead**: 1-2ms for typical UIs (mostly tessellation)
- **Memory**: Lightweight. Context + Memory ~few KB. Meshes are transient.
- **CPU when idle**: Near zero (smart repaint logic)
- **CPU during interaction**: Single-threaded layout, but very fast
- **Binary size**: ~1-2MB with default features (no huge dependencies)

**The secret sauce**: Precomputed circle vertices, incremental layout, and on-demand tessellation.

## How to Actually Use This

### Minimal Example
```rust
// examples/hello_world/src/main.rs
fn main() -> eframe::Result {
    eframe::run_native(
        "My App",
        eframe::NativeOptions::default(),
        Box::new(|_cc| Ok(Box::new(MyApp::default()))),
    )
}

struct MyApp { name: String, age: u32 }

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.heading("My egui Application");
            ui.text_edit_singleline(&mut self.name);
            ui.add(egui::Slider::new(&mut self.age, 0..=120));
            if ui.button("Increment").clicked() {
                self.age += 1;
            }
            ui.label(format!("Hello '{}', age {}", self.name, self.age));
        });
    }
}
```

### The Execution Flow (Traced)

1. **eframe event loop** calls `App::update()` 60fps (or on events)
2. Your code runs, calling `ui.button()`, `ui.label()`, etc.
3. Each widget call:
   - Generates a unique ID
   - Checks previous frame's interaction state from `Memory`
   - Allocates space via `Placer`
   - Adds `Shape`s to the current layer's graphics list
   - Returns a `Response`
4. After your code completes:
   - `Context::tessellate()` converts all `Shape`s to `ClippedPrimitive`s (meshes)
   - Integration (eframe) uploads meshes to GPU
   - GPU renders triangles
5. Next frame, repeat

## Why This Matters

egui proves that:
1. **Immediate mode scales**: Not just for game debug UIs, but real applications (see Rerun Viewer)
2. **Rust enables new GUI patterns**: No garbage collector, no callbacks, but still safe and ergonomic
3. **Simplicity is achievable**: Compare egui to Qt/GTK/WPF complexity

**What it challenges**: The assumption that complex UIs require complex frameworks.

## Code Conventions Worth Noting

From `Cargo.toml` and source code:

- **No unsafe code**: `unsafe_code = "deny"` in workspace lints
- **Builder pattern everywhere**: Verbose but explicit
- **Closure-based regions**: `ui.horizontal(|ui| { ... })` instead of `begin_horizontal()` / `end_horizontal()`
- **Premultiplied alpha**: ALL colors, always
- **Logical points, not pixels**: DPI scaling handled via `pixels_per_point`
- **Top-left origin**: (0,0) is top-left, Y increases downward

## Entry Points for Exploration

If you want to understand how egui works:

1. Start at `examples/hello_world/src/main.rs` - see the whole pattern
2. Read `crates/egui/src/lib.rs` docs (lines 1-250)
3. Trace a button: `crates/egui/src/widgets/button.rs`
4. Understand Context: `crates/egui/src/context.rs` (start with `Context::run()`)
5. See tessellation: `crates/epaint/src/tessellator.rs`

## The Dependencies Question

egui is surprisingly **minimal**:
- Core `egui` crate: `emath`, `epaint`, `ahash` (no-std compatible)
- No async runtime
- No huge dependencies
- Can run in WASM

This is by design. Heavy dependencies live in `eframe` and `egui_extras` (images, SVG, etc.).

## Conclusion: What We Learned

egui is **not** a new GUI paradigm—immediate mode has existed since the 1980s. What's novel:

1. **Rust + immediate mode** works beautifully (no GC, safety, performance)
2. **Selective state retention** solves immediate mode's historical problems
3. **Clean separation**: math → painting → GUI logic → integration
4. **Real-world viability**: Powers actual production software

The codebase is **pragmatic**: not purely functional, not over-abstracted, but carefully designed around the constraints of 60fps UI updates and Rust's ownership model.

---

*This document describes egui v0.33.0 as it exists, not as it should be.*
