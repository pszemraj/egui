# egui Quick Reference

Fast lookup for common tasks and patterns.

## Basic Widgets

```rust
// Text
ui.label("Static text");
ui.heading("Large heading");
ui.monospace("Code text");
ui.strong("Bold text");
ui.weak("Faint text");
ui.hyperlink("https://egui.rs");

// Buttons
if ui.button("Click me").clicked() { }
if ui.small_button("Small").clicked() { }
ui.add_enabled(enabled, egui::Button::new("Conditional"));

// Input
ui.text_edit_singleline(&mut string);
ui.text_edit_multiline(&mut string);
ui.checkbox(&mut boolean, "Label");
ui.radio_value(&mut choice, value, "Label");

// Sliders
ui.add(egui::Slider::new(&mut value, 0..=100));
ui.add(egui::DragValue::new(&mut value));

// Images
ui.image(egui::include_image!("path.png"));
ui.image(egui::ImageSource::Bytes { bytes, uri });

// Separator
ui.separator();

// Spacing
ui.add_space(10.0);
```

## Layout

```rust
// Horizontal
ui.horizontal(|ui| {
    ui.label("Left");
    ui.label("Right");
});

// Vertical (default)
ui.vertical(|ui| {
    ui.label("Top");
    ui.label("Bottom");
});

// Columns
ui.columns(2, |columns| {
    columns[0].label("Column 1");
    columns[1].label("Column 2");
});

// Grid
egui::Grid::new("my_grid").show(ui, |ui| {
    ui.label("Row 1, Col 1");
    ui.label("Row 1, Col 2");
    ui.end_row();

    ui.label("Row 2, Col 1");
    ui.label("Row 2, Col 2");
    ui.end_row();
});

// Centering
ui.vertical_centered(|ui| {
    ui.label("Centered");
});

// Right-aligned
ui.with_layout(egui::Layout::right_to_left(egui::Align::Center), |ui| {
    ui.label("Right");
});
```

## Containers

```rust
// Window
egui::Window::new("My Window")
    .resizable(true)
    .default_size([400.0, 300.0])
    .show(ctx, |ui| {
        ui.label("Window content");
    });

// Panels
egui::SidePanel::left("left_panel")
    .default_width(200.0)
    .show(ctx, |ui| { /* ... */ });

egui::TopBottomPanel::top("top_panel").show(ctx, |ui| { /* ... */ });

egui::CentralPanel::default().show(ctx, |ui| { /* ... */ });

// Collapsing header
egui::CollapsingHeader::new("Click to expand").show(ui, |ui| {
    ui.label("Hidden content");
});

// Scroll area
egui::ScrollArea::vertical()
    .max_height(200.0)
    .show(ui, |ui| {
        for i in 0..100 {
            ui.label(format!("Item {}", i));
        }
    });

// Frame
egui::Frame::none()
    .fill(egui::Color32::from_rgb(40, 40, 40))
    .inner_margin(10.0)
    .show(ui, |ui| {
        ui.label("Framed content");
    });

// Group (with automatic background)
ui.group(|ui| {
    ui.label("Grouped content");
});
```

## Interaction

```rust
// Check response
let response = ui.button("Click");
if response.clicked() { }
if response.hovered() { }
if response.dragged() { }
if response.has_focus() { }
if response.lost_focus() { }
if response.changed() { }

// Custom interaction
let (response, painter) = ui.allocate_painter(size, Sense::click());
if response.clicked() {
    println!("Clicked at {:?}", response.interact_pointer_pos());
}

// Tooltips
ui.label("Hover me").on_hover_text("Tooltip text");

// Context menu
ui.label("Right-click me").context_menu(|ui| {
    if ui.button("Menu item").clicked() {
        ui.close_menu();
    }
});

// Drag and drop
let response = ui.label("Drag me");
if response.dragged() {
    egui::DragAndDrop::set_payload(ctx, my_data);
}
if let Some(data) = egui::DragAndDrop::payload::<MyData>(ctx) {
    // Use dropped data
}
```

## Keyboard Input

```rust
// Single key
if ui.input(|i| i.key_pressed(egui::Key::Escape)) { }

// With modifiers
if ui.input_mut(|i| i.consume_key(egui::Modifiers::COMMAND, egui::Key::S)) {
    save_file();
}

// Check modifier state
if ui.input(|i| i.modifiers.ctrl) { }
if ui.input(|i| i.modifiers.shift) { }
if ui.input(|i| i.modifiers.alt) { }

// Text input
let text = ui.input(|i| i.events.iter()
    .filter_map(|e| match e {
        egui::Event::Text(s) => Some(s.clone()),
        _ => None,
    })
    .collect::<String>());
```

## Styling

```rust
// Colors
ui.visuals_mut().override_text_color = Some(egui::Color32::RED);
ui.style_mut().visuals.widgets.inactive.bg_fill = egui::Color32::from_rgb(50, 50, 50);

// Spacing
ui.spacing_mut().item_spacing = egui::vec2(10.0, 5.0);
ui.spacing_mut().button_padding = egui::vec2(8.0, 4.0);

// Fonts
ui.style_mut().text_styles.insert(
    egui::TextStyle::Body,
    egui::FontId::proportional(18.0)
);

// Temporary style change
ui.scope(|ui| {
    ui.spacing_mut().item_spacing.x = 0.0;
    ui.label("Tight");
    ui.label("Spacing");
});

// Colored text
ui.label(egui::RichText::new("Colored").color(egui::Color32::RED));
ui.label(egui::RichText::new("Large").size(24.0));
ui.label(egui::RichText::new("Styled").strong().italics());
```

## Custom Painting

```rust
// Allocate space and get painter
let (response, painter) = ui.allocate_painter(egui::vec2(100.0, 100.0), Sense::hover());

// Draw shapes
painter.circle_filled(center, radius, egui::Color32::RED);
painter.rect_filled(rect, corner_radius, egui::Color32::BLUE);
painter.line_segment([from, to], stroke);
painter.text(pos, align, "Text", font_id, color);

// Access painter in existing rect
let painter = ui.painter_at(rect);
painter.circle_stroke(center, radius, stroke);
```

## Memory / State

```rust
// Per-widget state
let my_state = ui.memory_mut(|mem| {
    mem.data.get_temp_mut_or_default::<MyState>(ui.id())
});

// Global state
ctx.data_mut(|d| d.insert_temp(egui::Id::NULL, my_global));
let global = ctx.data(|d| d.get_temp::<Global>(egui::Id::NULL));

// Persistent state (survives app restart with persistence feature)
ctx.data_mut(|d| d.insert_persisted(id, state));
let state = ctx.data(|d| d.get_persisted::<State>(id));

// Focus
ctx.memory_mut(|mem| mem.request_focus(id));
let has_focus = ctx.memory(|mem| mem.has_focus(id));
```

## Common Patterns

### Menu Bar
```rust
egui::TopBottomPanel::top("menu").show(ctx, |ui| {
    egui::menu::bar(ui, |ui| {
        ui.menu_button("File", |ui| {
            if ui.button("Open").clicked() {
                ui.close_menu();
            }
        });
    });
});
```

### Modal Dialog
```rust
if show_dialog {
    egui::Window::new("Confirm")
        .collapsible(false)
        .resizable(false)
        .show(ctx, |ui| {
            ui.label("Are you sure?");
            ui.horizontal(|ui| {
                if ui.button("Yes").clicked() {
                    confirmed = true;
                    show_dialog = false;
                }
                if ui.button("No").clicked() {
                    show_dialog = false;
                }
            });
        });
}
```

### Tab Bar
```rust
ui.horizontal(|ui| {
    ui.selectable_value(&mut active_tab, Tab::First, "First");
    ui.selectable_value(&mut active_tab, Tab::Second, "Second");
});
ui.separator();
match active_tab {
    Tab::First => { /* content */ }
    Tab::Second => { /* content */ }
}
```

### Table
```rust
use egui_extras::{TableBuilder, Column};

TableBuilder::new(ui)
    .column(Column::auto())
    .column(Column::remainder())
    .header(20.0, |mut header| {
        header.col(|ui| { ui.heading("Name"); });
        header.col(|ui| { ui.heading("Value"); });
    })
    .body(|mut body| {
        for item in &items {
            body.row(18.0, |mut row| {
                row.col(|ui| { ui.label(&item.name); });
                row.col(|ui| { ui.label(&item.value); });
            });
        }
    });
```

## Debugging

```rust
// Show widget rectangles on hover
ctx.set_debug_on_hover(true);

// Memory viewer
egui::Window::new("Memory").show(ctx, |ui| {
    ctx.memory(|mem| ui.label(format!("{:#?}", mem)));
});

// Settings UI
egui::Window::new("Settings").show(ctx, |ui| {
    ctx.settings_ui(ui);
});

// Inspection UI
egui::Window::new("Inspection").show(ctx, |ui| {
    ctx.inspection_ui(ui);
});
```

## Useful Constants

```rust
// Colors
egui::Color32::RED
egui::Color32::TRANSPARENT
egui::Color32::from_rgb(r, g, b)
egui::Color32::from_rgba_premultiplied(r, g, b, a)

// Vectors
egui::vec2(x, y)
egui::Vec2::ZERO
egui::Vec2::splat(v)

// Positions
egui::pos2(x, y)
egui::Pos2::ZERO

// Rectangles
egui::Rect::from_min_size(min, size)
egui::Rect::from_center_size(center, size)

// Keys
egui::Key::Enter
egui::Key::Escape
egui::Key::Space
egui::Key::Tab

// Modifiers
egui::Modifiers::NONE
egui::Modifiers::CTRL
egui::Modifiers::SHIFT
egui::Modifiers::ALT
egui::Modifiers::COMMAND  // Ctrl on Windows/Linux, Cmd on Mac
```

---

**More details**: See the [full guides](../guides/) or [API docs](https://docs.rs/egui).
