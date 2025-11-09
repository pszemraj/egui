# egui Architecture Documentation

Deep technical documentation on how egui works internally.

## Overview

egui is an immediate mode GUI library with a carefully layered architecture. Understanding this architecture helps you:
- Debug issues effectively
- Build custom integrations
- Contribute to the codebase
- Make performance optimizations
- Understand design decisions

## Available Documentation

### [Codebase Analysis](CODEBASE_ANALYSIS.md)
**Complete technical deep-dive** covering:

- **What Problem egui Solves**: The immediate mode philosophy
- **Core Insight**: How immediate mode with retained state works
- **Three-Phase Architecture**: Input → UI Logic → Tessellation
- **Crate Layering**: emath → ecolor → epaint → egui → integrations
- **Detailed Component Analysis**:
  - Context: The brain of egui
  - Ui: Layout and widget placement
  - Response: Interaction detection
  - Memory: State persistence
  - Painter: Shape tessellation
- **Novel Patterns**:
  - The Sense system
  - Smart repainting
  - Premultiplied alpha
  - The RwLock pattern
  - Frame delay solutions
- **Performance Characteristics**
- **What's Incomplete/Experimental**
- **Entry Points for Code Exploration**

**Read this first** to understand egui from first principles.

---

## Architecture Overview

### Dependency Hierarchy

```
emath (2D math: Vec2, Pos2, Rect)
  ↓
ecolor (Color types: Color32, Rgba)
  ↓
epaint (Shapes → Triangles: Tessellator, Mesh)
  ↓
egui (Immediate mode UI: Context, Ui, Widgets)
  ↓
Integrations (egui-winit, egui_glow, egui-wgpu)
  ↓
eframe (Complete framework)
```

### Execution Flow

Every frame follows this pattern:

```rust
loop {
    // 1. GATHER INPUT
    let raw_input = gather_mouse_keyboard_touch();

    // 2. RUN UI LOGIC
    let output = ctx.run(raw_input, |ctx| {
        // Your application code here
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.button("Click me");
        });
    });

    // 3. TESSELLATE & RENDER
    let primitives = ctx.tessellate(output.shapes, output.pixels_per_point);
    render_to_gpu(primitives);
}
```

### Key Data Structures

#### Context
- Single source of truth for the frame
- Stores `Memory` (persistent state)
- Manages viewports (OS windows)
- Tracks repainting needs
- Thread-safe via RwLock

#### Memory
- Persists between frames
- Stores:
  - Widget state (scroll positions, text cursors)
  - Window positions/sizes
  - User data via `IdTypeMap`
  - Focus state
  - Open popups

#### Ui
- Represents a screen region
- Has a `Placer` for layout
- Has a `Painter` for drawing
- Creates child Ui's for nested layouts

#### Response
- Returned by every widget
- Contains interaction state via bitflags
- Provides methods like:
  - `clicked()`, `hovered()`, `dragged()`
  - `changed()`, `has_focus()`, `lost_focus()`

### Widget Lifecycle

Widgets don't have objects—they're pure functions:

```rust
// This runs every frame
let response = ui.button("Click me");
if response.clicked() {
    // Handle click
}
```

**Under the hood** (simplified):

1. Generate stable ID from context
2. Allocate space via `Placer`
3. Load previous state from `Memory` (if any)
4. Check for interactions (clicks, hovers)
5. Add `Shape`s to the painter
6. Store state back to `Memory`
7. Return `Response`

### State Persistence

egui uses **stable IDs** to persist state:

```rust
// Automatic ID from position in UI hierarchy
ui.text_edit_singleline(&mut text);

// Explicit ID for stability
ui.text_edit_singleline(&mut text).id_source("my_editor");

// Window positions use window title as ID
egui::Window::new("Settings").show(ctx, |ui| { /* ... */ });
```

State is stored in `Memory::data` as an `IdTypeMap`:

```rust
ctx.memory_mut(|mem| {
    mem.data.insert_persisted(id, my_state);
});

let state: Option<MyState> = ctx.memory(|mem| {
    mem.data.get_persisted(id)
});
```

### Tessellation Pipeline

1. **Shapes**: High-level primitives
   - `Shape::Circle`, `Shape::Rect`, `Shape::Text`, etc.
2. **Clipped Shapes**: Shapes with clip rectangles
3. **Tessellator**: Converts shapes to meshes
   - Uses precomputed vertices for circles
   - Flattens Bézier curves
   - Adds feathering for anti-aliasing
4. **Meshes**: Triangle vertex/index arrays
5. **GPU Upload**: Integration renders meshes

### Repaint Logic

egui is **lazy** about repainting:

- Only repaints when:
  - User input occurs
  - Animations are running
  - `ctx.request_repaint()` is called
- When idle, CPU usage → 0
- Each repaint request schedules 2 frames (for settling)

### Font System

- Fonts stored in a `TextureAtlas`
- Glyphs rasterized on-demand
- Font texture ID is always `TextureId::default()`
- Multiple fonts supported via `FontDefinitions`

### Layer System

Widgets are painted in layers:

- **Background**: Behind everything
- **PanelResizeLine**: Panel resize handles
- **LayerId::new(Order, Id)**: Custom layers
- **Debug**: Debug overlays
- **Tooltip**: Always on top

## Design Principles

### No Unsafe Code
```toml
[workspace.lints.rust]
unsafe_code = "deny"
```

All safety guarantees via Rust's type system.

### Minimal Dependencies
Core `egui` only depends on:
- `emath`, `epaint` (internal)
- `ahash` (fast hashing)

Everything else is optional.

### Premultiplied Alpha Everywhere
All colors use premultiplied alpha for correct blending:
```rust
// RGB values are pre-multiplied by alpha
Color32::from_rgba_premultiplied(r, g, b, a)
```

### Builder Pattern
Most widgets use builders:
```rust
egui::TextEdit::multiline(&mut text)
    .font(egui::TextStyle::Monospace)
    .desired_rows(10)
    .hint_text("Enter code...")
    .show(ui);
```

### Closures for Regions
Avoids `begin`/`end` confusion:
```rust
ui.horizontal(|ui| {
    ui.label("Left");
    ui.label("Right");
}); // Automatic end
```

## Performance Characteristics

### Frame Budget (60fps = 16.6ms)
Typical breakdown:
- **Layout**: 0.5-2ms (incremental)
- **Tessellation**: 0.5-2ms (shapes → triangles)
- **GPU Upload**: 0.1-1ms
- **Rendering**: 1-5ms (GPU-dependent)

**Total UI overhead**: ~1-5ms for typical apps

### Memory Usage
- **Context + Memory**: ~few KB
- **Meshes**: Transient (recreated each frame)
- **Texture Atlas**: Font + user textures (~1-10 MB)

### Optimization Tips
1. **Avoid huge ScrollAreas**: Layout is O(n) per frame
2. **Cache expensive layouters**: Syntax highlighting
3. **Use `ui.is_rect_visible()`**: Skip hidden widgets
4. **Limit shapes**: Thousands of shapes → slow tessellation

## Common Patterns

### Global State
```rust
ctx.data_mut(|d| d.insert_temp(Id::NULL, my_global_state));
let state = ctx.data(|d| d.get_temp::<MyState>(Id::NULL));
```

### Frame Caching
```rust
use egui::util::cache::{ComputerMut, FrameCache};

ctx.memory_mut(|mem| {
    let cache = mem.caches.cache::<MyCache>();
    let result = cache.get(input);
});
```

### Custom Painting
```rust
let response = ui.allocate_response(size, Sense::click());
let painter = ui.painter_at(response.rect);
painter.circle_filled(center, radius, color);
```

## Source Code Navigation

### Start Here
- `crates/egui/src/lib.rs` - Overview and examples
- `crates/egui/src/context.rs` - The `Context` struct
- `crates/egui/src/ui.rs` - The `Ui` struct
- `crates/egui/src/response.rs` - The `Response` struct

### Widget Examples
- `crates/egui/src/widgets/button.rs` - Simple widget
- `crates/egui/src/widgets/text_edit/` - Complex stateful widget

### Rendering
- `crates/epaint/src/tessellator.rs` - Shape tessellation
- `crates/epaint/src/mesh.rs` - Mesh structure

### Integration Examples
- `crates/egui_glow/src/painter.rs` - OpenGL renderer
- `crates/egui-winit/src/lib.rs` - Window integration

## Debugging Tips

### Enable Debug Overlays
```rust
ctx.set_debug_on_hover(true);  // Show widget rects
```

### Inspect Memory
```rust
egui::Window::new("Debug").show(ctx, |ui| {
    ctx.memory(|mem| {
        ui.label(format!("Widgets: {}", mem.data.len()));
    });
});
```

### Trace Execution
```rust
puffin::profile_function!();  // Enable puffin profiler
```

### Check Repaint Causes
```rust
ctx.memory(|mem| {
    for cause in mem.repaint_causes() {
        println!("Repaint: {:?}", cause);
    }
});
```

## Further Reading

- **[Codebase Analysis](CODEBASE_ANALYSIS.md)** - Complete technical deep-dive
- **[Main README](../../README.md)** - Project overview
- **[ARCHITECTURE.md](../../ARCHITECTURE.md)** - Crate overview
- **API Docs**: <https://docs.rs/egui>
- **Source Code**: <https://github.com/emilk/egui>

---

**Questions about architecture?** Ask in [GitHub Discussions](https://github.com/emilk/egui/discussions).
