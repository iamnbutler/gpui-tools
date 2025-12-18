# GPUI App & Window Patterns

1. [High Level Patterns](#high-level-patterns)
2. [Application Creation](#application-creation)
3. [Window Creation](#window-creation)
4. [Window Options](#window-options)
5. [Window Bounds](#window-bounds)
6. [Titlebar Options](#titlebar-options)
7. [Window Kinds](#window-kinds)
8. [Background Appearance](#background-appearance)
9. [Asset Sources](#asset-sources)
10. [Key Bindings](#key-bindings)
11. [Menus](#menus)
12. [Window Lifecycle](#window-lifecycle)
13. [Multi-Window Patterns](#multi-window-patterns)

## High Level Patterns

### Application lifecycle

```rust
// Basic application flow:
// 1. Create Application
// 2. Configure with assets/http client (optional)
// 3. Call run() with initialization callback
// 4. In callback: bind keys, set menus, open windows

Application::new()
    .with_assets(MyAssets {})           // Optional: provide asset source
    .with_http_client(http_client)       // Optional: provide HTTP client
    .run(|cx: &mut App| {
        // Application initialization code
        cx.open_window(WindowOptions::default(), |window, cx| {
            cx.new(|cx| MyRootView::new(cx))
        }).unwrap();
        cx.activate(true);
    });
```

### Window creation context

```rust
// In App context:
cx.open_window(options, |window, cx| {
    // window: &mut Window - window-specific operations
    // cx: &mut App - app context for entity creation
    cx.new(|cx| RootView::new(window, cx))
})?;

// The closure must return Entity<V> where V: Render
```

### Window vs App context

```rust
// Window operations (window: &mut Window):
window.focus(&focus_handle);          // Focus management
window.bounds();                       // Get window bounds
window.viewport_size();                // Get viewport size
window.scale_factor();                 // Get display scale
window.activate_window();              // Bring to front

// App operations (cx: &mut App):
cx.new(|cx| MyEntity::new(cx));       // Create entities
cx.open_window(...);                   // Open windows
cx.activate(true);                     // Activate application
cx.quit();                             // Quit application
```

## Application Creation

### Basic application

```rust
use gpui::{App, Application};

fn main() {
    Application::new().run(|cx: &mut App| {
        // Initialize your app here
        cx.open_window(WindowOptions::default(), |_, cx| {
            cx.new(|_| MyView {})
        }).unwrap();
        cx.activate(true);
    });
}
```

### Headless application (no GUI)

```rust
// For CLI tools, testing, or server applications
Application::headless().run(|cx: &mut App| {
    // No windows can be opened
    // Useful for background processing
});
```

### Application with assets

```rust
use gpui::{App, Application, AssetSource, SharedString};
use anyhow::Result;
use std::borrow::Cow;

struct Assets {
    base: PathBuf,
}

impl AssetSource for Assets {
    fn load(&self, path: &str) -> Result<Option<Cow<'static, [u8]>>> {
        let full_path = self.base.join(path);
        std::fs::read(&full_path)
            .map(|data| Some(Cow::Owned(data)))
            .map_err(Into::into)
    }

    fn list(&self, path: &str) -> Result<Vec<SharedString>> {
        let full_path = self.base.join(path);
        Ok(std::fs::read_dir(&full_path)?
            .filter_map(|entry| {
                Some(SharedString::from(
                    entry.ok()?.path().to_string_lossy().into_owned()
                ))
            })
            .collect())
    }
}

fn main() {
    Application::new()
        .with_assets(Assets {
            base: PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("assets"),
        })
        .run(|cx: &mut App| {
            // Assets are now available via svg() and other asset-loading functions
        });
}
```

### Application with HTTP client

```rust
use gpui::{App, Application};
use std::sync::Arc;

fn main() {
    let http_client = ReqwestClient::user_agent("my-app/1.0").unwrap();

    Application::new()
        .with_http_client(Arc::new(http_client))
        .run(|cx: &mut App| {
            // HTTP client available via cx.http_client()
        });
}
```

### Application event handlers

```rust
Application::new()
    .on_open_urls(|urls| {
        // Handle URLs opened with this application
        // e.g., custom URL schemes like myapp://action
        for url in urls {
            println!("Opened URL: {}", url);
        }
    })
    .on_reopen(|cx| {
        // Called when application is reopened (e.g., dock icon clicked on macOS)
        // Typically show a window if none are visible
        if cx.windows().is_empty() {
            // Open a new window
        }
    })
    .run(|cx| {
        // ...
    });
```

## Window Creation

### Basic window creation

```rust
use gpui::{App, Bounds, WindowBounds, WindowOptions, px, size};

fn main() {
    Application::new().run(|cx: &mut App| {
        cx.open_window(
            WindowOptions::default(),
            |_, cx| cx.new(|_| MyView {}),
        ).unwrap();
        cx.activate(true);
    });
}
```

### Window with custom bounds

```rust
cx.open_window(
    WindowOptions {
        window_bounds: Some(WindowBounds::Windowed(Bounds::centered(
            None,                           // Display (None = primary)
            size(px(800.), px(600.)),       // Size
            cx,
        ))),
        ..Default::default()
    },
    |_, cx| cx.new(|_| MyView {}),
).unwrap();
```

### Window with full configuration

```rust
let options = WindowOptions {
    window_bounds: Some(WindowBounds::Windowed(bounds)),
    titlebar: Some(TitlebarOptions {
        title: Some("My Application".into()),
        appears_transparent: false,
        traffic_light_position: None,
    }),
    focus: true,                                    // Focus on creation
    show: true,                                     // Show immediately
    kind: WindowKind::Normal,                       // Window type
    is_movable: true,                               // Allow dragging
    is_resizable: true,                             // Allow resizing
    is_minimizable: true,                           // Allow minimize
    display_id: None,                               // Specific display
    window_background: WindowBackgroundAppearance::Opaque,
    app_id: Some("com.mycompany.myapp".into()),     // Linux app ID
    window_min_size: Some(size(px(400.), px(300.))), // Minimum size
    window_decorations: None,                        // Wayland only
    tabbing_identifier: None,                        // macOS tab grouping
};

cx.open_window(options, |window, cx| {
    cx.new(|cx| MyView::new(window, cx))
}).unwrap();
```

## Window Options

### WindowOptions struct

```rust
pub struct WindowOptions {
    /// Window state and bounds. None inherits from system.
    pub window_bounds: Option<WindowBounds>,

    /// Titlebar configuration. None uses system default.
    pub titlebar: Option<TitlebarOptions>,

    /// Whether window should be focused when created.
    pub focus: bool,

    /// Whether window should be shown when created.
    pub show: bool,

    /// The kind of window (Normal, PopUp, Floating).
    pub kind: WindowKind,

    /// Whether window can be moved by user.
    pub is_movable: bool,

    /// Whether window can be resized by user.
    pub is_resizable: bool,

    /// Whether window can be minimized by user.
    pub is_minimizable: bool,

    /// Display to create window on. None uses primary display.
    pub display_id: Option<DisplayId>,

    /// Background appearance (Opaque, Transparent, Blurred).
    pub window_background: WindowBackgroundAppearance,

    /// Application identifier (Linux desktop environments).
    pub app_id: Option<String>,

    /// Minimum window size.
    pub window_min_size: Option<Size<Pixels>>,

    /// Window decorations (Wayland only).
    pub window_decorations: Option<WindowDecorations>,

    /// Tab grouping identifier (macOS 10.12+).
    pub tabbing_identifier: Option<String>,
}

impl Default for WindowOptions {
    fn default() -> Self {
        Self {
            window_bounds: None,
            titlebar: Some(TitlebarOptions {
                title: Default::default(),
                appears_transparent: false,
                traffic_light_position: None,
            }),
            focus: true,
            show: true,
            kind: WindowKind::Normal,
            is_movable: true,
            is_resizable: true,
            is_minimizable: true,
            display_id: None,
            window_background: WindowBackgroundAppearance::Opaque,
            app_id: None,
            window_min_size: None,
            window_decorations: None,
            tabbing_identifier: None,
        }
    }
}
```

## Window Bounds

### WindowBounds enum

```rust
pub enum WindowBounds {
    /// Normal windowed state with specific bounds.
    Windowed(Bounds<Pixels>),

    /// Maximized state. Bounds are restore size.
    Maximized(Bounds<Pixels>),

    /// Fullscreen state. Bounds are restore size.
    Fullscreen(Bounds<Pixels>),
}
```

### Creating bounds

```rust
use gpui::{Bounds, Point, Size, px, point, size};

// Explicit bounds with origin and size
let bounds = Bounds {
    origin: point(px(100.), px(100.)),
    size: size(px(800.), px(600.)),
};

// Centered bounds on primary display
let centered = Bounds::centered(
    None,                           // Display (None = primary)
    size(px(800.), px(600.)),       // Size
    cx,                             // App context
);

// Centered bounds on specific display
let display = cx.primary_display();
let centered_on_display = Bounds::centered(
    Some(&display),
    size(px(800.), px(600.)),
    cx,
);
```

### Window state examples

```rust
// Normal window
WindowBounds::Windowed(Bounds::centered(None, size(px(800.), px(600.)), cx))

// Start maximized (bounds used when un-maximized)
WindowBounds::Maximized(Bounds::centered(None, size(px(800.), px(600.)), cx))

// Start fullscreen (bounds used when exiting fullscreen)
WindowBounds::Fullscreen(Bounds::centered(None, size(px(800.), px(600.)), cx))
```

### Window bounds helper

```rust
impl WindowBounds {
    /// Creates centered window bounds.
    pub fn centered(size: Size<Pixels>, cx: &App) -> Self {
        WindowBounds::Windowed(Bounds::centered(None, size, cx))
    }

    /// Get inner bounds regardless of state.
    pub fn get_bounds(&self) -> Bounds<Pixels> {
        match self {
            WindowBounds::Windowed(bounds) => *bounds,
            WindowBounds::Maximized(bounds) => *bounds,
            WindowBounds::Fullscreen(bounds) => *bounds,
        }
    }
}
```

## Titlebar Options

### TitlebarOptions struct

```rust
pub struct TitlebarOptions {
    /// Initial window title.
    pub title: Option<SharedString>,

    /// Hide system titlebar for custom titlebar (macOS/Windows).
    pub appears_transparent: bool,

    /// Position of traffic lights (macOS only).
    pub traffic_light_position: Option<Point<Pixels>>,
}
```

### Titlebar patterns

```rust
// Standard titlebar with title
let options = WindowOptions {
    titlebar: Some(TitlebarOptions {
        title: Some("My Application".into()),
        appears_transparent: false,
        traffic_light_position: None,
    }),
    ..Default::default()
};

// Custom titlebar (draw your own)
let options = WindowOptions {
    titlebar: Some(TitlebarOptions {
        title: Some("My Application".into()),
        appears_transparent: true,
        traffic_light_position: Some(point(px(12.), px(12.))),
    }),
    ..Default::default()
};

// No titlebar
let options = WindowOptions {
    titlebar: None,
    ..Default::default()
};
```

### Dynamic title updates

```rust
impl MyView {
    fn update_title(&self, window: &mut Window, _cx: &mut Context<Self>) {
        window.set_window_title(&format!("My App - {}", self.current_file));
    }
}
```

## Window Kinds

### WindowKind enum

```rust
pub enum WindowKind {
    /// A normal application window.
    Normal,

    /// A popup window that appears above others (alerts, popups).
    /// Use sparingly!
    PopUp,

    /// A floating window on top of its parent.
    Floating,

    /// Wayland LayerShell window (docks, notifications, wallpapers).
    #[cfg(all(target_os = "linux", feature = "wayland"))]
    LayerShell(LayerShellOptions),
}
```

### Window kind examples

```rust
// Normal application window
let options = WindowOptions {
    kind: WindowKind::Normal,
    ..Default::default()
};

// Popup window (stays on top, non-activating)
let options = WindowOptions {
    kind: WindowKind::PopUp,
    focus: false,
    titlebar: None,
    ..Default::default()
};

// Floating window (child of parent window)
let options = WindowOptions {
    kind: WindowKind::Floating,
    is_movable: true,
    ..Default::default()
};
```

### Notification window pattern

```rust
fn notification_window_options(
    screen: Rc<dyn PlatformDisplay>,
    size: Size<Pixels>,
    cx: &App,
) -> WindowOptions {
    let notification_margin = px(16.);

    // Position at top-right of screen
    let bounds = Bounds {
        origin: screen.bounds().top_right() - point(size.width + notification_margin, px(0.)),
        size,
    };

    WindowOptions {
        window_bounds: Some(WindowBounds::Windowed(bounds)),
        titlebar: None,
        focus: false,                                    // Don't steal focus
        show: true,
        kind: WindowKind::PopUp,
        is_movable: false,
        is_resizable: false,
        window_background: WindowBackgroundAppearance::Transparent,
        ..Default::default()
    }
}
```

## Background Appearance

### WindowBackgroundAppearance enum

```rust
pub enum WindowBackgroundAppearance {
    /// Opaque background (default).
    /// Allows window manager to skip drawing content behind window.
    Opaque,

    /// Transparent background (alpha blending).
    Transparent,

    /// Transparent with blur effect behind window.
    /// Not supported on all platforms.
    Blurred,

    /// Mica backdrop material (Windows 11 only).
    #[cfg(target_os = "windows")]
    Mica,
}
```

### Transparency examples

```rust
// Opaque window (most efficient)
let options = WindowOptions {
    window_background: WindowBackgroundAppearance::Opaque,
    ..Default::default()
};

// Transparent window
let options = WindowOptions {
    window_background: WindowBackgroundAppearance::Transparent,
    titlebar: None,  // Usually want no titlebar for transparent windows
    ..Default::default()
};

// Blurred background (vibrancy effect)
let options = WindowOptions {
    window_background: WindowBackgroundAppearance::Blurred,
    ..Default::default()
};
```

### Dynamic background appearance

```rust
impl MyView {
    fn set_transparency(&self, window: &mut Window, enabled: bool) {
        window.set_background_appearance(if enabled {
            WindowBackgroundAppearance::Transparent
        } else {
            WindowBackgroundAppearance::Opaque
        });
    }
}
```

## Asset Sources

### AssetSource trait

```rust
pub trait AssetSource: 'static + Send + Sync {
    /// Load asset data by path.
    fn load(&self, path: &str) -> Result<Option<Cow<'static, [u8]>>>;

    /// List assets in directory.
    fn list(&self, path: &str) -> Result<Vec<SharedString>>;
}
```

### Embedded assets

```rust
use rust_embed::RustEmbed;

#[derive(RustEmbed)]
#[folder = "assets/"]
struct Assets;

impl AssetSource for Assets {
    fn load(&self, path: &str) -> Result<Option<Cow<'static, [u8]>>> {
        Ok(Self::get(path).map(|file| file.data))
    }

    fn list(&self, path: &str) -> Result<Vec<SharedString>> {
        Ok(Self::iter()
            .filter(|name| name.starts_with(path))
            .map(|name| name.into())
            .collect())
    }
}
```

### Filesystem assets

```rust
struct FileAssets {
    base_path: PathBuf,
}

impl AssetSource for FileAssets {
    fn load(&self, path: &str) -> Result<Option<Cow<'static, [u8]>>> {
        let full_path = self.base_path.join(path);
        match std::fs::read(&full_path) {
            Ok(data) => Ok(Some(Cow::Owned(data))),
            Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(None),
            Err(e) => Err(e.into()),
        }
    }

    fn list(&self, path: &str) -> Result<Vec<SharedString>> {
        let full_path = self.base_path.join(path);
        Ok(std::fs::read_dir(&full_path)?
            .filter_map(|entry| {
                let path = entry.ok()?.path();
                let name = path.file_name()?.to_string_lossy().into_owned();
                Some(SharedString::from(name))
            })
            .collect())
    }
}
```

### Using assets

```rust
// SVG from assets
svg()
    .path("icons/my_icon.svg")
    .size_4()
    .text_color(gpui::black())

// Image from assets
img("images/logo.png")
    .size(px(100.), px(100.))
```

## Key Bindings

### Basic key binding setup

```rust
use gpui::{App, KeyBinding, actions};

actions!(my_app, [Quit, Save, Open, Undo, Redo]);

fn main() {
    Application::new().run(|cx: &mut App| {
        cx.bind_keys([
            KeyBinding::new("cmd-q", Quit, None),
            KeyBinding::new("cmd-s", Save, None),
            KeyBinding::new("cmd-o", Open, None),
            KeyBinding::new("cmd-z", Undo, None),
            KeyBinding::new("cmd-shift-z", Redo, None),
        ]);

        cx.on_action(|_: &Quit, cx| {
            cx.quit();
        });

        // ...
    });
}
```

### Context-specific bindings

```rust
// Key bindings with context
cx.bind_keys([
    // Global binding (no context)
    KeyBinding::new("cmd-q", Quit, None),

    // Binding only when Editor is focused
    KeyBinding::new("cmd-/", ToggleComment, Some("Editor")),

    // Binding only when search panel is open
    KeyBinding::new("escape", CloseSearch, Some("SearchPanel")),
]);
```

### Key binding syntax

```rust
// Modifiers: cmd, ctrl, alt, shift
// Platform-specific: on macOS 'cmd' is Command, on others it's Control

KeyBinding::new("cmd-s", Save, None)          // Cmd+S / Ctrl+S
KeyBinding::new("cmd-shift-s", SaveAs, None)  // Cmd+Shift+S / Ctrl+Shift+S
KeyBinding::new("alt-enter", Confirm, None)   // Alt+Enter
KeyBinding::new("ctrl-c", Copy, None)         // Ctrl+C (always Ctrl)
KeyBinding::new("tab", Tab, None)             // Tab key
KeyBinding::new("shift-tab", TabPrev, None)   // Shift+Tab
KeyBinding::new("escape", Cancel, None)       // Escape key
KeyBinding::new("enter", Submit, None)        // Enter key
KeyBinding::new("backspace", Delete, None)    // Backspace key
KeyBinding::new("up", MoveUp, None)           // Arrow up
KeyBinding::new("cmd-shift-p", CommandPalette, None)  // Multiple modifiers
```

### Handling actions in views

```rust
impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .track_focus(&self.focus_handle)
            .on_action(cx.listener(Self::handle_save))
            .on_action(cx.listener(Self::handle_quit))
            .child("Content")
    }
}

impl MyView {
    fn handle_save(&mut self, _: &Save, _window: &mut Window, cx: &mut Context<Self>) {
        // Handle save action
        cx.notify();
    }

    fn handle_quit(&mut self, _: &Quit, _window: &mut Window, cx: &mut Context<Self>) {
        cx.quit();
    }
}
```

## Menus

### Application menu setup

```rust
use gpui::{Menu, MenuItem};

fn main() {
    Application::new().run(|cx: &mut App| {
        cx.set_menus(vec![
            Menu {
                name: "My App".into(),
                items: vec![
                    MenuItem::action("About My App", About),
                    MenuItem::separator(),
                    MenuItem::action("Settings...", OpenSettings),
                    MenuItem::separator(),
                    MenuItem::action("Quit", Quit),
                ],
            },
            Menu {
                name: "File".into(),
                items: vec![
                    MenuItem::action("New", NewFile),
                    MenuItem::action("Open...", Open),
                    MenuItem::separator(),
                    MenuItem::action("Save", Save),
                    MenuItem::action("Save As...", SaveAs),
                ],
            },
            Menu {
                name: "Edit".into(),
                items: vec![
                    MenuItem::action("Undo", Undo),
                    MenuItem::action("Redo", Redo),
                    MenuItem::separator(),
                    MenuItem::action("Cut", Cut),
                    MenuItem::action("Copy", Copy),
                    MenuItem::action("Paste", Paste),
                ],
            },
        ]);

        // ...
    });
}
```

### MenuItem types

```rust
// Action item - triggers an action when clicked
MenuItem::action("Save", Save)

// Separator
MenuItem::separator()

// Submenu
MenuItem::submenu(Menu {
    name: "Recent Files".into(),
    items: vec![
        MenuItem::action("file1.txt", OpenRecent { path: "file1.txt".into() }),
        MenuItem::action("file2.txt", OpenRecent { path: "file2.txt".into() }),
    ],
})

// OS-specific items (handled by platform)
MenuItem::os_action("About", OsAction::About)
MenuItem::os_action("Hide", OsAction::Hide)
MenuItem::os_action("Quit", OsAction::Quit)
```

## Window Lifecycle

### Window close handling

```rust
fn main() {
    Application::new().run(|cx: &mut App| {
        // Handle all window closes
        cx.on_window_closed(|cx| {
            if cx.windows().is_empty() {
                cx.quit();  // Quit when last window closes
            }
        }).detach();

        // ...
    });
}
```

### Preventing window close

```rust
impl MyView {
    fn setup_close_handler(&self, window: &mut Window, cx: &mut Context<Self>) {
        window.on_window_should_close(cx.listener(|this, _window, cx| {
            if this.has_unsaved_changes {
                // Show confirmation dialog
                this.show_save_dialog(cx);
                false  // Prevent close
            } else {
                true   // Allow close
            }
        }));
    }
}
```

### Window activation

```rust
impl MyView {
    fn activate(&self, window: &mut Window) {
        window.activate_window();  // Bring window to front
    }
}
```

### Window state queries

```rust
impl MyView {
    fn check_window_state(&self, window: &Window) {
        let is_fullscreen = window.is_fullscreen();
        let is_maximized = window.is_maximized();
        let is_active = window.is_window_active();
        let bounds = window.bounds();
        let viewport = window.viewport_size();
        let scale = window.scale_factor();
    }
}
```

## Multi-Window Patterns

### Opening multiple windows

```rust
fn open_new_window(cx: &mut App) -> WindowHandle<MyView> {
    let bounds = Bounds::centered(None, size(px(800.), px(600.)), cx);

    cx.open_window(
        WindowOptions {
            window_bounds: Some(WindowBounds::Windowed(bounds)),
            ..Default::default()
        },
        |_, cx| cx.new(|_| MyView::new()),
    ).unwrap()
}
```

### Window communication

```rust
// Store window handles
struct AppState {
    windows: Vec<WindowHandle<MyView>>,
}

impl AppState {
    fn broadcast_message(&self, message: Message, cx: &mut App) {
        for window_handle in &self.windows {
            window_handle.update(cx, |_, window, cx| {
                // Send message to window
            }).ok();
        }
    }
}
```

### Getting all windows

```rust
fn main() {
    Application::new().run(|cx: &mut App| {
        // Get all open windows
        let windows = cx.windows();

        // Get active window
        let active = cx.active_window();

        // ...
    });
}
```

### Window positioning

```rust
fn tile_windows(cx: &mut App) {
    let windows = cx.windows();
    let display = cx.primary_display();
    let display_bounds = display.bounds();

    let window_count = windows.len() as f32;
    let window_width = display_bounds.size.width / window_count;

    for (i, window) in windows.iter().enumerate() {
        let bounds = Bounds {
            origin: point(
                display_bounds.origin.x + window_width * i as f32,
                display_bounds.origin.y,
            ),
            size: size(window_width, display_bounds.size.height),
        };

        window.update(cx, |_, window, _| {
            window.resize(bounds.size);
        }).ok();
    }
}
```

### Display management

```rust
fn main() {
    Application::new().run(|cx: &mut App| {
        // Get all displays
        let displays = cx.displays();

        // Get primary display
        let primary = cx.primary_display();

        // Find specific display
        let display = cx.find_display(|d| {
            d.bounds().size.width > px(2000.)  // Find wide display
        });

        // Open window on specific display
        if let Some(display) = display {
            cx.open_window(
                WindowOptions {
                    display_id: Some(display.id()),
                    window_bounds: Some(WindowBounds::Windowed(
                        Bounds::centered(Some(&display), size(px(800.), px(600.)), cx)
                    )),
                    ..Default::default()
                },
                |_, cx| cx.new(|_| MyView {}),
            ).unwrap();
        }
    });
}
```

### macOS window tabbing

```rust
// Group windows into native tabs (macOS 10.12+)
let options = WindowOptions {
    tabbing_identifier: Some("main-windows".into()),
    ..Default::default()
};

// Windows with same tabbing_identifier will be grouped together
```

### Complete application example

```rust
use gpui::{
    App, Application, Bounds, Context, KeyBinding, Menu, MenuItem, SharedString,
    Window, WindowBounds, WindowOptions, actions, div, prelude::*, px, rgb, size,
};

actions!(my_app, [Quit, NewWindow]);

struct MyView {
    counter: u32,
}

impl MyView {
    fn new() -> Self {
        Self { counter: 0 }
    }
}

impl Render for MyView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .size_full()
            .bg(rgb(0x1e1e1e))
            .text_color(rgb(0xffffff))
            .justify_center()
            .items_center()
            .child(format!("Counter: {}", self.counter))
    }
}

fn main() {
    Application::new().run(|cx: &mut App| {
        // Set up key bindings
        cx.bind_keys([
            KeyBinding::new("cmd-q", Quit, None),
            KeyBinding::new("cmd-n", NewWindow, None),
        ]);

        // Set up menus
        cx.set_menus(vec![
            Menu {
                name: "My App".into(),
                items: vec![
                    MenuItem::action("Quit", Quit),
                ],
            },
            Menu {
                name: "File".into(),
                items: vec![
                    MenuItem::action("New Window", NewWindow),
                ],
            },
        ]);

        // Handle actions
        cx.on_action(|_: &Quit, cx| cx.quit());
        cx.on_action(|_: &NewWindow, cx| {
            let bounds = Bounds::centered(None, size(px(800.), px(600.)), cx);
            cx.open_window(
                WindowOptions {
                    window_bounds: Some(
