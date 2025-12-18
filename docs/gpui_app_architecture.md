# GPUI App Architecture

This guide explains how to structure a GPUI application—the relationship between entities, views, components, and shared state. Understanding this architecture is key to building well-organized GPUI apps.

## Table of Contents

1. [The Four Layers](#the-four-layers)
2. [Entities: The Foundation](#entities-the-foundation)
3. [Views: Stateful UI](#views-stateful-ui)
4. [Components: Stateless UI](#components-stateless-ui)
5. [Shared Services](#shared-services)
6. [Passing Data Down](#passing-data-down)
7. [Choosing the Right Pattern](#choosing-the-right-pattern)
8. [Complete App Example](#complete-app-example)

---

## The Four Layers

A GPUI application typically has four conceptual layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Application Layer                            │
│   Global state, services, registries (Arc<T>, impl Global)          │
├─────────────────────────────────────────────────────────────────────┤
│                         Entity Layer                                │
│   Domain models, business logic (Entity<T>, no Render)              │
├─────────────────────────────────────────────────────────────────────┤
│                          View Layer                                 │
│   Stateful UI containers (Entity<T> + impl Render)                  │
├─────────────────────────────────────────────────────────────────────┤
│                        Component Layer                              │
│   Stateless UI pieces (impl RenderOnce, #[derive(IntoElement)])     │
└─────────────────────────────────────────────────────────────────────┘
```

### How They Interact

```
Application Layer
      │
      │ Arc<T> (shared by reference)
      │ Global (accessed via cx)
      ▼
Entity Layer ◄────────────────────────┐
      │                               │
      │ Entity<T> (owned/borrowed)    │ events, subscriptions
      │                               │
      ▼                               │
View Layer ───────────────────────────┘
      │
      │ Props (cloned data, callbacks)
      │
      ▼
Component Layer
```

---

## Entities: The Foundation

An `Entity<T>` is a **handle to state** managed by GPUI's runtime. Entities are the building blocks of GPUI applications.

### Pure Data Entities (No UI)

Some entities hold domain data and business logic without any UI concerns:

```rust
// A pure data entity - no Render impl
pub struct TodoList {
    todos: Vec<Todo>,
    filter: TodoFilter,
}

impl TodoList {
    pub fn new(_cx: &mut Context<Self>) -> Self {
        Self {
            todos: Vec::new(),
            filter: TodoFilter::All,
        }
    }

    pub fn add(&mut self, label: String, cx: &mut Context<Self>) {
        self.todos.push(Todo {
            id: TodoId::new(),
            label,
            completed: false,
        });
        cx.emit(TodoListEvent::Added);
        cx.notify();
    }

    pub fn toggle(&mut self, id: TodoId, cx: &mut Context<Self>) {
        if let Some(todo) = self.todos.iter_mut().find(|t| t.id == id) {
            todo.completed = !todo.completed;
            cx.emit(TodoListEvent::Updated(id));
            cx.notify();
        }
    }

    pub fn visible_todos(&self) -> impl Iterator<Item = &Todo> {
        self.todos.iter().filter(|t| match self.filter {
            TodoFilter::All => true,
            TodoFilter::Active => !t.completed,
            TodoFilter::Completed => t.completed,
        })
    }
}

impl EventEmitter<TodoListEvent> for TodoList {}
```

### Why Separate Data from UI?

1. **Testability**: Test business logic without UI
2. **Reusability**: Same data model, different views
3. **Clarity**: Clear separation of concerns
4. **Sharing**: Multiple views can share one data entity

```rust
// Multiple views can observe the same entity
let list = cx.new(|cx| TodoList::new(cx));

// A list view
let list_view = cx.new(|cx| TodoListView::new(list.clone(), cx));

// A stats view watching the same data
let stats_view = cx.new(|cx| TodoStatsView::new(list.clone(), cx));

// A sidebar counter
let counter = cx.new(|cx| TodoCounter::new(list.clone(), cx));
```

---

## Views: Stateful UI

A **view** is an `Entity<T>` where `T` implements `Render`. Views:

- Own their local UI state (selection, scroll position, focus)
- Hold references to data entities they need
- Subscribe to events from those entities
- Can be rendered as children of other elements

```rust
pub struct TodoListView {
    // Reference to shared data
    list: Entity<TodoList>,

    // Local UI state
    selected_id: Option<TodoId>,
    scroll_position: f32,
    focus_handle: FocusHandle,

    // Keep subscriptions alive
    _subscription: Subscription,
}

impl TodoListView {
    pub fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        // React to data changes
        let subscription = cx.subscribe(&list, Self::on_list_event);

        Self {
            list,
            selected_id: None,
            scroll_position: 0.0,
            focus_handle: cx.focus_handle(),
            _subscription: subscription,
        }
    }

    fn on_list_event(
        &mut self,
        _list: Entity<TodoList>,
        event: &TodoListEvent,
        cx: &mut Context<Self>,
    ) {
        match event {
            TodoListEvent::Removed(id) if self.selected_id == Some(*id) => {
                self.selected_id = None;
            }
            _ => {}
        }
        cx.notify();
    }

    fn select(&mut self, id: TodoId, cx: &mut Context<Self>) {
        self.selected_id = Some(id);
        cx.notify();
    }
}

impl Render for TodoListView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let list = self.list.read(cx);

        div()
            .track_focus(&self.focus_handle)
            .flex()
            .flex_col()
            .gap_2()
            .children(list.visible_todos().map(|todo| {
                TodoItem::new(todo.clone())
                    .selected(self.selected_id == Some(todo.id))
                    .on_click(cx.listener(move |this, _, _, cx| {
                        this.select(todo.id, cx);
                    }))
                    .on_toggle(cx.listener(move |this, _, _, cx| {
                        this.list.update(cx, |list, cx| {
                            list.toggle(todo.id, cx);
                        });
                    }))
            }))
    }
}
```

### When to Use a View (Entity + Render)

- UI that needs **local state** (selection, focus, scroll, expanded/collapsed)
- UI that **subscribes to events** from data entities
- UI that manages **async operations** (loading, tasks)
- Complex UI that benefits from **encapsulation**

---

## Components: Stateless UI

Components are **stateless UI building blocks** that implement `RenderOnce`. They:

- Receive all data through their constructor/builder
- Have no subscriptions or observations
- Are recreated every render (cheap and simple)
- Report interactions via callbacks

```rust
#[derive(IntoElement)]
pub struct TodoItem {
    todo: Todo,
    selected: bool,
    on_click: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
    on_toggle: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
}

impl TodoItem {
    pub fn new(todo: Todo) -> Self {
        Self {
            todo,
            selected: false,
            on_click: None,
            on_toggle: None,
        }
    }

    pub fn selected(mut self, selected: bool) -> Self {
        self.selected = selected;
        self
    }

    pub fn on_click(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_click = Some(Box::new(handler));
        self
    }

    pub fn on_toggle(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_toggle = Some(Box::new(handler));
        self
    }
}

impl RenderOnce for TodoItem {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div()
            .id(("todo-item", self.todo.id.0))
            .flex()
            .items_center()
            .gap_2()
            .p_2()
            .bg(if self.selected { rgb(0x3b82f6) } else { rgb(0x1e1e1e) })
            .rounded_md()
            .cursor_pointer()
            .when_some(self.on_click, |this, on_click| {
                this.on_click(on_click)
            })
            .child(
                Checkbox::new(self.todo.completed)
                    .when_some(self.on_toggle, |this, on_toggle| {
                        this.on_click(on_toggle)
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
    }
}
```

### When to Use RenderOnce Components

- **Presentational UI** that just displays data
- **Reusable widgets** (buttons, badges, cards, icons)
- UI that **doesn't need to persist** state between renders
- **List items** rendered from data

### This Is NOT Prop Drilling

In React, "prop drilling" is considered an anti-pattern because:
- Props pass through many intermediate components that don't use them
- Changes require modifying many components
- It creates tight coupling

**GPUI is different:**

1. **Components are recreated each render** - there's no "memorization" that prop changes would invalidate
2. **Data flows from Entities, not through component chains** - Views read directly from shared entities
3. **Callbacks go directly to the handler** - No bubbling through intermediate components
4. **It's explicit and traceable** - You can see exactly where data comes from

```rust
// This is fine in GPUI:
impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let settings = cx.global::<AppSettings>();
        let data = self.data_entity.read(cx);

        div()
            .child(
                Card::new()
                    .child(Header::new(data.title.clone()).size(settings.font_size))
                    .child(Content::new(data.body.clone()))
                    .child(
                        Footer::new()
                            .theme(settings.theme.clone())
                            .on_action(cx.listener(|this, _, _, cx| {
                                this.handle_action(cx);
                            }))
                    )
            )
    }
}
```

---

## Shared Services

For data that needs to be accessed across the entire application, use **`Arc<T>`** for immutable/interior-mutable services or **`impl Global`** for mutable global state.

### Arc Pattern (Zed's Approach)

Use `Arc<T>` for services, registries, and shared resources:

```rust
// A registry that's shared across the app
pub struct LanguageRegistry {
    languages: RwLock<Vec<Arc<Language>>>,
    // ...
}

impl LanguageRegistry {
    pub fn new() -> Arc<Self> {
        Arc::new(Self {
            languages: RwLock::new(Vec::new()),
        })
    }

    pub fn get_language(&self, name: &str) -> Option<Arc<Language>> {
        self.languages.read().iter()
            .find(|lang| lang.name() == name)
            .cloned()
    }

    pub fn add_language(&self, language: Arc<Language>) {
        self.languages.write().push(language);
    }
}

// App setup
struct AppState {
    language_registry: Arc<LanguageRegistry>,
    http_client: Arc<dyn HttpClient>,
    project: Entity<Project>,
}

impl AppState {
    fn new(cx: &mut App) -> Self {
        let language_registry = LanguageRegistry::new();
        let http_client = Arc::new(HttpClientImpl::new());

        Self {
            language_registry: language_registry.clone(),
            http_client,
            project: cx.new(|cx| Project::new(language_registry, cx)),
        }
    }
}
```

### When to Use Arc<T>

- **Registries** that hold collections of items (languages, themes, extensions)
- **Services** with internal caching (HTTP client, database pool)
- **Immutable configuration** that's read frequently
- Data that needs **interior mutability** (`RwLock`, `Mutex`)

### Passing Arc to Components

```rust
impl EditorView {
    fn new(
        language_registry: Arc<LanguageRegistry>,
        cx: &mut Context<Self>,
    ) -> Self {
        Self {
            language_registry,
            buffer: cx.new(|cx| Buffer::new(cx)),
        }
    }
}

// Passing to child views
impl Render for Workspace {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        EditorPanel::new(
            self.editor.clone(),
            self.language_registry.clone(),  // Arc is cheap to clone
        )
    }
}
```

### Global State Pattern

For state that truly needs to be globally accessible:

```rust
pub struct AppSettings {
    pub theme: Theme,
    pub font_size: f32,
    pub auto_save: bool,
}

impl Global for AppSettings {}

// Initialize at startup
fn main() {
    Application::new().run(|cx: &mut App| {
        cx.set_global(AppSettings {
            theme: Theme::dark(),
            font_size: 14.0,
            auto_save: true,
        });
    });
}

// Access from anywhere with a context
fn use_settings(cx: &App) -> &AppSettings {
    cx.global::<AppSettings>()
}

// React to changes
impl MyView {
    fn new(cx: &mut Context<Self>) -> Self {
        cx.observe_global::<AppSettings>(|this, cx| {
            cx.notify();  // Re-render when settings change
        }).detach();

        Self { /* ... */ }
    }
}
```

### When to Use Global

- **App-wide settings** (theme, preferences)
- **Singleton services** that need mutable access
- State where **every view needs to react** to changes

---

## Passing Data Down

### Pattern 1: Clone Data to Components

For components that just display data, clone what they need:

```rust
impl Render for ListView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let items = self.list.read(cx);

        div()
            .children(items.iter().map(|item| {
                // Clone the data the component needs
                ListItem::new(item.clone())
            }))
    }
}
```

### Pattern 2: Pass Entity References to Views

Child views that need to subscribe to changes should receive the entity:

```rust
impl Workspace {
    fn new(cx: &mut Context<Self>) -> Self {
        let project = cx.new(|cx| Project::new(cx));
        let file_tree = cx.new(|cx| FileTree::new(project.clone(), cx));
        let editor = cx.new(|cx| Editor::new(project.clone(), cx));

        Self { project, file_tree, editor }
    }
}
```

### Pattern 3: Pass Callbacks Up

Components report interactions through callbacks:

```rust
// Parent creates the callback
impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .children(self.items.iter().map(|item| {
                let item_id = item.id;

                ItemCard::new(item.clone())
                    .on_click(cx.listener(move |this, _, _, cx| {
                        this.select_item(item_id, cx);
                    }))
                    .on_delete(cx.listener(move |this, _, _, cx| {
                        this.delete_item(item_id, cx);
                    }))
            }))
    }
}
```

### Pattern 4: Arc for Read-Heavy Shared Data

When many components need access to the same data:

```rust
pub struct ThemeColors {
    pub background: Hsla,
    pub foreground: Hsla,
    pub accent: Hsla,
    pub border: Hsla,
    // ... many more colors
}

// Store as Arc to avoid cloning all fields
struct AppTheme {
    colors: Arc<ThemeColors>,
}

// Pass Arc to components
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let theme = cx.global::<AppTheme>();

        Panel::new()
            .colors(theme.colors.clone())  // Cheap Arc clone
            .child(/* ... */)
    }
}
```

---

## Choosing the Right Pattern

### Decision Flowchart

```
Does this piece of UI need to:
│
├─► Persist state between renders? (selection, scroll, focus)
│   │
│   ├─► YES ──► Use Entity + Render (View)
│   │           Can it be local to this view?
│   │           ├─► YES ──► Store in view struct
│   │           └─► NO ───► Create/receive separate Entity
│   │
│   └─► NO ───► Use RenderOnce (Component)
│
├─► Subscribe to events from another entity?
│   └─► YES ──► Use Entity + Render (View)
│
├─► Just display data and report clicks?
│   └─► YES ──► Use RenderOnce (Component)
│
└─► Be accessed from many unrelated parts of the app?
    │
    ├─► Mutable, changes trigger re-renders
    │   └─► impl Global
    │
    └─► Mostly immutable, or interior mutability
        └─► Arc<T>
```

### Quick Reference

| Pattern | When to Use |
|---------|-------------|
| `Entity<T>` (no Render) | Domain models, business logic, shared data |
| `Entity<T>` + `Render` | Views with local UI state, subscriptions |
| `RenderOnce` | Stateless UI components, list items |
| `Arc<T>` | Registries, services, heavy read-only data |
| `impl Global` | App-wide settings, singleton mutable state |

### State Location Guidance

| State Type | Location |
|------------|----------|
| Domain data (todos, documents) | `Entity<T>` (no Render) |
| Selection, focus, hover | View (`Entity<T>` + `Render`) |
| Scroll position | View |
| Expanded/collapsed | View |
| Loading state | View or Entity (depends on scope) |
| Form input values | View (or Entity if shared) |
| Theme/settings | Global |
| Services/registries | `Arc<T>` |

---

## Complete App Example

Here's how all the pieces fit together in a realistic todo application:

```rust
use gpui::{prelude::*, Entity, EventEmitter, Global, Subscription};
use std::sync::Arc;

// ════════════════════════════════════════════════════════════════════
// SHARED SERVICES (Arc<T>)
// ════════════════════════════════════════════════════════════════════

pub struct ApiClient {
    base_url: String,
}

impl ApiClient {
    pub fn new(base_url: &str) -> Arc<Self> {
        Arc::new(Self {
            base_url: base_url.to_string(),
        })
    }

    pub async fn fetch_todos(&self) -> anyhow::Result<Vec<Todo>> {
        // HTTP request implementation
        Ok(vec![])
    }

    pub async fn save_todo(&self, todo: &Todo) -> anyhow::Result<()> {
        // HTTP request implementation
        Ok(())
    }
}

// ════════════════════════════════════════════════════════════════════
// GLOBAL STATE (impl Global)
// ════════════════════════════════════════════════════════════════════

pub struct AppSettings {
    pub theme: Theme,
    pub show_completed: bool,
}

impl Global for AppSettings {}

pub struct Theme {
    pub background: Hsla,
    pub surface: Hsla,
    pub text: Hsla,
    pub accent: Hsla,
}

impl Theme {
    pub fn dark() -> Self {
        Self {
            background: rgb(0x1e1e1e).into(),
            surface: rgb(0x2d2d2d).into(),
            text: rgb(0xffffff).into(),
            accent: rgb(0x3b82f6).into(),
        }
    }
}

// ════════════════════════════════════════════════════════════════════
// DOMAIN MODELS
// ════════════════════════════════════════════════════════════════════

#[derive(Clone, Debug, PartialEq)]
pub struct TodoId(pub uuid::Uuid);

impl TodoId {
    pub fn new() -> Self {
        Self(uuid::Uuid::new_v4())
    }
}

#[derive(Clone, Debug)]
pub struct Todo {
    pub id: TodoId,
    pub label: String,
    pub completed: bool,
}

// ════════════════════════════════════════════════════════════════════
// DATA ENTITY (Entity without Render)
// ════════════════════════════════════════════════════════════════════

pub enum TodoListEvent {
    Added(TodoId),
    Updated(TodoId),
    Removed(TodoId),
    Loaded,
}

pub struct TodoList {
    todos: Vec<Todo>,
    api_client: Arc<ApiClient>,
    loading: bool,
}

impl EventEmitter<TodoListEvent> for TodoList {}

impl TodoList {
    pub fn new(api_client: Arc<ApiClient>, _cx: &mut Context<Self>) -> Self {
        Self {
            todos: Vec::new(),
            api_client,
            loading: false,
        }
    }

    pub fn todos(&self) -> &[Todo] {
        &self.todos
    }

    pub fn is_loading(&self) -> bool {
        self.loading
    }

    pub fn add(&mut self, label: String, cx: &mut Context<Self>) {
        let todo = Todo {
            id: TodoId::new(),
            label,
            completed: false,
        };
        let id = todo.id.clone();
        self.todos.push(todo);
        cx.emit(TodoListEvent::Added(id));
        cx.notify();
    }

    pub fn toggle(&mut self, id: &TodoId, cx: &mut Context<Self>) {
        if let Some(todo) = self.todos.iter_mut().find(|t| &t.id == id) {
            todo.completed = !todo.completed;
            cx.emit(TodoListEvent::Updated(id.clone()));
            cx.notify();
        }
    }

    pub fn remove(&mut self, id: &TodoId, cx: &mut Context<Self>) {
        self.todos.retain(|t| &t.id != id);
        cx.emit(TodoListEvent::Removed(id.clone()));
        cx.notify();
    }

    pub fn load(&mut self, cx: &mut Context<Self>) {
        self.loading = true;
        cx.notify();

        let api = self.api_client.clone();
        cx.spawn(|this, mut cx| async move {
            match api.fetch_todos().await {
                Ok(todos) => {
                    this.update(&mut cx, |this, cx| {
                        this.todos = todos;
                        this.loading = false;
                        cx.emit(TodoListEvent::Loaded);
                        cx.notify();
                    })?;
                }
                Err(e) => {
                    this.update(&mut cx, |this, cx| {
                        this.loading = false;
                        cx.notify();
                    })?;
                    log::error!("Failed to load todos: {}", e);
                }
            }
            anyhow::Ok(())
        })
        .detach_and_log_err(cx);
    }
}

// ════════════════════════════════════════════════════════════════════
// VIEW (Entity with Render)
// ════════════════════════════════════════════════════════════════════

pub struct TodoListView {
    // Reference to shared data entity
    list: Entity<TodoList>,

    // Local UI state
    selected_id: Option<TodoId>,
    input_text: String,
    focus_handle: FocusHandle,

    // Keep subscriptions alive
    _subscription: Subscription,
}

impl TodoListView {
    pub fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        let subscription = cx.subscribe(&list, Self::on_list_event);

        // Load initial data
        list.update(cx, |list, cx| list.load(cx));

        Self {
            list,
            selected_id: None,
            input_text: String::new(),
            focus_handle: cx.focus_handle(),
            _subscription: subscription,
        }
    }

    fn on_list_event(
        &mut self,
        _list: Entity<TodoList>,
        event: &TodoListEvent,
        cx: &mut Context<Self>,
    ) {
        match event {
            TodoListEvent::Removed(id) if self.selected_id.as_ref() == Some(id) => {
                self.selected_id = None;
            }
            _ => {}
        }
        cx.notify();
    }

    fn select(&mut self, id: TodoId, cx: &mut Context<Self>) {
        self.selected_id = Some(id);
        cx.notify();
    }

    fn add_todo(&mut self, cx: &mut Context<Self>) {
        if !self.input_text.is_empty() {
            let label = std::mem::take(&mut self.input_text);
            self.list.update(cx, |list, cx| {
                list.add(label, cx);
            });
            cx.notify();
        }
    }
}

impl Render for TodoListView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let settings = cx.global::<AppSettings>();
        let list = self.list.read(cx);

        div()
            .track_focus(&self.focus_handle)
            .size_full()
            .flex()
            .flex_col()
            .bg(settings.theme.background)
            .p_4()
            .gap_4()
            // Input area
            .child(
                div()
                    .flex()
                    .gap_2()
                    .child(
                        TextInput::new(self.input_text.clone())
                            .placeholder("Add a todo...")
                            .on_change(cx.listener(|this, text: &String, _, cx| {
                                this.input_text = text.clone();
                                cx.notify();
                            }))
                    )
                    .child(
                        Button::new("add", "Add")
                            .on_click(cx.listener(|this, _, _, cx| {
                                this.add_todo(cx);
                            }))
                    )
            )
            // List area
            .child(
                div()
                    .flex_1()
                    .overflow_y_scroll()
                    .flex()
                    .flex_col()
                    .gap_2()
                    .when(list.is_loading(), |this| {
                        this.child(
                            div()
                                .text_color(settings.theme.text)
                                .child("Loading...")
                        )
                    })
                    .when(!list.is_loading(), |this| {
                        this.children(
                            list.todos()
                                .iter()
                                .filter(|t| settings.show_completed || !t.completed)
                                .map(|todo| {
                                    let todo_id = todo.id.clone();
                                    let toggle_id = todo.id.clone();
                                    let delete_id = todo.id.clone();

                                    // Pass data down to component
                                    TodoItemComponent::new(todo.clone())
                                        .selected(self.selected_id.as_ref() == Some(&todo.id))
                                        .theme(&settings.theme)
                                        .on_click(cx.listener(move |this, _, _, cx| {
                                            this.select(todo_id.clone(), cx);
                                        }))
                                        .on_toggle(cx.listener(move |this, _, _, cx| {
                                            this.list.update(cx, |list, cx| {
                                                list.toggle(&toggle_id, cx);
                                            });
                                        }))
                                        .on_delete(cx.listener(move |this, _, _, cx| {
                                            this.list.update(cx, |list, cx| {
                                                list.remove(&delete_id, cx);
                                            });
                                        }))
                                })
                        )
                    })
            )
    }
}

// ════════════════════════════════════════════════════════════════════
// COMPONENT (RenderOnce)
// ════════════════════════════════════════════════════════════════════

#[derive(IntoElement)]
pub struct TodoItemComponent {
    // Data passed in
    todo: Todo,
    selected: bool,
    theme: Option<Theme>,

    // Callbacks
    on_click: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
    on_toggle: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
    on_delete: Option<Box<dyn Fn(&ClickEvent, &mut Window, &mut App) + 'static>>,
}

impl TodoItemComponent {
    pub fn new(todo: Todo) -> Self {
        Self {
            todo,
            selected: false,
            theme: None,
            on_click: None,
            on_toggle: None,
            on_delete: None,
        }
    }

    pub fn selected(mut self, selected: bool) -> Self {
        self.selected = selected;
        self
    }

    pub fn theme(mut self, theme: &Theme) -> Self {
        self.theme = Some(Theme {
            background: theme.background,
            surface: theme.surface,
            text: theme.text,
            accent: theme.accent,
        });
        self
    }

    pub fn on_click(
        mut self,
        handler: impl Fn(&ClickEvent, &mut Window, &mut App) + 'static,
    ) -> Self {
        self.on_click = Some(Box::new(handler));
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

impl RenderOnce for TodoItemComponent {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        let theme = self.theme.unwrap_or_else(|| Theme::dark());

        div()
            .id(("todo", self.todo.id.0))
            .flex()
            .items_center()
            .gap_2()
            .p_2()
            .bg(if self.selected { theme.accent } else { theme.surface })
            .rounded_md()
            .cursor_pointer()
            .when_some(self.on_click, |this, handler| {
                this.on_click(handler)
            })
            // Checkbox
            .child(
                div()
                    .id(("checkbox", self.todo.id.0))
                    .size_5()
                    .rounded_sm()
                    .border_1()
                    .border_color(theme.text.opacity(0.3))
                    .when(self.todo.completed, |this| this.bg(theme.accent))
                    .cursor_pointer()
                    .when_some(self.on_toggle, |this, handler| {
                        this.on_click(handler)
                    })
            )
            // Label
            .child(
                div()
                    .flex_1()
                    .text_color(theme.text)
                    .when(self.todo.completed, |this| {
                        this.line_through().text_color(theme.text.opacity(0.5))
                    })
                    .child(self.todo.label.clone())
            )
            // Delete button
            .child(
                div()
                    .id(("delete", self.todo.id.0))
                    .px_2()
                    .text_color(rgb(0xef4444))
                    .cursor_pointer()
                    .hover(|s| s.text_color(rgb(0xff6666)))
                    .when_some(self.on_delete, |this, handler| {
                        this.on_click(handler)
                    })
                    .child("×")
            )
    }
}

// ════════════════════════════════════════════════════════════════════
// APP INITIALIZATION
// ════════════════════════════════════════════════════════════════════

fn main() {
    Application::new().run(|cx: &mut App| {
        // Set up global state
        cx.set_global(AppSettings {
            theme: Theme::dark(),
            show_completed: true,
        });

        // Create shared services
        let api_client = ApiClient::new("https://api.example.com");

        // Create data entity
        let todo_list = cx.new(|cx| TodoList::new(api_client, cx));

        // Open window with view
        cx.open_window(
            WindowOptions::default(),
            |_, cx| cx.new(|cx| TodoListView::new(todo_list, cx)),
        )
        .unwrap();
    });
}
```

---

## Summary

Building a GPUI app is about choosing the right layer for each piece of your application:

| Layer | Type | Purpose |
|-------|------|---------|
| **Application** | `Arc<T>`, `impl Global` | Services, settings, registries |
| **Entity** | `Entity<T>` (no Render) | Domain data, business logic |
| **View** | `Entity<T>` + `Render` | Stateful UI containers |
| **Component** | `RenderOnce` | Stateless UI building blocks |

### Key Principles

1. **Entities own data** - Put your domain logic in Entity types without Render
2. **Views own UI state** - Selection, focus, scroll position belong in views
3. **Components are pure** - They receive data, render it, and report interactions
4. **Passing data down is fine** - Unlike React, this is the intended pattern in GPUI
5. **Arc for shared services** - Use Arc<T> for registries and services
6. **Global for app-wide state** - Settings and singletons that trigger re-renders

### Related Documentation

- [GPUI Components](gpui_components.md) - Deep dive on Render vs RenderOnce
- [GPUI Data Flow](gpui_data_flow.md) - Events, subscriptions, and async patterns
- [GPUI Styling](gpui_styling.md) - How to style your components
