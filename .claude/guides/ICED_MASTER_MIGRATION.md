# Migrating iced_aksel to iced master (0.15-dev)

Branch: `iced-master`. Reference guide (written while porting iced_widgets to the same revision):
`../iced_widgets/.claude/guides/ICED_0.15_MIGRATION.md`; section numbers below (§n) point into it.
The master-ready skills `../iced_widgets/.claude/skills/{iced,iced-custom-widget}/SKILL.md`
describe the target idioms (the globally installed `iced-custom-widget` skill still targets 0.14).

## Status: done (2026-11-07)

The library, all eight examples, the README and the doc comments are ported to the pinned
revision. `cargo build`, `cargo fmt --check`, `cargo clippy --all-targets --all-features -- -D
warnings`, `cargo test` (incl. 50 doctests) and `cargo doc --workspace --no-deps` all pass.
The step-by-step plan below is kept as the record of what changed and as the checklist for the
next revision bump; see [Outcome](#outcome) at the end for what actually happened.

## Setup

- All six iced crates in the workspace `Cargo.toml` (`iced`, `iced_core`, `iced_graphics`,
  `iced_renderer`, `iced_tiny_skia`, `iced_wgpu`) are pinned to
  `git = "https://github.com/iced-rs/iced", rev = "8ff48c42b0d5188223501ea1ccc506fd5beaf36a"`
  (master head on 2026-11-06, the same revision iced_widgets pins). Bump all six together.
- Every feature we use still exists at that revision: `tokio`, `debug`, `web-colors`,
  `thread-pool`, `webgl`, `fira-sans`; `iced_renderer`'s `wgpu-bare`, `tiny-skia`, `geometry`.
- **Lockfile on Windows:** `Cargo.lock` is git-ignored, so every Windows checkout resolves it
  fresh. The resolution pulls `windows` 0.58 for `gpu-allocator` next to `wgpu-hal` 30's 0.62,
  and `wgpu-hal`'s dx12 backend then fails to compile (`cargo check`/`clippy` may pass from cache;
  `cargo doc` and `cargo build` of the examples fail). Fix once per checkout:
  `cargo update -p windows@0.58.0 --precise 0.62.2`. Linux/macOS CI is unaffected.
- First compile: `cargo check -p iced_aksel --lib` stopped at 16 errors; the second wave was
  only 5 more.

## Scope

Much smaller than iced_widgets: **one `Widget` impl** (`Chart` in `iced_aksel/src/lib.rs`), **no
overlays**, no operations, no clipboard, no text editing. The work is concentrated in:

| Area | Files | Changes |
|---|---|---|
| Widget trait | `lib.rs` | `Meta`, `diff(&mut self)`, drop `children`/`size`, `layout` writes the tree, `update` loses `clipboard`, `Layout<'_>` → `Layout` |
| Layout model | `lib.rs`, `axis.rs`, `memory.rs` | `Node` gone; axes + plot area become `Tree::empty()` children |
| Renderer trait | `render.rs`, every `Renderer<Font = Font>` bound | Font associated type gone |
| Text | `axis.rs` (3 `Text` literals), `shape/label.rs`, `plot.rs`, `render/cache/path.rs`, `render/cache/mesh/tessellation/text.rs` | `ellipsis`, `hint_factor`, `default_font`, line height default, cosmic-text fork |
| Theme | `style.rs` + examples | `extended_palette()` → `palette()`, seed `palette()` → `seed()` |
| `Element` | `lib.rs` `From<Chart> for Element`, README, examples | `From` impls removed upstream; `.boxed()` / `impl Widget` |
| Examples | 8 crates in `examples/` | `view` returns, palette, `pick_list` arg order, `checkbox`/`text_input` API |

Unaffected (verified at the pinned revision): `iced_graphics::mesh::{Mesh, Cache, Renderer,
Indexed, SolidVertex2D}`, `color::pack`, `geometry::Frame::fill_text`, `fill_quad`/`Quad`, the
`fallback::Renderer` impls of `core::Renderer`, `text::Renderer`, `mesh::Renderer` and
`geometry::Renderer`, and `iced_graphics::text::{font_system, to_attributes, to_shaping, to_wrap}`.

## Plan

### Step 1: Renderer trait and font generics (§8)

- `render.rs`: drop the `Font` parameter from `crate::Renderer`:
  `pub trait Renderer: core::Renderer + core::text::Renderer + geometry::Renderer + mesh::Renderer`.
  Fallback impl: `impl<A: Renderer, B: Renderer<Paragraph = A::Paragraph, Editor = A::Editor>>`
  (check whether the `Paragraph`/`Editor` equality bounds are still what `fallback` requires).
- Every `iced_core::text::Renderer<Font = iced_core::Font>` bound (`lib.rs:998`, `:1501`,
  `axis.rs`) → just `crate::Renderer` (it already implies `text::Renderer`).
- Add `'static` to `Renderer` where the compiler asks (#3392).
- `renderer.default_font()` → `renderer.font()`; `default_size()` → `text_size()`
  (`axis.rs:401,544,804`, `plot.rs:96`). Consider renaming `Context::default_font` to `font`
  (public API; keep a deprecated alias or note it in the changelog).

### Step 2: Widget trait mechanics (§1)

In `lib.rs`:

- `impl Meta for Chart<..> {}` (no blanket impl).
- Delete `children()`. Keep `size()`: it exists unchanged on master (only the separate
  `size_hint()`, which `Chart` never overrode, was removed).
- `diff(&mut self, tree)`: `tree.children.resize_with(self.state.axes().len() + 1, Tree::empty)`.
  This replaces `children()`; the axis count can change between frames, so it must run every diff.
- `update`: drop `_clipboard`, `Clipboard` import.
- `Layout<'_>` / `layout::Layout<'_>` → `Layout` (`lib.rs:861,869,1100,1270,1420`,
  `axis.rs:319`, `memory.rs:48`). `CacheSignature::new` can take `Layout` by value (it's `Copy`).
- Delete `impl From<Chart> for Element` (`lib.rs:1495`), matching iced's removal of all such
  impls. Users call `.boxed()` or pass `chart` straight into `container(..)` / `column![..]`.

### Step 3: Layout in the widget tree (§2)

`Chart::layout` builds `Node::with_children(bounds, [axis nodes.., plot node])`. Rewrite as:

- `let bounds = limits.resolve(self.width, self.height, Size::ZERO)` stays (check `resolve`'s
  signature and the `Fit` default, Step 6).
- For each axis `i`: `tree.children[i].size = axis_size; tree.children[i].translation =
  Vector::new(x, y)`. The plot area is `tree.children[axis_count]`. Then `tree.size = bounds`.
- `Axis::layout(&self, &Limits) -> Node` → `-> Size`. It uses `limits.min()`/`max()`, which are
  now public fields (`limits.min`, `limits.max`).
- `layout_horizontal_axis`/`layout_vertical_axis` return `(Size, Vector)` or write the child tree
  directly.
- Every read of a child layout needs the tree: `layout.children().nth(i)` and `layout.child(idx)`
  → `layout.iter(&tree.children).nth(i)` (yields `(Layout, &Tree)`). `get_plot_layout(layout)`
  and `draw_spine_corners` gain a `tree: &Tree` parameter. `update` and `mouse_interaction` have
  the tree; inside `update`, take what you need from `tree.children` before
  `tree.state.downcast_mut()`, or index children by rectangle.
- Alternative if the borrow juggling gets ugly: store `axis_bounds: Vec<Rectangle>` and
  `plot_bounds` in `Memory` during `layout` (the reference guide allows plain rectangles for parts
  only used for hit tests and drawing). Prefer tree children first; it keeps `Layout` the single
  source of truth.

### Step 4: Text (§8)

- `iced_core::text::Text { .. }` literals (`axis.rs:396,539,799`, `shape/label.rs:258`) gain
  `ellipsis: Ellipsis::None` and `hint_factor: renderer.hint_factor()` (or `None` where the text
  is measured for canvas/mesh rendering, which is unhinted).
- Line height: `LineHeight::default()` is now 1.375 (was 1.3). Our label styles set explicit line
  heights, so check `style.rs`/`axis/label.rs` defaults and decide whether to follow the renderer
  (`renderer.line_height()`) when unset.
- `render/cache/path.rs` `frame.fill_text(Text { .. })`: the geometry `Text` may have gained
  fields (`ellipsis`, `wrapping`); fill them from the primitive.
- `render/cache/mesh/tessellation/text.rs` on the cosmic-text fork (rev `6adc236`):
  - `set_size(font_system, ..)` → `set_size(..)`, `set_wrap(font_system, ..)` → `set_wrap(..)`,
    `set_text(font_system, text, attrs, shaping, None)` → `set_text(text, attrs, shaping, None)`.
  - These setters are now lazy (dirty flags). Our `borrow_with(font_system).layout_runs()` calls
    `shape_until_scroll` first, so layout runs stay correct; don't switch to the bare
    `Buffer::layout_runs()`, which no longer shapes.
  - Re-check `font_system.db().face(..)`, `get_font(id, weight)` and `LayoutGlyph` fields against
    the fork.

### Step 5: Theme and palette (§10)

- `style.rs:180` `theme.extended_palette()` → `theme.palette()`; `palette::Extended` →
  `palette::Palette`. Field access (`background.strong.text`, `primary.base.color`) is unchanged.
- Generated colours changed: `primary.strong` is lighter than `primary.base` and its text is dark.
  Review every `.strong` use in `style.rs` for marker/tooltip label contrast.
- Doc examples / README / examples that use the old seed `theme.palette().primary` → 
  `theme.seed().primary` (or the extended `theme.palette().primary.base.color`, which is what most
  chart colours should use). 77 `palette` and 6 `extended_palette` uses in `examples/`, plus
  `plot.rs:44` and `README.md:120`.

### Step 6: Lengths and limits (§7)

- `Limits::min()/max()` → fields. `Limits::new` still exists.
- Default `width`/`height` are `Fill` (`lib.rs:217`). A chart has no intrinsic size, so `Fill`
  stays right; just make sure nothing uses `Shrink` meaning "intrinsic".
- Inside a `scrollable`, the scrolled axis now arrives as a finite `max` with
  `limits.infinite = true`. A `Fill` chart there should resolve to something sane (iced resolves
  `Fill` on an infinite axis to `max.max(intrinsic)`); test a chart in a `scrollable`.

### Step 7: Events and cursor (§12)

- `mouse::Cursor` gained `Obstructed` and `Levitating`; we only use `position()`/`position_over`,
  which handle them (`None` when obstructed). Check for exhaustive matches.
- A left press no widget captures now starts a text selection and makes every cursor move run a
  whole-tree `operate`. `handle_mouse_press` should `shell.capture_event()` for presses the chart
  acts on (drag/pan, clicks with handlers). Today only `WheelScrolled` captures.
- `RedrawRequested(Instant)` is snapped to the refresh grid: use `saturating_duration_since` in
  any animation code (spectrum/dashboard examples using `window::frames()`).

### Step 8: Library docs and README

- README is the crate docs (`build.rs` embeds it), so its code blocks are doctests:
  `fn view(&self) -> Element<Message> { ... .into() }` → `impl iced::Widget<Message> + '_` or
  `.boxed()`; palette calls as in Step 5.
- Same for doc comments in `lib.rs`, `plot.rs`, `axis/label.rs`, `shape/*`.
- `cargo test --doc -p iced_aksel` must pass.

### Step 9: Examples (§15, §11)

Per example (`core_axes`, `core_interactivity`, `core_scales`, `core_shapes`, `core_template`,
`showcase_candlestick`, `showcase_dashboard`, `showcase_spectrum`):

- `view` → `impl Widget<Message> + '_`, or `Element` built with `.boxed()`; no `.into()`.
  Helper fns returning `Element` (`bar.rs`, `gauge.rs`, `line.rs`, `chart.rs`) follow suit.
- `pick_list(options, selected, on_select)` → `pick_list(Some(selected), options,
  T::to_string).on_select(f)` (9 uses).
- `checkbox` (7), `text_input` (4), `slider` (2), `space` (2): check renamed setters
  (`text_line_height` → `line_height`, etc.).
- App builder: `.default_font` → `.font`, `.font(bytes)` → `.fonts([bytes])`; `Settings` renames.
- wasm targets: `cargo check --target wasm32-unknown-unknown` for one example to confirm the
  `webgl`/`fira-sans` path still builds.

### Step 10: Behavioural review (things that compile silently)

Run each example and compare with `main`:

- [ ] Axis label text height/baseline (line height 1.3 → 1.375; hinting off at this revision).
- [ ] Axis thickness and tick label placement unchanged.
- [ ] Colours from the new palette generator (markers, tooltips, crosshair labels).
- [ ] Mesh (wgpu) vs path (tiny-skia) backends both render: force tiny-skia with
  `ICED_BACKEND=tiny-skia`.
- [ ] Drag/pan/zoom and hover enter/exit still fire; no stray text-selection behaviour on press.
- [ ] Chart inside a `scrollable` and inside a `Fit`-sized container.
- [ ] Performance: tessellated text caches still hit (lazy cosmic-text shaping shouldn't reshape
  per frame).

### Step 11: CI and wrap-up

- `.github/workflows/check.yml`: `cargo clippy` with the workspace `nursery`/`style` deny lints
  will flag new things (`too_many_arguments`, `missing_const_for_fn`); fix, don't blanket-allow.
- `cargo nextest run`, `cargo test --doc`, `cargo doc --no-deps`.
- `flake.nix`: the git dependency needs network access in the Nix build; check `flake-check.yml`
  (may need `outputHashes` for the iced git source and the cosmic-text/cryoglyph forks).
- Bump crate version to a pre-release (e.g. `0.4.0-dev`); a git dependency can't be published
  to crates.io, so this branch stays unpublished until iced 0.15 ships.

## Order of work

Steps 1–2 together (they clear the first 16 errors), then 3 (the only structural change), 4–7 as
the compiler drives, then 8–9, then 10–11. Commit after each step that compiles, so the library
is green before the examples are touched.

## Outcome

What the port actually needed, versus the plan above:

- **`Widget::size` stays.** The plan above first misread "delete `size_hint()`" as deleting
  `size()`; 0.14 had both, master only drops `size_hint()`. `Chart` keeps `size()`.
- **Layout:** the axes and plot area are `Tree::empty()` children resized in `diff` to the
  current axis count (`layout` repeats the resize defensively before indexing). A private `ChartLayout<'a> { layout, children }`
  view (`lib.rs`) replaces `layout.children().nth(i)` / `get_plot_layout` with `.axis(i)` /
  `.plot()`, so every helper kept its shape. `Axis::layout` returns a `Size`;
  `CacheSignature::new` takes the bounds.
- **`From<Chart> for Element` removed** (iced's `Element::new` is private now). Users call
  `.boxed()` or pass the chart to `container(..)`/`column![..]` directly. This is the one
  breaking public API change; the README shows `fn view(&self) -> impl Widget<Message> + '_`.
- **`crate::Renderer`** lost its `Font` parameter; `renderer.default_font()` → `font()`.
  `plot::Context::default_font()` keeps its name.
- **Text:** axis labels set `hint_factor: renderer.hint_factor()` (drawn with
  `text::Renderer::fill_text`); shape labels measure with `hint_factor: None` (they are drawn as
  meshes/paths). The path backend now passes the primitive's `wrapping` to `Frame::fill_text`
  (the old comment said tiny-skia couldn't wrap). cosmic-text: dropped `font_system` from
  `set_size`/`set_wrap`/`set_text`; `layout_runs` stays on the `borrow_with` wrapper, which shapes.
- **Palette:** `style.rs` uses `theme.palette()` (ex-`extended_palette`); its `primary.base`
  colours are unchanged. Examples that used the old seed `theme.palette().x` now call
  `theme.seed().x`, which is the same seed colour.
- **Events:** presses on the plot and axes were already captured; nothing changed.
- **Examples:** `.into()` → `.boxed()` where a fn returns `Element`; dashboard helpers take
  `impl Widget<Message> + 'a` and return `Container<'a, Element<'a, Message>>`; `pick_list` uses
  `(selected, options, to_string).on_select(f)`; `Font::with_name` → `Font::new`.

Behavioural review (headless `iced_test` renders compared against `main`, then removed):

- [x] `core_axes`: axis labels, ticks, badges and grid identical; only iced's own controls are a
  few px taller (line height 1.375). The clipped top "100.00" label is the same on `main`.
- [x] `core_shapes` on wgpu (mesh) and tiny-skia (path): identical to `main`, including wrapped,
  rotated and zoom-scaled labels.
- [x] `core_interactivity`: background press → `BackgroundPressed`; press on the rectangle →
  `ShapeEnter` + `ShapeSelected`; dragging it emits `ShapeDragged` per move.
- [x] wasm: `core_template` checks for `wasm32-unknown-unknown`. Examples that use `rand` 0.10
  fail on wasm because `getrandom` 0.4 lacks the `wasm_js` feature; the same happens on `main` and
  is unrelated to iced (the examples only enable `wasm_js` for `getrandom` 0.3).
- [ ] Not checked: live animation timing in `showcase_dashboard`/`showcase_spectrum` (needs a
  window). The frame-clock instant can now be slightly in the past (§5). The examples' `now -
  last` (dashboard) and `duration_since` (spectrum) saturate to zero in today's std, and
  `showcase_spectrum` skips `dt == 0`, so at worst one frame advances by zero. std reserves the
  right to panic there again; switching both to `saturating_duration_since` would make it explicit
  (`gauge.rs` seeds `last_tick` with `Instant::now()`, then subtracts a frame instant from it).

## Findings shared with the iced_widgets guide

These were added to `../iced_widgets/.claude/guides/ICED_0.15_MIGRATION.md` (§n references
above point there):

- `size()` stays in the `Widget` recipe (§1). Synthetic-only children are created in `diff`, and a
  `Copy` layout-plus-children view keeps old `layout.children()` helpers intact (§1).
- The Windows `windows` 0.58/0.62 lockfile split can pass `check`/`clippy` and still fail
  `doc`/`build`. Git-ignored lockfiles hit it on every fresh checkout (header).
- `geometry::Text` literals need `wrapping` + `ellipsis`; renderer traits with a font parameter
  drop it, and the fallback impl keeps the `Paragraph`/`Editor` equality bounds (§8).
- The palette rename must run `.palette()` → `.seed()` before `.extended_palette()` →
  `.palette()`, and only once (§10).
- cosmic-text setters lost `font_system`; bare `Buffer::layout_runs()` doesn't shape, so use
  `borrow_with(..).layout_runs()` (Lessons / Text).
- `Instant` subtraction saturates in today's std, but stay explicit (§5).
- Headless tests pick the renderer with `ICED_TEST_BACKEND`; a worktree with `iced_test = "0.14"`
  gives before/after snapshots (§14).
