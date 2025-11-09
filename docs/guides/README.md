# egui Guides

Comprehensive guides for building applications with egui.

## Available Guides

### [Building a Text Editor](building-a-text-editor.md)
**Difficulty**: Intermediate
**Time**: 30-60 minutes

Learn how to build a text editor from scratch, covering:
- Basic text input (single-line and multiline)
- The TextBuffer trait and custom implementations
- State management (cursor position, undo/redo)
- Syntax highlighting with custom layouters
- Advanced features (line numbers, custom keybindings)
- Complete code editor example with syntect integration

Perfect for understanding how egui handles text, state persistence, and custom rendering.

---

## Coming Soon

### Getting Started with egui
**Difficulty**: Beginner

- Setting up your first egui project
- Understanding the application structure
- Basic widgets and layouts
- Handling user input
- Building a simple app

### Custom Widgets
**Difficulty**: Intermediate

- The Widget trait
- Creating reusable components
- Painter API for custom rendering
- Response handling
- Examples: Toggle switch, color picker, dial control

### State Management Patterns
**Difficulty**: Intermediate

- Using Memory and IdTypeMap
- Global application state
- Per-widget state
- State serialization
- Shared state between widgets

### Advanced Layouts
**Difficulty**: Advanced

- Custom layout algorithms
- Responsive design
- Grid and table layouts
- Dynamic sizing
- Performance optimization

### Integration Guide
**Difficulty**: Intermediate

- Integrating egui into game engines
- Writing custom backends
- Platform-specific features
- Custom rendering pipelines

## Guide Format

All guides follow this structure:

1. **Overview**: What you'll learn
2. **Prerequisites**: What you need to know
3. **Core Concepts**: Essential background
4. **Step-by-Step**: Build something real
5. **Advanced Topics**: Go deeper
6. **Complete Example**: Fully working code
7. **Further Reading**: Related docs and source

## How to Use These Guides

### For Beginners
Start with simple examples from [../examples/](../../examples/), then work through guides in order of difficulty.

### For Intermediate Users
Jump to specific guides that match your current task. Each guide is self-contained.

### For Advanced Users
Use guides as reference material and starting points for complex features.

## Tips for Learning egui

1. **Run the demos**: `cargo run --release -p egui_demo_app`
2. **Read the source**: egui's code is well-documented
3. **Experiment**: Immediate mode makes experimentation easy
4. **Check responses**: Always look at what `Response` tells you
5. **Use debug mode**: `ctx.set_debug_on_hover(true)` is your friend

## Contributing Guides

Want to write a guide? Great! Follow these steps:

1. Choose a topic not already covered
2. Follow the standard guide format
3. Include runnable code examples
4. Reference actual source code locations
5. Test everything thoroughly
6. Submit a PR

### Guide Writing Tips

- **Show, don't tell**: Code examples > explanations
- **Truth first**: Document what exists, not ideals
- **Build something real**: Not just API showcases
- **Explain why**: The reasoning matters
- **Keep it focused**: One topic per guide

---

**Questions?** Ask in [GitHub Discussions](https://github.com/emilk/egui/discussions) or [Discord](https://discord.gg/JFcEma9bJq).
