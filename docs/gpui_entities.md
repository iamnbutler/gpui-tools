# GPUI Entity Patterns

This document contains diverse examples of entity creation, manipulation, and lifecycle patterns in GPUI. Original code, unless noted.

## Source Attribution

Some examples in this document are derived from the [Zed editor](https://github.com/zed-industries/zed) codebase.

- Examples marked with `(GPL-3.0)` are from GPL-licensed crates (e.g., `assistant2`, `repl`)
- Examples marked with `(Apache-2.0)` are from Apache-licensed crates (e.g., `gpui`, `editor`, `workspace`)
- Please respect the original licensing when using these patterns

## Table of Contents

1. [Entity Creation Patterns](#entity-creation-patterns)
2. [Entity Reading Patterns](#entity-reading-patterns)
3. [Entity Updating Patterns](#entity-updating-patterns)
4. [WeakEntity Patterns](#weakentity-patterns)
5. [Entity Subscriptions](#entity-subscriptions)
6. [Entity Observations](#entity-observations)
7. [Entity Lifecycle](#entity-lifecycle)
8. [EventEmitter Patterns](#eventemitter-patterns)
9. [Entity Passing Patterns](#entity-passing-patterns)
10. [Complex Entity Patterns](#complex-entity-patterns)

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
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Using read_with from async context
let snapshot = buffer.read_with(cx, |buffer, _| buffer.text_snapshot())?;
```

### read_with with complex logic

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Complex read_with operation
let tool_call_results = self.acp_thread.read_with(cx, |this, cx| {
thread.read_with(cx, |thread, cx| {
    let term = thread.terminal(terminal_id.clone()).unwrap();
    term.read_with(cx, |t, cx| t.inner().read(cx).get_content())
})
```

### Read for extracting multiple values

```rust
// Source: zed/crates/assistant2/src/action_log.rs (GPL-3.0)
// Reading multiple fields from entity
let operation = action_log.read_with(&cx, |this, cx| {
let (diff, language, language_registry) = this.read_with(cx, |this, cx| {
    let tracked_buffer = this
        .tracked_buffers
        .get(buffer)
        .context("buffer not tracked")?;
    anyhow::Ok((
        tracked_buffer.diff.clone(),
        buffer.read(cx).language().cloned(),
        buffer.read(cx).language_registry(),
    ))
})??;
```

## Entity Updating Patterns

### Simple update pattern

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Basic update
self.next_entry.update(cx, |next_entry, cx| {
self.label.update(cx, |label, cx| {
    if let Some((first_line, _)) = title.split_once("\n") {
        label.replace(first_line.to_owned() + "…", cx)
    } else {
        label.replace(title, cx);
    }
});
```

### Update with return value

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Update that returns a value
let completion_task = self.update(cx, |this, cx| {
let buffer = project.update(cx, |project, cx| {
    project
        .project_path_for_absolute_path(&location.path, cx)
        .map(|path| project.open_buffer(path, cx))
})?
.await?;
```

### Nested entity updates

```rust
// Source: zed/crates/assistant2/src/action_log.rs (GPL-3.0)
// Updating nested entities
self.buffer.update(cx, |buffer, cx| {
entity.update(cx, |term, cx| {
    term.inner().update(cx, |inner, cx| {
        inner.write_output(&data, cx);
    })
});
```

### update_in for window context

```rust
// From thread_view.rs - Using update_in with window context
this.update_in(cx, |this, window, cx| {
    this.send_to_thread(contents, tracked_buffers, window, cx);
})?;
```

### Conditional update pattern

```rust
// From thread_view.rs - Conditional update
this.update_in(cx, |this, window, cx| {
    if this.editor.read(cx).text(cx).trim().is_empty() {
        return;
    }
    this.send_to_thread(window, cx);
})?;
```

### Update from async context

```rust
// From acp_thread.rs - Async entity update pattern
cx.spawn(async move |this, cx| {
    let resolved_locations = task.await;
    this.update(cx, |this, cx| {
        let project = this.project.clone();
        let Some((ix, tool_call)) = this.tool_call_mut(&id) else {
            return;
        };
        // Update state...
    })
})
```

## WeakEntity Patterns

### Basic WeakEntity creation

```rust
// Source: zed/crates/assistant2/src/agent.rs (GPL-3.0)
// Creating WeakEntity with downgrade
struct Session {
    thread: Entity<Thread>,
    acp_thread: WeakEntity<acp_thread::AcpThread>,
    // other fields...
}

// In usage:
acp_thread: acp_thread.downgrade(),
```

### WeakEntity in async functions

```rust
// Source: zed/crates/assistant2/src/action_log.rs (GPL-3.0)
// WeakEntity as function parameter
async fn maintain_diff(
    this: WeakEntity<Self>,
    buffer: Entity<Buffer>,
    mut buffer_updates: mpsc::UnboundedReceiver<(ChangeAuthor, text::BufferSnapshot)>,
    cx: &mut AsyncApp,
) -> Result<()> {
    // Implementation...
}
```

### WeakEntity update pattern

```rust
// Source: zed/crates/assistant2/src/thread.rs (GPL-3.0)
// Updating through WeakEntity
async fn run_turn_internal(
    this: &WeakEntity<Self>,
    model: Arc<dyn LanguageModel>,
    event_stream: &ThreadEventStream,
    cx: &mut AsyncApp,
) -> Result<()> {
    this.update(cx, |this, cx| {
        // Update logic that may fail if entity is released
    })?;
    Ok(())
}
```

### WeakEntity in struct fields

```rust
// Source: zed/crates/assistant2/src/tools/edit_file_tool.rs (GPL-3.0)
// Storing WeakEntity for lifecycle safety
pub struct EditFileTool {
    thread: WeakEntity<Thread>,
    language_registry: Arc<LanguageRegistry>,
    project: Entity<Project>,
    templates: Arc<Templates>,
}
```

### WeakEntity with AgentLocation pattern

```rust
// Source: zed/crates/assistant2/src/edit_agent.rs (GPL-3.0)
// Creating location with WeakEntity buffer
Some(AgentLocation {
    buffer: buffer.downgrade(),
    position: buffer.read_with(cx, |buffer, _| buffer.anchor_before(Point::new(0, 3)))
})
```

## Entity Subscriptions

### Basic subscription pattern

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Simple subscription
cx.subscribe(&thread, move |_thread, _event_thread, event: &AcpThreadEvent, _cx| {
    if matches!(event, AcpThreadEvent::Refusal) {
        *saw_refusal_event_captured.lock().unwrap() = true;
    }
})
.detach();
```

### Subscription with entity update

```rust
// Source: zed/crates/assistant2/src/action_log.rs (GPL-3.0)
// Subscription that updates entity
_subscription: cx.subscribe(&buffer, Self::handle_buffer_event),
```

### Multiple subscriptions pattern

```rust
// Source: zed/crates/assistant2/src/agent.rs (GPL-3.0)
// Managing multiple subscriptions
let subscriptions = vec![
    cx.observe_release(&acp_thread, |this, acp_thread, _cx| {
        this.sessions.remove(acp_thread.session_id());
    }),
    cx.subscribe(&thread_handle, Self::handle_thread_title_updated),
    cx.subscribe(&thread_handle, Self::handle_thread_token_usage_updated),
    cx.observe(&thread_handle, move |this, thread, cx| {
        this.save_thread(thread, cx)
    }),
];
```

### Subscription with closure capture

```rust
// Source: zed/crates/assistant2/src/agent_panel.rs (GPL-3.0)
// Subscription capturing entities
cx.subscribe(
    &LanguageModelRegistry::global(cx),
    |_, event: &language_model::Event, cx| match event {
        language_model::Event::ProviderStateChanged(_)
        | language_model::Event::AddedProvider(_)
        | language_model::Event::RemovedProvider(_) => {
            update_active_language_model_from_settings(cx);
        }
        _ => {}
    },
)
.detach();
```

## Entity Observations

### Basic observe pattern

```rust
// From diff.rs - Simple observation
_subscription: cx.observe(&buffer, |this, _, cx| {
    if let Diff::Pending(diff) = this {
        diff.update(cx);
    }
}),
```

### Observe with notification

```rust
// From acp_tools.rs - Observe and notify
let subscription = cx.observe(&connection_registry, |this, _, cx| {
    this.update_connection(cx);
    cx.notify();
});
```

### observe_release pattern

```rust
// Source: zed/crates/assistant2/src/agent.rs (GPL-3.0)
// Release observation
cx.observe_release(&acp_thread, |this, acp_thread, _cx| {
    this.sessions.remove(acp_thread.session_id());
}),
```

### Complex observe_release with buffer management

```rust
// From copilot.rs - Complex cleanup on release
cx.observe_release(buffer, move |this, _buffer, _cx| {
    this.buffers.remove(&weak_buffer);
    this.unregister_buffer(&weak_buffer);
}),
```

### Observe global state

```rust
// From text_thread_editor.rs - Observing global settings
cx.observe_global_in::<SettingsStore>(window, Self::settings_changed),
```

## Entity Lifecycle

### Entity with subscription cleanup

```rust
// Source: zed/crates/assistant2/src/agent_panel.rs (GPL-3.0)
// Managing subscriptions lifecycle
struct AgentPanel {
    _subscriptions: Vec<Subscription>,
    // other fields...
}

impl AgentPanel {
    fn new(cx: &mut Context<Self>) -> Self {
        let _subscriptions = vec![
            cx.subscribe(&some_entity, Self::handle_event),
            cx.observe(&other_entity, |_, _, cx| cx.notify()),
        ];

        Self {
            _subscriptions,
            // other fields...
        }
    }
}
```

### Task::ready for immediate entity operations

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Immediate entity task
pub fn cancel(&mut self, cx: &mut Context<Self>) -> Task<()> {
    let Some(send_task) = self.send_task.take() else {
        return Task::ready(());
    };
    // Continue with cancellation...
}
```

### Entity creation guard pattern

```rust
// Source: zed/crates/assistant2/src/thread_view.rs (GPL-3.0)
// Guard entity for lifecycle management
let guard = cx.new(|_| ());
cx.observe_release(&guard, |this, _guard, cx| {
    this.is_loading_contents = false;
    cx.notify();
})
.detach();
```

## EventEmitter Patterns

### Basic EventEmitter implementation

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Simple event emitter
impl EventEmitter<AcpThreadEvent> for AcpThread {}

#[derive(Debug, Clone)]
pub enum AcpThreadEvent {
    TitleUpdated,
    StreamedCompletion,
    // other events...
}
```

### Emitting events

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Emitting various events
cx.emit(AcpThreadEvent::TitleUpdated);
cx.emit(AcpThreadEvent::EntryUpdated(idx));
cx.emit(AcpThreadEvent::StreamedCompletion);
```

### Multiple event types for single entity

```rust
// Source: zed/crates/assistant2/src/thread.rs (GPL-3.0)
// Multiple EventEmitter implementations
pub struct TokenUsageUpdated(pub Option<acp_thread::TokenUsage>);
impl EventEmitter<TokenUsageUpdated> for Thread {}

pub struct TitleUpdated;
impl EventEmitter<TitleUpdated> for Thread {}
```

### Event with payload

```rust
// Source: zed/crates/assistant2/src/entry_view_state.rs (GPL-3.0)
// Event with data
pub struct EntryViewEvent {
    pub entry_index: usize,
    pub event: EditorEvent,
}

impl EventEmitter<EntryViewEvent> for EntryViewState {}
```

## Entity Passing Patterns

### Passing entity to functions

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Entity as parameter
pub fn open(
    channel_id: ChannelId,
    workspace: &Entity<Workspace>,
    window: &mut Window,
    cx: &mut App,
) -> Task<Result<Entity<Self>>> {
    // Implementation...
}
```

### Cloning entities for async operations

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Cloning for async
let project = self.project.clone();
let action_log = self.action_log.clone();
cx.spawn(async move |this, cx| {
    // Use cloned entities in async context
    project.update(cx, |project, cx| {
        // Project operations...
    })
})
```

### Entity in return types

```rust
// Source: zed/crates/assistant2/src/diff.rs (GPL-3.0)
// Returning new entity
pub fn new(buffer: Entity<Buffer>, cx: &mut Context<Self>) -> Self {
    let buffer_diff = cx.new(|cx| {
        // Create and return new entity
    });

    Self::Pending(PendingDiff {
        diff: buffer_diff,
        // other fields...
    })
}
```

### Storing entities in collections

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Entity collections
struct AcpThread {
    terminals: HashMap<TerminalId, Entity<Terminal>>,
    shared_buffers: HashMap<BufferId, Entity<Buffer>>,
    // other fields...
}
```

## Complex Entity Patterns

### Entity with async initialization

```rust
// Source: zed/crates/assistant2/src/channel_view.rs (GPL-3.0)
// Complex async entity loading
pub fn load(
    channel_id: ChannelId,
    workspace: &Entity<Workspace>,
    window: &mut Window,
    cx: &mut App,
) -> Task<Result<Entity<Self>>> {
    let channel_buffer =
        channel_store.update(cx, |store, cx| store.open_channel_buffer(channel_id, cx));

    window.spawn(cx, async move |cx| {
        let channel_buffer = channel_buffer.await?;
        let markdown = markdown.await;

        cx.update(|window, cx| {
            cx.new(|cx| {
                Self::new(
                    workspace,
                    channel_buffer,
                    markdown.ok(),
                    cx,
                )
            })
        })
    })
}
```

### Entity factory pattern

```rust
// Source: zed/crates/assistant2/src/agent.rs (GPL-3.0)
// Factory with environment injection
let thread_handle = cx.new(|cx| {
    Thread::new(
        id.clone(),
        model,
        tools,
        Rc::new(AcpThreadEnvironment {
            acp_thread: acp_thread.downgrade(),
        }) as _,
        cx,
    )
});
```

### Cross-entity communication pattern

```rust
// Source: zed/crates/assistant2/src/acp_tools.rs (GPL-3.0)
// Entity communication via channels
let mut receiver = connection.subscribe();
let task = cx.spawn(async move |this, cx| {
    while let Ok(message) = receiver.recv().await {
        this.update(cx, |this, cx| {
            this.push_stream_message(message, cx);
        })
        .ok(); // Continue even if update fails
    }
});
```

### Entity delegation pattern

```rust
// Source: zed/crates/assistant2/src/agent_configuration.rs (GPL-3.0)
// Delegating to context server store
cx.subscribe(&context_server_store, |_, _, _, cx| cx.notify())
    .detach();

// Later in render:
context_server_store.update(cx, |store, cx| {
    store.register_server(server_id, cx)
})
```

### Entity state machine pattern

```rust
// Source: zed/crates/assistant2/src/diff.rs (GPL-3.0)
// Entity with state transitions
enum Diff {
    Pending(PendingDiff),
    Finalized(FinalizedDiff),
}

impl Diff {
    pub fn finalized(...) -> Self {
        // Create in finalized state
        Self::Finalized(FinalizedDiff { ... })
    }

    pub fn new(buffer: Entity<Buffer>, cx: &mut Context<Self>) -> Self {
        // Create in pending state
        Self::Pending(PendingDiff { ... })
    }
}
```

### Entity registry pattern

```rust
// Source: zed/crates/assistant2/src/context_server_registry.rs (GPL-3.0)
// Registry managing entities
struct ContextServerRegistry {
    server_store: Entity<ContextServerStore>,
    registered_servers: HashMap<ContextServerId, RegisteredServer>,
    _subscription: Subscription,
}

impl ContextServerRegistry {
    pub fn new(
        server_store: Entity<ContextServerStore>,
        cx: &mut Context<Self>,
    ) -> Self {
        let mut this = Self {
            server_store: server_store.clone(),
            registered_servers: HashMap::default(),
            _subscription: cx.subscribe(&server_store, Self::handle_context_server_store_event),
        };

        this.register_existing_servers(cx);
        this
    }
}
```

## Key Patterns Summary

### Pattern: Entity creation context

```rust
// In Context<T>:
let entity = cx.new(|cx| SomeType::new(cx));

// In Window or App context:
cx.update(|window, cx| {
    cx.new(|cx| SomeType::new(cx))
})
```

### Pattern: Safe entity access from async

```rust
// Strong entity - will keep entity alive
entity.update(cx, |entity, cx| { /* ... */ })?

// Weak entity - may fail if released
weak_entity.update(cx, |entity, cx| { /* ... */ })?
```

### Pattern: Entity lifecycle management

```rust
// Store subscriptions to keep them alive
_subscriptions: Vec<Subscription>

// Use observe_release for cleanup
cx.observe_release(&entity, |this, entity, _| {
    // Cleanup code
})
```

### Pattern: Cross-context entity access

- `.read(cx)` - synchronous read in same context
- `.read_with(cx, |e, cx| ...)` - read from async context
- `.update(cx, |e, cx| ...)` - update in same context
- `.update_in(cx, |e, window, cx| ...)` - update with window context

### Pattern: Entity communication

- Use `EventEmitter` for events
- Use channels for streaming data
- Use subscriptions for reactive updates
- Use observations for state synchronization
