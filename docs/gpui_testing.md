# GPUI Testing Patterns

1. [Test Macros](#test-macros)
2. [Test Contexts](#test-contexts)
3. [Common Testing Patterns](#common-testing-patterns)
4. [Test Utilities](#test-utilities)
5. [Integration Testing Patterns](#integration-testing-patterns)
6. [Async Testing Patterns](#async-testing-patterns)
7. [UI Testing Patterns](#ui-testing-patterns)

## Test Macros

### Standard Rust Tests

Use `#[test]` for synchronous unit tests that don't require GPUI context:

```rust
// Testing todo struct logic without GPUI
#[test]
fn test_todo_creation() {
    let todo = Todo::new("Buy groceries".to_string());
    assert_eq!(todo.label, "Buy groceries");
    assert!(!todo.complete);
    assert!(todo.created_at <= SystemTime::now());
}

#[test]
fn test_todo_completion() {
    let mut todo = Todo::new("Write tests".to_string());
    todo.mark_complete();
    assert!(todo.complete);
    assert!(todo.completed_at.is_some());
}
```

### GPUI Tests

Use `#[gpui::test]` for tests requiring GPUI context and async operations:

```rust
// Testing TodoList entity with GPUI context
#[gpui::test]
async fn test_todo_list_operations(cx: &mut gpui::TestAppContext) {
    let todo_list = cx.new(|cx| TodoList::new("My Tasks".to_string(), cx));

    todo_list.update(cx, |list, cx| {
        list.add_todo("Complete documentation".to_string(), cx);
        list.add_todo("Review pull requests".to_string(), cx);
        assert_eq!(list.todos.len(), 2);
        cx.notify();
    });

    // Verify first todo
    assert_eq!(todo_list.read(cx).todos[0].label, "Complete documentation");
}
```

**CRITICAL**: Never import `test` from gpui at module level. Always use fully qualified paths:

```rust
// WRONG - Never do this:
use gpui::test;

// CORRECT - Always use fully qualified:
#[gpui::test]
async fn test_todo_sync(cx: &mut gpui::TestAppContext) {
    // test implementation
}
```

## Test Contexts

### TestAppContext

Provides test-specific methods for todo list operations:

```rust
#[gpui::test]
async fn test_todo_list_creation(cx: &mut gpui::TestAppContext) {
    // Create todo list entity
    let todo_list = cx.new(|cx| TodoList::new("Work Tasks".to_string(), cx));

    // Update global state
    cx.update(|cx| {
        cx.set_global(TodoSettings::default());
    });

    // Spawn async tasks
    cx.spawn(async move |cx| {
        // Async operations
        Ok(())
    }).detach();

    // Run all pending tasks
    cx.run_until_parked();

    // Simulate keyboard input for todo creation
    cx.simulate_keystrokes("New todo item");
}
```

### VisualTestContext

For testing todo list UI components with window operations:

```rust
#[gpui::test]
fn test_todo_list_view(cx: &mut TestAppContext) {
    let (todo_view, cx) = cx.add_window_view(|cx| {
        TodoListView::new(cx.new(|cx| TodoList::new("Test List".to_string(), cx)), cx)
    });

    // cx is now a &mut VisualTestContext
    cx.draw();

    // Simulate clicking on a todo item
    cx.simulate_click(point(50.0, 100.0));

    // Simulate scrolling through todo list
    cx.simulate_event(ScrollWheelEvent {
        position: point(100.0, 200.0),
        delta: ScrollDelta::Lines(point(0.0, 3.0)),
        modifiers: Modifiers::default(),
    });
}
```

## Common Testing Patterns

### Init Test Helper

Define an `init_test` function to set up todo app test dependencies:

```rust
fn init_todo_test(cx: &mut TestAppContext) {
    env_logger::try_init().ok();
    cx.update(|cx| {
        // Initialize todo settings
        let settings = TodoSettings::test_defaults();
        cx.set_global(settings);

        // Initialize storage backend
        let storage = TodoStorage::in_memory();
        cx.set_global(storage);

        // Initialize theme for UI tests
        let theme = TodoTheme::default();
        cx.set_global(theme);
    });
}

#[gpui::test]
async fn test_todo_persistence(cx: &mut TestAppContext) {
    init_todo_test(cx);

    let todo_list = cx.new(|cx| TodoList::new("Persistent List".to_string(), cx));

    // Test with initialized environment
    todo_list.update(cx, |list, cx| {
        list.save_to_storage(cx);
    });
}
```

### Testing Async Operations

```rust
#[gpui::test]
async fn test_todo_sync_service(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| TodoList::new("Synced List".to_string(), cx));

    // Add a todo and wait for sync event
    todo_list.update(cx, |list, cx| {
        list.add_todo("Sync this task".to_string(), cx);
        cx.emit(TodoListEvent::ItemAdded(list.todos.last().unwrap().clone()));
    });

    // Wait for sync completion event
    let event = todo_list.next_event::<TodoSyncEvent>(cx).await;
    assert_eq!(event, TodoSyncEvent::SyncCompleted);

    // Wait for condition
    todo_list.condition(cx, |list, _| {
        list.sync_status == SyncStatus::Synced
    }).await;

    // Wait for notification with timeout
    todo_list.next_notification(Duration::from_millis(100), cx).await;
}
```

### Testing Entity Updates

```rust
#[gpui::test]
async fn test_todo_list_updates(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| TodoList::new("My List".to_string(), cx));

    // Test adding todos
    todo_list.update(cx, |list, cx| {
        list.add_todo("First task".to_string(), cx);
        list.add_todo("Second task".to_string(), cx);
        assert_eq!(list.todos.len(), 2);
        cx.notify();
    });

    // Test completing a todo
    todo_list.update(cx, |list, cx| {
        list.complete_todo(list.todos[0].id, cx);
        assert!(list.todos[0].complete);
        cx.emit(TodoListEvent::ItemCompleted(list.todos[0].id));
    });

    // Verify state
    assert_eq!(todo_list.read(cx).completed_count(), 1);
}
```

### Testing Task Management

```rust
#[gpui::test]
async fn test_todo_import_task(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| TodoList::new("Import Test".to_string(), cx));

    todo_list.update(cx, |list, cx| {
        // Spawn import task
        cx.spawn(async move |this, cx| {
            // Simulate loading todos from file
            let imported_todos = vec![
                Todo::new("Imported task 1".to_string()),
                Todo::new("Imported task 2".to_string()),
            ];

            this.update(&cx, |list, cx| {
                list.merge_todos(imported_todos);
                list.import_status = ImportStatus::Completed;
                cx.notify();
            })?;

            Ok(())
        }).detach();
    });

    // Wait for tasks to complete
    cx.run_until_parked();

    // Verify import
    assert_eq!(todo_list.read(cx).todos.len(), 2);
    assert_eq!(todo_list.read(cx).import_status, ImportStatus::Completed);
}
```

## Test Utilities

### Simulating User Input

```rust
#[gpui::test]
async fn test_todo_input_handling(cx: &mut TestAppContext) {
    let (todo_view, cx) = cx.add_window_view(|cx| {
        TodoListView::new(cx.new(|cx| TodoList::new("Input Test".to_string(), cx)), cx)
    });

    // Simulate typing a new todo
    cx.simulate_keystrokes("Buy groceries");

    // Simulate pressing Enter to add todo
    cx.dispatch_action(todo_view.entity_id(), &AddTodoAction);

    // Simulate clicking checkbox to complete todo
    cx.simulate_click(point(25.0, 150.0)); // Click on checkbox

    // Simulate mouse hover over todo item
    cx.simulate_event(MouseMoveEvent {
        position: point(100.0, 150.0),
        pressed_button: None,
        modifiers: Modifiers::default(),
    });

    // Simulate scrolling todo list
    cx.simulate_event(ScrollWheelEvent {
        position: point(200.0, 300.0),
        delta: ScrollDelta::Lines(point(0.0, 5.0)),
        modifiers: Modifiers::default(),
    });
}
```

### Available Simulate Methods

```rust
#[gpui::test]
async fn test_todo_dialog_interactions(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| TodoList::new("Dialog Test".to_string(), cx));

    // Simulate text input for new todo
    cx.simulate_input("Complete project documentation");

    // Simulate keyboard shortcuts
    cx.simulate_keystrokes("cmd-enter"); // Add todo shortcut

    // Simulate file selection for import
    cx.simulate_new_path_selection(|_| Some(PathBuf::from("/path/to/todos.json")));

    // Simulate confirmation dialog
    cx.simulate_prompt_answer(0); // Click "Yes" to delete todo

    // Simulate window resize for responsive layout
    cx.simulate_window_resize(size(800.0, 600.0));
}
```

### Testing with Multiple Windows

```rust
#[gpui::test]
fn test_todo_multi_window(cx: &mut TestAppContext) {
    // Create main todo list window
    let main_window = cx.add_window(|cx| {
        TodoListView::new(cx.new(|cx| TodoList::new("Main List".to_string(), cx)), cx)
    });

    // Create secondary window for archived todos
    let archive_window = cx.add_window(|cx| {
        TodoArchiveView::new(cx.new(|cx| TodoList::archived(cx)), cx)
    });

    // Update main window
    main_window.update(cx, |view, cx| {
        view.update(cx, |view, cx| {
            view.add_todo("Active task".to_string(), cx);
        });
    });

    // Archive completed todos
    main_window.update(cx, |view, cx| {
        view.update(cx, |view, cx| {
            view.archive_completed(cx);
        });
    });

    // Verify in archive window
    archive_window.update(cx, |view, cx| {
        view.read(cx, |view, _| {
            assert!(view.has_archived_todos());
        });
    });
}
```

## Integration Testing Patterns

### Testing Entity Communication

```rust
#[gpui::test]
async fn test_todo_list_collaboration(cx: &mut TestAppContext) {
    init_todo_test(cx);

    // Create two todo lists that share updates
    let primary_list = cx.new(|cx| TodoList::new("Primary".to_string(), cx));
    let secondary_list = cx.new(|cx| TodoList::new("Secondary".to_string(), cx));

    // Subscribe secondary to primary's events
    secondary_list.update(cx, |list, cx| {
        cx.subscribe(&primary_list, |this, primary, event: &TodoListEvent, cx| {
            match event {
                TodoListEvent::ItemAdded(todo) => {
                    this.add_remote_todo(todo.clone(), cx);
                }
                TodoListEvent::ItemCompleted(id) => {
                    this.mark_remote_completed(*id, cx);
                }
                _ => {}
            }
        }).detach();
    });

    // Add todo to primary
    primary_list.update(cx, |list, cx| {
        list.add_todo("Shared task".to_string(), cx);
    });

    cx.run_until_parked();

    // Verify it appears in secondary
    assert_eq!(secondary_list.read(cx).todos.len(), 1);
    assert_eq!(secondary_list.read(cx).todos[0].label, "Shared task");
}
```

### Testing Storage Integration

```rust
#[gpui::test]
async fn test_todo_persistence_layer(cx: &mut TestAppContext) {
    let storage = cx.new(|_| TodoStorage::in_memory());
    let todo_list = cx.new(|cx| TodoList::with_storage(storage.clone(), cx));

    // Add todos and save
    todo_list.update(cx, |list, cx| {
        list.add_todo("Persistent task 1".to_string(), cx);
        list.add_todo("Persistent task 2".to_string(), cx);
    });

    let save_task = todo_list.update(cx, |list, cx| {
        list.save_to_storage(cx)
    });

    save_task.await.unwrap();

    // Create new list and load
    let restored_list = cx.new(|cx| TodoList::with_storage(storage, cx));
    let load_task = restored_list.update(cx, |list, cx| {
        list.load_from_storage(cx)
    });

    load_task.await.unwrap();

    // Verify todos were persisted
    assert_eq!(restored_list.read(cx).todos.len(), 2);
}
```

## Async Testing Patterns

### Testing Background Tasks

```rust
#[gpui::test]
async fn test_todo_background_sync(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| TodoList::new("Background Test".to_string(), cx));

    // Start background sync
    todo_list.update(cx, |list, cx| {
        list.start_background_sync(cx);
    });

    // Add todo which should trigger sync
    todo_list.update(cx, |list, cx| {
        list.add_todo("Auto-sync task".to_string(), cx);
    });

    // Wait for sync to complete
    todo_list.condition(cx, |list, _| {
        list.last_sync.is_some()
    }).await;

    // Verify sync timestamp
    let last_sync = todo_list.read(cx).last_sync.unwrap();
    assert!(last_sync <= SystemTime::now());
}
```

### Testing Timer-based Operations

```rust
#[gpui::test]
async fn test_todo_auto_save(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| TodoList::new("Auto-save Test".to_string(), cx));

    // Enable auto-save with 100ms delay
    todo_list.update(cx, |list, cx| {
        list.enable_auto_save(Duration::from_millis(100), cx);
        list.add_todo("Trigger auto-save".to_string(), cx);
    });

    // Wait for auto-save timer
    cx.background_executor().timer(Duration::from_millis(150)).await;
    cx.run_until_parked();

    // Verify save occurred
    assert!(todo_list.read(cx).last_save_time.is_some());
}
```

### Testing Error Scenarios

```rust
#[gpui::test]
async fn test_todo_sync_error_handling(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| {
        let mut list = TodoList::new("Error Test".to_string(), cx);
        // Configure to fail on sync
        list.sync_service = Some(SyncService::failing());
        list
    });

    // Attempt sync which should fail
    let sync_result = todo_list.update(cx, |list, cx| {
        list.sync_to_server(cx)
    }).await;

    // Verify error handling
    assert!(sync_result.is_err());
    assert_eq!(
        todo_list.read(cx).sync_status,
        SyncStatus::Failed("Sync service unavailable".to_string())
    );

    // Verify retry mechanism
    todo_list.update(cx, |list, cx| {
        list.retry_sync(cx);
    });

    cx.run_until_parked();
    assert_eq!(todo_list.read(cx).retry_count, 1);
}
```

## UI Testing Patterns

### Testing Component Rendering

```rust
#[gpui::test]
fn test_todo_item_rendering(cx: &mut TestAppContext) {
    let (todo_item, cx) = cx.add_window_view(|cx| {
        TodoItemView::new(
            Todo::new("Test rendering".to_string()),
            cx
        )
    });

    // Force render
    cx.draw();

    // Update todo state and re-render
    todo_item.update(cx, |item, cx| {
        item.todo.complete = true;
        cx.notify();
    });

    cx.draw();

    // Verify rendered state (would need actual assertions on rendered content)
}
```

### Testing Focus Management

```rust
#[gpui::test]
async fn test_todo_focus_navigation(cx: &mut TestAppContext) {
    let (todo_view, cx) = cx.add_window_view(|cx| {
        let list = cx.new(|cx| TodoList::new("Focus Test".to_string(), cx));
        TodoListView::new(list, cx)
    });

    // Add multiple todos
    todo_view.update(cx, |view, cx| {
        view.todo_list.update(cx, |list, cx| {
            list.add_todo("First".to_string(), cx);
            list.add_todo("Second".to_string(), cx);
            list.add_todo("Third".to_string(), cx);
        });
    });

    // Test keyboard navigation
    cx.simulate_keystrokes("down"); // Focus second item
    cx.simulate_keystrokes("down"); // Focus third item
    cx.simulate_keystrokes("space"); // Toggle completion

    // Verify focused item
    todo_view.read_with(cx, |view, _| {
        assert_eq!(view.focused_index, Some(2));
        assert!(view.todo_list.read(cx).todos[2].complete);
    });
}
```

### Testing Drag and Drop

```rust
#[gpui::test]
fn test_todo_drag_reorder(cx: &mut TestAppContext) {
    let (todo_view, cx) = cx.add_window_view(|cx| {
        TodoListView::new(cx.new(|cx| TodoList::new("Drag Test".to_string(), cx)), cx)
    });

    // Add todos
    todo_view.update(cx, |view, cx| {
        view.todo_list.update(cx, |list, cx| {
            list.add_todo("Task A".to_string(), cx);
            list.add_todo("Task B".to_string(), cx);
            list.add_todo("Task C".to_string(), cx);
        });
    });

    // Simulate drag from first to last position
    cx.simulate_event(DragEvent::Start {
        position: point(50.0, 100.0), // First todo position
        item_id: 0,
    });

    cx.simulate_event(DragEvent::Move {
        position: point(50.0, 300.0), // Third todo position
    });

    cx.simulate_event(DragEvent::End {
        position: point(50.0, 300.0),
    });

    // Verify reordering
    todo_view.read_with(cx, |view, cx| {
        let todos = &view.todo_list.read(cx).todos;
        assert_eq!(todos[0].label, "Task B");
        assert_eq!(todos[1].label, "Task C");
        assert_eq!(todos[2].label, "Task A");
    });
}
```

### Testing Context Menus

```rust
#[gpui::test]
async fn test_todo_context_menu(cx: &mut TestAppContext) {
    let (todo_view, cx) = cx.add_window_view(|cx| {
        TodoListView::new(cx.new(|cx| TodoList::new("Menu Test".to_string(), cx)), cx)
    });

    // Add a todo
    todo_view.update(cx, |view, cx| {
        view.todo_list.update(cx, |list, cx| {
            list.add_todo("Right-click me".to_string(), cx);
        });
    });

    // Simulate right-click on todo
    cx.simulate_event(MouseDownEvent {
        button: MouseButton::Right,
        position: point(100.0, 150.0),
        modifiers: Modifiers::default(),
        click_count: 1,
    });

    // Verify context menu is shown
    todo_view.read_with(cx, |view, _| {
        assert!(view.context_menu.is_some());
        let menu = view.context_menu.as_ref().unwrap();
        assert!(menu.has_item("Edit"));
        assert!(menu.has_item("Delete"));
        assert!(menu.has_item("Duplicate"));
    });

    // Select menu item
    cx.simulate_keystrokes("down"); // Select "Edit"
    cx.simulate_keystrokes("enter"); // Activate

    // Verify edit mode
    todo_view.read_with(cx, |view, _| {
        assert_eq!(view.editing_todo_id, Some(0));
    });
}
```

## Test Best Practices

### 1. Test Isolation

Each test should be independent and not rely on shared state:

```rust
#[gpui::test]
async fn test_isolated_todo_operations(cx: &mut TestAppContext) {
    // Create fresh instances for each test
    let todo_list = cx.new(|cx| TodoList::new("Isolated".to_string(), cx));

    // Test operates on its own data
    todo_list.update(cx, |list, cx| {
        list.add_todo("Test-specific task".to_string(), cx);
    });

    // Clean up is automatic when test ends
}
```

### 2. Deterministic Testing

Avoid time-dependent or random behavior in tests:

```rust
#[gpui::test]
async fn test_deterministic_todo_ids(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| {
        let mut list = TodoList::new("Deterministic".to_string(), cx);
        // Use deterministic ID generation for tests
        list.id_generator = Box::new(TestIdGenerator::new());
        list
    });

    todo_list.update(cx, |list, cx| {
        let id1 = list.add_todo("Task 1".to_string(), cx);
        let id2 = list.add_todo("Task 2".to_string(), cx);

        // IDs should be predictable in tests
        assert_eq!(id1, TodoId(1));
        assert_eq!(id2, TodoId(2));
    });
}
```

### 3. Testing Edge Cases

Always test boundary conditions and error cases:

```rust
#[gpui::test]
async fn test_todo_list_limits(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| {
        let mut list = TodoList::new("Limited".to_string(), cx);
        list.max_items = 3;
        list
    });

    // Test at capacity
    todo_list.update(cx, |list, cx| {
        list.add_todo("Task 1".to_string(), cx);
        list.add_todo("Task 2".to_string(), cx);
        list.add_todo("Task 3".to_string(), cx);
        assert_eq!(list.todos.len(), 3);
    });

    // Test over capacity
    todo_list.update(cx, |list, cx| {
        let result = list.try_add_todo("Task 4".to_string(), cx);
        assert!(result.is_err());
        assert_eq!(list.todos.len(), 3); // Should not exceed limit
    });

    // Test empty list operations
    let empty_list = cx.new(|cx| TodoList::new("Empty".to_string(), cx));
    empty_list.update(cx, |list, _| {
        assert!(list.complete_all().is_err());
        assert!(list.delete_todo(TodoId(999)).is_err());
    });
}
```

### 4. Async Test Patterns

Properly handle async operations in tests:

```rust
#[gpui::test]
async fn test_concurrent_todo_operations(cx: &mut TestAppContext) {
    let todo_list = cx.new(|cx| TodoList::new("Concurrent".to_string(), cx));

    // Spawn multiple concurrent operations
    let task1 = todo_list.update(cx, |list, cx| {
        list.import_from_file("todos1.json", cx)
    });

    let task2 = todo_list.update(cx, |list, cx| {
        list.import_from_file("todos2.json", cx)
    });

    // Wait for both to complete
    let (result1, result2) = futures::join!(task1, task2);

    assert!(result1.is_ok());
    assert!(result2.is_ok());

    // Verify final state is consistent
    todo_list.read_with(cx, |list, _| {
        assert!(list.is_consistent());
    });
}
```
