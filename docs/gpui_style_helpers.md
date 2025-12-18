# GPUI Style Helpers Reference

Quick reference for all style helper methods available on elements implementing the `Styled` trait.

## Table of Contents

1. [Units](#units)
2. [Display & Visibility](#display--visibility)
3. [Flexbox](#flexbox)
4. [Grid](#grid)
5. [Sizing](#sizing)
6. [Spacing](#spacing)
7. [Positioning](#positioning)
8. [Colors & Backgrounds](#colors--backgrounds)
9. [Borders](#borders)
10. [Border Radius](#border-radius)
11. [Shadows](#shadows)
12. [Typography](#typography)
13. [Overflow & Scrolling](#overflow--scrolling)
14. [Cursor](#cursor)
15. [Opacity](#opacity)
16. [Interactive States](#interactive-states)
17. [Conditional Styling](#conditional-styling)

---

## Units

```rust
use gpui::{px, rems, rems_from_px, relative, Length, Pixels, Rems};

px(16.)              // 16 pixels
rems(1.)             // 1rem (16px at default)
rems(0.5)            // 0.5rem (8px)
rems_from_px(12.)    // Convert pixels to rems
relative(0.5)        // 50%
relative(1.)         // 100%
Length::Auto         // auto
```

### Numeric Scale

| Scale | Rems | Pixels |
|-------|------|--------|
| `0` | 0 | 0 |
| `0p5` | 0.125rem | 2px |
| `1` | 0.25rem | 4px |
| `1p5` | 0.375rem | 6px |
| `2` | 0.5rem | 8px |
| `3` | 0.75rem | 12px |
| `4` | 1rem | 16px |
| `5` | 1.25rem | 20px |
| `6` | 1.5rem | 24px |
| `8` | 2rem | 32px |
| `10` | 2.5rem | 40px |
| `12` | 3rem | 48px |
| `16` | 4rem | 64px |
| `20` | 5rem | 80px |
| `24` | 6rem | 96px |
| `32` | 8rem | 128px |
| `40` | 10rem | 160px |
| `48` | 12rem | 192px |
| `56` | 14rem | 224px |
| `64` | 16rem | 256px |
| `72` | 18rem | 288px |
| `80` | 20rem | 320px |
| `96` | 24rem | 384px |

---

## Display & Visibility

### Display

| Method | CSS |
|--------|-----|
| `.flex()` | `display: flex` |
| `.block()` | `display: block` |
| `.grid()` | `display: grid` |
| `.hidden()` | `display: none` |

### Visibility

| Method | CSS |
|--------|-----|
| `.visible()` | `visibility: visible` |
| `.invisible()` | `visibility: hidden` |

---

## Flexbox

### Direction

| Method | CSS |
|--------|-----|
| `.flex_row()` | `flex-direction: row` |
| `.flex_row_reverse()` | `flex-direction: row-reverse` |
| `.flex_col()` | `flex-direction: column` |
| `.flex_col_reverse()` | `flex-direction: column-reverse` |

### Wrap

| Method | CSS |
|--------|-----|
| `.flex_wrap()` | `flex-wrap: wrap` |
| `.flex_wrap_reverse()` | `flex-wrap: wrap-reverse` |
| `.flex_nowrap()` | `flex-wrap: nowrap` |

### Flex Sizing

| Method | CSS | Description |
|--------|-----|-------------|
| `.flex_1()` | `flex: 1 1 0%` | Grow and shrink, ignore initial size |
| `.flex_auto()` | `flex: 1 1 auto` | Grow and shrink, use initial size |
| `.flex_initial()` | `flex: 0 1 auto` | Shrink only |
| `.flex_none()` | `flex: none` | Don't grow or shrink |
| `.flex_grow()` | `flex-grow: 1` | Allow growing |
| `.flex_shrink()` | `flex-shrink: 1` | Allow shrinking |
| `.flex_shrink_0()` | `flex-shrink: 0` | Prevent shrinking |
| `.flex_basis(val)` | `flex-basis: {val}` | Set flex basis |

### Align Items (cross axis)

| Method | CSS |
|--------|-----|
| `.items_start()` | `align-items: flex-start` |
| `.items_end()` | `align-items: flex-end` |
| `.items_center()` | `align-items: center` |
| `.items_baseline()` | `align-items: baseline` |

### Justify Content (main axis)

| Method | CSS |
|--------|-----|
| `.justify_start()` | `justify-content: flex-start` |
| `.justify_end()` | `justify-content: flex-end` |
| `.justify_center()` | `justify-content: center` |
| `.justify_between()` | `justify-content: space-between` |
| `.justify_around()` | `justify-content: space-around` |
| `.justify_evenly()` | `justify-content: space-evenly` |

### Align Content (multi-line)

| Method | CSS |
|--------|-----|
| `.content_start()` | `align-content: flex-start` |
| `.content_end()` | `align-content: flex-end` |
| `.content_center()` | `align-content: center` |
| `.content_between()` | `align-content: space-between` |
| `.content_around()` | `align-content: space-around` |
| `.content_normal()` | `align-content: normal` |

### Gap

| Method | CSS |
|--------|-----|
| `.gap_{n}()` | `gap: {scale}` |
| `.gap(val)` | `gap: {val}` |
| `.gap_x_{n}()` | `column-gap: {scale}` |
| `.gap_y_{n}()` | `row-gap: {scale}` |

---

## Grid

### Grid Template

| Method | CSS |
|--------|-----|
| `.grid_cols_1()` ... `.grid_cols_12()` | `grid-template-columns: repeat(n, 1fr)` |
| `.grid_rows_1()` ... `.grid_rows_12()` | `grid-template-rows: repeat(n, 1fr)` |

### Grid Item Placement

| Method | CSS |
|--------|-----|
| `.col_span_1()` ... `.col_span_12()` | `grid-column: span n` |
| `.col_span_full()` | `grid-column: 1 / -1` |
| `.col_start_1()` ... `.col_start_12()` | `grid-column-start: n` |
| `.row_span_1()` ... `.row_span_12()` | `grid-row: span n` |
| `.row_span_full()` | `grid-row: 1 / -1` |
| `.row_start_1()` ... `.row_start_12()` | `grid-row-start: n` |

---

## Sizing

### Width

| Method | CSS |
|--------|-----|
| `.w_{n}()` | `width: {scale}` |
| `.w(val)` | `width: {val}` |
| `.w_full()` | `width: 100%` |
| `.w_auto()` | `width: auto` |
| `.w_1_2()` | `width: 50%` |
| `.w_1_3()` | `width: 33.333%` |
| `.w_2_3()` | `width: 66.667%` |
| `.w_1_4()` | `width: 25%` |
| `.w_3_4()` | `width: 75%` |

### Height

| Method | CSS |
|--------|-----|
| `.h_{n}()` | `height: {scale}` |
| `.h(val)` | `height: {val}` |
| `.h_full()` | `height: 100%` |
| `.h_auto()` | `height: auto` |
| `.h_1_2()` | `height: 50%` |
| `.h_1_3()` | `height: 33.333%` |

### Size (width + height)

| Method | CSS |
|--------|-----|
| `.size_{n}()` | `width: {scale}; height: {scale}` |
| `.size(val)` | `width: {val}; height: {val}` |
| `.size_full()` | `width: 100%; height: 100%` |

### Min/Max Sizing

| Method | CSS |
|--------|-----|
| `.min_w_0()` | `min-width: 0` |
| `.min_w_full()` | `min-width: 100%` |
| `.min_w(val)` | `min-width: {val}` |
| `.max_w_full()` | `max-width: 100%` |
| `.max_w(val)` | `max-width: {val}` |
| `.min_h_0()` | `min-height: 0` |
| `.min_h_full()` | `min-height: 100%` |
| `.min_h(val)` | `min-height: {val}` |
| `.max_h_full()` | `max-height: 100%` |
| `.max_h(val)` | `max-height: {val}` |

---

## Spacing

### Padding

| Method | CSS |
|--------|-----|
| `.p_{n}()` | `padding: {scale}` |
| `.p(val)` | `padding: {val}` |
| `.px_{n}()` | `padding-left: {scale}; padding-right: {scale}` |
| `.py_{n}()` | `padding-top: {scale}; padding-bottom: {scale}` |
| `.pt_{n}()` | `padding-top: {scale}` |
| `.pr_{n}()` | `padding-right: {scale}` |
| `.pb_{n}()` | `padding-bottom: {scale}` |
| `.pl_{n}()` | `padding-left: {scale}` |
| `.pt(val)` | `padding-top: {val}` |
| `.pr(val)` | `padding-right: {val}` |
| `.pb(val)` | `padding-bottom: {val}` |
| `.pl(val)` | `padding-left: {val}` |
| `.px(val)` | `padding-left: {val}; padding-right: {val}` |
| `.py(val)` | `padding-top: {val}; padding-bottom: {val}` |

### Margin

| Method | CSS |
|--------|-----|
| `.m_{n}()` | `margin: {scale}` |
| `.m(val)` | `margin: {val}` |
| `.m_auto()` | `margin: auto` |
| `.mx_{n}()` | `margin-left: {scale}; margin-right: {scale}` |
| `.mx_auto()` | `margin-left: auto; margin-right: auto` |
| `.my_{n}()` | `margin-top: {scale}; margin-bottom: {scale}` |
| `.my_auto()` | `margin-top: auto; margin-bottom: auto` |
| `.mt_{n}()` | `margin-top: {scale}` |
| `.mr_{n}()` | `margin-right: {scale}` |
| `.mb_{n}()` | `margin-bottom: {scale}` |
| `.ml_{n}()` | `margin-left: {scale}` |
| `.mt_neg_{n}()` | `margin-top: -{scale}` |
| `.mr_neg_{n}()` | `margin-right: -{scale}` |
| `.mb_neg_{n}()` | `margin-bottom: -{scale}` |
| `.ml_neg_{n}()` | `margin-left: -{scale}` |

---

## Positioning

### Position Type

| Method | CSS |
|--------|-----|
| `.relative()` | `position: relative` |
| `.absolute()` | `position: absolute` |

### Inset

| Method | CSS |
|--------|-----|
| `.inset_0()` | `top: 0; right: 0; bottom: 0; left: 0` |
| `.inset_{n}()` | `inset: {scale}` |
| `.inset_x_0()` | `left: 0; right: 0` |
| `.inset_y_0()` | `top: 0; bottom: 0` |
| `.top_0()` | `top: 0` |
| `.top_{n}()` | `top: {scale}` |
| `.top(val)` | `top: {val}` |
| `.right_0()` | `right: 0` |
| `.right_{n}()` | `right: {scale}` |
| `.right(val)` | `right: {val}` |
| `.bottom_0()` | `bottom: 0` |
| `.bottom_{n}()` | `bottom: {scale}` |
| `.bottom(val)` | `bottom: {val}` |
| `.left_0()` | `left: 0` |
| `.left_{n}()` | `left: {scale}` |
| `.left(val)` | `left: {val}` |

### Z-Index

| Method | CSS |
|--------|-----|
| `.z_10()` | `z-index: 10` |
| `.z_20()` | `z-index: 20` |
| `.z_50()` | `z-index: 50` |

---

## Colors & Backgrounds

### Color Creation

```rust
use gpui::{rgb, rgba, hsla, Hsla};

rgb(0x1e1e1e)                     // #1e1e1e
rgba(0x1e1e1e80)                  // #1e1e1e with 50% alpha
hsla(210. / 360., 0.5, 0.5, 1.0)  // HSLA color

// Built-in colors
gpui::white()
gpui::black()
gpui::red()
gpui::green()
gpui::blue()
gpui::yellow()
gpui::transparent()
```

### Background

| Method | Description |
|--------|-------------|
| `.bg(color)` | Set background color |
| `.bg(linear_gradient(...))` | Set gradient background |

### Text Color

| Method | Description |
|--------|-------------|
| `.text_color(color)` | Set text color |

### Color Opacity

```rust
color.opacity(0.5)  // Returns color with 50% opacity
```

### Gradients

```rust
use gpui::{linear_gradient, linear_color_stop};

linear_gradient(
    180.,  // angle in degrees
    linear_color_stop(rgb(0x3b82f6), 0.),   // start color
    linear_color_stop(rgb(0x8b5cf6), 1.),   // end color
)
```

---

## Borders

### Border Width

| Method | CSS |
|--------|-----|
| `.border_0()` | `border-width: 0` |
| `.border_1()` | `border-width: 1px` |
| `.border_2()` | `border-width: 2px` |
| `.border_4()` | `border-width: 4px` |
| `.border_8()` | `border-width: 8px` |
| `.border_t_1()` | `border-top-width: 1px` |
| `.border_r_1()` | `border-right-width: 1px` |
| `.border_b_1()` | `border-bottom-width: 1px` |
| `.border_l_1()` | `border-left-width: 1px` |
| `.border_x_1()` | `border-left-width: 1px; border-right-width: 1px` |
| `.border_y_1()` | `border-top-width: 1px; border-bottom-width: 1px` |

### Border Color

| Method | Description |
|--------|-------------|
| `.border_color(color)` | Set border color |

### Border Style

| Method | CSS |
|--------|-----|
| `.border_solid()` | `border-style: solid` |
| `.border_dashed()` | `border-style: dashed` |
| `.border_dotted()` | `border-style: dotted` |

---

## Border Radius

### Preset Radii

| Method | CSS |
|--------|-----|
| `.rounded_none()` | `border-radius: 0` |
| `.rounded_sm()` | `border-radius: 0.125rem` (2px) |
| `.rounded_md()` | `border-radius: 0.375rem` (6px) |
| `.rounded_lg()` | `border-radius: 0.5rem` (8px) |
| `.rounded_xl()` | `border-radius: 0.75rem` (12px) |
| `.rounded_2xl()` | `border-radius: 1rem` (16px) |
| `.rounded_3xl()` | `border-radius: 1.5rem` (24px) |
| `.rounded_full()` | `border-radius: 9999px` |
| `.rounded(val)` | `border-radius: {val}` |

### Individual Corners

| Method | CSS |
|--------|-----|
| `.rounded_tl_{size}()` | `border-top-left-radius: {size}` |
| `.rounded_tr_{size}()` | `border-top-right-radius: {size}` |
| `.rounded_bl_{size}()` | `border-bottom-left-radius: {size}` |
| `.rounded_br_{size}()` | `border-bottom-right-radius: {size}` |
| `.rounded_t_{size}()` | `border-top-left-radius + border-top-right-radius` |
| `.rounded_b_{size}()` | `border-bottom-left-radius + border-bottom-right-radius` |
| `.rounded_l_{size}()` | `border-top-left-radius + border-bottom-left-radius` |
| `.rounded_r_{size}()` | `border-top-right-radius + border-bottom-right-radius` |

---

## Shadows

| Method | Description |
|--------|-------------|
| `.shadow_none()` | No shadow |
| `.shadow_sm()` | Small shadow |
| `.shadow()` | Default shadow |
| `.shadow_md()` | Medium shadow |
| `.shadow_lg()` | Large shadow |
| `.shadow_xl()` | Extra large shadow |
| `.shadow_2xl()` | 2x extra large shadow |

---

## Typography

### Font Size

| Method | CSS |
|--------|-----|
| `.text_xs()` | `font-size: 0.75rem` (12px) |
| `.text_sm()` | `font-size: 0.875rem` (14px) |
| `.text_base()` | `font-size: 1rem` (16px) |
| `.text_lg()` | `font-size: 1.125rem` (18px) |
| `.text_xl()` | `font-size: 1.25rem` (20px) |
| `.text_2xl()` | `font-size: 1.5rem` (24px) |
| `.text_3xl()` | `font-size: 1.875rem` (30px) |
| `.text_size(val)` | `font-size: {val}` |

### Font Weight

```rust
use gpui::FontWeight;

.font_weight(FontWeight::THIN)        // 100
.font_weight(FontWeight::EXTRA_LIGHT) // 200
.font_weight(FontWeight::LIGHT)       // 300
.font_weight(FontWeight::NORMAL)      // 400
.font_weight(FontWeight::MEDIUM)      // 500
.font_weight(FontWeight::SEMIBOLD)    // 600
.font_weight(FontWeight::BOLD)        // 700
.font_weight(FontWeight::EXTRA_BOLD)  // 800
.font_weight(FontWeight::BLACK)       // 900
```

### Font Family

| Method | Description |
|--------|-------------|
| `.font_family(name)` | Set font family |

### Text Alignment

| Method | CSS |
|--------|-----|
| `.text_left()` | `text-align: left` |
| `.text_center()` | `text-align: center` |
| `.text_right()` | `text-align: right` |

### Text Decoration

| Method | CSS |
|--------|-----|
| `.underline()` | `text-decoration: underline` |
| `.line_through()` | `text-decoration: line-through` |

### Text Overflow

| Method | Description |
|--------|-------------|
| `.whitespace_normal()` | Allow text wrapping |
| `.whitespace_nowrap()` | Prevent text wrapping |
| `.text_ellipsis()` | Truncate with "..." |
| `.truncate()` | `overflow-hidden + nowrap + ellipsis` |
| `.line_clamp(n)` | Limit to n lines with ellipsis |

### Line Height

| Method | Description |
|--------|-------------|
| `.line_height(val)` | Set line height |

---

## Overflow & Scrolling

### Overflow

| Method | CSS |
|--------|-----|
| `.overflow_hidden()` | `overflow: hidden` |
| `.overflow_visible()` | `overflow: visible` |
| `.overflow_scroll()` | `overflow: scroll` |
| `.overflow_auto()` | `overflow: auto` |
| `.overflow_x_hidden()` | `overflow-x: hidden` |
| `.overflow_x_visible()` | `overflow-x: visible` |
| `.overflow_x_scroll()` | `overflow-x: scroll` |
| `.overflow_x_auto()` | `overflow-x: auto` |
| `.overflow_y_hidden()` | `overflow-y: hidden` |
| `.overflow_y_visible()` | `overflow-y: visible` |
| `.overflow_y_scroll()` | `overflow-y: scroll` |
| `.overflow_y_auto()` | `overflow-y: auto` |

### Scroll Snap

| Method | Description |
|--------|-------------|
| `.snap_center()` | Snap to center |

---

## Cursor

| Method | CSS |
|--------|-----|
| `.cursor_default()` | `cursor: default` |
| `.cursor_pointer()` | `cursor: pointer` |
| `.cursor_text()` | `cursor: text` |
| `.cursor_move()` | `cursor: move` |
| `.cursor_not_allowed()` | `cursor: not-allowed` |
| `.cursor_grab()` | `cursor: grab` |
| `.cursor_grabbing()` | `cursor: grabbing` |
| `.cursor_crosshair()` | `cursor: crosshair` |
| `.cursor_col_resize()` | `cursor: col-resize` |
| `.cursor_row_resize()` | `cursor: row-resize` |
| `.cursor_n_resize()` | `cursor: n-resize` |
| `.cursor_e_resize()` | `cursor: e-resize` |
| `.cursor_s_resize()` | `cursor: s-resize` |
| `.cursor_w_resize()` | `cursor: w-resize` |
| `.cursor_ew_resize()` | `cursor: ew-resize` |
| `.cursor_ns_resize()` | `cursor: ns-resize` |
| `.cursor_nesw_resize()` | `cursor: nesw-resize` |
| `.cursor_nwse_resize()` | `cursor: nwse-resize` |

---

## Opacity

| Method | CSS |
|--------|-----|
| `.opacity_0()` | `opacity: 0` |
| `.opacity_5()` | `opacity: 0.05` |
| `.opacity_10()` | `opacity: 0.1` |
| `.opacity_20()` | `opacity: 0.2` |
| `.opacity_25()` | `opacity: 0.25` |
| `.opacity_30()` | `opacity: 0.3` |
| `.opacity_40()` | `opacity: 0.4` |
| `.opacity_50()` | `opacity: 0.5` |
| `.opacity_60()` | `opacity: 0.6` |
| `.opacity_70()` | `opacity: 0.7` |
| `.opacity_75()` | `opacity: 0.75` |
| `.opacity_80()` | `opacity: 0.8` |
| `.opacity_90()` | `opacity: 0.9` |
| `.opacity_95()` | `opacity: 0.95` |
| `.opacity_100()` | `opacity: 1` |
| `.opacity(val)` | `opacity: {val}` |

---

## Interactive States

### Hover

```rust
.hover(|style| style.bg(rgb(0x2563eb)))
```

### Active (pressed)

```rust
.active(|style| style.bg(rgb(0x1d4ed8)))
```

### Focus

```rust
.track_focus(&focus_handle)
.focus(|style| style.border_color(rgb(0x3b82f6)))
```

### Focus-Visible (keyboard only)

```rust
.track_focus(&focus_handle)
.focus_visible(|style| style.border_2().border_color(rgb(0x3b82f6)))
```

### Group Hover

```rust
// Parent
.group("card")

// Child
.group_hover("card", |style| style.opacity_100())
```

### Outline (for focus states)

| Method | Description |
|--------|-------------|
| `.outline_none()` | Remove outline |
| `.outline_1()` | 1px outline |
| `.outline_2()` | 2px outline |
| `.outline_color(color)` | Set outline color |
| `.outline_offset(val)` | Set outline offset |

---

## Conditional Styling

### when

```rust
.when(condition, |this| this.bg(rgb(0x3b82f6)))
```

### when_some

```rust
.when_some(option, |this, value| this.bg(value))
```

### map

```rust
.map(|this| {
    if condition {
        this.bg(rgb(0xff0000))
    } else {
        this.bg(rgb(0x00ff00))
    }
})
```
