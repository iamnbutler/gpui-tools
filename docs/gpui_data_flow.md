# GPUI Data Flow Patterns

1. [High Level Patterns](#high-level-patterns)
2. [State Management](#state-management)
3. [Entity Communication](#entity-communication)
4. [Event System](#event-system)
5. [Subscriptions & Observations](#subscriptions--observations)
6. [Global State](#global-state)
7. [Async Data Flow](#async-data-flow)
8. [Parent-Child Communication](#parent-child-communication)
9. [Cross-Window Communication](#cross-window-communication)
10. [Data Loading Patterns](#data-loading-patterns)
11. [Caching Patterns](#caching-patterns)

## High Level Patterns

### Data flow principles in GPUI

```rust
// GPUI follows a unidirectional data flow pattern:
// 1. State is owned by Entities
// 2. State changes trigger notifications
// 3. Notifications cause re-renders
// 4. UI reflects current state

// Key concepts:
// - Entities own state
// - Events communicate changes
// - Subscriptions react to events
// - Observations watch for any state change
// - cx.notify() triggers re-render
```

### Data flow diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Application                                  │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐           │
│  │   Entity A  │────▶│   Entity B  │────▶│   Entity C  │           │
│  │  (Model)    │emit │  (Service)  │emit │   (View)    │           │
│  └─────────────┘     └─────────────┘     └─────────────┘           │
│        │                   │                   │                    │
│        │ update            │ subscribe         │ observe            │
│        ▼                   ▼                   ▼                    │
│  ┌─────────────────────────────────────────────────────┐           │
│  │                    cx.notify()                       │           │
│  │              (triggers re-render)                    │           │
│  └─────────────────────────────────────────────────────┘           │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────┐           │
│  │                      render()                        │           │
│  │              (UI reflects new state)                 │           │
│  └─────────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────────┘
```

### Core data flow mechanisms

```rust
// 1. Direct updates - modify entity and notify
entity.update(cx, |entity, cx| {
    entity.data = new_value;
    cx.notify();
});

// 2. Events - emit typed events for subscribers
cx.emit(MyEvent::DataChanged { id: 1 });

// 3. Observations - react to any state change
cx.observe(&entity, |this, entity, cx| {
    // Called when entity notifies
});

// 4. Subscriptions - react to specific events
cx.subscribe(&entity, |this, emitter, event, cx| {
    // Called when entity emits event
});
```

## State Management

### Entity-owned state

```rust
// State is owned by Entities
struct TodoList {
    todos: Vec<Todo>,
    filter: TodoFilter,
    sort_order: SortOrder,
}

impl TodoList {
    pub fn new(_cx: &mut Context<Self>) -> Self {
        Self {
            todos: Vec::new(),
            filter: TodoFilter::All,
            sort_order: SortOrder::DateCreated,
        }
    }

    // State mutations go through methods
    pub fn add_todo(&mut self, label: String, cx: &mut Context<Self>) {
        self.todos.push(Todo {
            id: Uuid::new_v4(),
            label,
            completed: false,
            created_at: Utc::now(),
        });
        cx.emit(TodoListEvent::TodoAdded);
        cx.notify(); // Trigger re-render
    }

    pub fn toggle_todo(&mut self, id: Uuid, cx: &mut Context<Self>) {
        if let Some(todo) = self.todos.iter_mut().find(|t| t.id == id) {
            todo.completed = !todo.completed;
            cx.emit(TodoListEvent::TodoToggled { id });
            cx.notify();
        }
    }

    // Computed state (derived from owned state)
    pub fn visible_todos(&self) -> impl Iterator<Item = &Todo> {
        self.todos.iter().filter(|t| match self.filter {
            TodoFilter::All => true,
            TodoFilter::Active => !t.completed,
            TodoFilter::Completed => t.completed,
        })
    }

    pub fn completed_count(&self) -> usize {
        self.todos.iter().filter(|t| t.completed).count()
    }
}
```

### Reading state

```rust
// Synchronous read (same context)
let count = todo_list.read(cx).todos.len();
let completed = todo_list.read(cx).completed_count();

// Read with nested entity access
let stats = workspace.read(cx).todo_list.read(cx).completed_count();

// Async context read
let count = todo_list.read_with(&cx, |list, _| list.todos.len())?;

// Read with complex logic
let summary = todo_list.read_with(&cx, |list, cx| {
    TodoSummary {
        total: list.todos.len(),
        completed: list.completed_count(),
        categories: list.get_categories(),
    }
})?;
```

### Updating state

```rust
// Simple update
todo_list.update(cx, |list, cx| {
    list.filter = TodoFilter::Active;
    cx.notify();
});

// Update with return value
let todo_id = todo_list.update(cx, |list, cx| {
    let id = Uuid::new_v4();
    list.todos.push(Todo { id, ..Default::default() });
    cx.notify();
    id
});

// Update from async context
entity.update(&mut cx, |entity, cx| {
    entity.data = async_result;
    cx.notify();
})?;

// Conditional update
todo_list.update(cx, |list, cx| {
    if list.can_add_todo() {
        list.add_todo(label, cx);
    }
});
```

### Derived/computed state

```rust
impl TodoList {
    // Pure computed values - no caching needed
    pub fn active_count(&self) -> usize {
        self.todos.iter().filter(|t| !t.completed).count()
    }

    // Filtered views of data
    pub fn todos_by_category(&self, category: &str) -> Vec<&Todo> {
        self.todos.iter()
            .filter(|t| t.category.as_deref() == Some(category))
            .collect()
    }

    // Aggregated data
    pub fn statistics(&self) -> TodoStatistics {
        TodoStatistics {
            total: self.todos.len(),
            completed: self.completed_count(),
            active: self.active_count(),
            completion_rate: if self.todos.is_empty() {
                0.0
            } else {
                self.completed_count() as f64 / self.todos.len() as f64
            },
        }
    }
}
```

## Entity Communication

### Entity references

```rust
struct Workspace {
    todo_list: Entity<TodoList>,
    settings: Entity<Settings>,
    sync_service: Entity<SyncService>,
}

impl Workspace {
    pub fn new(cx: &mut Context<Self>) -> Self {
        // Create child entities
        let settings = cx.new(|cx| Settings::new(cx));
        let todo_list = cx.new(|cx| TodoList::new(cx));
        let sync_service = cx.new(|cx| SyncService::new(cx));

        Self {
            todo_list,
            settings,
            sync_service,
        }
    }

    // Access child entity
    pub fn todo_count(&self, cx: &App) -> usize {
        self.todo_list.read(cx).todos.len()
    }

    // Modify child entity
    pub fn add_todo(&self, label: String, cx: &mut App) {
        self.todo_list.update(cx, |list, cx| {
            list.add_todo(label, cx);
        });
    }
}
```

### WeakEntity for optional references

```rust
struct TodoItemView {
    todo: Todo,
    list: WeakEntity<TodoList>,  // May be released
}

impl TodoItemView {
    pub fn toggle(&mut self, cx: &mut Context<Self>) {
        // WeakEntity may fail if parent is released
        if let Some(list) = self.list.upgrade() {
            list.update(cx, |list, cx| {
                list.toggle_todo(self.todo.id, cx);
            });
        }
    }
}

// Creating weak reference
let weak_list = todo_list.downgrade();
```

### Passing entities between components

```rust
// Entity as parameter
impl SyncService {
    pub fn sync_list(&self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        let list_clone = list.clone();
        cx.spawn(|this, mut cx| async move {
            let todos = list_clone.read_with(&cx, |list, _| list.todos.clone())?;

            // Sync to server...
            let synced = sync_to_server(todos).await?;

            list_clone.update(&mut cx, |list, cx| {
                list.mark_synced();
                cx.notify();
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }
}
```

## Event System

### Defining events

```rust
// Simple event enum
pub enum TodoListEvent {
    TodoAdded,
    TodoRemoved { id: Uuid },
    TodoToggled { id: Uuid },
    TodoUpdated { id: Uuid },
    ListCleared,
    FilterChanged,
}

// Implement EventEmitter
impl EventEmitter<TodoListEvent> for TodoList {}

// Multiple event types for one entity
pub struct SyncCompleted {
    pub synced_count: usize,
    pub timestamp: DateTime<Utc>,
}
impl EventEmitter<SyncCompleted> for TodoList {}

pub struct SyncError {
    pub error: String,
}
impl EventEmitter<SyncError> for TodoList {}
```

### Emitting events

```rust
impl TodoList {
    pub fn add_todo(&mut self, label: String, cx: &mut Context<Self>) {
        let id = Uuid::new_v4();
        self.todos.push(Todo { id, label, completed: false });

        // Emit event for subscribers
        cx.emit(TodoListEvent::TodoAdded);

        // Notify observers (triggers re-render)
        cx.notify();
    }

    pub fn remove_todo(&mut self, id: Uuid, cx: &mut Context<Self>) {
        if let Some(index) = self.todos.iter().position(|t| t.id == id) {
            self.todos.remove(index);
            cx.emit(TodoListEvent::TodoRemoved { id });
            cx.notify();
        }
    }

    pub fn sync_completed(&mut self, count: usize, cx: &mut Context<Self>) {
        self.last_sync = Some(Utc::now());
        cx.emit(SyncCompleted {
            synced_count: count,
            timestamp: Utc::now(),
        });
        cx.notify();
    }
}
```

### Event payloads

```rust
// Rich event payload
pub struct TodoBatchUpdate {
    pub added: Vec<Todo>,
    pub removed: Vec<Uuid>,
    pub updated: Vec<(Uuid, TodoUpdate)>,
    pub source: UpdateSource,
}

pub enum UpdateSource {
    Local,
    Sync,
    Import,
    Undo,
}

impl EventEmitter<TodoBatchUpdate> for TodoList {}

// Emitting complex events
impl TodoList {
    pub fn apply_batch(&mut self, batch: TodoBatch, cx: &mut Context<Self>) {
        let mut added = Vec::new();
        let mut removed = Vec::new();

        for op in batch.operations {
            match op {
                BatchOp::Add(todo) => {
                    added.push(todo.clone());
                    self.todos.push(todo);
                }
                BatchOp::Remove(id) => {
                    if let Some(idx) = self.todos.iter().position(|t| t.id == id) {
                        self.todos.remove(idx);
                        removed.push(id);
                    }
                }
            }
        }

        cx.emit(TodoBatchUpdate {
            added,
            removed,
            updated: Vec::new(),
            source: UpdateSource::Local,
        });
        cx.notify();
    }
}
```

## Subscriptions & Observations

### Basic subscription

```rust
impl TodoListView {
    pub fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        // Subscribe to list events
        let subscription = cx.subscribe(&list, Self::handle_list_event);

        Self {
            list,
            _subscription: subscription,  // Keep alive
        }
    }

    fn handle_list_event(
        &mut self,
        _list: Entity<TodoList>,
        event: &TodoListEvent,
        cx: &mut Context<Self>,
    ) {
        match event {
            TodoListEvent::TodoAdded => {
                self.scroll_to_bottom = true;
            }
            TodoListEvent::TodoRemoved { id } => {
                if self.selected_id == Some(*id) {
                    self.selected_id = None;
                }
            }
            _ => {}
        }
        cx.notify();
    }
}
```

### Basic observation

```rust
impl TodoCounter {
    pub fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        // Observe any change to list
        let observation = cx.observe(&list, |this, list, cx| {
            // Called whenever list.notify() is called
            this.update_counts(list, cx);
        });

        Self {
            list,
            completed: 0,
            total: 0,
            _observation: observation,
        }
    }

    fn update_counts(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        let list = list.read(cx);
        self.total = list.todos.len();
        self.completed = list.completed_count();
        cx.notify();
    }
}
```

### Multiple subscriptions

```rust
struct Dashboard {
    lists: Vec<Entity<TodoList>>,
    sync_service: Entity<SyncService>,
    _subscriptions: Vec<Subscription>,
}

impl Dashboard {
    pub fn new(
        lists: Vec<Entity<TodoList>>,
        sync_service: Entity<SyncService>,
        cx: &mut Context<Self>,
    ) -> Self {
        let mut subscriptions = Vec::new();

        // Subscribe to all lists
        for list in &lists {
            subscriptions.push(
                cx.subscribe(list, Self::handle_list_event)
            );
        }

        // Subscribe to sync service
        subscriptions.push(
            cx.subscribe(&sync_service, Self::handle_sync_event)
        );

        Self {
            lists,
            sync_service,
            _subscriptions: subscriptions,
        }
    }

    fn handle_list_event(
        &mut self,
        _list: Entity<TodoList>,
        event: &TodoListEvent,
        cx: &mut Context<Self>,
    ) {
        // Handle list events...
        cx.notify();
    }

    fn handle_sync_event(
        &mut self,
        _service: Entity<SyncService>,
        event: &SyncEvent,
        cx: &mut Context<Self>,
    ) {
        // Handle sync events...
        cx.notify();
    }
}
```

### observe_release for cleanup

```rust
impl TodoListManager {
    pub fn track_list(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        let list_id = list.entity_id();
        self.lists.insert(list_id, list.clone());

        // Clean up when list is released
        cx.observe_release(&list, move |this, _, _cx| {
            this.lists.remove(&list_id);
            this.on_list_removed(list_id);
        }).detach();
    }
}
```

### Subscription vs Observation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Subscription vs Observation                       │
├─────────────────────┬───────────────────────────────────────────────┤
│ cx.subscribe()      │ cx.observe()                                  │
├─────────────────────┼───────────────────────────────────────────────┤
│ Reacts to events    │ Reacts to any notify()                        │
│ Typed events        │ No event payload                              │
│ Entity must impl    │ Works with any Entity                         │
│   EventEmitter      │                                               │
│ Good for specific   │ Good for general state                        │
│   state changes     │   synchronization                             │
│ Multiple event      │ Single callback for                           │
│   types possible    │   all changes                                 │
└─────────────────────┴───────────────────────────────────────────────┘
```

## Global State

### Setting up global state

```rust
// Define global state type
pub struct AppSettings {
    pub theme: Theme,
    pub font_size: f32,
    pub auto_save: bool,
}

impl Global for AppSettings {}

// Initialize in app startup
fn main() {
    Application::new().run(|cx: &mut App| {
        // Set global
        cx.set_global(AppSettings {
            theme: Theme::Dark,
            font_size: 14.0,
            auto_save: true,
        });

        // ...
    });
}
```

### Reading global state

```rust
// Read global
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let settings = cx.global::<AppSettings>();

        div()
            .text_size(px(settings.font_size))
            .bg(settings.theme.background())
            .child("Content")
    }
}

// Check if global exists
if cx.has_global::<AppSettings>() {
    let settings = cx.global::<AppSettings>();
}

// Try to get global
if let Some(settings) = cx.try_global::<AppSettings>() {
    // Use settings
}
```

### Updating global state

```rust
// Update global
cx.global_mut::<AppSettings>().theme = Theme::Light;

// Update with default
cx.default_global::<AppSettings>().font_size = 16.0;

// Remove global
let old_settings = cx.remove_global::<AppSettings>();
```

### Observing global changes

```rust
impl ThemeAwareView {
    pub fn new(cx: &mut Context<Self>) -> Self {
        // Watch for global changes
        cx.observe_global::<AppSettings>(|this, cx| {
            this.on_settings_changed(cx);
        }).detach();

        Self { /* ... */ }
    }

    fn on_settings_changed(&mut self, cx: &mut Context<Self>) {
        // React to settings change
        cx.notify();
    }
}
```

### Global services pattern

```rust
// Global service manager
pub struct Services {
    pub http_client: Arc<dyn HttpClient>,
    pub database: Arc<Database>,
    pub analytics: Arc<AnalyticsService>,
}

impl Global for Services {}

// Initialize services
fn init_services(cx: &mut App) {
    let http_client = Arc::new(ReqwestClient::new());
    let database = Arc::new(Database::open("app.db").unwrap());
    let analytics = Arc::new(AnalyticsService::new(http_client.clone()));

    cx.set_global(Services {
        http_client,
        database,
        analytics,
    });
}

// Use services
impl MyView {
    fn load_data(&self, cx: &mut Context<Self>) {
        let db = cx.global::<Services>().database.clone();

        cx.spawn(|this, mut cx| async move {
            let data = db.query_todos().await?;
            this.update(&mut cx, |this, cx| {
                this.todos = data;
                cx.notify();
            })
        }).detach_and_log_err(cx);
    }
}
```

## Async Data Flow

### Spawning async tasks

```rust
impl TodoList {
    pub fn load_from_server(&mut self, cx: &mut Context<Self>) {
        self.loading = true;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            // Async work
            let todos = fetch_todos_from_server().await?;

            // Update entity with results
            this.update(&mut cx, |this, cx| {
                this.todos = todos;
                this.loading = false;
                cx.emit(TodoListEvent::Loaded);
                cx.notify();
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Async data flow with channels

```rust
impl LiveDataView {
    pub fn start_streaming(&mut self, cx: &mut Context<Self>) {
        let (tx, mut rx) = futures::channel::mpsc::unbounded();

        // Spawn producer
        cx.spawn(|_, _| async move {
            let mut stream = connect_to_stream().await?;
            while let Some(item) = stream.next().await {
                if tx.unbounded_send(item).is_err() {
                    break;
                }
            }
            anyhow::Ok(())
        }).detach();

        // Spawn consumer
        cx.spawn(|this, mut cx| async move {
            while let Some(item) = rx.next().await {
                this.update(&mut cx, |this, cx| {
                    this.items.push(item);
                    cx.notify();
                })?;
            }
            anyhow::Ok(())
        }).detach();
    }
}
```

### Background tasks with progress

```rust
impl ImportView {
    pub fn import_file(&mut self, path: PathBuf, cx: &mut Context<Self>) {
        self.progress = Some(ImportProgress::default());
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            let file = tokio::fs::read(&path).await?;
            let total = file.len();
            let mut processed = 0;

            for chunk in file.chunks(1024) {
                // Process chunk...
                processed += chunk.len();

                // Update progress
                this.update(&mut cx, |this, cx| {
                    if let Some(ref mut progress) = this.progress {
                        progress.current = processed;
                        progress.total = total;
                    }
                    cx.notify();
                })?;
            }

            // Complete
            this.update(&mut cx, |this, cx| {
                this.progress = None;
                cx.emit(ImportEvent::Completed);
                cx.notify();
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Task cancellation

```rust
struct SearchView {
    query: String,
    results: Vec<SearchResult>,
    search_task: Option<Task<()>>,
}

impl SearchView {
    pub fn search(&mut self, query: String, cx: &mut Context<Self>) {
        // Cancel previous search
        self.search_task.take();

        self.query = query.clone();
        cx.notify();

        // Start new search
        self.search_task = Some(cx.spawn(|this, mut cx| async move {
            // Debounce
            cx.background_executor().timer(Duration::from_millis(300)).await;

            let results = perform_search(&query).await?;

            this.update(&mut cx, |this, cx| {
                this.results = results;
                cx.notify();
            })?;

            Ok(())
        }));
    }
}
```

## Parent-Child Communication

### Parent to child (props down)

```rust
// Parent view
impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .children(self.items.iter().map(|item| {
                // Pass data down to child component
                ItemCard::new(item.clone())
                    .selected(self.selected_id == Some(item.id))
                    .highlight_color(self.highlight_color)
            }))
    }
}

// Child component receives data via constructor/builder
#[derive(IntoElement)]
struct ItemCard {
    item: Item,
    selected: bool,
    highlight_color: Option<Hsla>,
}
```

### Child to parent (events up)

```rust
// Parent view with callback handlers
impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .children(self.items.iter().enumerate().map(|(index, item)| {
                let item_id = item.id;

                ItemCard::new(item.clone())
                    // Pass callbacks down
                    .on_select(cx.listener(move |this, _, _, cx| {
                        this.select_item(item_id, cx);
                    }))
                    .on_delete(cx.listener(move |this, _, _, cx| {
                        this.delete_item(item_id, cx);
                    }))
                    .on_edit(cx.listener(move |this, new_value: &String, _, cx| {
                        this.update_item(item_id, new_value.clone(), cx);
                    }))
            }))
    }
}

impl ParentView {
    fn select_item(&mut self, id: Uuid, cx: &mut Context<Self>) {
        self.selected_id = Some(id);
        cx.notify();
    }

    fn delete_item(&mut self, id: Uuid, cx: &mut Context<Self>) {
        self.items.retain(|item| item.id != id);
        cx.emit(ParentEvent::ItemDeleted(id));
        cx.notify();
    }
}
```

### Bidirectional with Entity children

```rust
struct ParentView {
    child_view: Entity<ChildView>,
    _subscription: Subscription,
}

impl ParentView {
    pub fn new(cx: &mut Context<Self>) -> Self {
        let child_view = cx.new(|cx| ChildView::new(cx));

        // Subscribe to child events
        let subscription = cx.subscribe(&child_view, Self::handle_child_event);

        Self {
            child_view,
            _subscription: subscription,
        }
    }

    fn handle_child_event(
        &mut self,
        _child: Entity<ChildView>,
        event: &ChildEvent,
        cx: &mut Context<Self>,
    ) {
        match event {
            ChildEvent::ValueChanged(value) => {
                self.handle_value_change(*value, cx);
            }
            ChildEvent::Submitted => {
                self.submit(cx);
            }
        }
    }

    // Send data to child
    fn update_child(&self, value: i32, cx: &mut App) {
        self.child_view.update(cx, |child, cx| {
            child.set_value(value, cx);
        });
    }
}

impl Render for ParentView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .child(self.child_view.clone())  // Render child entity
    }
}
```

### Context provider pattern

```rust
// Context/provider pattern using globals
pub struct TodoContext {
    pub list: Entity<TodoList>,
    pub filter: TodoFilter,
}

impl Global for TodoContext {}

// Provider sets up context
impl TodoProvider {
    pub fn new(cx: &mut Context<Self>) -> Self {
        let list = cx.new(|cx| TodoList::new(cx));

        cx.set_global(TodoContext {
            list: list.clone(),
            filter: TodoFilter::All,
        });

        Self { list }
    }
}

// Children access context
impl TodoConsumer {
    fn get_list(&self, cx: &App) -> Entity<TodoList> {
        cx.global::<TodoContext>().list.clone()
    }
}
```

## Cross-Window Communication

### Shared entities across windows

```rust
struct AppState {
    todo_list: Entity<TodoList>,
    windows: Vec<WindowHandle<MainWindow>>,
}

impl AppState {
    pub fn open_new_window(&mut self, cx: &mut App) {
        let list = self.todo_list.clone();

        let handle = cx.open_window(
            WindowOptions::default(),
            |_, cx| {
                cx.new(|cx| MainWindow::new(list, cx))
            },
        ).unwrap();

        self.windows.push(handle);
    }
}

// All windows share the same TodoList entity
// Changes in one window automatically reflect in others
// (via subscriptions/observations)
```

### Window messaging

```rust
// Global message bus
pub struct MessageBus {
    subscribers: Vec<WeakEntity<dyn MessageHandler>>,
}

impl Global for MessageBus {}

pub trait MessageHandler: 'static {
    fn handle_message(&mut self, message: &AppMessage, cx: &mut Context<Self>);
}

impl MessageBus {
    pub fn broadcast(&self, message: AppMessage, cx: &mut App) {
        for subscriber in &self.subscribers {
            if let Some(handler) = subscriber.upgrade() {
                handler.update(cx, |handler, cx| {
                    handler.handle_message(&message, cx);
                });
            }
        }
    }
}
```

### Window-specific state

```rust
struct WindowState {
    window_id: WindowId,
    sidebar_visible: bool,
    zoom_level: f32,
}

// Store per-window state
struct App {
    shared_data: Entity<SharedData>,
    window_states: HashMap<WindowId, WindowState>,
}

impl App {
    fn get_window_state(&self, window_id: WindowId) -> Option<&WindowState> {
        self.window_states.get(&window_id)
    }

    fn update_window_state(
        &mut self,
        window_id: WindowId,
        f: impl FnOnce(&mut WindowState),
    ) {
        if let Some(state) = self.window_states.get_mut(&window_id) {
            f(state);
        }
    }
}
```

## Data Loading Patterns

### Loading states

```rust
pub enum LoadState<T> {
    Idle,
    Loading,
    Loaded(T),
    Error(String),
}

struct DataView {
    data: LoadState<Vec<Item>>,
}

impl DataView {
    pub fn load(&mut self, cx: &mut Context<Self>) {
        self.data = LoadState::Loading;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            match fetch_data().await {
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
            Ok(())
        }).detach_and_log_err(cx);
    }
}

impl Render for DataView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        match &self.data {
            LoadState::Idle => div().child("Click to load"),
            LoadState::Loading => div().child("Loading..."),
            LoadState::Loaded(items) => {
                div().children(items.iter().map(|item| {
                    div().child(item.name.clone())
                }))
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

### Pagination pattern

```rust
struct PaginatedList {
    items: Vec<Item>,
    page: usize,
    per_page: usize,
    total: usize,
    loading: bool,
}

impl PaginatedList {
    pub fn load_page(&mut self, page: usize, cx: &mut Context<Self>) {
        self.loading = true;
        self.page = page;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            let response = fetch_page(page, this.read_with(&cx, |t, _| t.per_page)?).await?;

            this.update(&mut cx, |this, cx| {
                this.items = response.items;
                this.total = response.total;
                this.loading = false;
                cx.notify();
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }

    pub fn next_page(&mut self, cx: &mut Context<Self>) {
        if (self.page + 1) * self.per_page < self.total {
            self.load_page(self.page + 1, cx);
        }
    }

    pub fn prev_page(&mut self, cx: &mut Context<Self>) {
        if self.page > 0 {
            self.load_page(self.page - 1, cx);
        }
    }
}
```

### Infinite scroll pattern

```rust
struct InfiniteList {
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
            let response = fetch_items(cursor).await?;

            this.update(&mut cx, |this, cx| {
                this.items.extend(response.items);
                this.cursor = response.next_cursor;
                this.has_more = response.has_more;
                this.loading = false;
                cx.notify();
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }
}

impl Render for InfiniteList {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("infinite-list")
            .overflow_y_scroll()
            .on_scroll(cx.listener(|this, event: &ScrollEvent, _, cx| {
                // Load more when near bottom
                if event.scroll_position.y > event.content_size.height - px(200.) {
                    this.load_more(cx);
                }
            }))
            .children(self.items.iter().map(|item| {
                div().child(item.name.clone())
            }))
            .when(self.loading, |this| {
                this.child(div().child("Loading more..."))
            })
    }
}
```

## Caching Patterns

### Simple cache

```rust
struct CachedData<T> {
    data: Option<T>,
    fetched_at: Option<Instant>,
    ttl: Duration,
}

impl<T: Clone> CachedData<T> {
    pub fn new(ttl: Duration) -> Self {
        Self {
            data: None,
            fetched_at: None,
            ttl,
        }
    }

    pub fn is_valid(&self) -> bool {
        match self.fetched_at {
            Some(time) => time.elapsed() < self.ttl,
            None => false,
        }
    }

    pub fn get(&self) -> Option<&T> {
        if self.is_valid() {
            self.data.as_ref()
        } else {
            None
        }
    }

    pub fn set(&mut self, data: T) {
        self.data = Some(data);
        self.fetched_at = Some(Instant::now());
    }

    pub fn invalidate(&mut self) {
        self.fetched_at = None;
    }
}

// Usage
struct UserProfile {
    cache: CachedData<User>,
}

impl UserProfile {
    pub fn get_user(&mut self, cx: &mut Context<Self>) {
        if let Some(user) = self.cache.get() {
            // Use cached data
            return;
        }

        // Fetch fresh data
        cx.spawn(|this, mut cx| async move {
            let user = fetch_user().await?;
            this.update(&mut cx, |this, cx| {
                this.cache.set(user);
                cx.notify();
            })?;
            Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Entity-based cache

```rust
struct DataCache {
    items: HashMap<Uuid, CacheEntry>,
}

struct CacheEntry {
    data: Item,
    fetched_at: Instant,
}

impl DataCache {
    pub fn get(&self, id: Uuid) -> Option<&Item> {
        self.items.get(&id)
            .filter(|entry| entry.fetched_at.elapsed() < Duration::from_secs(300))
            .map(|entry| &entry.data)
    }

    pub fn set(&mut self, id: Uuid, data: Item) {
        self.items.insert(id, CacheEntry {
            data,
            fetched_at: Instant::now(),
        });
    }

    pub fn invalidate(&mut self, id: Uuid) {
        self.items.remove(&id);
    }

    pub fn invalidate_all(&mut self) {
        self.items.clear();
    }
}

impl EventEmitter<CacheEvent> for DataCache {}

pub enum CacheEvent {
    Updated(Uuid),
    Invalidated(Uuid),
    Cleared,
}
```

### Request deduplication

```rust
struct DataFetcher {
    pending_requests: HashMap<Uuid, Task<Result<Item>>>,
    cache: HashMap<Uuid, Item>,
}

impl DataFetcher {
    pub fn fetch(&mut self, id: Uuid, cx: &mut Context<Self>) -> Task<Result<Item>> {
        // Return cached data immediately
        if let Some(item) = self.cache.get(&id) {
            return Task::ready(Ok(item.clone()));
        }

        // Return existing pending request
        if let Some(task) = self.pending_requests.get(&id) {
            return task.clone();
        }

        // Create new request
        let task = cx.spawn(|this, mut cx| async move {
            let item = fetch_item(id).await?;

            this.update(&mut cx, |this, cx| {
                this.cache.insert(id, item.clone());
                this.pending_requests.remove(&id);
                cx.notify();
            })?;

            Ok(item)
        });

        self.pending_requests.insert(id, task.clone());
        task
    }
}
```

### Optimistic updates

```rust
impl TodoList {
    pub fn toggle_optimistic(&mut self, id: Uuid, cx: &mut Context<Self>) {
        // Optimistically update UI
        if let Some(todo) = self.todos.iter_mut().find(|t| t.id == id) {
            todo.completed = !todo.completed;
            let new_state = todo.completed;
            cx.notify();

            // Sync to server
            cx.spawn(|this, mut cx| async move {
                match update_todo_on_server(id, new_state).await {
                    Ok(_) => {
                        // Success - keep optimistic update
                    }
                    Err(_) => {
                        // Revert on failure
                        this.update(&mut cx, |this, cx| {
                            if let Some(todo) = this.todos.iter_mut().find(|t| t.id == id) {
                                todo.completed = !todo.completed; // Revert
                            }
                            cx.emit(TodoListEvent::SyncFailed(id));
                            cx.notify();
                        })?;
                    }
                }
                Ok(())
            }).detach_and_log_err(cx);
        }
    }
}
```

### Complete data flow example

```rust
use gpui::{prelude::*, Entity, EventEmitter, Subscription};
use std::collections::HashMap;
use uuid::Uuid;

// ═══════════════════════════════════════════════════════════════════
// Domain Models
// ═══════════════════════════════════════════════════════════════════

#[derive(Clone, Debug)]
pub struct Todo {
    pub id: Uuid,
    pub label: String,
    pub completed: bool,
}

// ═══════════════════════════════════════════════════════════════════
// Events
// ═══════════════════════════════════════════════════════════════════

pub enum TodoEvent {
    Added(Uuid),
    Removed(Uuid),
    Updated(Uuid),
    Loaded,
    SyncStarted,
    SyncCompleted,
    SyncFailed(String),
}

// ═══════════════════════════════════════════════════════════════════
// Todo List (Model)
// ═══════════════════════════════════════════════════════════════════

pub struct TodoList {
    todos: Vec<Todo>,
    loading: bool,
    syncing: bool,
}

impl EventEmitter<TodoEvent> for TodoList {}

impl TodoList {
    pub fn new(_cx: &mut Context<Self>) -> Self {
        Self {
            todos: Vec::new(),
            loading: false,
            syncing: false,
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
            id: Uuid::new_v4(),
            label,
            completed: false,
        };
        let id = todo.id;
        self.todos.push(todo);
        cx.emit(TodoEvent::Added(id));
        cx.notify();
    }

    pub fn toggle(&mut self, id: Uuid, cx: &mut Context<Self>) {
        if let Some(todo) = self.todos.iter_mut().find(|t| t.id == id) {
            todo.completed = !todo.completed;
            cx.emit(TodoEvent::Updated(id));
            cx.notify();
        }
    }

    pub fn remove(&mut self, id: Uuid, cx: &mut Context<Self>) {
        self.todos.retain(|t| t.id != id);
        cx.emit(TodoEvent::Removed(id));
        cx.notify();
    }

    pub fn load(&mut self, cx: &mut Context<Self>) {
        self.loading = true;
        cx.notify();

        cx.spawn(|this, mut cx| async move {
            // Simulate API call
            let todos = vec![
                Todo { id: Uuid::new_v4(), label: "Task 1".into(), completed: false },
                Todo { id: Uuid::new_v4(), label: "Task 2".into(), completed: true },
            ];

            this.update(&mut cx, |this, cx| {
                this.todos = todos;
                this.loading = false;
                cx.emit(TodoEvent::Loaded);
                cx.notify();
            })
        }).detach_and_log_err(cx);
    }
}

// ═══════════════════════════════════════════════════════════════════
// Todo List View (View)
// ═══════════════════════════════════════════════════════════════════

pub struct TodoListView {
    list: Entity<TodoList>,
    selected_id: Option<Uuid>,
    _subscription: Subscription,
}

impl TodoListView {
    pub fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        let subscription = cx.subscribe(&list, Self::on_list_event);

        Self {
            list,
            selected_id: None,
            _subscription: subscription,
        }
    }

    fn on_list_event(
        &mut self,
        _list: Entity<TodoList>,
        event: &TodoEvent,
        cx: &mut Context<Self>,
    ) {
        match event {
            TodoEvent::Removed(id) if self.selected_id == Some(*id) => {
                self.selected_id = None;
            }
            _ => {}
        }
        cx.notify();
    }

    fn select(&mut self, id: Uuid, cx: &mut Context<Self>) {
        self.selected_id = Some(id);
        cx.notify();
    }
}

impl Render for TodoListView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let list = self.list.read(cx);

        div()
            .flex()
            .flex_col()
            .gap_2()
            .when(list.is_loading(), |this| {
                this.child(div().child("Loading..."))
            })
            .when(!list.is_loading(), |this| {
                this.children(list.todos().iter().map(|todo| {
                    let id = todo.id;
                    let selected = self.selected_id == Some(id);

                    div()
                        .id(("todo", id))
                        .flex()
                        .items_center()
                        .gap_2()
                        .p_2()
                        .bg(if selected { rgb(0x3b82f6) } else { rgb(0x1e1e1e) })
                        .rounded_md()
                        .cursor_pointer()
                        .on_click(cx.listener(move |this, _, _, cx| {
                            this.select(id, cx);
                        }))
                        .child(
                            div()
                                .id(("toggle", id))
                                .size_5()
                                .rounded_sm()
                                .border_1()
                                .border_color(rgb(0x555555))
                                .when(todo.completed, |this| this.bg(rgb(0x22c55e)))
                                .on_click(cx.listener(move |this, _, _, cx| {
                                    this.list.update(cx, |list, cx| {
                                        list.toggle(id, cx);
                                    });
                                }))
                        )
                        .child(
                            div()
                                .flex_1()
                                .when(todo.completed, |this| {
                                    this.line_through().text_color(rgb(0x666666))
                                })
                                .child(todo.label.clone())
                        )
                        .child(
                            div()
                                .id(("delete", id))
                                .px_2()
                                .text_color(rgb(0xef4444))
                                .cursor_pointer()
                                .hover(|s| s.text_color(rgb(0xff6666)))
                                .on_click(cx.listener(move |this, _, _, cx| {
                                    this.list.update(cx, |list, cx| {
                                        list.remove(id, cx);
                                    });
                                }))
                                .child("×")
                        )
                }))
            })
    }
}
```
