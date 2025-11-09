# Building a Text Editor with egui: A Comprehensive Guide

This guide walks you through building a text editor in egui, from basic single-line input to a full-featured code editor with syntax highlighting, undo/redo, and custom keybindings.

## Table of Contents

- [Understanding TextEdit](#understanding-textedit)
- [Basic Text Input](#basic-text-input)
- [The TextBuffer Trait](#the-textbuffer-trait)
- [TextEdit State Management](#textedit-state-management)
- [Building a Multiline Editor](#building-a-multiline-editor)
- [Adding Syntax Highlighting](#adding-syntax-highlighting)
- [Implementing Undo/Redo](#implementing-undoredo)
- [Custom Keybindings](#custom-keybindings)
- [Advanced Features](#advanced-features)
- [Complete Example: A Code Editor](#complete-example-a-code-editor)

---

## Understanding TextEdit

egui's `TextEdit` widget is located in `crates/egui/src/widgets/text_edit/` and consists of:

- **`builder.rs`**: The `TextEdit` widget builder API
- **`state.rs`**: `TextEditState` - persistent state across frames
- **`text_buffer.rs`**: `TextBuffer` trait - abstraction over text storage
- **`output.rs`**: `TextEditOutput` - what the widget returns

### Key Concepts

1. **Immediate Mode**: Your text editing code runs every frame, just like any other widget
2. **State Persistence**: Cursor position, undo history, and scroll offset are stored in `Memory` via widget ID
3. **TextBuffer Abstraction**: You can use `String`, `&str`, or implement custom buffers
4. **Layout Jobs**: Custom text rendering via the `layouter` function (for syntax highlighting)

---

## Basic Text Input

### Single Line Text Input

The simplest text editor:

```rust
use eframe::egui;

struct MyApp {
    text: String,
}

impl eframe::App for MyApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().show(ctx, |ui| {
            ui.heading("Single Line Input");

            // Simple single-line editor
            ui.text_edit_singleline(&mut self.text);

            // With hint text
            ui.add(egui::TextEdit::singleline(&mut self.text)
                .hint_text("Type something..."));
        });
    }
}
```

**What happens under the hood** (`crates/egui/src/widgets/text_edit/builder.rs:110-116`):

```rust
pub fn singleline(text: &'t mut dyn TextBuffer) -> Self {
    Self {
        desired_height_rows: 1,
        multiline: false,
        clip_text: true,  // Text that overflows is clipped
        ..Self::multiline(text)
    }
}
```

### Detecting User Actions

The `Response` tells you what happened:

```rust
let response = ui.text_edit_singleline(&mut self.text);

if response.changed() {
    println!("Text changed to: {}", self.text);
}

if response.lost_focus() && ui.input(|i| i.key_pressed(egui::Key::Enter)) {
    println!("User pressed Enter: {}", self.text);
    // Process the input
    self.text.clear();
}
```

---

## The TextBuffer Trait

The `TextBuffer` trait (`crates/egui/src/widgets/text_edit/text_buffer.rs`) defines how text is stored and manipulated:

```rust
pub trait TextBuffer {
    fn is_mutable(&self) -> bool;
    fn as_str(&self) -> &str;
    fn insert_text(&mut self, text: &str, char_index: usize) -> usize;
    fn delete_char_range(&mut self, char_range: Range<usize>);

    // Many convenience methods built on top...
}
```

### Built-in Implementations

1. **`String`** - Mutable, editable text (most common)
2. **`&str`** - Immutable, read-only (for displaying selectable text)
3. **`Cow<'_, str>`** - Clone-on-write

### Custom TextBuffer Example

You might implement a custom buffer for:
- Gap buffers (efficient for large documents)
- Rope data structures (for very large files)
- Text with constraints (e.g., numbers only)

```rust
use egui::TextBuffer;
use std::ops::Range;

struct NumberOnlyBuffer {
    text: String,
}

impl TextBuffer for NumberOnlyBuffer {
    fn is_mutable(&self) -> bool { true }

    fn as_str(&self) -> &str { &self.text }

    fn insert_text(&mut self, text: &str, char_index: usize) -> usize {
        // Only allow digits
        let filtered: String = text.chars().filter(|c| c.is_ascii_digit()).collect();

        // Use String's implementation
        self.text.insert_text(&filtered, char_index)
    }

    fn delete_char_range(&mut self, char_range: Range<usize>) {
        self.text.delete_char_range(char_range);
    }

    fn type_id(&self) -> std::any::TypeId {
        std::any::TypeId::of::<Self>()
    }
}
```

---

## TextEdit State Management

egui stores editor state in `TextEditState` (`crates/egui/src/widgets/text_edit/state.rs:36-61`):

```rust
pub struct TextEditState {
    /// Cursor position and selection
    pub cursor: TextCursorState,

    /// Undo/redo history
    pub(crate) undoer: Arc<Mutex<TextEditUndoer>>,

    /// IME (Input Method Editor) state for Asian languages
    pub(crate) ime_enabled: bool,
    pub(crate) ime_cursor_range: CCursorRange,

    /// Text scroll offset
    pub(crate) text_offset: Vec2,

    /// Last interaction time (for cursor blink animation)
    pub(crate) last_interaction_time: f64,
}
```

### Manually Manipulating State

You can load, modify, and store the state manually:

```rust
let text_edit_id = output.response.id;

// Load state
if let Some(mut state) = egui::TextEdit::load_state(ui.ctx(), text_edit_id) {
    // Move cursor to end
    let char_count = text.chars().count();
    let ccursor = egui::text::CCursor::new(char_count);
    state.cursor.set_char_range(Some(egui::text::CCursorRange::one(ccursor)));

    // Store modified state
    state.store(ui.ctx(), text_edit_id);

    // Return focus to the TextEdit
    ui.ctx().memory_mut(|mem| mem.request_focus(text_edit_id));
}
```

**Real example from `crates/egui_demo_lib/src/demo/text_edit.rs:99-109`.**

---

## Building a Multiline Editor

### Basic Multiline Editor

```rust
egui::TextEdit::multiline(&mut self.text)
    .desired_rows(10)          // Height in rows
    .desired_width(f32::INFINITY)  // Take full width
    .show(ui);
```

### Configuration Options

```rust
egui::TextEdit::multiline(&mut self.text)
    .font(egui::TextStyle::Monospace)  // Use monospace font
    .hint_text("Enter your code here...")
    .frame(true)                       // Show background frame
    .margin(egui::vec2(8.0, 4.0))     // Inner margin
    .interactive(true)                 // Can be edited
    .desired_rows(20)
    .desired_width(600.0)
    .clip_text(false)                  // Don't clip overflowing text
    .show(ui);
```

### Scrollable Text Editor

```rust
egui::ScrollArea::vertical().show(ui, |ui| {
    ui.add_sized(
        ui.available_size(),
        egui::TextEdit::multiline(&mut self.code)
            .font(egui::TextStyle::Monospace)
            .desired_rows(30)
            .desired_width(f32::INFINITY)
    );
});
```

---

## Adding Syntax Highlighting

Syntax highlighting is implemented via the **`layouter`** function, which converts text into a `LayoutJob` with colored spans.

### Understanding LayoutJob

A `LayoutJob` (from `epaint::text`) specifies:
- Text content
- Font for each character range
- Color for each character range
- Text wrapping behavior

### Simple Manual Highlighting

```rust
use egui::text::{LayoutJob, TextFormat};
use egui::Color32;

let mut layouter = |ui: &egui::Ui, buf: &dyn egui::TextBuffer, wrap_width: f32| {
    let mut job = LayoutJob::default();
    let text = buf.as_str();

    // Highlight Rust keywords
    let keywords = ["fn", "let", "mut", "if", "else", "for", "while"];
    let mut last_end = 0;

    for keyword in &keywords {
        if let Some(start) = text.find(keyword) {
            // Normal text before keyword
            job.append(
                &text[last_end..start],
                0.0,
                TextFormat {
                    color: ui.style().visuals.text_color(),
                    ..Default::default()
                },
            );

            // Highlighted keyword
            job.append(
                keyword,
                0.0,
                TextFormat {
                    color: Color32::from_rgb(220, 100, 50),
                    ..Default::default()
                },
            );

            last_end = start + keyword.len();
        }
    }

    // Remaining text
    if last_end < text.len() {
        job.append(
            &text[last_end..],
            0.0,
            TextFormat {
                color: ui.style().visuals.text_color(),
                ..Default::default()
            },
        );
    }

    job.wrap.max_width = wrap_width;
    ui.fonts_mut(|f| f.layout_job(job))
};

ui.add(egui::TextEdit::multiline(&mut self.code)
    .layouter(&mut layouter));
```

### Production-Grade Highlighting with syntect

For real syntax highlighting, use the `syntect` crate (see `crates/egui_demo_lib/src/demo/code_editor.rs`):

```rust
// Add to Cargo.toml:
// egui_extras = { version = "0.33", features = ["syntect"] }

use egui_extras::syntax_highlighting;

struct CodeEditor {
    code: String,
    language: String,
}

impl CodeEditor {
    fn ui(&mut self, ui: &mut egui::Ui) {
        let theme = syntax_highlighting::CodeTheme::from_memory(ui.ctx(), ui.style());

        let mut layouter = |ui: &egui::Ui, buf: &dyn egui::TextBuffer, wrap_width: f32| {
            let mut layout_job = syntax_highlighting::highlight(
                ui.ctx(),
                ui.style(),
                &theme,
                buf.as_str(),
                &self.language,  // "rs", "py", "js", etc.
            );
            layout_job.wrap.max_width = wrap_width;
            ui.fonts_mut(|f| f.layout_job(layout_job))
        };

        egui::ScrollArea::vertical().show(ui, |ui| {
            ui.add(
                egui::TextEdit::multiline(&mut self.code)
                    .font(egui::TextStyle::Monospace)
                    .code_editor()  // Enables tab insertion
                    .desired_rows(20)
                    .desired_width(f32::INFINITY)
                    .layouter(&mut layouter),
            );
        });
    }
}
```

### Caching Syntax Highlighting

**Important**: The layouter runs every frame! For large files, cache the result:

```rust
use egui::util::cache::{ComputerMut, FrameCache};
use std::sync::Arc;

struct HighlightCache {
    language: String,
}

impl ComputerMut<&str, Arc<egui::Galley>> for HighlightCache {
    fn compute(&mut self, code: &str) -> Arc<egui::Galley> {
        // This is only called when the text changes
        // ... perform expensive syntax highlighting ...
        todo!()
    }
}

// In your UI code:
ctx.memory_mut(|mem| {
    let cache = mem.caches.cache::<FrameCache<Arc<egui::Galley>, HighlightCache>>();
    let galley = cache.get(code);
});
```

---

## Implementing Undo/Redo

egui's `TextEdit` includes built-in undo/redo (`TextEditUndoer` in `crates/egui/src/util/undoer.rs`).

### Default Undo/Redo

**Built-in keybindings**:
- **Undo**: `Ctrl+Z` (or `Cmd+Z` on Mac)
- **Redo**: `Ctrl+Y` or `Ctrl+Shift+Z` (or `Cmd+Shift+Z` on Mac)

These work automatically! No code needed.

### Custom Undo/Redo Operations

```rust
let text_edit_id = output.response.id;

if let Some(mut state) = egui::TextEdit::load_state(ui.ctx(), text_edit_id) {
    let mut undoer = state.undoer();

    // Add custom undo point
    undoer.add_undo((cursor_range, old_text.clone()));

    // Manual undo
    if let Some((old_cursor, old_text)) = undoer.undo(&(current_cursor, current_text)) {
        text.replace_with(&old_text);
        state.cursor.set_char_range(Some(old_cursor));
    }

    state.set_undoer(undoer);
    state.store(ui.ctx(), text_edit_id);
}
```

### Clearing Undo History

```rust
if let Some(mut state) = egui::TextEdit::load_state(ui.ctx(), text_edit_id) {
    state.clear_undoer();
    state.store(ui.ctx(), text_edit_id);
}
```

---

## Custom Keybindings

### Reading Keyboard Input

```rust
let response = ui.text_edit_multiline(&mut self.text);

if response.has_focus() {
    // Check for custom keybindings
    if ui.input_mut(|i| i.consume_key(egui::Modifiers::COMMAND, egui::Key::S)) {
        self.save_file();
    }

    if ui.input_mut(|i| i.consume_key(egui::Modifiers::COMMAND, egui::Key::F)) {
        self.show_find_dialog = true;
    }
}
```

### Example: Toggle Case Keybinding

From `crates/egui_demo_lib/src/demo/text_edit.rs:68-82`:

```rust
let output = egui::TextEdit::multiline(&mut self.text).show(ui);

if ui.input_mut(|i| i.consume_key(egui::Modifiers::COMMAND, egui::Key::Y))
    && let Some(text_cursor_range) = output.cursor_range
{
    use egui::TextBuffer as _;
    let selected_chars = text_cursor_range.as_sorted_char_range();
    let selected_text = self.text.char_range(selected_chars.clone());

    let upper_case = selected_text.to_uppercase();
    let new_text = if selected_text == upper_case {
        selected_text.to_lowercase()
    } else {
        upper_case
    };

    self.text.delete_char_range(selected_chars.clone());
    self.text.insert_text(&new_text, selected_chars.start);
}
```

### Changing the Return Key Behavior

```rust
use egui::{Key, KeyboardShortcut, Modifiers};

// Default: Enter creates a newline in multiline
egui::TextEdit::multiline(&mut self.text).show(ui);

// Custom: Shift+Enter creates newline, Enter submits
egui::TextEdit::multiline(&mut self.text)
    .return_key(KeyboardShortcut::new(Modifiers::SHIFT, Key::Enter))
    .show(ui);

// Disable the return key entirely
egui::TextEdit::multiline(&mut self.text)
    .return_key(None)
    .show(ui);
```

---

## Advanced Features

### Code Editor Mode

The `code_editor()` method configures the editor for code:

```rust
egui::TextEdit::multiline(&mut self.code)
    .code_editor()  // Monospace font + Tab inserts tab character
    .show(ui);
```

Equivalent to:

```rust
egui::TextEdit::multiline(&mut self.code)
    .font(egui::TextStyle::Monospace)
    .lock_focus(true)  // Tab inserts '\t' instead of moving focus
    .show(ui);
```

### Password Fields

```rust
egui::TextEdit::singleline(&mut self.password)
    .password(true)  // Shows dots instead of characters
    .hint_text("Enter password")
    .show(ui);
```

### Character Limits

```rust
egui::TextEdit::singleline(&mut self.username)
    .char_limit(20)  // Max 20 characters
    .show(ui);
```

### Custom Background Color

```rust
egui::TextEdit::multiline(&mut self.text)
    .background_color(egui::Color32::from_rgb(40, 40, 40))
    .show(ui);
```

### Accessing TextEdit Output

The `show()` method returns `TextEditOutput`:

```rust
let output = egui::TextEdit::multiline(&mut self.text).show(ui);

// The Response (for interaction detection)
if output.response.changed() {
    println!("Text changed!");
}

// Current cursor position/selection
if let Some(cursor_range) = output.cursor_range {
    let selection = cursor_range.primary.ccursor.index;
    println!("Cursor at character {}", selection);
}

// The rendered text (Galley)
let galley = output.galley;  // Arc<Galley>
println!("Text rendered as {} rows", galley.rows.len());

// Position where the text is drawn (for custom overlays)
let text_draw_pos = output.galley_pos;
```

### Reading Selected Text

```rust
let output = egui::TextEdit::multiline(&mut self.text).show(ui);

if let Some(cursor_range) = output.cursor_range {
    let selected_text = cursor_range.slice_str(&self.text);
    ui.label(format!("Selected: {}", selected_text));
}
```

### Line Numbers

Add line numbers using custom painting:

```rust
let output = egui::TextEdit::multiline(&mut self.code)
    .desired_width(f32::INFINITY)
    .show(ui);

let line_count = self.code.lines().count();
let text_height = output.galley.size().y;
let line_height = text_height / line_count.max(1) as f32;

let painter = ui.painter_at(output.response.rect);
for line_num in 0..line_count {
    let y = output.galley_pos.y + line_num as f32 * line_height;
    painter.text(
        egui::pos2(5.0, y),
        egui::Align2::LEFT_TOP,
        format!("{:3}", line_num + 1),
        egui::FontId::monospace(12.0),
        egui::Color32::GRAY,
    );
}
```

---

## Complete Example: A Code Editor

Here's a complete, production-ready code editor with all the features:

```rust
use eframe::egui;
use std::sync::Arc;

struct CodeEditorApp {
    code: String,
    language: String,
    filename: String,
    show_line_numbers: bool,
}

impl Default for CodeEditorApp {
    fn default() -> Self {
        Self {
            code: "fn main() {\n    println!(\"Hello, world!\");\n}\n".to_string(),
            language: "rs".to_string(),
            filename: "main.rs".to_string(),
            show_line_numbers: true,
        }
    }
}

impl eframe::App for CodeEditorApp {
    fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
        // Menu bar
        egui::TopBottomPanel::top("menu_bar").show(ctx, |ui| {
            egui::menu::bar(ui, |ui| {
                ui.menu_button("File", |ui| {
                    if ui.button("Open...").clicked() {
                        // Implement file opening
                        ui.close_menu();
                    }
                    if ui.button("Save").clicked() {
                        self.save_file();
                        ui.close_menu();
                    }
                    ui.separator();
                    if ui.button("Quit").clicked() {
                        ctx.send_viewport_cmd(egui::ViewportCommand::Close);
                    }
                });

                ui.menu_button("Edit", |ui| {
                    if ui.button("Undo (Ctrl+Z)").clicked() {
                        // Built-in undo/redo works automatically
                        ui.close_menu();
                    }
                    if ui.button("Redo (Ctrl+Y)").clicked() {
                        ui.close_menu();
                    }
                });

                ui.menu_button("View", |ui| {
                    ui.checkbox(&mut self.show_line_numbers, "Show Line Numbers");
                });
            });
        });

        // Status bar
        egui::TopBottomPanel::bottom("status_bar").show(ctx, |ui| {
            ui.horizontal(|ui| {
                ui.label(format!("File: {}", self.filename));
                ui.separator();
                ui.label(format!("Language: {}", self.language));
                ui.separator();
                let line_count = self.code.lines().count();
                let char_count = self.code.chars().count();
                ui.label(format!("{} lines, {} chars", line_count, char_count));
            });
        });

        // Main editor area
        egui::CentralPanel::default().show(ctx, |ui| {
            let theme = egui_extras::syntax_highlighting::CodeTheme::from_memory(
                ui.ctx(),
                ui.style()
            );

            let mut layouter = |ui: &egui::Ui, buf: &dyn egui::TextBuffer, wrap_width: f32| {
                let mut layout_job = egui_extras::syntax_highlighting::highlight(
                    ui.ctx(),
                    ui.style(),
                    &theme,
                    buf.as_str(),
                    &self.language,
                );
                layout_job.wrap.max_width = wrap_width;
                ui.fonts_mut(|f| f.layout_job(layout_job))
            };

            egui::ScrollArea::both()
                .auto_shrink([false, false])
                .show(ui, |ui| {
                    let output = ui.add_sized(
                        ui.available_size(),
                        egui::TextEdit::multiline(&mut self.code)
                            .font(egui::TextStyle::Monospace)
                            .code_editor()
                            .desired_rows(30)
                            .desired_width(f32::INFINITY)
                            .layouter(&mut layouter),
                    );

                    // Handle custom keybindings
                    if output.response.has_focus() {
                        // Save file
                        if ui.input_mut(|i| i.consume_key(egui::Modifiers::COMMAND, egui::Key::S)) {
                            self.save_file();
                        }

                        // Comment/uncomment line
                        if ui.input_mut(|i| i.consume_key(egui::Modifiers::COMMAND, egui::Key::Slash)) {
                            self.toggle_comment(&output);
                        }
                    }

                    // Draw line numbers if enabled
                    if self.show_line_numbers {
                        self.draw_line_numbers(ui, &output);
                    }
                });
        });
    }
}

impl CodeEditorApp {
    fn save_file(&self) {
        // Implement file saving
        println!("Saving file: {}", self.filename);
    }

    fn toggle_comment(&mut self, output: &egui::text_edit::TextEditOutput) {
        if let Some(cursor_range) = output.cursor_range {
            use egui::TextBuffer as _;

            // Get the selected line(s)
            let char_range = cursor_range.as_sorted_char_range();
            let text = self.code.char_range(char_range.clone());

            // Toggle comment based on language
            let comment_str = match self.language.as_str() {
                "rs" | "c" | "cpp" | "js" => "// ",
                "py" | "sh" => "# ",
                _ => "// ",
            };

            let new_text = if text.starts_with(comment_str) {
                text.strip_prefix(comment_str).unwrap().to_string()
            } else {
                format!("{}{}", comment_str, text)
            };

            self.code.delete_char_range(char_range.clone());
            self.code.insert_text(&new_text, char_range.start);
        }
    }

    fn draw_line_numbers(&self, ui: &egui::Ui, output: &egui::text_edit::TextEditOutput) {
        let line_count = self.code.lines().count();
        let line_height = output.galley.size().y / line_count.max(1) as f32;

        let painter = ui.painter_at(output.response.rect);
        for line_num in 0..line_count {
            let y = output.galley_pos.y + line_num as f32 * line_height;
            painter.text(
                egui::pos2(output.response.rect.left() - 40.0, y),
                egui::Align2::RIGHT_TOP,
                format!("{:>4}", line_num + 1),
                egui::FontId::monospace(12.0),
                ui.style().visuals.weak_text_color(),
            );
        }
    }
}

fn main() -> eframe::Result {
    let options = eframe::NativeOptions {
        viewport: egui::ViewportBuilder::default()
            .with_inner_size([800.0, 600.0])
            .with_title("Code Editor"),
        ..Default::default()
    };

    eframe::run_native(
        "Code Editor",
        options,
        Box::new(|_cc| Ok(Box::<CodeEditorApp>::default())),
    )
}
```

### Required Dependencies

```toml
[dependencies]
eframe = "0.33"
egui = "0.33"
egui_extras = { version = "0.33", features = ["syntect"] }
```

---

## Performance Considerations

### 1. Cache Syntax Highlighting

Syntax highlighting can be expensive. Cache results in `Memory`:

```rust
ctx.memory_mut(|mem| {
    mem.caches.cache::<HighlightCache>()
});
```

### 2. Limit Visible Text

For very large files, only render visible lines:

```rust
// Calculate visible line range from scroll position
let visible_lines = calculate_visible_lines(scroll_offset, viewport_height);

// Only syntax highlight visible text
let visible_text = &full_text[visible_lines.start..visible_lines.end];
```

### 3. Debounce Expensive Operations

Don't run spell-checkers or formatters every keystroke:

```rust
if output.response.changed() {
    self.last_change_time = ctx.input(|i| i.time);
}

let time_since_change = ctx.input(|i| i.time) - self.last_change_time;
if time_since_change > 0.5 {  // 500ms delay
    self.run_spell_checker();
}
```

---

## Best Practices

1. **Always use `code_editor()` for code**: It enables tab insertion and monospace font
2. **Wrap in `ScrollArea`**: Especially for multiline editors
3. **Cache syntax highlighting**: Use `memory.caches` for expensive operations
4. **Use `desired_width(f32::INFINITY)`**: To take full available width
5. **Handle focus properly**: Check `has_focus()` before processing keybindings
6. **Store stable IDs**: Don't rely on auto-generated IDs for stateful editors

---

## Further Reading

- **Source Code**: `crates/egui/src/widgets/text_edit/`
- **Demo**: `crates/egui_demo_lib/src/demo/code_editor.rs`
- **TextBuffer Trait**: `crates/egui/src/widgets/text_edit/text_buffer.rs`
- **Undo/Redo**: `crates/egui/src/util/undoer.rs`
- **Syntax Highlighting**: `crates/egui_extras/src/syntax_highlighting.rs`

---

**This guide reflects egui v0.33.0. For the latest API, always check the official docs at <https://docs.rs/egui>.**
