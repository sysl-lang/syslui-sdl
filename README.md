# syslui-sdl

**The window, the frame loop and the events for [syslUI](https://github.com/sysl-lang/syslui) —
one driver, for a desktop and for a phone.**

```
dependencies {
  syslui-sdl { git = "github.com/sysl-lang/syslui-sdl", version = "0.2.15" }
}
```

**One, because imports are transitive** — naming this package reaches syslUI, SDL3 and PlutoVG
through it, so a manifest names what a project takes rather than everything it can see. That needs
**sysl 0.0.73 or newer**; on 0.0.72 a program had to name all four, which is what prompted the
language change.

## What an application is

```
import sh.sysl.ui.*
import sh.sysl.ui_sdl.{app, run}

val count: &Signal[int] = signal(0)

screen() -> &View =
    column(spacing = 10):
        text(s"count: ${count.read()}")
        button("increment", () -> count.set(count.read() + 1))
        text_field(name, 1, 260, "your name")

main(args: []string) -> Result[unit, string] =
    run(app("Counter", () -> screen(), () -> 0x12141C))
```

The same program on Android is the same call under a different name, because Android looks up
`SDL_main` rather than calling `main`:

```
@export("SDL_main")
sdl_main(argc: i32, argv: **u8) -> i32
    run(app("Counter", () -> screen(), () -> 0x12141C))
    0
```

## Why it is one driver and not two

Measured on the two applications that exist. Of the ~120 lines an Android frame loop took, **about
seventeen were genuinely platform-specific**: the entry point, the system bars, and one hint about
orientation. Everything else was identical — and most of what *looked* like phone code was a desktop
program taking shortcuts a fixed-size, density-unaware window allows.

| what | really per-platform? |
|---|---|
| SDL init, window, renderer, event pump, frame | **no** — identical |
| density → canvas scale, touch → points | **no** — a retina Mac is 2.0 |
| surface and texture rebuilt on resize | **no** — any resizable window |
| `start`/`stop_text_input` on focus | **no** — needed on a desktop too, and `stop` is a no-op there |
| `@export("SDL_main")` instead of `main` | **yes** |
| the JNI method reporting the system bars | **yes**, and it is `sysl-lang/skitter`'s now |
| `SDL_ORIENTATIONS` + `WINDOW_RESIZABLE` | **yes**, and Skitter's too |

Two packages would be two copies of one loop kept in sync by hand.

**Two of those three rows moved out of this package at 0.2.0**, which is worth reading as a
correction to the measurement rather than a change of mind about it. Seventeen platform-specific
lines was right; what was wrong was assuming they had to live in the *driver*. The system bars and
the orientation pair are facts about Android, not about SDL or about drawing, and
[`skitter`](https://github.com/sysl-lang/skitter) is where they belong — so this package is down to
the one row that genuinely cannot move, the entry point, which is the application's anyway.

**From 0.2.8 skitter is taken only on Android** (`if = "android"` in the manifest, `#if android` around
the imports), so a desktop program's binary has none of its JNI exports.

## Why it is not part of syslUI

**A link directive is never pruned.** Putting SDL3 in the toolkit's own manifest would put `-lSDL3`
on every consumer's link line, including a panel driver on a microcontroller with no window system at
all. That is the same argument that makes SDL3 four packages in this org rather than one, and it is
why `sh.sysl.ui` imports nothing but `sysl.*` and PlutoVG.

## The three things SDL does not make platform-independent

SDL does nearly all of it: a tap is a mouse event, typed text is a `TextInput`, the renderer and the
texture are the same call. Three things are left, and all three are handled here:

- **The density.** The tree is built and measured in *points*; the canvas is scaled before anything
  is drawn, so a rounded corner and a glyph are rasterized at the pixel size they end up at. Scaling
  the finished picture instead is one bilinear blur over the whole interface. **A pointer is reported
  in window units and the drawing is in pixels**, and which convention a platform uses is the thing
  nothing documents the same way twice — so the ratio is measured from `size_in_pixels` against
  `size` rather than assumed.
- **The system bars.** From Android API 35 an app draws edge to edge whether it asks to or not.
  `window.safe_area()` is the obvious answer and the wrong rectangle: SDL builds it from five inset
  types at once because it answers *where can a button go*, which on a gesture-navigation phone is 78
  pixels off each side. So the Java side reports `WindowInsets.Type.systemBars()` — and as of 0.2.0
  this driver reads them straight from Skitter, once a frame, at the one place they are used. **An
  application forwards nothing.** All zero on a desktop, which is the honest answer for a program
  with a whole window.
- **The font.** PlutoVG rasterizes glyphs itself and wants a file; there is no fontconfig on a phone
  and no API anywhere that answers "a sans-serif face, please". `system_font()` tries the paths the
  common platforms use, and an application that cares passes its own.

## Scrolling: a finger, the wheel and a trackpad (since 0.2.2)

syslUI 0.1.5's `scroll` follows a dragged finger and the wheel, and this driver feeds it both:

- **A tap lands at the release.** The left button coming up calls the canvas's `release`, which taps
  only if the press was not taken as a drag — so a finger scrolling a list of buttons presses none of
  them. On a phone a finger *is* the mouse here, since SDL synthesises mouse events from touch.
- **Only the left button reaches a control.** The right and middle buttons reach a view that asks for
  them through `Canvas.held`, `grab_by` and `drag_by` (syslUI's README has them), and a side button
  reaches nothing. `mouse_down` and `mouse_up` are the mapping, and the tests send them events through
  a recorder.
- **`Canvas.modifiers` is SDL's modifier state, read once a frame**, so a key pressed or let go during
  a drag shows in the next frame. Either side of the keyboard counts; Command or the Windows key is
  `cmd`; Caps Lock and Num Lock are not modifiers. `modifiers_of` is the conversion.
- **A trackpad's scroll cannot be told from a wheel's, because SDL does not say**: its wheel event
  carries amounts and a direction and no kind of device. Both are `Canvas.wheel`.
- **`SDL_EVENT_MOUSE_WHEEL` becomes points**: one notch is 40, and a trackpad's fractional deltas pass
  through as fractions, which the canvas adds up so a slow glide still moves. `wheel_points` is the
  conversion, and the tests reach it as a function.
- **SDL's `direction` is not read, on purpose.** `FLIPPED` means the platform has already applied
  natural scrolling, so the values are the ones the person chose; flipping them back would undo it.
- **A Mac trackpad glides on after the fingers lift**, because the driver sets
  `SDL_MAC_SCROLL_MOMENTUM` before `init`. A dragged finger does not: there is no fling yet.

## What stays with the application

- **The entry point** — `main`, or an `@export("SDL_main")`.
- **~~The JNI method that reports the insets.~~ Not any more, and it is the reason for 0.2.0.** Its
  symbol is mangled from a class's package, so while the activity had to be renamed into each
  application's package the export had to be hand-written to match — and this driver kept four
  `int`s and a `set_insets` for it to call. Skitter fixes the activity at
  `sh.sysl.skitter.SkitterActivity`, which fixes the symbol, which lets the bridge be a library's.
  **A program upgrading from 0.1.1 deletes its export and its `set_insets` call together**; there is
  nothing to replace them with, which is the point.
- **Its shortcuts.** `on_event` sees every event before the driver does and answers `true` to say it
  has dealt with one — which is where ⌘C and escape go. A driver that guessed at those would be
  wrong for the second program that used it. The wheel is the driver's since 0.2.2, because it
  scrolls a list; a program wanting it for something else answers `true` to it here.
- **Its own work, once a frame.** `on_frame` (since 0.2.1) is handed the frame's step in seconds —
  clamped, like the animations' — after the events and before the loop decides whether to rebuild,
  so a signal written there rebuilds the tree on the same frame. It is for a program that is not
  driven by events: a tuner reading the microphone, a game stepping its world. It runs on the loop's
  thread, so it may write signals freely, and it must not block — a frame is about sixteen
  milliseconds.

  ```
  run(app("Tuner", () -> screen(), () -> 0x12141C, on_frame = dt -> listen(mic, dt)))
  ```
- **What it gives back while it is away.** `on_pause` and `on_resume` (since 0.2.3) run once each
  way when the program leaves the screen and comes back — on Android, the Home button, another app
  in front, the screen going off. A tuner closes its recording stream in one and opens it in the
  other, so the system's microphone indicator goes out while it is in the background. SDL sends each
  direction as a will/did pair and the hook belongs to whichever half arrives first; `Terminating`
  pauses too, if nothing paused first. **Nothing is drawn and `on_frame` is not called in between**:
  under SDL's default `SDL_ANDROID_BLOCK_ON_PAUSE` the loop is parked in the event pump anyway, and
  with that hint off the pump wakes ten times a second and those wake-ups are not frames.
  **A desktop never fires either**: minimising a window is `WindowMinimized`, not a pause, and the
  program keeps running — `on_event` sees it if a program wants to act on it.

  ```
  run(app("Tuner", () -> screen(), () -> 0x12141C, on_frame = dt -> listen(mic, dt),
          on_pause = () -> mic.close(), on_resume = () -> mic.open()))
  ```

## Drawing 3D in part of the window (since 0.2.12)

An application opts in by naming the shader formats it supplies. Without `gpu`, it has today's window
and renderer and no GPU device.

```
val view = viewport(f ->
    val pass = f.pass(0.1, 0.1, 0.1).unwrap()

    pass.bind_pipeline(pipeline)
    pass.draw(36)
    pass.finish()
, (c, r) ->
    val (dx, dy) = c.drag_by(r, 1, PointerButton.Primary, 2, 2)
)

run(app("Cube", () -> row([view.flex(), panel]), ground, gpu = GPU_SHADERFORMAT_MSL))
```

- **Make the viewport once, outside the tree function**, and place the same one at every rebuild: it
  holds the targets. It takes whatever size layout offers.
- **`draw` gets a `ViewportFrame`**: `device`, `cmd`, `color`, `depth` and `rect`, its place in the window
  in pixels. `color` and `depth` are that size and are remade when it changes. `pass(r, g, b)` opens a
  pass over both, cleared. Record into `cmd` and leave it open; the driver submits it. A pipeline's
  formats are `VIEWPORT_COLOR_FORMAT` and `viewport_depth_format(device)`.
- **`input` gets the canvas and the rectangle in points**, as it is painted, so `held`, `modifiers`,
  `grab_by`, `drag_by` and `wheel` work as they do in any view. A grab and a drag on the same slot are
  alternatives: a slot claimed by `grab_by` does not then report movement through `drag_by`.
- **The interface composites over the 3D with premultiplied alpha.** PlutoVG's surface really is
  premultiplied: half-white over nothing has every channel equal to the alpha, and no soft edge has a
  channel above its alpha, both asserted. A viewport cuts its rectangle out of the surface when it is
  painted, on whole device pixels. What the tree painted before it is gone from there, so a background
  behind it does not hide it; what is painted after it lands on top; a clip in force limits the hole.
- **Such a window asks for the display's real density**, which an application that did not opt in does
  not get on a Mac.
- **`save_frame(file)`** writes the next frame to a PNG, read back from the renderer, in either kind of
  application. **`frame_timing()`** says where the last frame's time went.

**Time on a 2880 by 1800 window on an M-series laptop's 120 Hz display**, from `examples/cube`, averaged
over 300 frames: interface repaint 3.0 ms, upload 2.2 ms, recording the 3D 0.05 ms, and 3.8 ms in present,
which includes the wait for the display; 9.2 ms a frame. The repaint and upload are of the whole surface
every frame, so they scale with the window and not with what changed.

```
cd examples/cube
sysl build . --lib ../..
./cube
./cube --frames 300 --shot frame.png
```

The example's shader is Metal Shading Language, so it runs on a Mac.

## Driving a running application over a socket (since 0.2.15)

Any application can be driven from outside, the way a browser is through its developer tools: HTTP on a
private Unix socket, with `curl` as the whole client. Name a socket and `run` opens it; name none and
nothing listens.

```sysl
run(app("Editor", tree, ground, control = "/tmp/editor.sock"))
```

```sh
S=/tmp/editor.sock
curl --unix-socket $S http://app/snapshot
curl --unix-socket $S http://app/click -d x=1320 -d y=95
curl --unix-socket $S http://app/type --data-urlencode "text=hello"
curl --unix-socket $S http://app/shot -o shot.png
```

- **`GET /snapshot`** is syslUI's snapshot of the frame last painted, as its JSON: every control on screen
  with its role, label, name, value, rectangle, state and the point a click should land on. `.named("x")`
  gives a control a `ref` that holds across rebuilds. It is 503 before the first frame and 405 for any
  method but `GET`.
- **`POST /click?ref=<key>`** (since 0.2.16) clicks the control the snapshot calls `key`: its name, or
  the `@n` reference. The middle of what shows of it is converted to the window coordinate `/click`
  takes, and the request goes on to sdl3's own `/click`, so the pointer glides, presses and the answer
  comes when the release is done; `button` and `clicks` are passed on. An unknown key is 404 naming it,
  a key that names two controls or one under something raised above it is 409 with the reason, and
  before the first frame it is 503. A `/click` with `x` and `y` and no `ref` is untouched.
  `curl --unix-socket $S -X POST 'http://app/click?ref=colour'`. There is no `/type?ref=`: sdl3 can
  forward one request and `/type` types where the focus is, so click by `ref`, then `POST /type`.
- **Everything else is `sh.sysl.sdl3.control`'s** — `/click`, `/drag`, `/key`, `/type`, `/wait`, `/state`
  and `/shot`; its README has the table. Its input goes in through SDL's own event queue at a pace a person
  can follow, so the application cannot tell it from a mouse and keyboard.
- **The pointer is drawn over the frame**: a magenta dot ringed in white where it is, and a square round it
  while a button is held. Input pushed into SDL's queue moves nothing on the screen, so without this a
  person watching would see controls react to nothing. A `/shot` shows it too. **Without a socket nothing
  is drawn**, and a window under control looks exactly as before until the first command moves the pointer.
- **A snapshot is taken every frame while a socket is open**, right after the frame is painted — a second
  paint pass through a canvas that draws nothing. An application without a socket never takes one. **A
  viewport sits that pass out** (since 0.2.17): it neither cuts its hole in the interface again, which
  would clear a menu raised over it, nor runs its `input` hook, which would see the pointer let go and end
  a drag every frame.
- **The application's own routes** (since 0.2.17): `app(..., routes = req -> ...)` answers a
  `sh.sysl.sdl3.control.Request` with a `Routed`. It is asked **first**, before `/snapshot`, `/click?ref=`
  and sdl3's built-ins: `Respond(...)` answers, `Forward(req.rewritten(...))` hands a rewritten request
  on, and `Pass` (the default) lets the request through to the rest. A path of syslui-sdl's own can be
  answered by the application this way, so a route should match only what it means to.

  ```sysl
  val routes: &Fn(Request) -> Routed = req ->
      if req.path == "/editor/state" then Respond(json(200, state_json())) else Pass
  run(app("Editor", tree, ground, control = "/tmp/editor.sock", routes = routes))
  ```
- **A hidden window** (since 0.2.17): `app(..., hidden = true)` creates the window with SDL's hidden flag,
  for a controlled run that should not appear on screen. It paints every frame as a visible one does, and
  `/shot` reads the frame back; sdl3's `/state` says `"hidden":true`. The default is a visible window.
- **Coordinates.** `/click` and `/drag` take window coordinates, which is what SDL's mouse events carry;
  a snapshot is in the interface's points. They agree when the window asked for the display's density (an
  application with a `gpu`) or when the display scale is 1. Where they do not, divide a snapshot's point
  by the ratio the loop turns the pointer into points with (the display scale, when a window is in pixels).
  `/click?ref=` does this for you.
- **The socket's path must be short**: a Unix socket's address holds about a hundred bytes, so a path deep
  in a temporary directory is refused at `listen` with "too long for a socket's address".

`examples/cube` takes `--control <socket>`; its button is named `colour` and retitles the window, and its
View menu (`menu.View`) drops down over the viewport. `--hidden` runs it with the window hidden, and
`--still` stops the cube turning by itself, so two runs draw the same frame.

## The window, for a dialog or a title (since 0.2.15)

**`window()`** answers the window `run` opened while it runs, and `None` before and after: what sdl3's file
dialogs and message boxes take as their parent. **`set_title(title)`** retitles it, and answers `false`
when there is no window.

```sysl
show_message("Exported", "The file was written.", Information, window())
_ = set_title("Editor — untitled")
```

## Tests

```
sysl test .
```

**The viewport's tests** check the layout's rectangle, pointer input, what PlutoVG's surface holds, and,
against a real device with no window, the composite by exact pixels: the premultiplied equation on every
pixel of a frame with text on it, the panel's edge at twice the density, a clip, a fractional scale, the
depth target's resizes and its effect, and a button changing what is drawn. **The GPU ones run only where
`launchctl managername` says `Aqua`**, asked before SDL starts because SDL could block for ever otherwise;
anywhere else they print that they did not run. An application with no `gpu` is run through the whole loop
on SDL's dummy driver, and its frame must equal its surface byte for byte.

The other eighteen cover **the part that can be wrong without crashing**: the key mapping, the scale,
the pointer units, the frame rectangle, the application's share of a frame — that `on_frame` is
handed the clamped step and that a signal it writes rebuilds the tree on that same frame — the
wheel's sense and its fractions, and which events pause and resume, with each will/did pair firing
its hook once. A wrong scale is a whole interface at the wrong size and a
wrong key is a field that will not take a backspace, while a frame loop needs a display server, a
font and a compositor — a test that mocked all three would be asserting the mock. **The loop itself
is proved by the two applications that use it**, `syslui-demo` on a desktop and `syslui-android` on
a phone.

**The control socket's tests** run `run` itself on SDL's dummy driver with the test as the client, sending
from `on_frame`: `/snapshot` names the button, a `/click` at its point reaches its handler through SDL's
events, the `/shot` that follows has the pointer's dot at that point and nowhere else, the window is handed
out and retitled while it runs and gone after, and the socket file goes with it. Without a socket the window
is handed out all the same and no frame has a mark. The route's refusals (405, 503) and the marks — the dot,
the ring, the press square, their scale to pixels, nothing before the pointer has moved, and the draw colour
left black for the next clear — are checked by exact pixels. An application's own route answers ahead of
the built-ins and its `Pass` falls through to them, and a hidden window, driven over its socket, answers its
route, `/snapshot`, `/state` (hidden) and a `/shot` holding the painted ground. On the viewport side, a menu
raised over a viewport keeps its pixels with control on and off and the viewport is placed once, and the
snapshot pass never runs a viewport's `input`, so a press held across it is still held on the next frame.

## Licence

ISC. SDL3 is zlib and PlutoVG is MIT; neither is carried here.
