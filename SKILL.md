---
name: fractional-scaling
description: Rules for crisp, stable UI at fractional display scales (125%, 150%, 175%...), for both UI library/framework implementers and developers who build apps with a UI library. Use this skill whenever writing, designing or reviewing code that involves layout, rendering, borders, 1px lines, dividers, icons, centering, HiDPI, devicePixelRatio, scale_factor, custom GPU renderers (wgpu, Vulkan, GL, Skia), WebView or native view embedding, Wayland compositors or server-side decorations, or any bug report about blurry edges, uneven border widths, hairline gaps, or elements that jitter while moving or resizing. Use it even if "fractional scaling" is never mentioned.
---

# Fractional scaling

## Why this matters

At scale 1.5, a 1px logical line is 1.5 device pixels. Anything that is not a
whole device pixel gets anti-aliased across two pixel columns: edges blur, the
four sides of one box render with different thicknesses, hairline gaps appear
between neighbours, and sizes wobble by 1px as elements move.

At scales 1.0 and 2.0 every logical integer is a device integer, so all of this
is invisible there. Most developers work on a 1.0 or 2.0 screen, and most test
setups default to one of them, which is why these bugs ship.

There is no rounding scheme that keeps every position exact, every length
stable, and every shared edge closed at the same time. Every UI stack picks
which property to give up. The job is to pick deliberately and apply it
consistently.

## Step 1: Identify the role

- **Implementer**: writing or changing the layout engine, renderer, style
  resolution, or the boundary between native and embedded content (a UI
  library, framework, game UI, compositor).
- **User**: building an app or page on top of an existing UI library, browser
  engine, or toolkit, where rounding happens below the code being written.

A project can be both (e.g. an app with a custom canvas widget). Apply each
section to the code it covers.

## Step 2: Raise it at design time

Fractional scaling problems are cheap to avoid at design time and expensive to
fix afterwards, because the fix usually means changing where rounding happens
across the whole pipeline. So when designing new layout, rendering, or
component code, explain the issue to the user and work out together which
approach to take, before implementing.

Whether to stop and ask for a decision, or to proceed with a clearly stated
assumption, follows whatever other skills or instructions govern consulting the
user. This skill only requires that the topic is raised and explained, not that
work is blocked on it.

Keep this proportionate. For a small edit inside an existing design, apply the
rules below quietly and mention the topic only when the edit touches it.

---

## For implementers

### Default approach when no strategy exists yet

If the project has no deliberate rounding strategy, use **pre-layout snapping
plus integer layout**, modelled on XAML's `UseLayoutRounding` (WPF, UWP,
WinUI):

1. Convert every authored length to whole device pixels once, when styles are
   resolved: `dp = round(logical * scale)`. Non-zero strokes become at least
   1 dp.
2. Run layout entirely in integer device pixels.
3. Paint integer rects only.

This keeps line thickness perfectly stable and edges crisp, and the algorithm
stays small. The cost is slight ratio distortion (at 1.25, 1px becomes 1dp and
3px becomes 4dp). Accept it: people notice a 1dp wobble in line thickness far
more than a sub-pixel error in position. Do not "correct" it by reintroducing
fractional positions.

If the project already has a deliberate strategy (for example browser-style
edge snapping), respect it and do not mix in a second one. Mixed strategies are
worse than either alone.

### Rules

1. **One snapping point, one rounding function.** Convert logical to device
   pixels in exactly one place. Use a single rounding function everywhere, with
   one decided midpoint rule (is 1.5dp drawn as 1 or 2?).

2. **Integer distribution.** For flex and grid, subtract all fixed integer
   lengths first, split the remaining integer space among flexible children,
   and hand out leftover pixels deterministically (e.g. one each, in order).
   Positions are sums of integers, so siblings always touch.

3. **Never derive a size from two rounded positions.** `round(far) - round(near)`
   changes by ±1dp as the origin moves. Position = parent origin + integer
   offset; size is its own integer.

4. **Measured content rounds up.** Ceil text and other measured sizes to whole
   device pixels so content is never clipped.

5. **Strokes inside by default.** A stroke centred on an edge puts half its
   width outside; at odd scales that is always a half pixel. Draw borders
   inside the box.

6. **Motion: float state, integer paint.** Track animation, scroll, and window
   positions in floats, but round only the painted origin. Keep sizes from
   layout. Rendering at sub-pixel offsets blurs the whole element while it
   moves.

7. **Separate coordinate spaces by type.** Use distinct types for logical and
   device pixels (newtypes in Rust, branded types in TypeScript) so mixing them
   fails at compile time.

8. **Align external boundaries to the same grid.** Embedded native views,
   WebViews, and client buffers must be placed with the same rounding rules.
   Where an external size can differ by about 1 device pixel (e.g. a Wayland
   client buffer is `round(logical * scale)`), cover the gap explicitly instead
   of letting the two drift apart.

9. **Give users a device-pixel hairline.** Expose something like SwiftUI's
   `pixelLength` so users can draw exactly 1 device pixel without knowing the
   scale.

10. **Fix the source, not the symptom.** When something is blurry or uneven,
    find where a fractional value entered the integer pipeline. Do not add a
    local `round()` at the paint site; scattered rounding causes double
    rounding and moves the bug elsewhere.

11. **Document the policy.** State the rounding strategy, the midpoint rule,
    and the accepted trade-offs, so contributors do not "fix" them away.

### Tests

Run with scales `[1.0, 1.25, 1.5, 1.75, 2.0, 2.5, 3.0]` and many random origins.
Never default the test environment to a single scale.

- Every painted rect and border width is an integer in device pixels.
- All four border widths of a uniformly bordered box are equal in device pixels.
- Siblings in a flex/grid container have no gap and no overlap.
- Children fill their container exactly.
- Translating a component does not change any of its internal sizes.

Verify rendering by reading back the framebuffer and comparing pixel values
numerically. Downscaled screenshots hide 1px errors.

---

## For users

Rounding happens inside the library or engine, so the goal is to stay on its
well-snapped paths and avoid values that are guaranteed to land on half pixels.

1. **Check at 125% and 150%.** On Windows change the display scale; in a
   browser use zoom. Developing only on a Mac (always integer scale) hides
   these bugs entirely.

2. **Centre with layout, not transforms.** `transform: translate(-50%, -50%)`
   puts odd-sized elements on half pixels and can blur them as a layer. Use
   flex or grid centring.

3. **Avoid values that are half pixels by construction.** Do not hard-code
   0.5px or centre odd-pixel elements. For hairlines, use the library's
   device-pixel unit if it has one.

4. **Draw lines as borders.** A 1px-tall element is snapped as a layout box and
   can render as 1 or 2 device pixels depending on position. Border widths are
   usually snapped more consistently.

5. **One owner per boundary.** Do not put a border on both neighbours, or on
   both a WebView frame and its content. Give each visible line to exactly one
   element.

6. **Avoid seams between same-coloured neighbours.** Put the background on the
   parent instead of on adjacent children.

7. **No bitmaps at non-integer magnification.** Use SVG icons or per-scale
   assets.

8. **Use platform primitives that respect the grid.** For example `strokeBorder`
   instead of `stroke` in SwiftUI, `pixelLength` for hairlines,
   `backingAlignedRect` in AppKit.

9. **Device-pixel drawing is a last resort.** Do not move UI into canvas, WebGL,
   or other manual device-pixel drawing just to escape rounding: it gives up
   accessibility, text rendering, and the library's own layout. Exhaust the
   options above first, and if the library itself is wrong, report it upstream.
   If the app already draws with canvas/WebGL for its own reasons (charts,
   games), allocate the backing store in device pixels (`devicePixelRatio`,
   and `ResizeObserver`'s `devicePixelContentBoxSize` where supported).

10. **Ask for the scale in bug reports.** For "it looks blurry" reports, get the
    display scale and zoom level, and view screenshots at 100%.

---

## Anti-patterns (both roles)

- Multiplying logical rects by `scale` at paint time and passing floats to the GPU.
- Layout in floats with "rounding at the end" for sizes.
- Different rounding functions (`round`, `floor`, `ceil`, truncating casts) for
  the same kind of value in different places.
- Adding `round()` where the symptom shows up.
- Testing or reviewing only at scale 1.0 or 2.0.
- Judging correctness from a reduced-size screenshot.

## Platform notes

- **Browsers / WebViews**: sub-pixel layout with edge snapping at paint time.
  WebView engines differ by OS (Chromium on Windows and Android, WebKit on
  Apple platforms and Linux), so the same CSS can snap differently. The
  native/WebView boundary rounds twice.
- **Apple**: apps always see an integer scale; fractional "looks like"
  resolutions are handled by scaling the whole frame. Fractional device pixels
  still arise from centred strokes and values that are not multiples of
  1/scale.
- **Android**: fractional density factors, but views lay out in integer pixels.
- **Wayland**: surface positions and sizes are integer logical pixels, so
  movement steps unevenly in device pixels at fractional scale. Keep positions
  as floats internally and snap at paint.
