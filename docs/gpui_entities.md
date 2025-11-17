# GPUI Entity Patterns

1. [High Level Patterns](#high-level-patterns)
2. [Entity Creation Patterns](#entity-creation-patterns)
3. [Entity Reading Patterns](#entity-reading-patterns)
4. [Entity Updating Patterns](#entity-updating-patterns)
5. [WeakEntity Patterns](#weakentity-patterns)
6. [Entity Subscriptions](#entity-subscriptions)
7. [Entity Observations](#entity-observations)
8. [Entity Lifecycle](#entity-lifecycle)
9. [EventEmitter Patterns](#eventemitter-patterns)
10. [Entity Passing Patterns](#entity-passing-patterns)
11. [Complex Entity Patterns](#complex-entity-patterns)

## High Level Patterns

### Entity creation context

```rust
// In Context<T>:
let entity = cx.new(|cx| SomeType::new(cx));

// In Window or App context:
cx.update(|window, cx| {
    cx.new(|cx| SomeType::new(cx))
})
```

### Safe entity access from async

```rust
// Strong entity - will keep entity alive
entity.update(cx, |entity, cx| { /* ... */ })?

// Weak entity - may fail if released
weak_entity.update(cx, |entity, cx| { /* ... */ })?
```

### Entity lifecycle management

```rust
// Store subscriptions to keep them alive
_subscriptions: Vec<Subscription>

// Use observe_release for cleanup
cx.observe_release(&entity, |this, entity, _| {
    // Cleanup code
})
```

### Cross-context entity access

- `.read(cx)` - synchronous read in same context
- `.read_with(cx, |e, cx| ...)` - read from async context
- `.update(cx, |e, cx| ...)` - update in same context
- `.update_in(cx, |e, window, cx| ...)` - update with window context

### Entity communication

- Use `EventEmitter` for events
- Use channels for streaming data
- Use subscriptions for reactive updates
- Use observations for state synchronization

## Entity Creation Patterns

### Basic entity creation with cx.new

```rust
struct Todo { label: SharedString, complete: bool }
struct TodoList { items: Vec<Todo> }

impl TodoList {
    // OR if you need a window: pub fn new(window: &mut Window, cx: &mut Context<Self>) -> Self {
    pub fn new(cx: &mut Context<Self>) -> Self {
        Self { items: vec![] }
    }
}

let todo_list = cx.new(|cx| TodoList::new(cx));
```

## Entity Reading Patterns

### Simple read pattern

```rust
// where `list` is a method that lists all todos
let todos = todo_list.read(cx).list();
```

### Read with nested entity access

```rust
// where TodoList has a field `subscribed: Entity<SubscribedUsers>`
let subscribed_users = todo_list.read(cx).subscribed.read(cx).list(cx);

// or - if using the read multiple times it might be useful to do this:

let todo_list = todo_list.read(cx);
let subscribed = todo_list.subscribed.read(cx);
let subscribed_users = subscribed.list(cx);
```

### read_with for async contexts

```rust
// Using read_with from async context
let todo_count = todo_list.read_with(&cx, |list, _| list.todos.len())?;
let list_name = todo_list.read_with(&cx, |list, _| list.name.clone())?;
```

### read_with with complex logic

```rust
// Complex read_with operation with nested reads
let summary = todo_workspace.read_with(&cx, |workspace, cx| {
    workspace.active_list.as_ref().and_then(|list| {
        list.read_with(cx, |list, cx| {
            let stats = list.stats_tracker.read(cx);
            Some(TodoSummary {
                total: stats.total_count,
                completed: stats.completed_count,
                categories: list.get_categories(),
            })
        })
    })
})?;
```

### Read for extracting multiple values

```rust
// Reading multiple fields from entity
let list_data = todo_list.read_with(&cx, |list, cx| {
    let filter = list.filter_manager.read(cx);
    let sort_config = list.sort_engine.read(cx);

    anyhow::Ok(TodoListData {
        todos: list.get_filtered_todos(filter),
        sort_order: sort_config.current_order.clone(),
        last_sync: list.sync_metadata.last_sync_time,
        has_unsaved: list.has_unsaved_changes(),
    })
})?;
```

## Entity Updating Patterns

### Simple update pattern

```rust
// Basic update to modify todo items
todo_list.update(cx, |list, cx| {
    list.todos.push(Todo::new("New task".to_string()));
    list.mark_unsaved();
    cx.notify();
});

// Updating display label
todo_label.update(cx, |label, cx| {
    let display_text = if text.len() > 50 {
        format!("{}...", &text[..47])
    } else {
        text.to_string()
    };
    label.set_text(display_text, cx);
});
```

### Update with return value

```rust
// Update that returns a value
let todo_id = todo_list.update(cx, |list, cx| {
    let todo = Todo::new(label);
    let id = todo.id;
    list.todos.push(todo);
    cx.notify();
    id
});

// Async update with return value
let export_path = todo_list.update(cx, |list, cx| {
    list.export_service.update(cx, |service, cx| {
        service.prepare_export(&list.todos, ExportFormat::Json, cx)
    })
})?;
```

### Nested entity updates

```rust
// Updating nested entities
todo_workspace.update(cx, |workspace, cx| {
    workspace.active_list.update(cx, |list, cx| {
        list.filter_manager.update(cx, |filter, cx| {
            filter.add_category("Work");
            filter.set_active(true);
            cx.notify();
        });
        cx.notify();
    });
});
```

### update_in for window context

```rust
// Using update_in with window context
todo_view.update_in(cx, |view, window, cx| {
    view.handle_drop_event(dropped_items, window, cx);
    window.activate(cx);
})?;
```

### Conditional update pattern

```rust
// Conditional update based on state
todo_list.update_in(cx, |list, window, cx| {
    if list.input_field.read(cx).text().trim().is_empty() {
        window.show_notification("Todo label cannot be empty", cx);
        return;
    }
    list.create_todo_from_input(window, cx);
})?;
```

### Update from async context

```rust
// Async entity update pattern
cx.spawn(async move |this, cx| {
    let imported_todos = load_from_file(path).await?;

    this.update(&cx, |list, cx| {
        // Check if we still want to import
        if list.import_cancelled {
            return Ok(());
        }

        list.merge_todos(imported_todos);
        list.last_import = Some(SystemTime::now());
        cx.notify();
        Ok(())
    })?
})
```

## WeakEntity Patterns

### Basic WeakEntity creation

```rust
// Creating WeakEntity with downgrade
struct TodoSession {
    active_list: Entity<TodoList>,
    backup_list: WeakEntity<TodoList>,
    undo_manager: WeakEntity<UndoManager>,
}

impl TodoSession {
    pub fn new(list: Entity<TodoList>, undo_manager: Entity<UndoManager>, cx: &mut Context<Self>) -> Self {
        Self {
            active_list: list.clone(),
            backup_list: list.downgrade(),
            undo_manager: undo_manager.downgrade(),
        }
    }
}
```

### WeakEntity in async functions

```rust
// WeakEntity as function parameter for background sync
async fn sync_todos_to_server(
    list: WeakEntity<TodoList>,
    mut changes: mpsc::UnboundedReceiver<TodoChange>,
    cx: &mut AsyncApp,
) -> Result<()> {
    while let Some(change) = changes.next().await {
        list.update(&cx, |list, cx| {
            list.mark_syncing(&change);
            cx.notify();
        })?;

        // Sync to server...
    }
    Ok(())
}
```

### WeakEntity update pattern

```rust
// Updating through WeakEntity in async context
impl TodoList {
    async fn auto_save(
        weak_self: &WeakEntity<Self>,
        storage: Arc<dyn TodoStorage>,
        cx: &mut AsyncApp,
    ) -> Result<()> {
        // Try to update - fails gracefully if entity was dropped
        weak_self.update(&cx, |list, cx| {
            if list.has_unsaved_changes() {
                list.mark_saving();
                cx.notify();
            }
        })?;

        // Perform save...
        storage.save_todos().await?;

        weak_self.update(&cx, |list, cx| {
            list.mark_saved();
            cx.notify();
        })?;

        Ok(())
    }
}
```

### WeakEntity in struct fields

```rust
// Storing WeakEntity for lifecycle safety
pub struct TodoExporter {
    source_list: WeakEntity<TodoList>,
    export_format: ExportFormat,
    output_buffer: Entity<Buffer>,
    template_engine: Arc<TemplateEngine>,
}

impl TodoExporter {
    pub fn export(&mut self, cx: &mut App) -> Result<String> {
        // Safe access - returns error if source list was dropped
        self.source_list.read_with(cx, |list, _| {
            list.todos.clone()
        })
        .map(|todos| self.template_engine.render(todos))
    }
}
```

### WeakEntity for optional dependencies

```rust
// Using WeakEntity for optional/circular references
struct TodoListPosition {
    list: WeakEntity<TodoList>,
    item_index: usize,
    cursor_offset: usize,
}

impl TodoListPosition {
    pub fn from_list(list: &Entity<TodoList>, index: usize, cx: &App) -> Self {
        Self {
            list: list.downgrade(),
            item_index: index,
            cursor_offset: list.read(cx).todos.get(index).map(|t| t.label.len()).unwrap_or(0),
        }
    }
}
```

## Entity Subscriptions

### Basic subscription pattern

```rust
// Simple subscription to todo list events
impl TodoListView {
    fn subscribe_to_list(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        cx.subscribe(&list, move |this, list, event: &TodoListEvent, cx| {
            match event {
                TodoListEvent::ItemAdded(item) => {
                    this.flash_item(item.id, cx);
                    cx.notify();
                }
                TodoListEvent::ItemCompleted(id) => {
                    this.animate_completion(*id, cx);
                }
                _ => {}
            }
        })
        .detach();
    }
}
```

### Subscription with entity update

```rust
// Subscription that updates another entity
struct TodoCounter {
    list: Entity<TodoList>,
    _subscription: Subscription,
    completed_count: usize,
    total_count: usize,
}

impl TodoCounter {
    fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        Self {
            list: list.clone(),
            _subscription: cx.subscribe(&list, Self::handle_list_event),
            completed_count: 0,
            total_count: 0,
        }
    }

    fn handle_list_event(&mut self, _list: Entity<TodoList>, event: &TodoListEvent, cx: &mut Context<Self>) {
        match event {
            TodoListEvent::ItemAdded(_) => self.total_count += 1,
            TodoListEvent::ItemCompleted(_) => self.completed_count += 1,
            TodoListEvent::ItemRemoved(_) => self.total_count -= 1,
            _ => {}
        }
        cx.notify();
    }
}
```

### Multiple subscriptions pattern

```rust
// Managing multiple subscriptions for a TodoList dashboard
struct TodoDashboard {
    lists: Vec<Entity<TodoList>>,
    sync_service: Entity<SyncService>,
    undo_manager: Entity<UndoManager>,
    _subscriptions: Vec<Subscription>,
}

impl TodoDashboard {
    fn new(lists: Vec<Entity<TodoList>>, sync_service: Entity<SyncService>, cx: &mut Context<Self>) -> Self {
        let undo_manager = cx.new(|_| UndoManager::new());

        let subscriptions = lists.iter().flat_map(|list| {
            vec![
                cx.observe_release(list, |this, list, _cx| {
                    this.handle_list_removed(list);
                }),
                cx.subscribe(list, Self::handle_list_change),
                cx.observe(list, move |this, list, cx| {
                    this.auto_save(list, cx)
                }),
            ]
        }).chain(vec![
            cx.subscribe(&sync_service, Self::handle_sync_event),
            cx.observe(&undo_manager, |_, _, cx| cx.notify()),
        ]).collect();

        Self {
            lists,
            sync_service,
            undo_manager,
            _subscriptions: subscriptions,
        }
    }
}
```

### Subscription with closure capture

```rust
// Subscription capturing local state
impl TodoListSettings {
    fn watch_theme_changes(&mut self, cx: &mut Context<Self>) {
        let current_theme = self.theme.clone();
        let list_entity = self.list.clone();

        cx.subscribe(
            &ThemeRegistry::global(cx),
            move |this, _, event: &ThemeEvent, cx| {
                match event {
                    ThemeEvent::ThemeChanged(new_theme) => {
                        if new_theme.name != current_theme {
                            list_entity.update(cx, |list, cx| {
                                list.apply_theme(new_theme);
                                cx.notify();
                            });
                        }
                    }
                    ThemeEvent::ColorsUpdated => {
                        this.refresh_colors(cx);
                    }
                    _ => {}
                }
            },
        )
        .detach();
    }
}
```

## Entity Observations

### Basic observe pattern

```rust
// Simple observation to track changes
impl TodoListView {
    fn watch_list(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        self._list_observer = cx.observe(&list, |this, _, cx| {
            this.refresh_display();
            cx.notify();
        });
    }
}
```

### Observe with notification

```rust
// Observe and propagate updates
impl TodoSidebar {
    fn observe_active_list(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        let subscription = cx.observe(&list, |this, list, cx| {
            this.update_stats(list, cx);
            this.highlight_active_items(list, cx);
            cx.notify();
        });

        self._active_list_observer = Some(subscription);
    }
}
```

### observe_release pattern

```rust
// Clean up when entity is released
impl TodoWorkspace {
    fn track_list(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        let list_id = list.read(cx).id.clone();

        cx.observe_release(&list, move |this, _list, _cx| {
            this.open_lists.remove(&list_id);
            this.cleanup_list_resources(&list_id);
        })
        .detach();
    }
}
```

### Complex observe_release with buffer management

```rust
// Complex cleanup when todo list is closed
impl TodoSyncManager {
    fn register_list(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) {
        let weak_list = list.downgrade();
        let list_id = list.read(cx).id.clone();

        self.active_lists.insert(list_id.clone(), list.clone());

        cx.observe_release(&list, move |this, _list, _cx| {
            this.active_lists.remove(&list_id);
            this.pending_syncs.remove(&weak_list);
            this.stop_sync_timer(&list_id);
            this.cleanup_temp_files(&list_id);
        })
        .detach();
    }
}
```

### Observe global state

```rust
// Observing global settings changes
impl TodoListView {
    fn watch_settings(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        cx.observe_global_in::<TodoSettings>(window, Self::on_settings_changed)
            .detach();
    }

    fn on_settings_changed(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        self.update_sort_order(cx);
        self.apply_filter_settings(cx);
        cx.notify();
    }
}
```

## Entity Lifecycle

### Entity with subscription cleanup

```rust
// Managing subscriptions lifecycle
struct TodoPanel {
    todo_list: Entity<TodoList>,
    filter: TodoFilter,
    _subscriptions: Vec<Subscription>,
}

impl TodoPanel {
    fn new(todo_list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        let _subscriptions = vec![
            cx.subscribe(&todo_list, Self::handle_list_event),
            cx.observe(&todo_list, |this, _, cx| {
                this.update_stats(cx);
                cx.notify()
            }),
        ];

        Self {
            todo_list,
            filter: TodoFilter::All,
            _subscriptions,
        }
    }

    fn handle_list_event(&mut self, _list: Entity<TodoList>, event: &TodoListEvent, cx: &mut Context<Self>) {
        match event {
            TodoListEvent::ItemAdded(_) | TodoListEvent::ItemRemoved(_) => {
                self.refresh_view(cx);
            }
            _ => {}
        }
    }
}
```

### Task::ready for immediate entity operations

```rust
// Immediate task return when no async work needed
impl TodoList {
    pub fn save(&mut self, cx: &mut Context<Self>) -> Task<Result<()>> {
        if !self.has_unsaved_changes() {
            // No changes to save, return immediately
            return Task::ready(Ok(()));
        }

        let todos = self.todos.clone();
        cx.background_spawn(async move {
            save_to_disk(todos).await
        })
    }
}
```

### Entity creation guard pattern

```rust
// Guard entity for scoped operations
impl TodoImporter {
    fn import_with_progress(&mut self, file_path: PathBuf, cx: &mut Context<Self>) {
        self.is_importing = true;
        cx.notify();

        // Create a guard entity that will clean up when dropped
        let guard = cx.new(|_| ());

        cx.observe_release(&guard, |this, _guard, cx| {
            this.is_importing = false;
            this.clear_progress();
            cx.notify();
        })
        .detach();

        // The guard will be dropped when import completes or errors
        cx.spawn(async move |this, cx| {
            let _guard = guard; // Keep guard alive until async completes

            let todos = load_todos_from_file(file_path).await?;

            this.update(&cx, |this, cx| {
                this.merge_imported_todos(todos);
                cx.notify();
            })?;

            Ok(())
        })
        .detach_and_log_err(cx);
    }
}
```

## EventEmitter Patterns

### Basic EventEmitter implementation

```rust
// Simple event emitter for TodoList
impl EventEmitter<TodoListEvent> for TodoList {}

#[derive(Debug, Clone)]
pub enum TodoListEvent {
    ItemAdded(Todo),
    ItemRemoved(TodoId),
    ItemCompleted(TodoId),
    ItemUpdated { id: TodoId, old: Todo, new: Todo },
    ListCleared,
    ListReordered,
}
```

### Emitting events

```rust
// Emitting events from TodoList methods
impl TodoList {
    pub fn add_todo(&mut self, label: String, cx: &mut Context<Self>) -> TodoId {
        let todo = Todo::new(label);
        let id = todo.id;
        self.todos.push(todo.clone());

        cx.emit(TodoListEvent::ItemAdded(todo));
        cx.notify();

        id
    }

    pub fn complete_todo(&mut self, id: TodoId, cx: &mut Context<Self>) {
        if let Some(todo) = self.todos.iter_mut().find(|t| t.id == id) {
            todo.complete = true;
            cx.emit(TodoListEvent::ItemCompleted(id));
            cx.notify();
        }
    }
}
```

### Multiple event types for single entity

```rust
// Multiple EventEmitter implementations for different concerns
pub struct TodoStatsUpdated {
    pub completed_count: usize,
    pub total_count: usize,
}
impl EventEmitter<TodoStatsUpdated> for TodoList {}

pub struct TodoListSaved {
    pub path: PathBuf,
    pub timestamp: SystemTime,
}
impl EventEmitter<TodoListSaved> for TodoList {}

pub struct TodoListError(pub String);
impl EventEmitter<TodoListError> for TodoList {}
```

### Event with payload

```rust
// Event with complex data payload
pub struct TodoChangeEvent {
    pub change_type: ChangeType,
    pub affected_ids: Vec<TodoId>,
    pub metadata: ChangeMetadata,
}

#[derive(Debug, Clone)]
pub enum ChangeType {
    BatchUpdate,
    Reorder { old_positions: Vec<usize>, new_positions: Vec<usize> },
    Import { source: String },
}

#[derive(Debug, Clone)]
pub struct ChangeMetadata {
    pub timestamp: SystemTime,
    pub user_action: bool,
    pub description: Option<String>,
}

impl EventEmitter<TodoChangeEvent> for TodoList {}
```

## Entity Passing Patterns

### Passing entity to functions

```rust
// Entity as parameter for operations
impl TodoList {
    pub fn create_linked_list(
        parent: &Entity<TodoList>,
        name: String,
        window: &mut Window,
        cx: &mut App,
    ) -> Entity<Self> {
        let parent_id = parent.read(cx).id.clone();

        cx.new(|cx| {
            let mut list = Self::new(name);
            list.parent_id = Some(parent_id);
            list.inherit_settings_from(parent, cx);
            list
        })
    }
}
```

### Cloning entities for async operations

```rust
// Cloning entities for use in async tasks
impl TodoSyncService {
    fn sync_lists(&mut self, cx: &mut Context<Self>) {
        let primary_list = self.primary_list.clone();
        let backup_list = self.backup_list.clone();
        let sync_log = self.sync_log.clone();

        cx.spawn(async move |this, cx| {
            // Use cloned entities in async context
            let todos = primary_list.read_with(&cx, |list, _| list.todos.clone())?;

            backup_list.update(&cx, |backup, cx| {
                backup.replace_todos(todos);
                cx.notify();
            })?;

            sync_log.update(&cx, |log, cx| {
                log.record_sync(SystemTime::now());
                cx.notify();
            })?;

            Ok(())
        })
        .detach_and_log_err(cx);
    }
}
```

### Entity in return types

```rust
// Returning new entity from factory method
impl TodoList {
    pub fn split_into_sublists(&mut self, cx: &mut Context<Self>) -> Vec<Entity<TodoList>> {
        let mut sublists = Vec::new();
        let categories = self.group_by_category();

        for (category, todos) in categories {
            let sublist = cx.new(|cx| {
                let mut list = TodoList::new(category.to_string());
                list.todos = todos;
                list.parent_id = Some(self.id.clone());
                list
            });

            sublists.push(sublist);
        }

        sublists
    }
}
```

### Storing entities in collections

```rust
// Entity collections for managing multiple lists
struct TodoWorkspace {
    active_lists: HashMap<ListId, Entity<TodoList>>,
    archived_lists: Vec<Entity<TodoList>>,
    list_views: HashMap<ViewId, Entity<TodoListView>>,
    shared_filters: BTreeMap<String, Entity<TodoFilter>>,
}

impl TodoWorkspace {
    pub fn get_list(&self, id: &ListId) -> Option<&Entity<TodoList>> {
        self.active_lists.get(id)
    }

    pub fn archive_list(&mut self, id: ListId, cx: &mut Context<Self>) -> Result<()> {
        if let Some(list) = self.active_lists.remove(&id) {
            list.update(cx, |list, cx| {
                list.archived = true;
                list.archived_at = Some(SystemTime::now());
                cx.notify();
            });

            self.archived_lists.push(list);
        }
        Ok(())
    }
}
```

## Complex Entity Patterns

### Entity with async initialization

```rust
// Complex async entity loading from multiple sources
impl TodoList {
    pub fn load_from_cloud(
        list_id: ListId,
        workspace: &Entity<TodoWorkspace>,
        window: &mut Window,
        cx: &mut App,
    ) -> Task<Result<Entity<Self>>> {
        let sync_service = workspace.read(cx).sync_service.clone();
        let template_loader = cx.background_spawn(async move {
            load_templates_for_list(list_id).await
        });

        window.spawn(cx, async move |cx| {
            // Load todos from cloud
            let todos = sync_service.fetch_todos(list_id).await?;
            let templates = template_loader.await.ok();

            // Load user preferences
            let preferences = load_user_preferences().await.unwrap_or_default();

            cx.update(|window, cx| {
                cx.new(|cx| {
                    let mut list = Self::new(todos.name);
                    list.todos = todos.items;
                    list.templates = templates;
                    list.apply_preferences(preferences);
                    list.mark_synced();
                    list
                })
            })
        })
    }
}
```

### Entity factory pattern

```rust
// Factory with configuration injection
impl TodoListFactory {
    pub fn create_with_config(
        config: TodoListConfig,
        environment: Arc<TodoEnvironment>,
        cx: &mut App,
    ) -> Entity<TodoList> {
        cx.new(|cx| {
            let mut list = TodoList::new(config.name.clone());

            // Inject environment dependencies
            list.storage_backend = environment.storage.clone();
            list.sync_service = environment.sync_service.clone();
            list.notification_service = environment.notifications.clone();

            // Apply configuration
            list.auto_save_enabled = config.auto_save;
            list.sync_interval = config.sync_interval;
            list.max_items = config.max_items;

            // Initialize with default todos if specified
            if let Some(template) = config.template {
                list.apply_template(template, cx);
            }

            list
        })
    }
}
```

### Cross-entity communication pattern

```rust
// Entity communication via channels for real-time collaboration
impl TodoList {
    pub fn enable_collaboration(&mut self, cx: &mut Context<Self>) {
        let (sender, mut receiver) = mpsc::unbounded_channel();
        self.collaboration_sender = Some(sender);

        cx.spawn(async move |this, cx| {
            while let Some(change) = receiver.next().await {
                match this.update(&cx, |list, cx| {
                    match change {
                        CollaborationChange::TodoAdded(todo) => {
                            list.merge_remote_todo(todo);
                        }
                        CollaborationChange::TodoUpdated(id, updates) => {
                            list.apply_remote_updates(id, updates);
                        }
                        CollaborationChange::UserJoined(user) => {
                            list.add_collaborator(user);
                        }
                    }
                    cx.notify();
                }) {
                    Ok(_) => {},
                    Err(_) => break, // Entity was dropped
                }
            }
        })
        .detach();
    }
}
```

### Entity delegation pattern

```rust
// Delegating operations to specialized entities
struct TodoListController {
    list: Entity<TodoList>,
    filter_manager: Entity<FilterManager>,
    sort_engine: Entity<SortEngine>,
    export_service: Entity<ExportService>,
}

impl TodoListController {
    fn new(list: Entity<TodoList>, cx: &mut Context<Self>) -> Self {
        let filter_manager = cx.new(|_| FilterManager::new());
        let sort_engine = cx.new(|_| SortEngine::default());
        let export_service = cx.new(|_| ExportService::new());

        // Delegate change notifications
        cx.subscribe(&list, |this, _, event, cx| {
            // Forward events to specialized handlers
            this.filter_manager.update(cx, |filter, cx| {
                filter.handle_list_change(event, cx);
            });
            cx.notify();
        })
        .detach();

        Self {
            list,
            filter_manager,
            sort_engine,
            export_service,
        }
    }

    // Delegate operations to specialized entities
    fn apply_filter(&mut self, filter: TodoFilter, cx: &mut Context<Self>) {
        self.filter_manager.update(cx, |manager, cx| {
            manager.set_filter(filter, cx);
        });
    }
}
```

### Entity state machine pattern

```rust
// Entity with state transitions for todo list lifecycle
enum TodoListState {
    Loading(LoadingState),
    Active(ActiveState),
    Syncing(SyncingState),
    Archived(ArchivedState),
}

struct LoadingState {
    source: DataSource,
    progress: f32,
}

struct ActiveState {
    todos: Vec<Todo>,
    last_modified: SystemTime,
}

struct SyncingState {
    local_todos: Vec<Todo>,
    pending_changes: Vec<Change>,
    sync_task: Task<Result<()>>,
}

struct ArchivedState {
    todos: Vec<Todo>,
    archived_at: SystemTime,
}

impl TodoList {
    pub fn new_loading(source: DataSource) -> Self {
        Self {
            state: TodoListState::Loading(LoadingState {
                source,
                progress: 0.0,
            }),
            // other fields...
        }
    }

    pub fn transition_to_active(&mut self, todos: Vec<Todo>, cx: &mut Context<Self>) {
        self.state = TodoListState::Active(ActiveState {
            todos,
            last_modified: SystemTime::now(),
        });
        cx.emit(TodoListEvent::StateChanged);
        cx.notify();
    }

    pub fn start_sync(&mut self, cx: &mut Context<Self>) -> Result<()> {
        let TodoListState::Active(active) = &self.state else {
            return Err(anyhow!("Can only sync from active state"));
        };

        let sync_task = self.create_sync_task(cx);

        self.state = TodoListState::Syncing(SyncingState {
            local_todos: active.todos.clone(),
            pending_changes: Vec::new(),
            sync_task,
        });

        cx.notify();
        Ok(())
    }
}
```

### Entity registry pattern

```rust
// Registry managing todo list entities
struct TodoListRegistry {
    lists: HashMap<ListId, Entity<TodoList>>,
    list_metadata: HashMap<ListId, ListMetadata>,
    active_list: Option<ListId>,
    _subscription: Subscription,
}

impl TodoListRegistry {
    pub fn new(
        storage: Entity<TodoStorage>,
        cx: &mut Context<Self>,
    ) -> Self {
        let mut registry = Self {
            lists: HashMap::default(),
            list_metadata: HashMap::default(),
            active_list: None,
            _subscription: cx.subscribe(&storage, Self::handle_storage_event),
        };

        registry.load_saved_lists(cx);
        registry
    }

    pub fn register_list(&mut self, list: Entity<TodoList>, cx: &mut Context<Self>) -> ListId {
        let id = ListId::generate();
        let metadata = list.read(cx).create_metadata();

        self.lists.insert(id.clone(), list.clone());
        self.list_metadata.insert(id.clone(), metadata);

        cx.observe_release(&list, {
            let id = id.clone();
            move |this, _, _| {
                this.unregister_list(&id);
            }
        })
        .detach();

        cx.emit(RegistryEvent::ListRegistered(id.clone()));
        id
    }

    pub fn get_list(&self, id: &ListId) -> Option<Entity<TodoList>> {
        self.lists.get(id).cloned()
    }

    pub fn get_active_list(&self) -> Option<Entity<TodoList>> {
        self.active_list.as_ref().and_then(|id| self.lists.get(id).cloned())
    }

    fn handle_storage_event(
        &mut self,
        _storage: Entity<TodoStorage>,
        event: &StorageEvent,
        cx: &mut Context<Self>,
    ) {
        match event {
            StorageEvent::ListSaved(id) => {
                if let Some(metadata) = self.list_metadata.get_mut(id) {
                    metadata.last_saved = SystemTime::now();
                }
            }
            StorageEvent::ListDeleted(id) => {
                self.unregister_list(id);
            }
        }
        cx.notify();
    }
}
```
