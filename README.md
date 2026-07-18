# plumage

[![CI](https://github.com/martian56/plumage/actions/workflows/ci.yml/badge.svg)](https://github.com/martian56/plumage/actions/workflows/ci.yml)

A terminal UI framework for [Raven](https://github.com/martian56/raven).
Build full-screen, keyboard and mouse driven apps that render flicker-free,
from a model and three functions.

plumage owns the terminal, the event loop, and the drawing. You describe
what the screen should look like for a given model; it figures out which
cells changed since the last frame and repaints only those.

```rust
import "github.com/martian56/plumage" { App, Event, Frame, Step, next, quit, run }
import "github.com/martian56/plumage/widgets/paragraph" { Paragraph }
import "github.com/martian56/perch/keys" { Key }

struct Model {
    count: Int,
}

fun init() -> Model {
    return Model { count: 0 }
}

fun update(m: Model, e: Event) -> Step<Model> {
    match e {
        KeyPress(k) -> match k {
            Char(c) -> {
                if c == "q" {
                    return quit(m)
                }
                return next(Model { count: m.count + 1 })
            },
            _ -> {},
        },
        _ -> {},
    }
    return next(m)
}

fun view(m: Model, f: Frame) {
    Paragraph.new("pressed ${m.count} keys, q quits").render(f.area(), f.buf)
}

fun main() {
    run<Model>(App { init: init, update: update, view: view, tick_ms: 200 })
}
```

## Install

```
rvpm add github.com/martian56/plumage@v0.2.0
rvpm add github.com/martian56/perch@v0.2.0
```

perch is plumage's terminal layer; your code imports its `Key` and mouse
types to match on input, which is why it sits in your manifest too.
Building needs a C toolchain, the same one Raven already uses to link.

## How an app works

Three functions around one model type:

- `init() -> M` builds the starting model.
- `update(m, event) -> Step<M>` is the only place state changes. Return
  `next(model)` to keep going or `quit(model)` to stop.
- `view(m, frame)` describes one frame by rendering widgets into regions.

`Event` covers `KeyPress(Key)`, `Mouse(MouseEvent)`, `Paste(String)`,
`Resize(w, h)`, and `Tick`. Ticks fire whenever `tick_ms` passes without
input, which is what makes spinners spin and gauges move while the app sits
idle. Ctrl+C is not special: it arrives as `Ctrl("c")` and quitting is your
decision.

`Paste` is one whole bracketed paste with newlines normalized to `\n`: the
run loop switches the terminal mode on, so pasted text reaches `update` as
a single event instead of a keystroke stream where a pasted newline would
act as Enter. Insert it into a `TextInput` with `input.insert(text)`:

```rust
Paste(text) -> {
    m.entry.insert(text)
    return next(m)
},
```

## Layout

Split any rect into rows or columns with constraints, then render into the
pieces:

```rust
import "github.com/martian56/plumage/layout" { Constraint, split_v }

let rows = split_v(f.area(), [
    Constraint.Len(1),     // exactly one row
    Constraint.Fill(1),    // everything left over
    Constraint.Len(1),
])
```

`Len` is exact, `Pct` is a share of the whole, `Fill(weight)` divides the
leftovers, `Min(n)` is a floor plus one share. Splits nest: split a column
of a split to build grids.

## Widgets

| Widget | Import under `plumage/widgets/` | What it does |
|---|---|---|
| `Block` | `block` | Borders (plain, rounded, double, thick) and a title; `inner(area)` is where content goes |
| `Paragraph` | `paragraph` | Styled multi-line text, alignment, word wrap, scrolling |
| `ListView` | `list` | Selectable list that scrolls to keep the selection visible |
| `Table` | `table` | Header plus rows, content-sized columns, selectable |
| `Tabs` | `tabs` | One-row tab bar |
| `Gauge` | `gauge` | Progress bar with a centered label |
| `TextInput` | `input` | Single-line editor: cursor, insert, backspace, delete, arrows, home, end; pasted newlines render as a return glyph while the value keeps them |
| `Scrollbar` | `scrollbar` | Proportional vertical thumb |
| `Spinner` | `spinner` | Braille activity indicator driven by ticks |

Widgets are values built fresh from your model each frame; selection and
scroll positions are plain `Int`s you keep in the model. `TextInput` is the
one stateful exception: keep it in your model and feed keys to
`handle_key`. Every widget draws with
`widget.render(area, f.buf)`.

Styling lives in `plumage/style`: named colors, the 256-color palette,
RGB, and the usual attributes, as immutable builders:

```rust
Style.new().fg(Color.Cyan).bold()
Style.new().bg(Color.Rgb(30, 30, 46))
```

Text is measured in terminal columns, not bytes, so CJK and emoji line up;
that logic is [wingspan](https://github.com/martian56/wingspan), usable on
its own.

## Demos

The repo doubles as a runnable demo. In a real terminal:

```
rvpm run -- dashboard   # tabs, animated gauges, a table, a spinner
rvpm run -- todo        # text input, list with selection and toggles
```

Their sources under `examples/` are the recommended reading order after
this file.

## The stack

plumage is the top of three small packages, each useful alone:

- [wingspan](https://github.com/martian56/wingspan): display width for
  terminal text. Pure Raven.
- [perch](https://github.com/martian56/perch): raw mode, decoded key and
  mouse events, terminal size, unbuffered writes. One small C shim,
  everything above it Raven.
- plumage: buffers, layout, styles, widgets, and the runtime. Pure Raven.

## Known limits

No grapheme clustering: a ZWJ emoji sequence measures as the sum of its
parts. Alt combinations decode as the plain key. Mouse support needs a
terminal that speaks SGR mouse reporting, which is every mainstream
emulator and Windows Terminal. Legacy conhost without VT processing is not
supported.

## Development

```
rvpm build    # type-check the library and compile the demo
rvpm test     # buffer, layout, text, and widget tests, no terminal needed
rvpm fmt
```

Design notes live in [docs/design.md](docs/design.md).

## License

MIT
