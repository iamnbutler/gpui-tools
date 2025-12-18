# GPUI Styling Patterns

This guide covers how styling works in GPUI and common patterns for styling elements. For a complete reference of available style methods, see [GPUI Style Helpers](gpui_style_helpers.md).

## Table of Contents

1. [Styling Philosophy](#styling-philosophy)
2. [The Styled Trait](#the-styled-trait)
3. [Units in GPUI](#units-in-gpui)
4. [Layout Fundamentals](#layout-fundamentals)
5. [Conditional Styling](#conditional-styling)
6. [Interactive States](#interactive-states)
7. [Theme Integration](#theme-integration)

## Styling Philosophy

GPUI uses a **Tailwind CSS-inspired styling API**. Styles are applied using method chaining on elements, and most style methods follow Tailwind naming conventions. This approach provides:

- **Composability**: Styles chain together naturally
- **Discoverability**: Autocomplete helps find available styles
- **Consistency**: Familiar patterns for web developers
- **Type safety**: Compile-time checking of style values

```rust
// Basic styling pattern
div()
    .flex()                    // display: flex
    .flex_col()               // flex-direction: column
    .gap_2()                  // gap: 0.5rem (8px)
    .p_4()                    // padding: 1rem (16px)
    .bg(rgb(0x1e1e1e))        // background-color
    .text_color(rgb(0xffffff)) // color
    .rounded_md()             // border-radius: 0.375rem
    .shadow_lg()              // box-shadow
    .child("Hello World")
```

### Numeric Scale

GPUI uses a numeric scale following Tailwind conventions. This scale is used for spacing (padding, margin, gap), sizing, and positioning:

| Scale | Rem Value | Pixel Value |
|-------|-----------|-------------|
| 0 | 0 | 0 |
| 0p5 | 0.125rem | 2px |
| 1 | 0.25rem | 4px |
| 2 | 0.5rem | 8px |
| 3 | 0.75rem | 12px |
| 4 | 1rem | 16px |
| 6 | 1.5rem | 24px |
| 8 | 2rem | 32px |
| 12 | 3rem | 48px |
| 16 | 4rem | 64px |

Example usage:

```rust
.p_4()    // padding: 1rem (16px)
.m_2()    // margin: 0.5rem (8px)
.gap_3()  // gap: 0.75rem (12px)
.w_48()   // width: 12rem (192px)
```

## The Styled Trait

The `Styled` trait provides the fluent styling API. Any element implementing `Styled` gets access to all style methods.

```rust
pub trait Styled: Sized {
    /// Returns a mutable reference to the style.
    fn style(&mut self) -> &mut StyleRefinement;

    // All style methods are provided by macros and trait defaults
}
```

### Built-in Styled Elements

```rust
div()      // Div implements Styled
img()      // Img implements Styled
svg()      // Svg implements Styled
```

### Custom Styled Elements

You can implement `Styled` on your own types:

```rust
struct MyElement {
    style: StyleRefinement,
}

impl Styled for MyElement {
    fn style(&mut self) -> &mut StyleRefinement {
        &mut self.style
    }
}

// Now MyElement can use all style methods
let my_element = MyElement { style: StyleRefinement::default() }
    .p_4()
    .bg(rgb(0x333333));
```

### StyleRefinement

`StyleRefinement` holds optional style overrides. Only set values are applied; `None` values inherit from the parent. This allows partial style application and overrides:

```rust
pub struct StyleRefinement {
    pub display: Option<Display>,
    pub visibility: Option<Visibility>,
    pub overflow: Point<Option<Overflow>>,
    pub position: Option<Position>,
    pub flex_direction: Option<FlexDirection>,
    pub flex_wrap: Option<FlexWrap>,
    pub align_items: Option<AlignItems>,
    pub justify_content: Option<JustifyContent>,
    // ... many more fields
}
```

## Units in GPUI

GPUI provides several unit types for expressing dimensions:

```rust
use gpui::{px, rems, relative, Pixels, Rems};

// Pixels - absolute values
px(16.)                       // 16 pixels
px(1.)                        // 1 pixel

// Rems - relative to root font size (default 16px)
rems(1.)                      // 1rem = 16px
rems(0.5)                     // 0.5rem = 8px
rems_from_px(12.)             // Convert 12px to rems

// Relative - percentages and fractions
relative(0.5)                 // 50%
relative(1.)                  // 100%

// Auto
Length::Auto                  // auto
```

### When to Use Each Unit

- **Pixels (`px`)**: Use for precise, fixed sizes like borders, icons, or specific layout requirements
- **Rems (`rems`)**: Use for spacing and typography that should scale with user preferences
- **Relative**: Use for responsive layouts that should adapt to container size

## Layout Fundamentals

GPUI uses flexbox as its primary layout model, with grid support for more complex layouts.

### Flexbox Basics

```rust
// Horizontal layout (default flex direction is row)
div()
    .flex()
    .gap_2()
    .child("Item 1")
    .child("Item 2")

// Vertical layout
div()
    .flex()
    .flex_col()
    .gap_2()
    .child("Item 1")
    .child("Item 2")
```

### Common Layout Helpers

These helper functions encapsulate common patterns:

```rust
// Horizontal flex container with centered items
fn h_flex() -> Div {
    div().flex().flex_row().items_center()
}

// Vertical flex container
fn v_flex() -> Div {
    div().flex().flex_col()
}

// Usage
h_flex().gap_2().child("A").child("B")  // Horizontal row with gap
v_flex().gap_4().child("A").child("B")  // Vertical column with gap
```

### Layout Example: App Shell

```rust
fn app_shell() -> impl IntoElement {
    div()
        .flex()
        .flex_col()
        .size_full()
        .child(
            // Header
            div()
                .h_12()
                .w_full()
                .bg(rgb(0x333333))
                .child("Header")
        )
        .child(
            // Main content area
            div()
                .flex_1()
                .flex()
                .flex_row()
                .child(
                    // Sidebar
                    div()
                        .w_64()
                        .bg(rgb(0x2a2a2a))
                        .child("Sidebar")
                )
                .child(
                    // Content
                    div()
                        .flex_1()
                        .bg(rgb(0x1e1e1e))
                        .child("Content")
                )
        )
}
```

### Flex Item Sizing

Understanding flex grow/shrink is essential for responsive layouts:

```rust
// flex: 1 1 0% - grow and shrink equally, ignore initial size
div().flex_1()

// flex: 1 1 auto - grow and shrink, respect initial size
div().flex_auto()

// flex: 0 1 auto - can shrink but won't grow
div().flex_initial()

// flex: none - fixed size, won't grow or shrink
div().flex_none()
```

### Grid Layout

For complex two-dimensional layouts, use grid:

```rust
div()
    .grid()
    .grid_cols_3()    // 3 columns
    .grid_rows_2()    // 2 rows
    .gap_4()
    .children((0..6).map(|i| {
        div().bg(rgb(0x333333)).p_4().child(format!("Item {}", i))
    }))
```

## Conditional Styling

GPUI provides several patterns for applying styles conditionally.

### Using `when` and `when_some`

```rust
// Apply styles based on a boolean condition
div()
    .when(is_selected, |this| this.bg(rgb(0x3b82f6)))
    .when(!is_selected, |this| this.bg(rgb(0x333333)))

// Apply styles when an Option has a value
div()
    .when_some(maybe_color, |this, color| this.bg(color))
    .when(maybe_color.is_none(), |this| this.bg(rgb(0x333333)))
```

### Using `map` for Complex Transformations

```rust
div()
    .map(|this| {
        if some_condition {
            this.bg(rgb(0xff0000)).text_color(rgb(0xffffff))
        } else {
            this.bg(rgb(0x00ff00)).text_color(rgb(0x000000))
        }
    })
```

### Conditional Children

```rust
div()
    .flex()
    .gap_2()
    .when(show_icon, |this| this.child(
        svg().path("icons/check.svg").size_4()
    ))
    .child("Label")
```

### Example: Disabled State Pattern

```rust
fn button(label: &str, disabled: bool) -> impl IntoElement {
    div()
        .id(label)
        .px_4()
        .py_2()
        .rounded_md()
        .when(disabled, |this| {
            this
                .bg(rgb(0x666666))
                .text_color(rgb(0x999999))
                .cursor_not_allowed()
        })
        .when(!disabled, |this| {
            this
                .bg(rgb(0x3b82f6))
                .text_color(rgb(0xffffff))
                .cursor_pointer()
                .hover(|style| style.bg(rgb(0x2563eb)))
                .active(|style| style.bg(rgb(0x1d4ed8)))
        })
        .child(label)
}
```

## Interactive States

GPUI supports CSS-like pseudo-class states for interactive elements.

### Hover State

```rust
div()
    .id("button")  // ID required for interactive states
    .bg(rgb(0x3b82f6))
    .hover(|style| style.bg(rgb(0x2563eb)))  // Darker on hover
    .child("Hover me")
```

### Active State (Pressed)

```rust
div()
    .id("button")
    .bg(rgb(0x3b82f6))
    .hover(|style| style.bg(rgb(0x2563eb)))
    .active(|style| style.bg(rgb(0x1d4ed8)))  // Even darker when pressed
    .child("Press me")
```

### Focus States

```rust
// Focus state (any focus)
div()
    .id("input")
    .track_focus(&focus_handle)
    .border_1()
    .border_color(rgb(0x333333))
    .focus(|style| {
        style.border_color(rgb(0x3b82f6))
    })

// Focus-visible (keyboard focus only)
div()
    .id("button")
    .track_focus(&focus_handle)
    .focus_visible(|style| {
        style
            .outline_2()
            .outline_color(rgb(0x3b82f6))
    })
```

### Group Hover

Coordinate hover states between parent and child elements:

```rust
div()
    .group("card")  // Define a group
    .p_4()
    .bg(rgb(0x1e1e1e))
    .hover(|style| style.bg(rgb(0x2a2a2a)))
    .child(
        // Child responds to parent group hover
        div()
            .opacity_0()
            .group_hover("card", |style| style.opacity_100())
            .child("Shows on parent hover")
    )
```

### Complete Interactive Button

```rust
fn interactive_button(
    label: &str,
    focus_handle: &FocusHandle,
) -> impl IntoElement {
    div()
        .id(label)
        .track_focus(focus_handle)
        .px_4()
        .py_2()
        .bg(rgb(0x3b82f6))
        .text_color(rgb(0xffffff))
        .rounded_md()
        .cursor_pointer()
        .hover(|style| style.bg(rgb(0x2563eb)))
        .active(|style| style.bg(rgb(0x1d4ed8)))
        .focus_visible(|style| {
            style
                .outline_2()
                .outline_color(rgb(0xffffff))
                .outline_offset(px(2.))
        })
        .child(label)
}
```

## Theme Integration

GPUI applications typically use a theme system to manage colors and styling consistently.

### Basic Theme Pattern

```rust
pub struct Theme {
    pub bg_primary: Hsla,
    pub bg_secondary: Hsla,
    pub text_primary: Hsla,
    pub text_secondary: Hsla,
    pub accent: Hsla,
    pub border: Hsla,
}

impl Theme {
    pub fn dark() -> Self {
        Self {
            bg_primary: rgb(0x1e1e1e).into(),
            bg_secondary: rgb(0x2a2a2a).into(),
            text_primary: rgb(0xffffff).into(),
            text_secondary: rgb(0x999999).into(),
            accent: rgb(0x3b82f6).into(),
            border: rgb(0x333333).into(),
        }
    }

    pub fn light() -> Self {
        Self {
            bg_primary: rgb(0xffffff).into(),
            bg_secondary: rgb(0xf5f5f5).into(),
            text_primary: rgb(0x000000).into(),
            text_secondary: rgb(0x666666).into(),
            accent: rgb(0x2563eb).into(),
            border: rgb(0xe5e5e5).into(),
        }
    }
}
```

### Using Theme in Components

```rust
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let theme = cx.global::<Theme>();

        div()
            .bg(theme.bg_primary)
            .text_color(theme.text_primary)
            .border_color(theme.border)
            .child("Themed content")
    }
}
```

### Semantic Color Helpers

Create an abstraction for semantic colors:

```rust
pub enum Color {
    Default,
    Muted,
    Accent,
    Success,
    Warning,
    Error,
    Disabled,
}

impl Color {
    pub fn color(&self, cx: &App) -> Hsla {
        let theme = cx.global::<Theme>();
        match self {
            Color::Default => theme.text_primary,
            Color::Muted => theme.text_secondary,
            Color::Accent => theme.accent,
            // ... other mappings
        }
    }
}
```

### Dark/Light Mode Detection

```rust
impl Render for MyView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let is_dark = matches!(
            window.appearance(),
            WindowAppearance::Dark | WindowAppearance::VibrantDark
        );

        div()
            .bg(if is_dark { rgb(0x1e1e1e) } else { rgb(0xffffff) })
            .text_color(if is_dark { rgb(0xffffff) } else { rgb(0x000000) })
            .child("Adapts to system theme")
    }
}
```

### Observing Appearance Changes

```rust
impl MyView {
    fn new(window: &mut Window, cx: &mut Context<Self>) -> Self {
        window.observe_window_appearance(cx, |this, window, cx| {
            cx.notify();  // Re-render when appearance changes
        }).detach();

        Self {}
    }
}
```

### Themed Component Example

```rust
fn themed_card(theme: &Theme, title: &str, content: &str) -> impl IntoElement {
    div()
        .p_4()
        .bg(theme.bg_secondary)
        .border_1()
        .border_color(theme.border)
        .rounded_lg()
        .shadow_md()
        .child(
            div()
                .text_lg()
                .font_weight(FontWeight::BOLD)
                .text_color(theme.text_primary)
                .mb_2()
                .child(title)
        )
        .child(
            div()
                .text_sm()
                .text_color(theme.text_secondary)
                .child(content)
        )
}
```

## Summary

Key takeaways for GPUI styling:

1. **Chain methods** for a fluent, composable styling API
2. **Use the numeric scale** for consistent spacing
3. **Prefer flexbox** for most layouts, grid for complex 2D layouts
4. **Use `when`/`when_some`** for conditional styling
5. **Add `id()`** to elements that need interactive states
6. **Create a theme system** for consistent, maintainable colors
7. **Use semantic color helpers** to abstract color meanings from values

For a complete reference of all available style methods, see [GPUI Style Helpers](gpui_style_helpers.md).
