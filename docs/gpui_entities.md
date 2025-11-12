# GPUI Entity Patterns

This document contains diverse examples of entity creation, manipulation, and lifecycle patterns in GPUI, extracted from the Zed codebase.

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
// From acp_thread.rs - Simple entity creation
let content = cx.new(|cx| Markdown::new(entry.content.into(), None, None, cx));
```

### Entity creation with complex initialization

```rust
// From diff.rs - Creating entity with multiple nested entities
let multibuffer = cx.new(|_cx| MultiBuffer::without_headers(Capability::ReadOnly));
let new_buffer = cx.new(|cx| Buffer::local(new_text, cx));

let buffer_diff = cx.new(|cx| {
    let mut diff = BufferDiff::new_unchanged(&buffer_text_snapshot, base_text_snapshot);
    let snapshot = diff.snapshot(cx);
    let secondary_diff = cx.new(|cx| {
        let mut diff = BufferDiff::new(&buffer_text_snapshot, cx);
        diff.set_snapshot(snapshot, &buffer_text_snapshot, cx);
        diff
    });
    diff.set_secondary_diff(secondary_diff);
    diff
});
```

### Entity creation as part of struct initialization

````rust
// From terminal.rs - Entity creation in struct field
Self {
    id,
    command: cx.new(|cx| {
        Markdown::new(
            format!("```\n{}\n```", command_label).into(),
            Some(language_registry.clone()),
            None,
            cx,
        )
    }),
    working_dir,
    // other fields...
}
````

## Entity Reading Patterns

### Simple read pattern

```rust
// From acp_thread.rs - Basic entity read
let source = self.label.read(cx).source();
```

### Read with nested entity access

```rust
// From acp_thread.rs - Reading nested entities
let buffer_id = channel_view.read(cx).channel_buffer.read(cx).remote_id(cx);
```

### Conditional read pattern

```rust
// From acp_thread.rs - Conditional entity read
if let Some(last_entry) = self.entries.last_mut()
    && let AgentThreadEntry::UserMessage(UserMessage { content, .. }) = last_entry
{
    let language_registry = self.project.read(cx).languages().clone();
    // Use the read data...
}
```

### read_with for async contexts

```rust
// From acp_thread.rs - Using read_with from async context
let snapshot = buffer.read_with(cx, |buffer, _| buffer.text_snapshot())?;
```

### read_with with complex logic

```rust
// From acp_thread.rs - Complex read_with usage
thread.read_with(cx, |thread, cx| {
    let term = thread.terminal(terminal_id.clone()).unwrap();
    term.read_with(cx, |t, cx| t.inner().read(cx).get_content())
})
```

### Read for extracting multiple values

```rust
// From action_log.rs - Reading multiple values from entity
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
// From acp_thread.rs - Basic entity update
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
// From acp_thread.rs - Update returning a value
let buffer = project.update(cx, |project, cx| {
    project
        .project_path_for_absolute_path(&location.path, cx)
        .map(|path| project.open_buffer(path, cx))
})?
.await?;
```

### Nested entity updates

```rust
// From acp_thread.rs - Updating nested entities
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
// From agent.rs - Creating WeakEntity with downgrade
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
// From action_log.rs - WeakEntity as function parameter
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
// From thread.rs - Updating through WeakEntity
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
// From edit_file_tool.rs - Storing WeakEntity for lifecycle safety
pub struct EditFileTool {
    thread: WeakEntity<Thread>,
    language_registry: Arc<LanguageRegistry>,
    project: Entity<Project>,
    templates: Arc<Templates>,
}
```

### WeakEntity with AgentLocation pattern

```rust
// From edit_agent.rs - Creating location with WeakEntity buffer
Some(AgentLocation {
    buffer: buffer.downgrade(),
    position: buffer.read_with(cx, |buffer, _| buffer.anchor_before(Point::new(0, 3)))
})
```

## Entity Subscriptions

### Basic subscription pattern

```rust
// From acp_thread.rs - Simple subscription
cx.subscribe(&thread, move |_thread, _event_thread, event: &AcpThreadEvent, _cx| {
    if matches!(event, AcpThreadEvent::Refusal) {
        *saw_refusal_event_captured.lock().unwrap() = true;
    }
})
.detach();
```

### Subscription with entity update

```rust
// From action_log.rs - Subscription that updates entity
_subscription: cx.subscribe(&buffer, Self::handle_buffer_event),
```

### Multiple subscriptions pattern

```rust
// From agent.rs - Managing multiple subscriptions
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
// From agent_ui.rs - Subscription with captured state
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
// From agent.rs - Cleanup on entity release
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
// From agent_panel.rs - Entity with _subscriptions field
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
// From acp_thread.rs - Immediate entity task
pub fn cancel(&mut self, cx: &mut Context<Self>) -> Task<()> {
    let Some(send_task) = self.send_task.take() else {
        return Task::ready(());
    };
    // Continue with cancellation...
}
```

### Entity creation guard pattern

```rust
// From thread_view.rs - Guard entity for lifecycle management
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
// From acp_thread.rs - Simple event emitter
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
// From acp_thread.rs - Emitting various events
cx.emit(AcpThreadEvent::TitleUpdated);
cx.emit(AcpThreadEvent::EntryUpdated(idx));
cx.emit(AcpThreadEvent::StreamedCompletion);
```

### Multiple event types for single entity

```rust
// From thread.rs - Multiple EventEmitter implementations
pub struct TokenUsageUpdated(pub Option<acp_thread::TokenUsage>);
impl EventEmitter<TokenUsageUpdated> for Thread {}

pub struct TitleUpdated;
impl EventEmitter<TitleUpdated> for Thread {}
```

### Event with payload

```rust
// From entry_view_state.rs - Event with data
pub struct EntryViewEvent {
    pub entry_index: usize,
    pub event: EditorEvent,
}

impl EventEmitter<EntryViewEvent> for EntryViewState {}
```

## Entity Passing Patterns

### Passing entity to functions

```rust
// From acp_thread.rs - Entity as parameter
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
// From acp_thread.rs - Cloning for async
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
// From diff.rs - Returning new entity
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
// From acp_thread.rs - HashMap of entities
struct AcpThread {
    terminals: HashMap<acp::TerminalId, Entity<Terminal>>,
    shared_buffers: HashMap<Entity<Buffer>, BufferSnapshot>,
    // other fields...
}
```

## Complex Entity Patterns

### Entity with async initialization

```rust
// From channel_view.rs - Complex async entity loading
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
// From agent.rs - Factory with environment injection
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
// From acp_tools.rs - Entity communication via channels
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
// From agent_configuration.rs - Delegating to context server store
cx.subscribe(&context_server_store, |_, _, _, cx| cx.notify())
    .detach();

// Later in render:
context_server_store.update(cx, |store, cx| {
    store.register_server(server_id, cx)
})
```

### Entity state machine pattern

```rust
// From diff.rs - Entity with state transitions
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
// From context_server_registry.rs - Registry managing entities
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
