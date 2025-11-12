# GPUI Task Examples

This document contains diverse examples of task spawning and entity update patterns in GPUI.

## Source Attribution

Examples in this document are derived from the [Zed editor](https://github.com/zed-industries/zed) codebase.

- Examples marked with `(GPL-3.0)` are from GPL-licensed crates (e.g., `assistant2`, `repl`)
- Examples marked with `(Apache-2.0)` are from Apache-licensed crates (e.g., `gpui`, `editor`, `workspace`)
- Please respect the original licensing when using these patterns

## Table of Contents

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
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Basic entity update after async operation
cx.spawn(async move |this, cx| {
    let resolved_locations = task.await;
    this.update(cx, |this, cx| {
        let project = this.project.clone();
        let Some((ix, tool_call)) = this.tool_call_mut(&id) else {
            return;
        };
        // ... update state
    })
})
```

### Spawn with WeakEntity handle

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Using WeakEntity in spawn
cx.spawn(async move |handle, cx| {
    let new_checkpoint = new_checkpoint.await;
    handle.update(cx, |this, cx| {
        this.update_last_checkpoint(new_checkpoint, cx);
    })
})
```

### Spawn returning a value

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Returning value from spawn
project.update(cx, |_, cx| {
    cx.spawn(async move |project, cx| {
        let mut new_locations = Vec::new();
        for location in locations {
            new_locations.push(Self::resolve_location(location, project.clone(), cx).await);
        }
        new_locations
    })
})
```

### Spawn with async move pattern (CORRECT)

```rust
// Source: zed/crates/assistant2/src/diff.rs (GPL-3.0)
// Correct async move syntax
self.update_diff = cx.spawn(async move |diff, cx| {
    let text_snapshot = buffer.read_with(cx, |buffer, _| buffer.text_snapshot())?;
    let diff_snapshot = BufferDiff::update_diff(
        buffer_diff.clone(),
        text_snapshot.clone(),
        Some(base_text),
        false,
    ).await?;

    diff.update(cx, |diff, cx| {
        diff.apply_diff_snapshot(diff_snapshot, cx);
    })?;

    Ok(())
});
```

## Window Spawn Patterns

### Basic window.spawn

```rust
// Source: zed/crates/assistant2/src/thread_view.rs (GPL-3.0)
// Window spawn with entity update
window.spawn(cx, async move |cx| {
    let markdown_language = markdown_language_task.await?;
    // Work with the result
    Ok(())
})
```

### Window spawn with workspace operations

```rust
// Source: zed/crates/editor/src/editor.rs (Apache-2.0)
// Opening editor at anchor
window.spawn(cx, async move |cx| {
    let Some(editor) = item.await?.downcast::<Editor>() else {
        return Err(anyhow::anyhow!("Expected editor"));
    };
    editor.update(cx, |editor, cx| {
        editor.go_to_range(target..target, window, cx);
    })?;
    Ok(())
})
```

### Window spawn for deserialization

```rust
// Source: zed/crates/editor/src/items.rs (Apache-2.0)
// Deserializing editor state
window.spawn(cx, async move |cx| {
    let language_registry =
        project.read_with(cx, |project, _| project.languages().clone())?;

    let language = if let Some(language_name) = language {
        let language_task = language_registry.language_for_name(language_name);
        Some(language_task.await?)
    } else {
        None
    };

    let buffer = project
        .update(cx, |project, cx| {
            project.create_buffer_with_language(language, cx)
        })?
        .await?;

    cx.update(|window, cx| {
        cx.new(|cx| {
            let mut editor = Editor::for_buffer(buffer, Some(project), cx);
            editor.read_scroll_position_from_db(item_id, workspace_id, cx);
            editor
        })
    })
})
```

## cx.spawn_in Patterns

### spawn_in with workspace update

```rust
// Source: zed/crates/activity_indicator/src/activity_indicator.rs (Apache-2.0)
// Creating buffer and updating UI
cx.spawn_in(window, async move |workspace, cx| {
    let buffer = create_buffer.await?;
    buffer.update(cx, |buffer, cx| {
        buffer.edit(
            [(0..0, format!("Language server {server_name}:\n\n{status}"))],
            None,
            cx,
        );
    })?;

    workspace.update_in(cx, |workspace, window, cx| {
        let project = workspace.project().clone();
        let buffer = cx.new(|cx| {
            MultiBuffer::singleton(buffer, cx).with_title(title)
        });
        workspace.add_item_to_active_pane(
            Box::new(cx.new(|cx| Editor::for_multibuffer(buffer, Some(project), true, cx))),
            None,
            true,
            window,
            cx,
        );
    })?;
    Ok(())
})
```

### spawn_in for async UI updates

```rust
// Source: zed/crates/assistant2/src/thread_view.rs (GPL-3.0)
// Authentication with UI feedback
self.auth_task = Some(cx.spawn_in(window, {
    let agent = self.agent.clone();
    async move |this, cx| {
        let result = authenticate.await;

        match &result {
            Ok(_) => telemetry::event!(
                "Authenticate Agent Succeeded",
                agent = agent.telemetry_id()
            ),
            Err(error) => telemetry::event!(
                "Authenticate Agent Failed",
                agent = agent.telemetry_id(),
                error_message = error.to_string()
            ),
        }

        this.update(cx, |this, cx| {
            this.auth_task = None;
            cx.notify();
        })?;

        result
    }
}));
```

### spawn_in with background work

```rust
// From tool_picker.rs - Background filtering with UI update
cx.spawn_in(window, async move |this, cx| {
    let filtered_items = cx
        .background_spawn(async move {
            let mut tools_by_provider: BTreeMap<Option<Arc<str>>, Vec<Arc<str>>> =
                BTreeMap::default();

            for item in all_items.iter() {
                if let PickerItem::Tool { server_id, name } = item.clone() {
                    // Filtering logic...
                }
            }

            tools_by_provider
        })
        .await;

    this.update(cx, |this, cx| {
        this.delegate.filtered = filtered_items;
        cx.notify();
    })
})
```

## Background Spawn Patterns

### Simple background computation

```rust
// From diff.rs - Background rope creation
let old_text_rope = cx
    .background_spawn({
        let old_text = old_text.clone();
        async move { Rope::from(old_text.as_str()) }
    })
    .await;
```

### Background spawn with complex computation

```rust
// From action_log.rs - Background diff computation
let rebase = cx.background_spawn({
    let mut base_text = tracked_buffer.diff_base.clone();
    let old_snapshot = tracked_buffer.snapshot.clone();
    let new_snapshot = buffer_snapshot.clone();
    let unreviewed_edits = tracked_buffer.unreviewed_edits.clone();

    async move {
        let mut unreviewed_edits = unreviewed_edits.into_iter().peekable();
        // Complex diff computation...
        Result::<_>::Ok((new_diff_base, new_unreviewed_edits))
    }
});
```

### Background spawn with streaming

```rust
// From edit_agent.rs - Streaming parser
let output = cx.background_spawn(async move {
    pin_mut!(chunks);

    let mut parser = EditParser::new(edit_format);
    let mut raw_edits = String::new();

    while let Some(chunk) = chunks.next().await {
        match chunk {
            Ok(chunk) => {
                raw_edits.push_str(&chunk);
                parser.push_str(&chunk);
                while let Some(event) = parser.next() {
                    tx.send(event).await.ok();
                }
            }
            Err(error) => {
                tx.send(EditParserEvent::Error(error)).await.ok();
                break;
            }
        }
    }

    raw_edits
});
```

### Background spawn for database operations

```rust
// From history_store.rs - Database operations
pub fn load_thread(&self, id: HistoryEntryId, cx: &mut App) -> Task<Result<Option<DbThread>>> {
    let database_future = ThreadsDatabase::connect(cx);
    cx.background_spawn(async move {
        let database = database_future.await.map_err(|err| anyhow!(err))?;
        database.load_thread(id).await
    })
}
```

## Entity Update Patterns

### Update from async context with error handling

```rust
// From thread_view.rs - Updating after async operation
cx.spawn_in(window, async move |this, cx| {
    let (contents, tracked_buffers) = contents.await?;

    this.update_in(cx, |this, window, cx| {
        this.send_to_thread(contents, tracked_buffers, window, cx);
    })?;

    anyhow::Ok(())
})
```

### Chained updates

```rust
// From acp_thread.rs - Multiple sequential updates
cx.spawn(async move |this, cx| {
    cx.update(|cx| truncate.run(id.clone(), cx))?.await?;
    this.update(cx, |this, cx| {
        if let Some((ix, _)) = this.user_message_mut(&id) {
            let range = ix..this.entries.len();
            this.entries.truncate(ix);
            cx.emit(AcpThreadEvent::StreamedCompletion);
        }
    })?;
    Ok(())
})
```

### Update with subscription

```rust
// From acp_tools.rs - Subscribing to stream updates
let task = cx.spawn(async move |this, cx| {
    while let Ok(message) = receiver.recv().await {
        this.update(cx, |this, cx| {
            this.push_stream_message(message, cx);
        })
        .ok(); // Continue even if update fails
    }
});
```

### Conditional update pattern

```rust
// From thread_view.rs - Conditional entity update
cx.spawn_in(window, async move |this, cx| {
    cancelled.await;

    this.update_in(cx, |this, window, cx| {
        if this.editor.read(cx).text(cx).trim().is_empty() {
            return;
        }
        this.send_to_thread(window, cx);
    })?;

    Ok(())
})
```

## Task Storage Patterns

### Task stored in struct field

```rust
// From diff.rs - Storing update task
struct PendingDiff {
    update_diff: Task<Result<()>>,
    // other fields...
}

impl PendingDiff {
    pub fn update(&mut self, cx: &mut Context<Diff>) {
        self.update_diff = cx.spawn(async move |diff, cx| {
            // Async work...
            Ok(())
        });
    }
}
```

### Optional task field

```rust
// From thread_view.rs - Optional authentication task
struct AcpThreadView {
    auth_task: Option<Task<Result<()>>>,
    // other fields...
}

impl AcpThreadView {
    fn authenticate(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        self.auth_task = Some(cx.spawn_in(window, async move |this, cx| {
            // Authentication logic...
            this.update(cx, |this, cx| {
                this.auth_task = None; // Clear task when done
                cx.notify();
            })?;
            Ok(())
        }));
    }
}
```

### Task collection

```rust
// From action_log.rs - Collecting multiple tasks
pub fn reject_all_edits(&self, cx: &mut App) -> Task<()> {
    let futures = self
        .tracked_buffers
        .iter()
        .filter_map(|(buffer, tracked_buffer)| {
            // Create tasks...
        })
        .collect::<Vec<_>>();

    let task = futures::future::join_all(futures);

    cx.spawn(async move |_, _| {
        task.await;
    })
}
```

## Error Handling Patterns

### detach_and_log_err pattern

```rust
// From thread_view.rs - Fire-and-forget with error logging
thread
    .update(cx, |thread, cx| {
        thread.set_title(new_title.into(), cx)
    })
    .detach_and_log_err(cx);
```

### Manual error logging with detach

```rust
// From agent.rs - Manual error handling before detach
cx.background_spawn(async move {
    if let acp::RequestPermissionOutcome::Selected { option_id } = outcome_task.await {
        response
            .send(option_id)
            .map(|_| anyhow!("authorization receiver was dropped"))
            .log_err();
    }
})
.detach();
```

### Propagating errors through tasks

```rust
// From thread_view.rs - Error propagation
window.spawn(cx, async move |cx| {
    let workspace = workspace.upgrade().context("workspace was released")?;
    let editor = editor.upgrade().context("editor was released")?;
    let range = editor.update(cx, |editor, cx| {
        editor.buffer().update(cx, |multibuffer, cx| {
            // Get range...
        })
    })?;
    // Continue processing...
    Ok(range)
})
```

### Task with fallback on error

```rust
// From action_log.rs - Fallback to Task::ready on error
buffer
    .read(cx)
    .entry_id(cx)
    .and_then(|entry_id| {
        self.project.update(cx, |project, cx| {
            project.delete_entry(entry_id, false, cx)
        })
    })
    .unwrap_or(Task::ready(Ok(())))
```

## Task Lifecycle Patterns

### Task::ready for immediate values

```rust
// From acp_thread.rs - Immediate task completion
pub fn cancel(&mut self, cx: &mut Context<Self>) -> Task<()> {
    let Some(send_task) = self.send_task.take() else {
        return Task::ready(());
    };
    // Continue with actual cancellation...
}
```

### Shared task pattern

```rust
// From acp_thread.rs - Sharing task result
let env = match worktree {
    Some(wt) => wt.update(cx, |wt, cx| {
        project.directory_environment(&shell, dir.as_path().into(), cx)
    }),
    None => Task::ready(None).shared(),
};
```

### Task with timer

```rust
// From acp_tools.rs - Timer-based task
cx.spawn(async move |this, cx| {
    cx.background_executor().timer(Duration::from_secs(2)).await;
    this.update(cx, |this, cx| {
        this.just_copied = false;
        cx.notify();
    })
})
.detach();
```

### Task with channel communication

```rust
// From acp_thread.rs - Using oneshot channel
let (tx, rx) = oneshot::channel();
let cancel_task = self.cancel(cx);

self.send_task = Some(cx.spawn(async move |this, cx| {
    cancel_task.await;
    tx.send(f(this, cx).await).ok();
}));

cx.spawn(async move |this, cx| {
    let response = rx.await;
    // Handle response...
})
```

## Complex Async Patterns

### Join multiple tasks

```rust
// From items.rs - Joining multiple buffer loads
Some(window.spawn(cx, async move |cx| {
    let mut buffers = futures::future::try_join_all(buffers?)
        .await
        .context("Failed to join buffers")?;
    // Process joined results...
    Ok(())
}))
```

### Nested spawns

```rust
// From model_selector.rs - Nested async operations
let refresh_models_task = {
    cx.spawn_in(window, {
        async move |this, cx| {
            async fn refresh(
                this: &WeakEntity<Picker<AcpModelPickerDelegate>>,
                cx: &mut AsyncWindowContext,
            ) -> Result<()> {
                let (models_task, selected_model_task) = this.update(cx, |this, cx| {
                    (
                        this.delegate.selector.list_models(cx),
                        this.delegate.selector.selected_model(cx),
                    )
                })?;

                let (models, selected_model) =
                    futures::join!(models_task, selected_model_task);

                // Update with results...
                Ok(())
            }

            refresh(&this, cx).await
        }
    })
};
```

### Stream processing pattern

```rust
// From edit_agent.rs - Processing async stream
let (tx, rx) = mpsc::unbounded();
let output = cx.background_spawn(async move {
    pin_mut!(chunks);

    let mut parser = EditParser::new(edit_format);
    let mut raw_edits = String::new();

    while let Some(chunk) = chunks.next().await {
        match chunk {
            Ok(chunk) => {
                raw_edits.push_str(&chunk);
                parser.push_str(&chunk);

                while let Some(event) = parser.next() {
                    tx.send(event).await.ok();
                }
            }
            Err(error) => {
                tx.send(EditParserEvent::Error(error)).await.ok();
                break;
            }
        }
    }

    raw_edits
});
```

### Task with retry logic

```rust
// From connection.rs - Polling with retry
cx.spawn(async move |_| {
    loop {
        if let Some((tool_call, options)) = permission_request {
            thread.update(cx, |thread, cx| {
                thread.request_tool_call_authorization(
                    tool_call.clone(),
                    options.clone(),
                    false,
                    cx,
                )
            })?;
        }

        if let Some(result) = check_result() {
            return Ok(result);
        }

        cx.background_executor().timer(Duration::from_millis(100)).await;
    }
})
```

### Concurrent task management

```rust
// From agent_panel.rs - Managing multiple concurrent path lookups
cx.spawn_in(window, async move |this, cx| {
    let mut paths = vec![];
    let mut added_worktrees = vec![];

    for task in tasks {
        if let Some((path, worktree)) = task.await.log_err() {
            paths.push(path);
            if let Some(worktree) = worktree {
                added_worktrees.push(worktree);
            }
        }
    }

    this.update(cx, |this, cx| {
        // Update with all collected results
        cx.notify();
    })
})
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
