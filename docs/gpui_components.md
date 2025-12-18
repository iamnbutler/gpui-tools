# GPUI Component Design Patterns

This guide covers the details of creating UI components in GPUI—the `Render` and `RenderOnce` traits, the builder pattern, and composition. For guidance on when to use each type and how they fit into an app, see [GPUI App Architecture](gpui_app_architecture.md).

## Table of Contents

1. [High Level Patterns](#high-level-patterns)
2. [Render vs RenderOnce](#render-vs-renderonce)
3. [Render Trait](#render-trait)
4. [RenderOnce Trait](#renderonce-trait)
5. [IntoElement Derive](#intoelement-derive)
6. [Component Structure](#component-structure)
7. [Builder Pattern](#builder-pattern)
8. [Stateful Components](#stateful-components)
9. [Stateless Components](#stateless-components)
10. [Component Composition](#component-composition)
11. [Event Handling](#event-handling)
12. [Focus Management](#focus-management)
13. [Component Libraries](#component-libraries)

## High Level Patterns

### When to use Render vs RenderOnce

```rust
// Use Render (Entity-backed views) when:
// - Component needs to maintain internal state across renders
// - Component needs to subscribe to events or observe entities
// - Component needs its own focus handle
// - Component needs to spawn async tasks
// - Component needs lifecycle management (created once, updated many times)

// Use RenderOnce (stateless components) when:
// - Component is purely presentational
// - Component derives all data from props
// - Component doesn't need to maintain internal state
// - Component is rendered inline and recreated each frame
// - You want maximum performance for simple UI elements
```

### Decision flowchart

```
Does the component need:
├── Internal mutable state? → Render
├── Event subscriptions? → Render
├── Focus handle management? → Render
├── Async task spawning? → Render
├── Lifecycle callbacks? → Render
└── None of the above? → RenderOnce
```

### Performance considerations

```rust
// RenderOnce: Created fresh each render, then consumed
// - Lower memory overhead (no Entity allocation)
// - No subscription management
// - Ideal for lists with many items

// Render: Entity-backed, persisted between renders
// - Has identity (Entity<T>)
// - Can be observed, subscribed to
// - State preserved across renders
// - Higher memory overhead per instance
```

## Render vs RenderOnce

### Side-by-side comparison

```rust
// ═══════════════════════════════════════════════════════
// RENDER: Entity-backed view with persistent state
// ═══════════════════════════════════════════════════════

struct Counter {
    count: u32,
    focus_handle: FocusHandle,
}

impl Counter {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            count: 0,
            focus_handle: cx.focus_handle(),
        }
    }

    fn increment(&mut self, _: &Click, _window: &mut Window, cx: &mut Context<Self>) {
        self.count += 1;
        cx.notify();
    }
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("counter")
            .track_focus(&self.focus_handle)
            .on_click(cx.listener(Self::increment))
            .child(format!("Count: {}", self.count))
    }
}

// Usage: must be created as Entity
let counter: Entity<Counter> = cx.new(|cx| Counter::new(cx));

// ═══════════════════════════════════════════════════════
// RENDERONCE: Stateless component consumed on render
// ═══════════════════════════════════════════════════════

#[derive(IntoElement)]
struct CounterDisplay {
    count: u32,
    on_click: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
}

impl CounterDisplay {
    fn new(count: u32) -> Self {
        Self { count, on_click: None }
    }

    fn on_click(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static
    ) -> Self {
        self.on_click = Some(Box::new(handler));
        self
    }
}

impl RenderOnce for CounterDisplay {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let element = div()
            .id("counter-display")
            .child(format!("Count: {}", self.count));

        if let Some(handler) = self.on_click {
            element.on_click(move |event, window, cx| handler(event, window, cx))
        } else {
            element
        }
    }
}

// Usage: created inline, consumed immediately
div().child(CounterDisplay::new(42).on_click(|_, _, _| {}))
```

### Key differences table

```
┌────────────────────┬─────────────────────────┬─────────────────────────┐
│ Aspect             │ Render                  │ RenderOnce              │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ Backing            │ Entity<T>               │ None (consumed)         │
│ Self reference     │ &mut self               │ self (owned)            │
│ Context            │ Context<Self>           │ App                     │
│ State persistence  │ Yes                     │ No                      │
│ Event subscriptions│ Yes                     │ No                      │
│ Focus handles      │ Yes (via Context)       │ No (passed in)          │
│ Async tasks        │ Yes (cx.spawn)          │ No                      │
│ Lifecycle          │ Created once            │ Created each render     │
│ Memory             │ Higher (Entity)         │ Lower (stack)           │
│ Usage              │ cx.new(|cx| T::new(cx)) │ T::new().into_element() │
└────────────────────┴─────────────────────────┴─────────────────────────┘
```

## Render Trait

### Trait definition

```rust
pub trait Render: 'static + Sized {
    /// Render this view into an element tree.
    /// Takes &mut self - state can be read and modified.
    /// Has access to Context<Self> for subscriptions, spawning, etc.
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement;
}
```

### Complete Render example

```rust
use gpui::{
    div, prelude::*, App, Context, Entity, EventEmitter, FocusHandle, Focusable,
    Subscription, Window,
};

// Events this view can emit
pub enum CounterEvent {
    Changed(u32),
    Reset,
}

struct CounterView {
    count: u32,
    max_count: Option<u32>,
    focus_handle: FocusHandle,
    label: SharedString,
    _subscriptions: Vec<Subscription>,
}

impl EventEmitter<CounterEvent> for CounterView {}

impl CounterView {
    pub fn new(cx: &mut Context<Self>) -> Self {
        Self {
            count: 0,
            max_count: None,
            focus_handle: cx.focus_handle(),
            label: "Counter".into(),
            _subscriptions: Vec::new(),
        }
    }

    pub fn with_label(mut self, label: impl Into<SharedString>) -> Self {
        self.label = label.into();
        self
    }

    pub fn with_max(mut self, max: u32) -> Self {
        self.max_count = Some(max);
        self
    }

    pub fn observe_changes(
        &mut self,
        target: &Entity<SomeOtherEntity>,
        cx: &mut Context<Self>,
    ) {
        let subscription = cx.subscribe(target, |this, _emitter, event, cx| {
            // Handle event from target
            this.handle_external_event(event, cx);
        });
        self._subscriptions.push(subscription);
    }

    fn handle_external_event(&mut self, _event: &SomeEvent, cx: &mut Context<Self>) {
        cx.notify();
    }

    fn increment(&mut self, _: &Click, _window: &mut Window, cx: &mut Context<Self>) {
        if let Some(max) = self.max_count {
            if self.count >= max {
                return;
            }
        }
        self.count += 1;
        cx.emit(CounterEvent::Changed(self.count));
        cx.notify();
    }

    fn reset(&mut self, _: &Click, _window: &mut Window, cx: &mut Context<Self>) {
        self.count = 0;
        cx.emit(CounterEvent::Reset);
        cx.notify();
    }
}

impl Focusable for CounterView {
    fn focus_handle(&self, _cx: &App) -> FocusHandle {
        self.focus_handle.clone()
    }
}

impl Render for CounterView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let at_max = self.max_count.map(|m| self.count >= m).unwrap_or(false);

        div()
            .id("counter-view")
            .track_focus(&self.focus_handle)
            .flex()
            .flex_col()
            .gap_2()
            .p_4()
            .bg(rgb(0x1e1e1e))
            .rounded_md()
            .child(
                div()
                    .text_sm()
                    .text_color(rgb(0x888888))
                    .child(self.label.clone())
            )
            .child(
                div()
                    .text_2xl()
                    .font_weight(FontWeight::BOLD)
                    .child(format!("{}", self.count))
            )
            .child(
                div()
                    .flex()
                    .gap_2()
                    .child(
                        div()
                            .id("increment")
                            .px_3()
                            .py_1()
                            .bg(if at_max { rgb(0x666666) } else { rgb(0x3b82f6) })
                            .rounded_md()
                            .cursor(if at_max { CursorStyle::NotAllowed } else { CursorStyle::PointingHand })
                            .when(!at_max, |this| {
                                this.on_click(cx.listener(Self::increment))
                            })
                            .child("+")
                    )
                    .child(
                        div()
                            .id("reset")
                            .px_3()
                            .py_1()
                            .bg(rgb(0xef4444))
                            .rounded_md()
                            .cursor_pointer()
                            .on_click(cx.listener(Self::reset))
                            .child("Reset")
                    )
            )
    }
}

// Usage
let counter = cx.new(|cx| {
    CounterView::new(cx)
        .with_label("My Counter")
        .with_max(10)
});
```

### When render updates

```rust
// Render is called when:
// 1. cx.notify() is called
// 2. Window needs to redraw
// 3. Parent view re-renders and includes this view

impl Render for MyView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // This closure runs every time the view updates
        // Keep it efficient - avoid heavy computations here

        div().child("Content")
    }
}

impl MyView {
    fn update_something(&mut self, cx: &mut Context<Self>) {
        self.data = compute_new_data();
        cx.notify(); // Triggers re-render
    }
}
```

## RenderOnce Trait

### Trait definition

```rust
pub trait RenderOnce: 'static {
    /// Render this component into an element tree.
    /// Takes ownership of self - component is consumed.
    /// Has access to App context only (no subscriptions, no spawning).
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement;
}
```

### Complete RenderOnce example

```rust
use gpui::{div, prelude::*, rgb, App, SharedString, Window};

/// A reusable button component
#[derive(IntoElement)]
pub struct Button {
    id: ElementId,
    label: SharedString,
    icon: Option<IconName>,
    disabled: bool,
    style: ButtonStyle,
    on_click: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
}

#[derive(Default, Clone, Copy)]
pub enum ButtonStyle {
    #[default]
    Primary,
    Secondary,
    Danger,
    Ghost,
}

impl Button {
    pub fn new(id: impl Into<ElementId>, label: impl Into<SharedString>) -> Self {
        Self {
            id: id.into(),
            label: label.into(),
            icon: None,
            disabled: false,
            style: ButtonStyle::default(),
            on_click: None,
        }
    }

    pub fn icon(mut self, icon: IconName) -> Self {
        self.icon = Some(icon);
        self
    }

    pub fn disabled(mut self, disabled: bool) -> Self {
        self.disabled = disabled;
        self
    }

    pub fn style(mut self, style: ButtonStyle) -> Self {
        self.style = style;
        self
    }

    pub fn on_click(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_click = Some(Box::new(handler));
        self
    }
}

impl RenderOnce for Button {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let (bg_color, hover_color, text_color) = match self.style {
            ButtonStyle::Primary => (rgb(0x3b82f6), rgb(0x2563eb), rgb(0xffffff)),
            ButtonStyle::Secondary => (rgb(0x6b7280), rgb(0x4b5563), rgb(0xffffff)),
            ButtonStyle::Danger => (rgb(0xef4444), rgb(0xdc2626), rgb(0xffffff)),
            ButtonStyle::Ghost => (rgb(0x00000000), rgb(0x374151), rgb(0xd1d5db)),
        };

        let mut element = div()
            .id(self.id)
            .flex()
            .items_center()
            .gap_2()
            .px_4()
            .py_2()
            .rounded_md()
            .text_sm()
            .font_weight(FontWeight::MEDIUM);

        if self.disabled {
            element = element
                .bg(rgb(0x4b5563))
                .text_color(rgb(0x9ca3af))
                .cursor_not_allowed();
        } else {
            element = element
                .bg(bg_color)
                .text_color(text_color)
                .cursor_pointer()
                .hover(|style| style.bg(hover_color));

            if let Some(handler) = self.on_click {
                element = element.on_click(move |event, window, cx| {
                    handler(event, window, cx);
                });
            }
        }

        if let Some(icon) = self.icon {
            element = element.child(
                svg()
                    .path(icon.path())
                    .size_4()
                    .text_color(text_color)
            );
        }

        element.child(self.label)
    }
}

// Usage - inline, consumed immediately
div()
    .child(
        Button::new("submit", "Submit")
            .style(ButtonStyle::Primary)
            .on_click(|_, _, _| {
                println!("Clicked!");
            })
    )
    .child(
        Button::new("cancel", "Cancel")
            .style(ButtonStyle::Ghost)
            .on_click(|_, _, _| {
                println!("Cancelled!");
            })
    )
```

### RenderOnce with external state

```rust
#[derive(IntoElement)]
pub struct TodoItem {
    todo: Todo,
    is_selected: bool,
    on_toggle: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
    on_delete: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
}

impl TodoItem {
    pub fn new(todo: Todo) -> Self {
        Self {
            todo,
            is_selected: false,
            on_toggle: None,
            on_delete: None,
        }
    }

    pub fn selected(mut self, selected: bool) -> Self {
        self.is_selected = selected;
        self
    }

    pub fn on_toggle(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_toggle = Some(Box::new(handler));
        self
    }

    pub fn on_delete(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_delete = Some(Box::new(handler));
        self
    }
}

impl RenderOnce for TodoItem {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let todo_id = self.todo.id;

        div()
            .id(("todo", todo_id))
            .flex()
            .items_center()
            .gap_3()
            .p_2()
            .bg(if self.is_selected { rgb(0x2a2a2a) } else { rgb(0x1e1e1e) })
            .hover(|s| s.bg(rgb(0x333333)))
            .rounded_md()
            .child(
                div()
                    .id(("toggle", todo_id))
                    .size_5()
                    .rounded_sm()
                    .border_1()
                    .border_color(rgb(0x555555))
                    .cursor_pointer()
                    .when(self.todo.completed, |this| {
                        this.bg(rgb(0x22c55e))
                            .child(svg().path("icons/check.svg").size_4())
                    })
                    .when_some(self.on_toggle, |this, handler| {
                        this.on_click(move |e, w, cx| handler(e, w, cx))
                    })
            )
            .child(
                div()
                    .flex_1()
                    .when(self.todo.completed, |this| {
                        this.line_through().text_color(rgb(0x666666))
                    })
                    .child(self.todo.label.clone())
            )
            .child(
                div()
                    .id(("delete", todo_id))
                    .size_6()
                    .flex()
                    .items_center()
                    .justify_center()
                    .rounded_sm()
                    .cursor_pointer()
                    .hover(|s| s.bg(rgb(0xef4444)))
                    .when_some(self.on_delete, |this, handler| {
                        this.on_click(move |e, w, cx| handler(e, w, cx))
                    })
                    .child(svg().path("icons/trash.svg").size_4())
            )
    }
}

// Usage in a Render view
impl Render for TodoListView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .gap_1()
            .children(self.todos.iter().map(|todo| {
                let todo_id = todo.id;
                TodoItem::new(todo.clone())
                    .selected(self.selected_id == Some(todo_id))
                    .on_toggle(cx.listener(move |this, _, _, cx| {
                        this.toggle_todo(todo_id, cx);
                    }))
                    .on_delete(cx.listener(move |this, _, _, cx| {
                        this.delete_todo(todo_id, cx);
                    }))
            }))
    }
}
```

## IntoElement Derive

### Using the derive macro

```rust
// The #[derive(IntoElement)] macro automatically implements:
// - IntoElement trait
// - Wraps your RenderOnce in a Component

#[derive(IntoElement)]
pub struct MyComponent {
    // fields...
}

impl RenderOnce for MyComponent {
    fn render(self, window: &mut Window, cx: &mut App) -> impl IntoElement {
        div().child("My Component")
    }
}

// This is equivalent to manually implementing:
impl IntoElement for MyComponent {
    type Element = Component<Self>;

    fn into_element(self) -> Self::Element {
        Component::new(self)
    }
}
```

### Generic components with IntoElement

```rust
#[derive(IntoElement)]
pub struct Container<C: IntoElement> {
    child: C,
    padding: Pixels,
}

impl<C: IntoElement + 'static> Container<C> {
    pub fn new(child: C) -> Self {
        Self {
            child,
            padding: px(16.),
        }
    }

    pub fn padding(mut self, padding: Pixels) -> Self {
        self.padding = padding;
        self
    }
}

impl<C: IntoElement + 'static> RenderOnce for Container<C> {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div()
            .p(self.padding)
            .child(self.child)
    }
}

// Usage
Container::new(
    div().child("Inner content")
).padding(px(24.))
```

## Component Structure

### Recommended file structure

```
src/
├── components/
│   ├── mod.rs
│   ├── button.rs         # Button, IconButton, etc.
│   ├── input.rs          # TextInput, NumberInput, etc.
│   ├── list.rs           # List, ListItem, etc.
│   └── modal.rs          # Modal, Dialog, etc.
├── views/
│   ├── mod.rs
│   ├── main_view.rs      # Root view (Render)
│   ├── sidebar.rs        # Sidebar view (Render)
│   └── editor.rs         # Editor view (Render)
└── main.rs
```

### Component module pattern

```rust
// components/button.rs

mod button;
mod button_like;
mod icon_button;

pub use button::*;
pub use button_like::*;
pub use icon_button::*;

// Re-export common types
pub use crate::theme::ButtonStyle;
```

### Prelude pattern

```rust
// components/prelude.rs
pub use crate::components::{
    Button, ButtonStyle, IconButton,
    Input, InputStyle,
    Label, LabelSize,
    // ... etc
};

// Re-export gpui prelude
pub use gpui::prelude::*;

// Usage in other files
use crate::components::prelude::*;
```

## Builder Pattern

### Standard builder pattern for components

```rust
#[derive(IntoElement)]
pub struct Card {
    title: Option<SharedString>,
    subtitle: Option<SharedString>,
    children: Vec<AnyElement>,
    padding: Pixels,
    rounded: bool,
    shadow: bool,
    border: bool,
}

impl Card {
    pub fn new() -> Self {
        Self {
            title: None,
            subtitle: None,
            children: Vec::new(),
            padding: px(16.),
            rounded: true,
            shadow: true,
            border: false,
        }
    }

    pub fn title(mut self, title: impl Into<SharedString>) -> Self {
        self.title = Some(title.into());
        self
    }

    pub fn subtitle(mut self, subtitle: impl Into<SharedString>) -> Self {
        self.subtitle = Some(subtitle.into());
        self
    }

    pub fn padding(mut self, padding: Pixels) -> Self {
        self.padding = padding;
        self
    }

    pub fn rounded(mut self, rounded: bool) -> Self {
        self.rounded = rounded;
        self
    }

    pub fn shadow(mut self, shadow: bool) -> Self {
        self.shadow = shadow;
        self
    }

    pub fn border(mut self, border: bool) -> Self {
        self.border = border;
        self
    }

    pub fn child(mut self, child: impl IntoElement) -> Self {
        self.children.push(child.into_any_element());
        self
    }

    pub fn children(mut self, children: impl IntoIterator<Item = impl IntoElement>) -> Self {
        self.children.extend(children.into_iter().map(|c| c.into_any_element()));
        self
    }
}

impl RenderOnce for Card {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let mut card = div()
            .p(self.padding)
            .bg(rgb(0x2a2a2a));

        if self.rounded {
            card = card.rounded_lg();
        }
        if self.shadow {
            card = card.shadow_md();
        }
        if self.border {
            card = card.border_1().border_color(rgb(0x404040));
        }

        let mut content = div().flex().flex_col().gap_2();

        if let Some(title) = self.title {
            content = content.child(
                div()
                    .text_lg()
                    .font_weight(FontWeight::SEMIBOLD)
                    .child(title)
            );
        }

        if let Some(subtitle) = self.subtitle {
            content = content.child(
                div()
                    .text_sm()
                    .text_color(rgb(0x888888))
                    .child(subtitle)
            );
        }

        content = content.children(self.children);

        card.child(content)
    }
}

// Usage
Card::new()
    .title("Settings")
    .subtitle("Configure your preferences")
    .padding(px(24.))
    .shadow(true)
    .child(
        Button::new("save", "Save Changes")
            .style(ButtonStyle::Primary)
    )
```

### Builder with Into trait for flexibility

```rust
#[derive(IntoElement)]
pub struct Label {
    text: SharedString,
    size: LabelSize,
    color: LabelColor,
}

#[derive(Default, Clone, Copy)]
pub enum LabelSize {
    XSmall,
    Small,
    #[default]
    Medium,
    Large,
    XLarge,
}

#[derive(Default, Clone, Copy)]
pub enum LabelColor {
    #[default]
    Default,
    Muted,
    Accent,
    Success,
    Warning,
    Error,
}

impl Label {
    pub fn new(text: impl Into<SharedString>) -> Self {
        Self {
            text: text.into(),
            size: LabelSize::default(),
            color: LabelColor::default(),
        }
    }

    pub fn size(mut self, size: impl Into<LabelSize>) -> Self {
        self.size = size.into();
        self
    }

    pub fn color(mut self, color: impl Into<LabelColor>) -> Self {
        self.color = color.into();
        self
    }
}

// Allow Option<LabelSize> to be passed
impl From<Option<LabelSize>> for LabelSize {
    fn from(opt: Option<LabelSize>) -> Self {
        opt.unwrap_or_default()
    }
}
```

## Stateful Components

### Stateful component with Entity

```rust
pub struct Accordion {
    sections: Vec<AccordionSection>,
    expanded_index: Option<usize>,
    focus_handle: FocusHandle,
    allow_multiple: bool,
}

struct AccordionSection {
    title: SharedString,
    content: AnyElement,
    is_expanded: bool,
}

impl Accordion {
    pub fn new(cx: &mut Context<Self>) -> Self {
        Self {
            sections: Vec::new(),
            expanded_index: None,
            focus_handle: cx.focus_handle(),
            allow_multiple: false,
        }
    }

    pub fn section(
        mut self,
        title: impl Into<SharedString>,
        content: impl IntoElement,
    ) -> Self {
        self.sections.push(AccordionSection {
            title: title.into(),
            content: content.into_any_element(),
            is_expanded: false,
        });
        self
    }

    pub fn allow_multiple(mut self, allow: bool) -> Self {
        self.allow_multiple = allow;
        self
    }

    fn toggle_section(&mut self, index: usize, cx: &mut Context<Self>) {
        if self.allow_multiple {
            self.sections[index].is_expanded = !self.sections[index].is_expanded;
        } else {
            let was_expanded = self.sections[index].is_expanded;
            for section in &mut self.sections {
                section.is_expanded = false;
            }
            self.sections[index].is_expanded = !was_expanded;
        }
        cx.notify();
    }
}

impl Focusable for Accordion {
    fn focus_handle(&self, _cx: &App) -> FocusHandle {
        self.focus_handle.clone()
    }
}

impl Render for Accordion {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .track_focus(&self.focus_handle)
            .flex()
            .flex_col()
            .children(self.sections.iter().enumerate().map(|(index, section)| {
                div()
                    .flex()
                    .flex_col()
                    .border_b_1()
                    .border_color(rgb(0x333333))
                    .child(
                        div()
                            .id(("header", index))
                            .flex()
                            .items_center()
                            .justify_between()
                            .p_3()
                            .cursor_pointer()
                            .hover(|s| s.bg(rgb(0x2a2a2a)))
                            .on_click(cx.listener(move |this, _, _, cx| {
                                this.toggle_section(index, cx);
                            }))
                            .child(section.title.clone())
                            .child(
                                svg()
                                    .path("icons/chevron.svg")
                                    .size_4()
                                    .when(section.is_expanded, |this| {
                                        this.with_transformation(Transformation::rotate(percentage(0.25)))
                                    })
                            )
                    )
                    .when(section.is_expanded, |this| {
                        this.child(
                            div()
                                .p_3()
                                .bg(rgb(0x1a1a1a))
                                .child("Content goes here")
                        )
                    })
            }))
    }
}

// Usage
let accordion = cx.new(|cx| {
    Accordion::new(cx)
        .section("Section 1", div().child("Content 1"))
        .section("Section 2", div().child("Content 2"))
        .section("Section 3", div().child("Content 3"))
});
```

## Stateless Components

### Pure presentational components

```rust
/// A badge component - purely presentational, no state
#[derive(IntoElement)]
pub struct Badge {
    label: SharedString,
    variant: BadgeVariant,
}

#[derive(Default, Clone, Copy)]
pub enum BadgeVariant {
    #[default]
    Default,
    Success,
    Warning,
    Error,
    Info,
}

impl Badge {
    pub fn new(label: impl Into<SharedString>) -> Self {
        Self {
            label: label.into(),
            variant: BadgeVariant::default(),
        }
    }

    pub fn variant(mut self, variant: BadgeVariant) -> Self {
        self.variant = variant;
        self
    }

    pub fn success(label: impl Into<SharedString>) -> Self {
        Self::new(label).variant(BadgeVariant::Success)
    }

    pub fn error(label: impl Into<SharedString>) -> Self {
        Self::new(label).variant(BadgeVariant::Error)
    }
}

impl RenderOnce for Badge {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let (bg, text) = match self.variant {
            BadgeVariant::Default => (rgb(0x374151), rgb(0xd1d5db)),
            BadgeVariant::Success => (rgb(0x065f46), rgb(0x6ee7b7)),
            BadgeVariant::Warning => (rgb(0x92400e), rgb(0xfcd34d)),
            BadgeVariant::Error => (rgb(0x991b1b), rgb(0xfca5a5)),
            BadgeVariant::Info => (rgb(0x1e40af), rgb(0x93c5fd)),
        };

        div()
            .px_2()
            .py_0p5()
            .bg(bg)
            .text_color(text)
            .text_xs()
            .font_weight(FontWeight::MEDIUM)
            .rounded_full()
            .child(self.label)
    }
}
```

### Stateless wrapper components

```rust
/// A centered container - just layout, no state
#[derive(IntoElement)]
pub struct Center {
    child: AnyElement,
}

impl Center {
    pub fn new(child: impl IntoElement) -> Self {
        Self {
            child: child.into_any_element(),
        }
    }
}

impl RenderOnce for Center {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div()
            .size_full()
            .flex()
            .items_center()
            .justify_center()
            .child(self.child)
    }
}

// Usage
Center::new(
    div().child("Centered content")
)
```

## Component Composition

### Composing RenderOnce components

```rust
#[derive(IntoElement)]
pub struct UserCard {
    user: User,
    show_actions: bool,
    on_edit: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
}

impl UserCard {
    pub fn new(user: User) -> Self {
        Self {
            user,
            show_actions: true,
            on_edit: None,
        }
    }

    pub fn hide_actions(mut self) -> Self {
        self.show_actions = false;
        self
    }

    pub fn on_edit(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_edit = Some(Box::new(handler));
        self
    }
}

impl RenderOnce for UserCard {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        Card::new()
            .padding(px(16.))
            .child(
                div()
                    .flex()
                    .items_center()
                    .gap_3()
                    .child(
                        Avatar::new(self.user.avatar_url.clone())
                            .size(px(48.))
                    )
                    .child(
                        div()
                            .flex()
                            .flex_col()
                            .child(
                                Label::new(self.user.name.clone())
                                    .size(LabelSize::Large)
                            )
                            .child(
                                Label::new(self.user.email.clone())
                                    .size(LabelSize::Small)
                                    .color(LabelColor::Muted)
                            )
                    )
            )
            .when(self.show_actions, |card| {
                card.child(
                    div()
                        .flex()
                        .gap_2()
                        .mt_3()
                        .child(
                            Button::new("message", "Message")
                                .style(ButtonStyle::Primary)
                        )
                        .when_some(self.on_edit, |div, handler| {
                            div.child(
                                Button::new("edit", "Edit")
                                    .style(ButtonStyle::Secondary)
                                    .on_click(move |e, w, cx| handler(e, w, cx))
                            )
                        })
                )
            })
    }
}
```

### Embedding Entity views in RenderOnce

```rust
#[derive(IntoElement)]
pub struct EditorPanel {
    editor: Entity<Editor>,
    show_line_numbers: bool,
}

impl EditorPanel {
    pub fn new(editor: Entity<Editor>) -> Self {
        Self {
            editor,
            show_line_numbers: true,
        }
    }
}

impl RenderOnce for EditorPanel {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .size_full()
            .child(
                div()
                    .h_8()
                    .px_2()
                    .flex()
                    .items_center()
                    .bg(rgb(0x252525))
                    .child("Editor")
            )
            .child(
                div()
                    .flex_1()
                    .overflow_hidden()
                    // Embed the Entity view
                    .child(self.editor)
            )
    }
}
```

## Event Handling

### Click handlers in RenderOnce

```rust
#[derive(IntoElement)]
pub struct ClickableItem {
    label: SharedString,
    on_click: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
    on_double_click: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
}

impl ClickableItem {
    pub fn new(label: impl Into<SharedString>) -> Self {
        Self {
            label: label.into(),
            on_click: None,
            on_double_click: None,
        }
    }

    pub fn on_click(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_click = Some(Box::new(handler));
        self
    }

    pub fn on_double_click(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_double_click = Some(Box::new(handler));
        self
    }
}

impl RenderOnce for ClickableItem {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let mut element = div()
            .id(self.label.clone())
            .p_2()
            .cursor_pointer()
            .hover(|s| s.bg(rgb(0x333333)))
            .child(self.label);

        if let Some(handler) = self.on_click {
            element = element.on_click(move |event, window, cx| {
                handler(event, window, cx);
            });
        }

        if let Some(handler) = self.on_double_click {
            element = element.on_double_click(move |event, window, cx| {
                handler(event, window, cx);
            });
        }

        element
    }
}
```

### Event handlers in Render views with cx.listener

```rust
impl MyView {
    fn handle_click(&mut self, event: &ClickEvent, window: &mut Window, cx: &mut Context<Self>) {
        // Access self, modify state
        self.click_count += 1;
        cx.notify();
    }

    fn handle_key_down(&mut self, event: &KeyDownEvent, window: &mut Window, cx: &mut Context<Self>) {
        match event.keystroke.key.as_str() {
            "enter" => self.submit(cx),
            "escape" => self.cancel(cx),
            _ => {}
        }
    }
}

impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("my-view")
            .on_click(cx.listener(Self::handle_click))
            .on_key_down(cx.listener(Self::handle_key_down))
            .child("Content")
    }
}
```

### Passing callbacks from parent to child

```rust
// Parent view (Render)
impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .children(self.items.iter().enumerate().map(|(index, item)| {
                // Create listener that captures index
                ListItem::new(item.clone())
                    .on_select(cx.listener(move |this, _, _, cx| {
                        this.select_item(index, cx);
                    }))
                    .on_delete(cx.listener(move |this, _, _, cx| {
                        this.delete_item(index, cx);
                    }))
            }))
    }
}

// Child component (RenderOnce)
#[derive(IntoElement)]
pub struct ListItem {
    item: Item,
    on_select: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
    on_delete: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
}

impl ListItem {
    pub fn new(item: Item) -> Self {
        Self {
            item,
            on_select: None,
            on_delete: None,
        }
    }

    pub fn on_select(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_select = Some(Box::new(handler));
        self
    }

    pub fn on_delete(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_delete = Some(Box::new(handler));
        self
    }
}
```

## Focus Management

### Focus in Render views

```rust
struct FocusableView {
    focus_handle: FocusHandle,
    items: Vec<FocusHandle>,
}

impl FocusableView {
    fn new(cx: &mut Context<Self>) -> Self {
        let items = (0..5)
            .map(|i| cx.focus_handle().tab_index(i).tab_stop(true))
            .collect();

        Self {
            focus_handle: cx.focus_handle(),
            items,
        }
    }
}

impl Focusable for FocusableView {
    fn focus_handle(&self, _cx: &App) -> FocusHandle {
        self.focus_handle.clone()
    }
}

impl Render for FocusableView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .track_focus(&self.focus_handle)
            .flex()
            .flex_col()
            .gap_2()
            .children(self.items.iter().enumerate().map(|(i, handle)| {
                div()
                    .id(("item", i))
                    .track_focus(handle)
                    .p_2()
                    .border_1()
                    .border_color(rgb(0x333333))
                    .focus(|style| style.border_color(rgb(0x3b82f6)))
                    .child(format!("Focusable item {}", i))
            }))
    }
}
```

### Passing focus handles to RenderOnce

```rust
#[derive(IntoElement)]
pub struct FocusableInput {
    focus_handle: FocusHandle,
    placeholder: SharedString,
    value: SharedString,
}

impl FocusableInput {
    pub fn new(focus_handle: FocusHandle) -> Self {
        Self {
            focus_handle,
            placeholder: "".into(),
            value: "".into(),
        }
    }

    pub fn placeholder(mut self, placeholder: impl Into<SharedString>) -> Self {
        self.placeholder = placeholder.into();
        self
    }

    pub fn value(mut self, value: impl Into<SharedString>) -> Self {
        self.value = value.into();
        self
    }
}

impl RenderOnce for FocusableInput {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div()
            .id("input")
            .track_focus(&self.focus_handle)
            .px_3()
            .py_2()
            .bg(rgb(0x1e1e1e))
            .border_1()
            .border_color(rgb(0x333333))
            .rounded_md()
            .focus(|style| {
                style
                    .border_color(rgb(0x3b82f6))
                    .bg(rgb(0x252525))
            })
            .child(if self.value.is_empty() {
                div().text_color(rgb(0x666666)).child(self.placeholder)
            } else {
                div().child(self.value)
            })
    }
}

// Usage in a Render view
impl Render for FormView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(
                FocusableInput::new(self.name_focus.clone())
                    .placeholder("Enter name")
                    .value(self.name.clone())
            )
            .child(
                FocusableInput::new(self.email_focus.clone())
                    .placeholder("Enter email")
                    .value(self.email.clone())
            )
    }
}
```

## Component Libraries

### Creating a component library (Zed UI pattern)

```rust
// lib.rs
mod components;
mod prelude;
mod theme;

pub use components::*;
pub use prelude::*;
pub use theme::*;

// prelude.rs
pub use gpui::prelude::*;
pub use gpui::{div, svg, img, px, rems, rgb, rgba, hsla};

pub use crate::components::{
    Button, ButtonStyle, IconButton,
    Card,
    Input,
    Label, LabelSize, LabelColor,
    Badge, BadgeVariant,
    Avatar,
    Icon, IconName, IconSize,
    Divider,
    Tooltip,
};

pub use crate::theme::{Theme, ThemeColors};

// components/mod.rs
mod avatar;
mod badge;
mod button;
mod card;
mod divider;
mod icon;
mod input;
mod label;
mod tooltip;

pub use avatar::*;
pub use badge::*;
pub use button::*;
pub use card::*;
pub use divider::*;
pub use icon::*;
pub use input::*;
pub use label::*;
pub use tooltip::*;
```

### Component documentation pattern

````rust
/// A button component for triggering actions.
///
/// # Examples
///
/// Basic button:
/// ```
/// Button::new("btn", "Click me")
///     .on_click(|_, _, _| println!("Clicked!"))
/// ```
///
/// Button with icon:
/// ```
/// Button::new("save", "Save")
///     .icon(IconName::Save)
///     .style(ButtonStyle::Primary)
/// ```
///
/// Disabled button:
/// ```
/// Button::new("submit", "Submit")
///     .disabled(true)
/// ```
#[derive(IntoElement)]
pub struct Button {
    // ...
}
````

### Component testing

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use gpui::TestAppContext;

    #[gpui::test]
    fn test_button_renders(cx: &mut TestAppContext) {
        cx.update(|cx| {
            let button = Button::new("test", "Test Button")
                .style(ButtonStyle::Primary);

            // Button should be creatable
            let element = button.into_any_element();
            assert!(element.is_some());
        });
    }

    #[gpui::test]
    fn test_button_click_handler(cx: &mut TestAppContext) {
        let clicked = std::sync::Arc::new(std::sync::atomic::AtomicBool::new(false));
        let clicked_clone = clicked.clone();

        cx.update(|cx| {
            let button = Button::new("test", "Click")
                .on_click(move |_, _, _| {
                    clicked_clone.store(true, std::sync::atomic::Ordering::SeqCst);
                });

            // Verify handler was set
            assert!(button.on_click.is_some());
        });
    }
}
```

### Complete component example with all patterns

```rust
use gpui::{prelude::*, div, rgb, px, SharedString, Window, App, FocusHandle};

/// Icon names available in the component library
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum IconName {
    Check,
    Close,
    ChevronDown,
    ChevronRight,
    Plus,
    Minus,
    Search,
    Settings,
}

impl IconName {
    pub fn path(&self) -> &'static str {
        match self {
            IconName::Check => "icons/check.svg",
            IconName::Close => "icons/close.svg",
            IconName::ChevronDown => "icons/chevron_down.svg",
            IconName::ChevronRight => "icons/chevron_right.svg",
            IconName::Plus => "icons/plus.svg",
            IconName::Minus => "icons/minus.svg",
            IconName::Search => "icons/search.svg",
            IconName::Settings => "icons/settings.svg",
        }
    }
}

/// Size variants for icons
#[derive(Default, Clone, Copy)]
pub enum IconSize {
    XSmall,
    Small,
    #[default]
    Medium,
    Large,
}

impl IconSize {
    pub fn px(&self) -> Pixels {
        match self {
            IconSize::XSmall => px(12.),
            IconSize::Small => px(14.),
            IconSize::Medium => px(16.),
            IconSize::Large => px(20.),
        }
    }
}

/// A flexible icon component
#[derive(IntoElement)]
pub struct Icon {
    name: IconName,
    size: IconSize,
    color: Option<Hsla>,
}

impl Icon {
    pub fn new(name: IconName) -> Self {
        Self {
            name,
            size: IconSize::default(),
            color: None,
        }
    }

    pub fn size(mut self, size: IconSize) -> Self {
        self.size = size;
        self
    }

    pub fn color(mut self, color: impl Into<Hsla>) -> Self {
        self.color = Some(color.into());
        self
    }
}

impl RenderOnce for Icon {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let size = self.size.px();

        let mut icon = svg()
            .path(self.name.path())
            .size(size);

        if let Some(color) = self.color {
            icon = icon.text_color(color);
        }

        icon
    }
}

// Usage examples:
// Icon::new(IconName::Check).size(IconSize::Small).color(rgb(0x22c55e))
// Icon::new(IconName::Close).size(IconSize::Large)
```
