# syslui-sdl

**The window, the frame loop and the events for [syslUI](https://github.com/sysl-lang/syslui) —
one driver, for a desktop and for a phone.**

```
dependencies {
  syslui     { git = "github.com/sysl-lang/syslui",     version = "0.1.0" }
  syslui-sdl { git = "github.com/sysl-lang/syslui-sdl", version = "0.1.0" }
}
```

**Both, and that is a rule rather than an oversight: a dependency is not transitive.** Naming this
package fetches syslUI with it and does *not* let a program `import sh.sysl.ui` — a module is
importable only from the package that names it, so what a program uses it declares. A program that
built a tree would have named the toolkit anyway.

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
| the JNI method reporting the system bars | **yes** |
| `SDL_ORIENTATIONS` + `WINDOW_RESIZABLE` | **yes**, and harmless elsewhere |

Two packages would be two copies of one loop kept in sync by hand.

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
  pixels off each side. So the Java side reports `WindowInsets.Type.systemBars()` and the application
  hands them over with `set_insets`.
- **The font.** PlutoVG rasterizes glyphs itself and wants a file; there is no fontconfig on a phone
  and no API anywhere that answers "a sans-serif face, please". `system_font()` tries the paths the
  common platforms use, and an application that cares passes its own.

## What stays with the application

- **The entry point** — `main`, or an `@export("SDL_main")`.
- **The JNI method that reports the insets.** Its symbol is mangled from the *application's* own
  package name (`Java_<package>_MainActivity_nativeSetSystemBars`), so it cannot live in a library.
  Eight lines, and it calls `set_insets`.
- **Its shortcuts.** `on_event` sees every event before the driver does and answers `true` to say it
  has dealt with one — which is where ⌘C, escape and the mouse wheel go. A driver that guessed at
  those would be wrong for the second program that used it.

## Tests

```
sysl test .
```

Ten, and they cover **the part that can be wrong without crashing**: the key mapping, the scale, the
pointer units and the frame rectangle. A wrong scale is a whole interface at the wrong size and a
wrong key is a field that will not take a backspace, while a frame loop needs a display server, a
font and a compositor — a test that mocked all three would be asserting the mock. **The loop itself
is proved by the two applications that use it**, `syslui-demo` on a desktop and `syslui-android` on
a phone.

## Licence

ISC. SDL3 is zlib and PlutoVG is MIT; neither is carried here.
