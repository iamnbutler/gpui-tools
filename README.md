# GPUI Tools Documentation

A collection of documentation and patterns for working with GPUI, the GPU-accelerated UI framework used by Zed.

## Purpose

This repository contains documentation, patterns, and examples for developers working with GPUI. The documentation covers:

- Entity patterns and lifecycle management
- Task management and async operations
- Testing patterns and best practices

## Source Attribution

This documentation includes examples from various open-source projects. We gratefully acknowledge the following sources:

### Primary Sources

#### Zed Editor

- **Repository**: https://github.com/zed-industries/zed
- **License**: Multiple licenses (GPL-3.0, Apache-2.0, AGPL-3.0)
- **Usage**: Examples from both GPL-licensed and Apache-licensed crates

**Important Note**: Some examples in this documentation are derived from GPL-licensed code in the Zed repository. These examples are used for educational purposes. When using code patterns from this documentation in your own projects, please be aware of the licensing implications.

### GPL-Licensed Crates in Zed (GPL-3.0 or AGPL-3.0)

The following Zed crates are GPL/AGPL-licensed. Examples from these should be used with caution:

- `crates/assistant2/*` - AI assistant functionality (GPL-3.0)
- `crates/repl/*` - REPL implementation (GPL-3.0)
- `crates/assistant/*` - Legacy assistant (GPL-3.0)
- `crates/assistant_slash_command/*` - Assistant commands (GPL-3.0)
- `crates/assistant_tool/*` - Assistant tools (GPL-3.0)
- `crates/inline_completion_button/*` - UI components (GPL-3.0)
- `crates/zed/*` - Main application (GPL-3.0)

### Apache-Licensed Crates in Zed (Apache-2.0)

These Zed crates are Apache-licensed and safer to reference for patterns:

- `crates/gpui/*` - Core UI framework (Apache-2.0)
- `crates/editor/*` - Text editor implementation (Apache-2.0)
- `crates/workspace/*` - Workspace management (Apache-2.0)
- `crates/project/*` - Project management (Apache-2.0)
- `crates/language/*` - Language server support (Apache-2.0)
- `crates/theme/*` - Theming system (Apache-2.0)
- `crates/ui/*` - UI components (Apache-2.0)
- `crates/settings/*` - Settings management (Apache-2.0)
- `crates/db/*` - Database layer (Apache-2.0)
- `crates/rpc/*` - RPC implementation (Apache-2.0)
- Most other crates not listed above are Apache-2.0

### How to Identify License in Examples

Each example in our documentation includes a source comment indicating:

1. The source file path
2. The license of that crate (GPL-3.0 or Apache-2.0)

Example:

```rust
// Source: zed/crates/assistant2/src/acp_thread.rs (GPL-3.0)
// Source: zed/crates/editor/src/editor.rs (Apache-2.0)
```

## Documentation Files

### `docs/gpui_entities.md`

Comprehensive guide to GPUI entity patterns including:

- Entity creation and initialization
- Reading and updating entity state
- WeakEntity patterns for async operations
- Subscriptions and observations
- Event emission patterns
- Entity lifecycle management

### `docs/gpui_tasks.md`

Guide to task management in GPUI including:

- Task spawning patterns
- Background task management
- Async/await patterns
- Task cancellation and cleanup

### `docs/gpui_testing.md`

Testing patterns and best practices for GPUI applications including:

- Test context setup
- Entity testing
- Async test patterns
- Mock and stub patterns

## License Compliance

This documentation repository itself is for educational purposes. Individual code examples maintain their original licensing from their source repositories.

### Guidelines for Using Examples:

1. **Apache-2.0 Licensed Examples**: Can be freely used in most projects with attribution
2. **GPL-3.0 Licensed Examples**:
   - Use only the patterns/concepts, not the actual code
   - If you copy code directly, your project may need to be GPL-licensed
   - Consider finding alternative implementations from Apache-licensed crates

3. **Best Practices**:
   - Always check the source comment on each example
   - Prefer examples from Apache-licensed crates when available
   - When in doubt, implement your own version inspired by the pattern
   - Provide attribution as required by the license

## Contributing

When contributing examples or documentation:

1. **Always add source attribution** in this format:

   ```rust
   // Source: zed/crates/[crate_name]/src/[file].rs ([License])
   ```

2. **Prefer Apache-2.0 licensed examples** when multiple options exist

3. **For GPL examples**, ensure they demonstrate unique patterns not available in Apache-licensed code

4. **Document the pattern**, not just the code - explain:
   - What problem it solves
   - When to use it
   - Alternative approaches if available

5. **Test your examples** to ensure they compile and work correctly

## Additional Resources

- [GPUI Repository](https://github.com/zed-industries/zed/tree/main/crates/gpui)
- [Zed Documentation](https://zed.dev/docs)
- [GPUI API Documentation](https://docs.rs/gpui/latest/gpui/)

## Disclaimer

This is an independent documentation project and is not officially affiliated with or endorsed by Zed Industries. All trademarks and project names are the property of their respective owners.
