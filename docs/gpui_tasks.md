# GPUI Task Patterns

1. [Basic Spawn Patterns](#basic-spawn-patterns)
2. [Window Spawn Patterns](#window-spawn-patterns)
3. [Background Spawn Patterns](#background-spawn-patterns)
4. [Entity Update Patterns](#entity-update-patterns)
5. [Task Storage Patterns](#task-storage-patterns)
6. [Error Handling Patterns](#error-handling-patterns)
7. [Task Lifecycle Patterns](#task-lifecycle-patterns)
8. [Complex Async Patterns](#complex-async-patterns)

## Basic Spawn Patterns

### Simple spawn from Context<T>

```rust
// Basic entity update after async operation
impl TodoList {
    fn load_todos(&mut self, cx: &mut Context<Self>) {
        cx.spawn(async move |this, cx| {
            let todos = fetch_todos_from_api().await?;

            this.update(&cx, |list, cx| {
                list.todos = todos;
                list.mark_synced();
                cx.emit(TodoListEvent::TodosLoaded);
                cx.notify();
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Spawn with WeakEntity handle

```rust
// Using WeakEntity in spawn for safe background updates
impl TodoSync {
    fn sync_periodically(&mut self, list: WeakEntity<TodoList>, cx: &mut Context<Self>) {
        cx.spawn(async move |handle, cx| {
            let sync_result = perform_sync().await;

            // WeakEntity update can fail if entity was dropped
            handle.update(&cx, |sync, cx| {
                list.update(cx, |list, cx| {
                    list.last_sync = sync_result.timestamp;
                    list.sync_status = SyncStatus::Completed;
                    cx.notify();
                })
            })?;

            Ok(())
        }).detach();
    }
}
```

### Spawn returning a value

```rust
// Returning value from spawn for batch operations
impl TodoList {
    fn import_todos(&mut self, sources: Vec<ImportSource>, cx: &mut Context<Self>) -> Task<Vec<Todo>> {
        cx.spawn(async move |this, cx| {
            let mut imported_todos = Vec::new();

            for source in sources {
                let todos = match source {
                    ImportSource::File(path) => load_from_file(path).await?,
                    ImportSource::Url(url) => fetch_from_url(url).await?,
                    ImportSource::Clipboard => parse_clipboard_content().await?,
                };
                imported_todos.extend(todos);
            }

            this.update(&cx, |list, cx| {
                list.merge_todos(&imported_todos);
                cx.notify();
            })?;

            Ok(imported_todos)
        })
    }
}
```

### Spawn with async move pattern (CORRECT)

```rust
// Correct async move syntax - async move comes BEFORE parameters
impl TodoDiff {
    fn compute_changes(&mut self, cx: &mut Context<Self>) {
        let old_todos = self.snapshot.clone();
        let current_todos = self.current.clone();

        self.diff_task = cx.spawn(async move |this, cx| {
            // Compute diff in background
            let changes = compute_todo_diff(old_todos, current_todos).await?;

            this.update(&cx, |diff, cx| {
                diff.pending_changes = changes;
                diff.state = DiffState::Computed;
                cx.emit(TodoDiffEvent::ChangesComputed);
                cx.notify();
            })?;

            Ok(())
        });
    }
}
```

## Window Spawn Patterns

### Basic window.spawn

```rust
// Window spawn for UI-related async operations
impl TodoListView {
    fn load_templates(&mut self, window: &mut Window, cx: &mut App) -> Task<Result<()>> {
        window.spawn(cx, async move |cx| {
            let templates = load_todo_templates().await?;

            cx.update(|window, cx| {
                // Update UI with loaded templates
                window.show_notification("Templates loaded", cx);
            })?;

            Ok(())
        })
    }
}
```

### Window spawn with workspace operations

```rust
// Opening todo editor at specific item
impl TodoWorkspace {
    fn open_todo_editor(&mut self, todo_id: TodoId, window: &mut Window, cx: &mut App) {
        window.spawn(cx, async move |cx| {
            let Some(editor) = find_or_create_editor(todo_id).await? else {
                return Err(anyhow::anyhow!("Failed to create todo editor"));
            };

            cx.update(|window, cx| {
                editor.update(cx, |editor, cx| {
                    editor.focus_todo(todo_id, window, cx);
                    editor.expand_details(cx);
                })
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Window spawn for deserialization

```rust
// Deserializing todo list state from storage
impl TodoList {
    fn deserialize(
        state: TodoListState,
        workspace: Entity<TodoWorkspace>,
        window: &mut Window,
        cx: &mut App,
    ) -> Task<Result<Entity<Self>>> {
        window.spawn(cx, async move |cx| {
            // Load any async dependencies
            let storage = TodoStorage::connect().await?;
            let sync_service = if state.sync_enabled {
                Some(SyncService::initialize(state.sync_config).await?)
            } else {
                None
            };

            // Restore todos from persistent state
            let todos = storage.load_todos(state.list_id).await?;

            cx.update(|window, cx| {
                cx.new(|cx| {
                    let mut list = TodoList::new(state.name);
                    list.todos = todos;
                    list.filter = state.filter;
                    list.sort_order = state.sort_order;
                    list.sync_service = sync_service;
                    list.restore_selection(state.selected_ids, cx);
                    list
                })
            })
        })
    }
}
```

## cx.spawn_in Patterns

### spawn_in with workspace update

```rust
// Creating todo report and adding to workspace
impl TodoReporter {
    fn generate_report(&mut self, window: &Window, cx: &mut Context<Self>) {
        cx.spawn_in(window, async move |workspace, cx| {
            let report = generate_todo_report().await?;

            // Create buffer with report content
            let buffer = cx.update(|_, cx| {
                let mut buffer = Buffer::new(0, cx);
                buffer.edit(
                    [(0..0, format!("Todo Report Generated at {}\n\n{}",
                        chrono::Local::now().format("%Y-%m-%d %H:%M:%S"),
                        report))],
                    None,
                    cx,
                );
                buffer
            })?;

            // Add report to workspace
            workspace.update_in(cx, |workspace, window, cx| {
                let report_view = cx.new(|cx| {
                    TodoReportView::new(buffer, workspace.todo_list.clone(), cx)
                });

                workspace.add_item_to_active_pane(
                    Box::new(report_view),
                    None,
                    true,
                    window,
                    cx,
                );
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### spawn_in for async UI updates

```rust
// Authentication with UI feedback for todo sync service
impl TodoSyncView {
    fn authenticate(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        let service_name = self.sync_service.name();

        self.auth_task = Some(cx.spawn_in(window, {
            let credentials = self.pending_credentials.clone();

            async move |this, cx| {
                let result = authenticate_sync_service(credentials).await;

                match &result {
                    Ok(token) => {
                        log::info!("Todo sync authenticated successfully");
                        telemetry::event!("Todo Sync Auth Success", service = service_name);
                    }
                    Err(error) => {
                        log::error!("Todo sync authentication failed: {}", error);
                        telemetry::event!(
                            "Todo Sync Auth Failed",
                            service = service_name,
                            error = error.to_string()
                        );
                    }
                }

                this.update(&cx, |view, cx| {
                    view.auth_task = None;
                    view.auth_state = match result {
                        Ok(token) => AuthState::Authenticated(token),
                        Err(e) => AuthState::Failed(e.to_string()),
                    };
                    cx.notify();
                })?;

                result
            }
        }));
    }
}
```

### spawn_in with background work

```rust
// Background filtering of todos with UI update
impl TodoFilterPicker {
    fn filter_todos(&mut self, query: String, window: &Window, cx: &mut Context<Self>) {
        let all_todos = self.todos.clone();

        cx.spawn_in(window, async move |this, cx| {
            // Perform heavy filtering in background
            let filtered_results = cx
                .background_spawn(async move {
                    let mut categories: BTreeMap<Option<String>, Vec<Todo>> = BTreeMap::default();

                    for todo in all_todos.iter() {
                        if todo.matches_query(&query) {
                            categories
                                .entry(todo.category.clone())
                                .or_default()
                                .push(todo.clone());
                        }
                    }

                    categories
                })
                .await;

            // Update UI on main thread
            this.update(&cx, |picker, cx| {
                picker.filtered_todos = filtered_results;
                picker.update_display(cx);
                cx.notify();
            })?;

            Ok(())
        }).detach();
    }
}
```

## Background Spawn Patterns

### Simple background computation

```rust
// Background text processing for large todo descriptions
impl TodoList {
    async fn prepare_export_text(&self, cx: &mut App) -> String {
        let todos_text = self.todos.iter()
            .map(|t| t.to_export_string())
            .collect::<Vec<_>>()
            .join("\n");

        // Process large text in background
        let processed = cx
            .background_spawn({
                let text = todos_text.clone();
                async move {
                    // Heavy text processing (formatting, sanitization, etc.)
                    format_and_sanitize_export(text)
                }
            })
            .await;

        processed
    }
}
```

### Background spawn with complex computation

```rust
// Background computation for todo analytics
impl TodoAnalytics {
    fn compute_statistics(&mut self, cx: &mut Context<Self>) -> Task<TodoStats> {
        let todos = self.todos.clone();
        let history = self.history.clone();
        let date_range = self.current_range.clone();

        cx.background_spawn({
            async move {
                let mut stats = TodoStats::default();

                // Complex analytics computation
                for todo in todos.iter() {
                    stats.total_count += 1;
                    if todo.complete {
                        stats.completed_count += 1;
                        stats.completion_times.push(todo.completed_at);
                    }

                    // Calculate velocity, burndown, etc.
                    if date_range.contains(&todo.created_at) {
                        stats.velocity_data.record(todo);
                    }
                }

                // Process historical trends
                stats.trends = compute_trends(history, date_range).await;

                Result::<_>::Ok(stats)
            }
        })
    }
}
```

### Background spawn with streaming

```rust
// Streaming todo import from large CSV
impl TodoImporter {
    fn import_streaming(&mut self, stream: impl Stream<Item = Result<String>>, cx: &mut Context<Self>) {
        let (tx, mut rx) = mpsc::unbounded();

        let import_task = cx.background_spawn(async move {
            pin_mut!(stream);

            let mut parser = TodoCsvParser::new();
            let mut total_imported = 0;

            while let Some(chunk) = stream.next().await {
                match chunk {
                    Ok(data) => {
                        parser.push_str(&data);

                        while let Some(todo_result) = parser.next_todo() {
                            match todo_result {
                                Ok(todo) => {
                                    total_imported += 1;
                                    tx.send(ImportEvent::TodoParsed(todo)).await.ok();
                                }
                                Err(e) => {
                                    tx.send(ImportEvent::ParseError(e)).await.ok();
                                }
                            }
                        }
                    }
                    Err(error) => {
                        tx.send(ImportEvent::StreamError(error)).await.ok();
                        break;
                    }
                }
            }

            tx.send(ImportEvent::Complete(total_imported)).await.ok();
            total_imported
        });

        // Handle import events in UI
        cx.spawn(async move |this, cx| {
            while let Some(event) = rx.next().await {
                this.update(&cx, |importer, cx| {
                    importer.handle_import_event(event, cx);
                })?;
            }
            Ok(())
        }).detach();
    }
}
```

### Background spawn for database operations

```rust
// Database operations for todo persistence
impl TodoStorage {
    pub fn load_list(&self, list_id: ListId, cx: &mut App) -> Task<Result<Option<TodoList>>> {
        let db_connection = self.get_connection(cx);

        cx.background_spawn(async move {
            let database = db_connection.await
                .map_err(|e| anyhow!("Failed to connect to database: {}", e))?;

            // Load todos from database
            let todos = database.query_todos(list_id).await?;
            let metadata = database.query_list_metadata(list_id).await?;

            Ok(metadata.map(|meta| TodoList {
                id: list_id,
                name: meta.name,
                todos,
                created_at: meta.created_at,
                modified_at: meta.modified_at,
            }))
        })
    }
}
```

## Entity Update Patterns

### Update from async context with error handling

```rust
// Updating todo list after async validation
impl TodoListView {
    fn validate_and_save(&mut self, window: &Window, cx: &mut Context<Self>) {
        cx.spawn_in(window, async move |this, cx| {
            // Async validation
            let validation_result = validate_todos_async().await?;

            if !validation_result.is_valid {
                return Err(anyhow!("Validation failed: {}", validation_result.message));
            }

            // Update UI after successful validation
            this.update_in(&cx, |view, window, cx| {
                view.apply_validation_results(validation_result, window, cx);
                view.save_todos(cx);
            })?;

            anyhow::Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Chained updates

```rust
// Multiple sequential updates for todo archiving
impl TodoList {
    fn archive_completed(&mut self, cx: &mut Context<Self>) {
        let completed_ids = self.get_completed_ids();

        cx.spawn(async move |this, cx| {
            // First, move todos to archive storage
            let archive_result = archive_todos(completed_ids.clone()).await?;

            // Then update the list
            this.update(&cx, |list, cx| {
                // Remove archived todos from active list
                list.todos.retain(|t| !completed_ids.contains(&t.id));

                // Update archive metadata
                list.archive_count += completed_ids.len();
                list.last_archive_date = Some(SystemTime::now());

                cx.emit(TodoListEvent::TodosArchived(completed_ids));
                cx.notify();
            })?;

            // Finally, trigger sync if enabled
            this.update(&cx, |list, cx| {
                if list.auto_sync {
                    list.trigger_sync(cx);
                }
            })?;

            Ok(())
        }).detach_and_log_err(cx);
    }
}
```

### Update with subscription

```rust
// Subscribing to real-time todo updates
impl TodoCollaborationView {
    fn subscribe_to_updates(&mut self, cx: &mut Context<Self>) {
        let (tx, mut rx) = mpsc::unbounded();
        self.subscribe_to_collaboration_events(tx);

        let task = cx.spawn(async move |this, cx| {
            while let Some(update) = rx.next().await {
                // Continue processing even if individual update fails
                this.update(&cx, |view, cx| {
                    match update {
                        CollabEvent::TodoAdded(todo) => view.add_remote_todo(todo, cx),
                        CollabEvent::TodoUpdated(id, changes) => view.update_todo(id, changes, cx),
                        CollabEvent::UserCursorMoved(user, position) => view.update_cursor(user, position, cx),
                    }
                    cx.notify();
                })
                .ok(); // Don't stop on individual failures
            }
        });

        self.collab_task = Some(task);
    }
}
```

### Conditional update pattern

```rust
// Conditional entity update based on validation
impl TodoInputView {
    fn submit_todo(&mut self, window: &Window, cx: &mut Context<Self>) {
        let input_text = self.input_field.text().clone();

        cx.spawn_in(window, async move |this, cx| {
            // Wait for any pending validation
            validate_input(&input_text).await?;

            this.update_in(&cx, |view, window, cx| {
                // Only submit if input is still valid
                if view.input_field.text().trim().is_empty() {
                    window.show_notification("Cannot add empty todo", cx);
                    return;
                }
                view.create_todo_from_input(window, cx);
                view.clear_input(cx);
            })?;

            Ok(())
        }).detach();
    }
}
```

## Task Storage Patterns

### Task stored in struct field

```rust
// Storing update task for todo synchronization
struct TodoSyncState {
    sync_task: Task<Result<()>>,
    last_sync: SystemTime,
    pending_changes: Vec<TodoChange>,
}

impl TodoSyncState {
    pub fn update(&mut self, cx: &mut Context<TodoList>) {
        let changes = self.pending_changes.clone();

        self.sync_task = cx.spawn(async move |list, cx| {
            // Sync changes to server
            let result = sync_changes_to_server(changes).await?;

            list.update(&cx, |list, cx| {
                list.mark_synced(result.timestamp);
                cx.notify();
            })?;

            Ok(())
        });
    }
}
```

### Optional task field

```rust
// Optional task for todo export operations
struct TodoExportView {
    export_task: Option<Task<Result<PathBuf>>>,
    export_progress: f32,
    export_format: ExportFormat,
}

impl TodoExportView {
    fn start_export(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        // Cancel any existing export
        self.export_task = None;

        let todos = self.get_todos_to_export();
        let format = self.export_format.clone();

        self.export_task = Some(cx.spawn_in(window, async move |this, cx| {
            let file_path = export_todos(todos, format).await?;

            this.update(&cx, |view, cx| {
                view.export_task = None; // Clear task when done
                view.export_progress = 1.0;
                view.show_export_complete(file_path.clone(), cx);
                cx.notify();
            })?;

            Ok(file_path)
        }));
    }
}
```

### Task collection

```rust
// Collecting multiple todo operation tasks
impl TodoBatchOperations {
    pub fn update_all_todos(&self, updates: Vec<TodoUpdate>, cx: &mut App) -> Task<()> {
        let tasks = updates
            .into_iter()
            .map(|update| {
                cx.background_spawn(async move {
                    apply_todo_update(update).await
                })
            })
            .collect::<Vec<_>>();

        cx.spawn(async move |_, _| {
            let results = futures::future::join_all(tasks).await;

            // Log any failures but continue
            for (idx, result) in results.into_iter().enumerate() {
                if let Err(e) = result {
                    log::error!("Failed to update todo {}: {}", idx, e);
                }
            }
        })
    }
}
```

## Error Handling Patterns

### detach_and_log_err pattern

```rust
// Fire-and-forget with error logging for todo updates
impl TodoList {
    fn auto_save(&mut self, cx: &mut Context<Self>) {
        // Fire and forget auto-save with error logging
        self.save_to_storage(cx)
            .detach_and_log_err(cx);

        // Also update UI asynchronously
        cx.spawn(async move |this, cx| {
            this.update(&cx, |list, cx| {
                list.update_save_indicator(cx)
            })
        })
        .detach_and_log_err(cx);
    }
}
```

### Manual error logging with detach

```rust
// Manual error handling before detach for todo notifications
impl TodoNotificationService {
    fn send_reminder(&mut self, todo: Todo, cx: &mut Context<Self>) {
        cx.background_spawn(async move {
            match send_notification(todo.clone()).await {
                Ok(notification_id) => {
                    log::debug!("Sent reminder for todo {}: {}", todo.id, notification_id);
                }
                Err(e) => {
                    log::error!("Failed to send reminder for todo {}: {}", todo.id, e);
                    // Try fallback notification method
                    if let Err(fallback_err) = send_fallback_notification(todo).await {
                        log::error!("Fallback notification also failed: {}", fallback_err);
                    }
                }
            }
        })
        .detach();
    }
}
```

### Propagating errors through tasks

```rust
// Error propagation through todo operations
impl TodoWorkspace {
    fn open_todo_in_editor(&self, todo_id: TodoId, window: &mut Window, cx: &mut App) -> Task<Result<()>> {
        let workspace = self.weak_self.clone();
        let list = self.active_list.clone();

        window.spawn(cx, async move |cx| {
            let workspace = workspace.upgrade().context("Workspace was dropped")?;
            let list = list.upgrade().context("Todo list was dropped")?;

            let todo = list.update(&cx, |list, _| {
                list.find_todo(todo_id)
                    .ok_or_else(|| anyhow!("Todo {} not found", todo_id))
            })??;

            workspace.update(&cx, |workspace, cx| {
                workspace.open_todo_editor(todo, cx)
            })?;

            Ok(())
        })
    }
}
```

### Task with fallback on error

```rust
// Fallback to Task::ready for optional todo operations
impl TodoList {
    fn delete_todo(&mut self, todo_id: TodoId, cx: &mut Context<Self>) -> Task<Result<()>> {
        // Try to delete from storage, fallback to immediate success if not connected
        self.storage
            .as_ref()
            .and_then(|storage| {
                storage.update(cx, |storage, cx| {
                    storage.delete_todo(todo_id, cx)
                })
            })
            .unwrap_or_else(|| {
                // No storage connected, just remove locally
                self.todos.retain(|t| t.id != todo_id);
                cx.notify();
                Task::ready(Ok(()))
            })
    }
}
```

## Task Lifecycle Patterns

### Task::ready for immediate values

```rust
// Immediate task completion for todo operations
impl TodoSyncService {
    pub fn cancel_sync(&mut self, cx: &mut Context<Self>) -> Task<()> {
        // If no sync in progress, return immediately
        let Some(sync_task) = self.active_sync.take() else {
            return Task::ready(());
        };

        // Cancel the active sync
        cx.spawn(async move {
            sync_task.abort();
            log::info!("Todo sync cancelled");
        })
    }
}
```

### Shared task pattern

```rust
// Sharing task result for todo template loading
impl TodoTemplateLoader {
    fn load_templates(&self, cx: &mut App) -> Shared<Task<Result<Vec<TodoTemplate>>>> {
        // Check if templates are already cached
        if let Some(cached) = self.cached_templates.clone() {
            return Task::ready(Ok(cached)).shared();
        }

        // Load templates and share the result
        cx.background_spawn(async move {
            let templates = load_templates_from_disk().await?;
            Ok(templates)
        })
        .shared()
    }
}
```

### Task with timer

```rust
// Timer-based task for todo notifications
impl TodoReminderView {
    fn show_temporary_notification(&mut self, message: String, cx: &mut Context<Self>) {
        self.notification_visible = true;
        self.notification_message = message;
        cx.notify();

        cx.spawn(async move |this, cx| {
            // Wait 3 seconds then hide notification
            cx.background_executor().timer(Duration::from_secs(3)).await;

            this.update(&cx, |view, cx| {
                view.notification_visible = false;
                cx.notify();
            })
        })
        .detach();
    }
}
```

### Task with channel communication

```rust
// Using oneshot channel for todo dialog response
impl TodoDeleteDialog {
    fn show(&mut self, todo: Todo, cx: &mut Context<Self>) -> Task<bool> {
        let (tx, rx) = oneshot::channel();

        self.pending_response = Some(tx);
        self.todo_to_delete = Some(todo.clone());
        self.visible = true;
        cx.notify();

        cx.spawn(async move |this, cx| {
            // Wait for user response
            let confirmed = rx.await.unwrap_or(false);

            if confirmed {
                this.update(&cx, |dialog, cx| {
                    dialog.delete_todo(todo, cx);
                })?;
            }

            Ok(confirmed)
        })
    }
}
```

## Complex Async Patterns

### Join multiple tasks

```rust
// Joining multiple todo list loads
impl TodoWorkspace {
    fn load_all_lists(&self, window: &mut Window, cx: &mut App) -> Task<Result<Vec<TodoList>>> {
        let list_ids = self.get_list_ids();
        let load_tasks: Vec<_> = list_ids
            .into_iter()
            .map(|id| self.storage.load_list(id, cx))
            .collect();

        window.spawn(cx, async move |cx| {
            let lists = futures::future::try_join_all(load_tasks)
                .await
                .context("Failed to load todo lists")?;

            // Filter out None values and collect valid lists
            let valid_lists: Vec<TodoList> = lists.into_iter().flatten().collect();

            Ok(valid_lists)
        })
    }
}
```

### Nested spawns

```rust
// Nested async operations for todo filtering
impl TodoFilterPicker {
    fn refresh_filters(&mut self, window: &mut Window, cx: &mut Context<Self>) -> Task<Result<()>> {
        cx.spawn_in(window, async move |this, cx| {
            async fn load_filter_data(
                picker: &WeakEntity<TodoFilterPicker>,
                cx: &mut AsyncWindowContext,
            ) -> Result<()> {
                // Get multiple async operations
                let (categories_task, tags_task, dates_task) = picker.update(&cx, |picker, cx| {
                    (
                        picker.load_categories(cx),
                        picker.load_tags(cx),
                        picker.load_date_ranges(cx),
                    )
                })?;

                // Wait for all data in parallel
                let (categories, tags, date_ranges) =
                    futures::join!(categories_task, tags_task, dates_task);

                // Update picker with all results
                picker.update(&cx, |picker, cx| {
                    picker.available_categories = categories?;
                    picker.available_tags = tags?;
                    picker.date_ranges = date_ranges?;
                    picker.refresh_display(cx);
                    cx.notify();
                    Ok(())
                })?
            }

            load_filter_data(&this, cx).await
        })
    }
}
```

### Stream processing pattern

```rust
// Processing async stream for real-time todo updates
impl TodoStreamProcessor {
    fn process_update_stream(&mut self, stream: impl Stream<Item = Result<TodoUpdate>>, cx: &mut Context<Self>) {
        let (tx, mut rx) = mpsc::unbounded();

        let process_task = cx.background_spawn(async move {
            pin_mut!(stream);

            let mut update_batch = Vec::new();
            let mut total_processed = 0;

            while let Some(update_result) = stream.next().await {
                match update_result {
                    Ok(update) => {
                        update_batch.push(update.clone());
                        total_processed += 1;

                        // Send batch every 10 updates
                        if update_batch.len() >= 10 {
                            tx.send(StreamEvent::Batch(update_batch.clone())).await.ok();
                            update_batch.clear();
                        }
                    }
                    Err(error) => {
                        tx.send(StreamEvent::Error(error)).await.ok();
                        break;
                    }
                }
            }

            // Send remaining updates
            if !update_batch.is_empty() {
                tx.send(StreamEvent::Batch(update_batch)).await.ok();
            }

            tx.send(StreamEvent::Complete(total_processed)).await.ok();
            total_processed
        });

        self.stream_task = Some(process_task);
    }
}
```

### Task with retry logic

```rust
// Polling with retry for todo sync status
impl TodoSyncMonitor {
    fn monitor_sync_progress(&mut self, sync_id: SyncId, cx: &mut Context<Self>) -> Task<Result<SyncResult>> {
        let max_retries = 10;
        let poll_interval = Duration::from_millis(500);

        cx.spawn(async move |this, cx| {
            for retry in 0..max_retries {
                // Check sync status
                let status = this.update(&cx, |monitor, cx| {
                    monitor.check_sync_status(sync_id, cx)
                })?;

                match status {
                    SyncStatus::Complete(result) => return Ok(result),
                    SyncStatus::Failed(error) => {
                        if retry < max_retries - 1 {
                            log::warn!("Sync failed, retrying ({}/{}): {}", retry + 1, max_retries, error);
                            // Continue to retry
                        } else {
                            return Err(anyhow!("Sync failed after {} retries: {}", max_retries, error));
                        }
                    }
                    SyncStatus::InProgress => {
                        // Continue polling
                    }
                }

                cx.background_executor().timer(poll_interval).await;
            }

            Err(anyhow!("Sync timed out after {} retries", max_retries))
        })
    }
}
```

### Concurrent task management

```rust
// Managing multiple concurrent todo imports
impl TodoImportManager {
    fn import_from_multiple_sources(&mut self, sources: Vec<ImportSource>, window: &mut Window, cx: &mut Context<Self>) {
        let import_tasks: Vec<_> = sources
            .into_iter()
            .map(|source| {
                cx.background_spawn(async move {
                    match source {
                        ImportSource::File(path) => import_todos_from_file(path).await,
                        ImportSource::Url(url) => import_todos_from_url(url).await,
                        ImportSource::Database(conn) => import_todos_from_db(conn).await,
                    }
                })
            })
            .collect();

        cx.spawn_in(window, async move |this, cx| {
            let mut imported_lists = Vec::new();
            let mut failed_imports = Vec::new();

            for (idx, task) in import_tasks.into_iter().enumerate() {
                match task.await {
                    Ok(todos) => {
                        log::info!("Successfully imported {} todos from source {}", todos.len(), idx);
                        imported_lists.push(todos);
                    }
                    Err(e) => {
                        log::error!("Failed to import from source {}: {}", idx, e);
                        failed_imports.push((idx, e));
                    }
                }
            }

            this.update(&cx, |manager, cx| {
                manager.process_imported_todos(imported_lists, failed_imports, cx);
                cx.notify();
            })
        }).detach();
    }
}
```

## Key Patterns Summary

### Pattern: Always use `async move` BEFORE the closure parameters

```rust
// CORRECT
cx.spawn(async move |handle, cx| { ... })

// INCORRECT - will not compile
cx.spawn(|handle, cx| async move { ... })
```

### Pattern: Entity updates must use the inner `cx`

```rust
cx.spawn(async move |this, cx| {
    // Use the inner cx, not the outer one
    this.update(cx, |this, cx| {
        // Again, use the inner cx from this closure
        this.some_field = value;
        cx.notify();
    })
})
```

### Pattern: WeakEntity requires error handling

```rust
// WeakEntity operations return Result
handle.update(cx, |this, cx| {
    // Update logic
})?; // Note the ? operator
```

### Pattern: Choose the right context

- Use `cx.spawn()` when you have a `Context<T>`
- Use `window.spawn()` when you have a `Window` reference
- Use `cx.spawn_in()` when you need to spawn in a specific window from an async context
- Use `cx.background_spawn()` for CPU-intensive work off the main thread

### Pattern: Task lifecycle management

- Store tasks in fields when you need to cancel them or await their completion
- Use `.detach()` for fire-and-forget tasks
- Use `.detach_and_log_err()` when errors should be logged but not fatal
- Use `Task::ready()` for immediate values without async work
