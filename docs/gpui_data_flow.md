# GPUI Data Flow Patterns

This guide covers how data flows through a GPUI application—events, subscriptions, observations, and async operations. For architectural guidance on structuring your app, see [GPUI App Architecture](gpui_app_architecture.md).

## Table of Contents

1. [Events](#events)
2. [Subscriptions](#subscriptions)
3. [Observations](#observations)
4. [Async Operations](#async-operations)
5. [Loading States](#loading-states)
6. [Task Management](#task-management)
7. [Common Patterns](#common-patterns)

---

## Events

Events are typed messages that entities emit to notify subscribers of specific state changes.

### Defining Events

```rust
// Simple event enum
pub enum TodoListEvent {
    Added(TodoId),
    Removed(TodoId),
    Updated(TodoId),
    Cleared,
    Loaded,
}

// Implement EventEmitter for your entity
impl EventEmitter<TodoListEvent> for TodoList {}
```

### Multiple Event Types

An entity can emit multiple event types:

```rust
// Separate event types for different concerns
pub struct SyncStarted;
pub struct SyncCompleted {
    pub count: usize,
}
pub struct SyncFailed {
    pub error: String,
}

impl EventEmitter<SyncStarted> for TodoList {}
impl EventEmitter<SyncCompleted> for TodoList {}
impl EventEmitter<SyncFailed> for TodoList {}
```

### Emitting Events

```rust
impl TodoList {
    pub fn add(&mut self, label: String, cx: &mut Context<Self>) {
        let id = TodoId::new();
        self.todos.push(Todo { id, label, completed: false });
        
        // Emit event for subscribers
        cx.emit(TodoListEvent::Added(id));
        
        // Notify observers (triggers re-render)
        cx.notify();
    }

    pub fn sync(&mut self, cx: &mut Context<Self>) {
        cx.emit(SyncStarted);
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            match sync_to_server().await {
                Ok(count) => {
                    this.update(&mut cx, |this, cx| {
                        cx.emit(SyncCompleted { count });
                        cx.notify();
                    })?;
                }
                Err(e) => {
                    this.update(&mut cx, |this, cx| {
                        cx.emit(SyncFailed { error: e.to_string() });
                        cx.notify();
                    })?;
                }
            }
            anyhow::Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Event vs Notify

| `cx.emit(event)` | `cx.notify()` |
|------------------|---------------|
| Sends typed event to subscribers | Triggers re-render of observers |
| Use for specific state changes | Use when UI should update |
| Subscribers get event payload | Observers just know "something changed" |
| Often used together | Can be used alone |

---

## Subscriptions

Subscriptions let you react to specific events from an entity.

### Basic Subscription

```rust
pub struct TodoListView {
    list: Entity<TodoList>,
    _subscription: Subscription,  // Must keep alive
}

impl TodoListView {
    pub fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        // Subscribe to events from the list
        let subscription = cx.subscribe(&list, Self::on_list_event);

        Self {
            list,
            _subscription: subscription,
        }
    }

    fn on_list_event(
        &mut self,
        list: Entity<TodoList>,
        event: &TodoListEvent,
        cx: &mut Context<Self>,
    ) {
        match event {
            TodoListEvent::Added(id) => {
                // Maybe scroll to new item
                self.scroll_to_item(*id);
            }
            TodoListEvent::Removed(id) => {
                // Clear selection if deleted
                if self.selected_id == Some(*id) {
                    self.selected_id = None;
                }
            }
            TodoListEvent::Loaded => {
                // Reset view state
                self.selected_id = None;
                self.scroll_position = 0.0;
            }
            _ => {}
        }
        cx.notify();  // Re-render to reflect changes
    }
}
```

### Multiple Subscriptions

```rust
pub struct Dashboard {
    project: Entity<Project>,
    sync_service: Entity<SyncService>,
    _subscriptions: Vec<Subscription>,
}

impl Dashboard {
    pub fn new(
        project: Entity<Project>,
        sync_service: Entity<SyncService>,
        cx: &mut Context<Self>,
    ) -> Self {
        let subscriptions = vec![
            cx.subscribe(&project, Self::on_project_event),
            cx.subscribe(&sync_service, Self::on_sync_event),
        ];

        Self {
            project,
            sync_service,
            _subscriptions: subscriptions,
        }
    }

    fn on_project_event(
        &mut self,
        _project: Entity<Project>,
        event: &ProjectEvent,
        cx: &mut Context<Self>,
    ) {
        // Handle project events
        cx.notify();
    }

    fn on_sync_event(
        &mut self,
        _service: Entity<SyncService>,
        event: &SyncEvent,
        cx: &mut Context<Self>,
    ) {
        // Handle sync events
        cx.notify();
    }
}
```

### Subscribing to Multiple Event Types

```rust
impl MyView {
    pub fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        let subscriptions = vec![
            // Different handlers for different event types
            cx.subscribe(&list, Self::on_todo_event),
            cx.subscribe(&list, Self::on_sync_started),
            cx.subscribe(&list, Self::on_sync_completed),
        ];

        Self {
            list,
            _subscriptions: subscriptions,
        }
    }

    fn on_todo_event(&mut self, _: Entity<TodoList>, event: &TodoListEvent, cx: &mut Context<Self>) {
        // Handle TodoListEvent
    }

    fn on_sync_started(&mut self, _: Entity<TodoList>, _: &SyncStarted, cx: &mut Context<Self>) {
        self.show_sync_indicator = true;
        cx.notify();
    }

    fn on_sync_completed(&mut self, _: Entity<TodoList>, event: &SyncCompleted, cx: &mut Context<Self>) {
        self.show_sync_indicator = false;
        self.last_sync_count = Some(event.count);
        cx.notify();
    }
}
```

---

## Observations

Observations react to *any* change in an entity (whenever `cx.notify()` is called).

### Basic Observation

```rust
pub struct TodoCounter {
    list: Entity<TodoList>,
    count: usize,
    completed: usize,
    _observation: Subscription,
}

impl TodoCounter {
    pub fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        // Observe any change to the list
        let observation = cx.observe(&list, |this, list, cx| {
            this.update_counts(list, cx);
        });

        // Initialize counts
        let (count, completed) = {
            let list = list.read(cx);
            (list.todos().len(), list.completed_count())
        };

        Self {
            list,
            count,
            completed,
            _observation: observation,
        }
    }

    fn update_counts(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        let list = list.read(cx);
        self.count = list.todos().len();
        self.completed = list.completed_count();
        cx.notify();
    }
}
```

### Subscription vs Observation

```
┌────────────────────────────┬────────────────────────────────────────┐
│ cx.subscribe()             │ cx.observe()                           │
├────────────────────────────┼────────────────────────────────────────┤
│ Reacts to specific events  │ Reacts to any notify()                 │
│ Receives typed event data  │ No event payload                       │
│ Entity must impl           │ Works with any Entity                  │
│   EventEmitter             │                                        │
│ Good for specific changes  │ Good for state synchronization         │
│ "User added a todo"        │ "Something changed, update display"    │
└────────────────────────────┴────────────────────────────────────────┘
```

### When to Use Each

**Use `subscribe`** when you need to:
- React to specific events (Added vs Removed vs Updated)
- Access event payload data
- Perform different actions based on event type

**Use `observe`** when you need to:
- Keep derived state in sync
- Update a display whenever anything changes
- Simple "refresh when changed" behavior

### Observing Global State

```rust
impl ThemeAwareView {
    pub fn new(cx: &mut Context<Self>) -> Self {
        // React to global changes
        cx.observe_global::<AppSettings>(|this, cx| {
            // Settings changed, re-render
            cx.notify();
        }).detach();

        Self { /* ... */ }
    }
}
```

### Observing Entity Release

Clean up when an entity is dropped:

```rust
impl EntityTracker {
    pub fn track(&mut self, entity: Entity<TrackedThing>, cx: &mut Context<Self>) {
        let id = entity.entity_id();
        self.tracked.insert(id, entity.clone());

        // Clean up when entity is released
        cx.observe_release(&entity, move |this, _, _cx| {
            this.tracked.remove(&id);
        }).detach();
    }
}
```

---

## Async Operations

GPUI provides `cx.spawn()` for foreground async tasks and `cx.background_spawn()` for background work.

### Basic Async Pattern

```rust
impl TodoList {
    pub fn load(&mut self, cx: &mut Context<Self>) {
        self.loading = true;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            // Async work
            let todos = fetch_todos().await?;

            // Update entity with results
            this.update(&mut cx, |this, cx| {
                this.todos = todos;
                this.loading = false;
                cx.emit(TodoListEvent::Loaded);
                cx.notify();
            })?;

            anyhow::Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Background Tasks

For CPU-intensive work, use background executor:

```rust
impl DataProcessor {
    pub fn process(&mut self, data: Vec<u8>, cx: &mut Context<Self>) {
        self.processing = true;
        cx.notify();

        // Spawn background work
        let process_task = cx.background_executor().spawn(async move {
            // Heavy computation on background thread
            expensive_computation(&data)
        });

        // Spawn foreground task to await result
        cx.spawn(|this, mut cx| async move {
            let result = process_task.await;

            this.update(&mut cx, |this, cx| {
                this.result = Some(result);
                this.processing = false;
                cx.notify();
            })
        }).detach_and_log_err(cx);
    }
}
```

### Async with Error Handling

```rust
impl DataView {
    pub fn load(&mut self, cx: &mut Context<Self>) {
        self.state = LoadState::Loading;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            match fetch_data().await {
                Ok(data) => {
                    this.update(&mut cx, |this, cx| {
                        this.state = LoadState::Loaded(data);
                        cx.notify();
                    })?;
                }
                Err(e) => {
                    this.update(&mut cx, |this, cx| {
                        this.state = LoadState::Error(e.to_string());
                        cx.notify();
                    })?;
                }
            }
            anyhow::Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Spawning from Views

When spawning from a view (entity with Render), the spawn closure receives a `WeakEntity`:

```rust
impl MyView {
    fn fetch_data(&mut self, cx: &mut Context<Self>) {
        cx.spawn(|this, mut cx| async move {
            let data = fetch().await?;

            // `this` is WeakEntity<Self>
            this.update(&mut cx, |this, cx| {
                this.data = data;
                cx.notify();
            })?;

            anyhow::Ok(())
        }).detach_and_log_err(cx);
    }
}
```

---

## Loading States

A common pattern for async data:

```rust
pub enum LoadState<T> {
    Idle,
    Loading,
    Loaded(T),
    Error(String),
}

pub struct DataView {
    data: LoadState<Vec<Item>>,
}

impl DataView {
    pub fn load(&mut self, cx: &mut Context<Self>) {
        self.data = LoadState::Loading;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            match fetch_items().await {
                Ok(items) => {
                    this.update(&mut cx, |this, cx| {
                        this.data = LoadState::Loaded(items);
                        cx.notify();
                    })?;
                }
                Err(e) => {
                    this.update(&mut cx, |this, cx| {
                        this.data = LoadState::Error(e.to_string());
                        cx.notify();
                    })?;
                }
            }
            anyhow::Ok(())
        }).detach_and_log_err(cx);
    }
}

impl Render for DataView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        match &self.data {
            LoadState::Idle => div().child("Click to load"),
            LoadState::Loading => div().child(Spinner::new()),
            LoadState::Loaded(items) => {
                div().children(items.iter().map(|item| ItemRow::new(item.clone())))
            }
            LoadState::Error(err) => {
                div()
                    .text_color(rgb(0xef4444))
                    .child(format!("Error: {}", err))
            }
        }
    }
}
```

---

## Task Management

### Storing Tasks

Store tasks to prevent them from being cancelled:

```rust
pub struct DataFetcher {
    fetch_task: Option<Task<()>>,
}

impl DataFetcher {
    pub fn fetch(&mut self, cx: &mut Context<Self>) {
        // Store task to keep it alive
        self.fetch_task = Some(cx.spawn(|this, mut cx| async move {
            let data = fetch_data().await?;
            this.update(&mut cx, |this, cx| {
                this.data = data;
                cx.notify();
            })
        }));
    }
}
```

### Task Cancellation

Cancel previous tasks when starting new ones:

```rust
pub struct SearchView {
    query: String,
    results: Vec<SearchResult>,
    search_task: Option<Task<()>>,
}

impl SearchView {
    pub fn search(&mut self, query: String, cx: &mut Context<Self>) {
        // Cancel any pending search
        self.search_task.take();

        self.query = query.clone();
        cx.notify();

        if query.is_empty() {
            self.results.clear();
            return;
        }

        // Start new search
        self.search_task = Some(cx.spawn(|this, mut cx| async move {
            // Debounce
            cx.background_executor()
                .timer(Duration::from_millis(300))
                .await;

            let results = perform_search(&query).await?;

            this.update(&mut cx, |this, cx| {
                this.results = results;
                cx.notify();
            })
        }));
    }
}
```

### Detaching vs Storing Tasks

| Pattern | Use When |
|---------|----------|
| `.detach()` | Fire-and-forget, task should run to completion |
| `.detach_and_log_err(cx)` | Same, but log any errors |
| Store in field | Need to cancel task, or prevent drop |
| Await in another task | Need the result |

---

## Common Patterns

### Debouncing

```rust
impl SearchInput {
    pub fn on_input_changed(&mut self, text: String, cx: &mut Context<Self>) {
        self.text = text.clone();
        self.debounce_task.take();  // Cancel previous

        self.debounce_task = Some(cx.spawn(|this, mut cx| async move {
            cx.background_executor()
                .timer(Duration::from_millis(300))
                .await;

            this.update(&mut cx, |this, cx| {
                this.perform_search(cx);
            })
        }));

        cx.notify();
    }
}
```

### Optimistic Updates

```rust
impl TodoList {
    pub fn toggle_optimistic(&mut self, id: TodoId, cx: &mut Context<Self>) {
        // Optimistically update UI
        let Some(todo) = self.todos.iter_mut().find(|t| t.id == id) else {
            return;
        };
        todo.completed = !todo.completed;
        let new_state = todo.completed;
        cx.notify();

        // Sync to server in background
        cx.spawn(|this, mut cx| async move {
            match update_todo_on_server(id, new_state).await {
                Ok(_) => {
                    // Success - keep the optimistic update
                }
                Err(_) => {
                    // Revert on failure
                    this.update(&mut cx, |this, cx| {
                        if let Some(todo) = this.todos.iter_mut().find(|t| t.id == id) {
                            todo.completed = !todo.completed;
                        }
                        cx.emit(TodoListEvent::SyncFailed);
                        cx.notify();
                    })?;
                }
            }
            anyhow::Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Progress Updates

```rust
impl ImportView {
    pub fn import(&mut self, file: PathBuf, cx: &mut Context<Self>) {
        self.progress = Some(Progress { current: 0, total: 0 });
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            let data = tokio::fs::read(&file).await?;
            let total = data.len();

            for (i, chunk) in data.chunks(1024).enumerate() {
                // Process chunk...
                process_chunk(chunk).await?;

                // Update progress
                this.update(&mut cx, |this, cx| {
                    if let Some(ref mut p) = this.progress {
                        p.current = (i + 1) * 1024;
                        p.total = total;
                    }
                    cx.notify();
                })?;
            }

            this.update(&mut cx, |this, cx| {
                this.progress = None;
                cx.emit(ImportEvent::Completed);
                cx.notify();
            })?;

            anyhow::Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Request Deduplication

```rust
pub struct DataCache {
    cache: HashMap<String, Item>,
    pending: HashMap<String, Shared<Task<Result<Item>>>>,
}

impl DataCache {
    pub fn get(&mut self, key: String, cx: &mut Context<Self>) -> Task<Result<Item>> {
        // Return cached value
        if let Some(item) = self.cache.get(&key) {
            return Task::ready(Ok(item.clone()));
        }

        // Return pending request
        if let Some(pending) = self.pending.get(&key) {
            return Task::ready(pending.clone().await);
        }

        // Start new request
        let task = cx.spawn(|this, mut cx| async move {
            let item = fetch_item(&key).await?;

            this.update(&mut cx, |this, _cx| {
                this.cache.insert(key.clone(), item.clone());
                this.pending.remove(&key);
            })?;

            Ok(item)
        }).shared();

        self.pending.insert(key, task.clone());
        Task::ready(task.await)
    }
}
```

### Pagination

```rust
pub struct PaginatedList {
    items: Vec<Item>,
    page: usize,
    per_page: usize,
    total: Option<usize>,
    loading: bool,
}

impl PaginatedList {
    pub fn load_page(&mut self, page: usize, cx: &mut Context<Self>) {
        self.loading = true;
        self.page = page;
        cx.notify();

        let per_page = self.per_page;
        cx.spawn(|this, mut cx| async move {
            let response = fetch_page(page, per_page).await?;

            this.update(&mut cx, |this, cx| {
                this.items = response.items;
                this.total = Some(response.total);
                this.loading = false;
                cx.notify();
            })
        }).detach_and_log_err(cx);
    }

    pub fn next_page(&mut self, cx: &mut Context<Self>) {
        if let Some(total) = self.total {
            if (self.page + 1) * self.per_page < total {
                self.load_page(self.page + 1, cx);
            }
        }
    }

    pub fn prev_page(&mut self, cx: &mut Context<Self>) {
        if self.page > 0 {
            self.load_page(self.page - 1, cx);
        }
    }
}
```

### Infinite Scroll

```rust
pub struct InfiniteList {
    items: Vec<Item>,
    cursor: Option<String>,
    loading: bool,
    has_more: bool,
}

impl InfiniteList {
    pub fn load_more(&mut self, cx: &mut Context<Self>) {
        if self.loading || !self.has_more {
            return;
        }

        self.loading = true;
        cx.notify();

        let cursor = self.cursor.clone();
        cx.spawn(|this, mut cx| async move {
            let response = fetch_next(cursor).await?;

            this.update(&mut cx, |this, cx| {
                this.items.extend(response.items);
                this.cursor = response.next_cursor;
                this.has_more = response.has_more;
                this.loading = false;
                cx.notify();
            })
        }).detach_and_log_err(cx);
    }
}

impl Render for InfiniteList {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("infinite-list")
            .overflow_y_scroll()
            .on_scroll_to_bottom(cx.listener(|this, _, _, cx| {
                this.load_more(cx);
            }))
            .children(self.items.iter().map(|item| {
                ItemRow::new(item.clone())
            }))
            .when(self.loading, |this| {
                this.child(div().child("Loading more..."))
            })
    }
}
```

---

## Summary

| Mechanism | Purpose |
|-----------|---------|
| `cx.emit(event)` | Send typed event to subscribers |
| `cx.notify()` | Trigger re-render for observers |
| `cx.subscribe()` | React to specific events |
| `cx.observe()` | React to any state change |
| `cx.spawn()` | Foreground async task |
| `cx.background_executor().spawn()` | Background thread work |

### Related Documentation

- [GPUI App Architecture](gpui_app_architecture.md) - How to structure your app
- [GPUI Components](gpui_components.md) - Render vs RenderOnce patterns
