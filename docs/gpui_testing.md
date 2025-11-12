## Testing

### Test Macros

There are two ways to write tests in gpui:

#### Standard Rust Tests

Use `#[test]` for synchronous unit tests that don't require GPUI context:

```rust
#[test]
fn test_basic_logic() {
    assert_eq!(2 + 2, 4);
}
```

#### GPUI Tests

Use `#[gpui::test]` for tests requiring GPUI context and async operations:

```rust
#[gpui::test]
async fn test_entity_updates(cx: &mut gpui::TestAppContext) {
    let entity = cx.new(|_| MyEntity::default());
    entity.update(cx, |entity, cx| {
        entity.process();
        cx.notify();
    });
}
```

**CRITICAL**: Never import `test` from gpui at module level. Always use fully qualified paths:

```rust
// WRONG - Never do this:
use gpui::test;

// CORRECT - Always use fully qualified:
#[gpui::test]
async fn my_test(cx: &mut gpui::TestAppContext) {
    // test implementation
}
```

### Test Contexts

#### TestAppContext

Provides test-specific methods for GPUI applications:

- `cx.new(|cx| ...)` - Create entities
- `cx.update(|cx| ...)` - Update global state
- `cx.spawn(...)` - Spawn tasks
- `cx.run_until_parked()` - Run all pending tasks
- `cx.simulate_keystrokes(text)` - Simulate keyboard input

#### VisualTestContext

For testing UI components with window-specific operations:

```rust
#[gpui::test]
fn test_ui_component(cx: &mut TestAppContext) {
    let (view, cx) = cx.add_window_view(|cx| MyView::new(cx));

    // cx is now a &mut VisualTestContext
    cx.draw();
    cx.simulate_click(point(10.0, 10.0));
    cx.simulate_event(ScrollWheelEvent { ... });
}
```

### Common Testing Patterns

#### Init Test Helper

Define an `init_test` function to set up common test dependencies:

```rust
fn init_test(cx: &mut TestAppContext) {
    env_logger::try_init().ok();
    cx.update(|cx| {
        let settings_store = SettingsStore::test(cx);
        cx.set_global(settings_store);
        Project::init_settings(cx);
        language::init(cx);
    });
}

#[gpui::test]
async fn test_feature(cx: &mut TestAppContext) {
    init_test(cx);
    // test implementation
}
```

#### Testing Async Operations

```rust
#[gpui::test]
async fn test_async_entity(cx: &mut TestAppContext) {
    let entity = cx.new(|_| MyEntity::default());

    // Wait for next event
    let event = entity.next_event::<MyEvent>(cx).await;

    // Wait for condition
    entity.condition(cx, |entity, _| entity.is_ready()).await;

    // Wait for notification
    entity.next_notification(Duration::from_millis(100), cx).await;
}
```

#### Testing Entity Updates

```rust
#[gpui::test]
async fn test_entity_behavior(cx: &mut TestAppContext) {
    let entity = cx.new(|_| Counter { value: 0 });

    entity.update(cx, |counter, cx| {
        counter.value += 1;
        cx.notify();
    });

    assert_eq!(entity.read(cx).value, 1);
}
```

#### Testing Task Management

```rust
#[gpui::test]
async fn test_task_spawning(cx: &mut TestAppContext) {
    let entity = cx.new(|_| MyEntity::default());

    entity.update(cx, |entity, cx| {
        cx.spawn(async move |this, cx| {
            // Async work
            this.update(&cx, |this, cx| {
                this.process_result();
                cx.notify();
            })?;
            Ok(())
        }).detach();
    });

    cx.run_until_parked();
}
```

### Test Utilities

#### Simulating User Input

```rust
// Keyboard input
cx.simulate_keystrokes("hello world");
cx.dispatch_action(window, &MyAction);

// Mouse input
cx.simulate_click(point(x, y));
cx.simulate_event(MouseMoveEvent { position, ... });

// Scroll events
cx.simulate_event(ScrollWheelEvent { delta, ... });
```

#### Testing with Multiple Windows

```rust
#[gpui::test]
fn test_multiple_windows(cx: &mut TestAppContext) {
    let window1 = cx.add_empty_window();
    let window2 = cx.add_empty_window();

    window1.update(|window, cx| {
        // Window 1 operations
    });

    window2.update(|window, cx| {
        // Window 2 operations
    });
}
```
