# plumage design

Status: implemented in v0.1.0. This records the decisions and the reasons,
so future changes argue with the reasons instead of guessing.

## Goal

A terminal UI framework for Raven that can carry a full application:
keyboard and mouse input, layout, a widget set, animation, and resize
handling, with flicker-free drawing.

## Shape: model, update, view

An application is one model type and three functions:

    init() -> M
    update(M, Event) -> Step<M>
    view(M, Frame)

update is the only place state changes. view is a pure description of one
frame, rebuilt from the model every time. The runtime (lib.rv) owns the
terminal, the loop, and the drawing.

Why this shape: Raven's strengths line up with it. Enums with payloads and
exhaustive match make the Event type safe to extend. Structs are cheap to
rebuild, so reconstructing widgets each frame costs little and removes a
whole class of stale-state bugs. And because the runtime polls rather than
registering callbacks, nothing depends on the C FFI holding Raven closures
alive across frames.

Step<M> is a struct (model plus a quit flag, built with next(m) or
quit(m)), not an enum. A payload-less variant of a generic enum cannot be
inferred at the call site in current Raven, so an enum here would force
type annotations on every return.

## The stack

Three packages, each useful without the ones above it:

- wingspan: display width for text. Pure Raven.
- perch: raw mode, key and mouse decoding, terminal size, unbuffered
  writes. The only native code in the stack, one C shim.
- plumage: buffer, layout, style, text, widgets, runtime.

The split exists because the lower layers answer questions other programs
ask too. raven-table can measure CJK correctly with wingspan; an installer
prompt can read arrows with perch and never think about buffers.

## Rendering: diffed cell grid

Widgets draw into a Buffer of cells (character plus style). Each frame the
runtime diffs the new buffer against the previous one and emits ANSI only
for cells that changed, skipping cursor moves while runs stay contiguous
and repeating styles only when they change. First frame and resizes do a
full clear and repaint.

Wide characters occupy two cells; the second holds an empty marker the
renderer skips. Width comes from wingspan, so the buffer, the widgets, and
the wrap logic never disagree about how many columns text takes.

## Events and timing

perch delivers InputEvent (KeyPress, Mouse, Idle, Closed) from one
read_event call with a timeout. The runtime maps that onto the app Event:
Idle becomes Tick, which drives animation; a size change becomes Resize;
Closed ends the loop. There are no threads and no channels: one loop, one
timeout. Raven has goroutines, but its channels carry only Int today, and
a poll loop needs neither.

## Widgets

Stateless display widgets are built fresh each frame from the model
(Block, Paragraph, ListView, Table, Tabs, Gauge, Scrollbar, Spinner).
Selection and scroll positions are plain Ints in the model. TextInput is
the one stateful widget: it owns its value and cursor and edits itself
through handle_key, because line editing is fiddly enough that every app
would otherwise reimplement it.

Widgets implement the Widget trait and are called directly:
widget.render(area, f.buf). There is no generic Frame.render helper:
Raven's checker does not currently resolve trait bounds for impls that
live in another module, so a generic call site cannot prove Block: Widget.
Direct method calls dispatch fine. If cross-module bounds land in the
compiler, the sugar can come back without breaking anything.

## Compiler notes that shaped the code

Recorded here because they will bite any contributor who does not know:

- Lambdas are expression-bodied; a return inside a lambda block targets
  the enclosing function. Named top-level functions are the way to fill
  App's fields.
- Generic type arguments are explicit at call sites that cannot infer
  them: run<Model>(app).
- An enum variant name that collides with an imported type name loses;
  that is why perch says KeyPress(Key) instead of Key(Key).
- A Unit function whose last statement is an expression with a value needs
  a bare return after it.
- Empty list literals need a typed binding: let xs: List<Int> = [].

## Out of scope for v0.1.0

Grapheme clustering (ZWJ emoji measure as their parts), themes, focus
management beyond what apps do by hand, scrolling regions, double-width
borders, Windows legacy conhost without VT support.
