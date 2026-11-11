# Platforms — building Liquid Glass outside the browser

The design rules in SKILL.md (two layers, regular vs clear, rationed colour,
rim, refraction, accessibility fallbacks) are the same everywhere. What changes
per platform is *how* you get three things:

1. **Backdrop** — can the glass see the pixels behind it?
2. **Blur** — can it blur them?
3. **Bend** — can it displace them (refraction)? This is the part people notice
   when it is missing.

APIs below were checked on 2026-10-12 against the linked pages. Re-check before
big work: these move every release.

Contents: [Choosing a route](#choosing-a-route) · [Apple native](#apple-native-swiftui-uikit) ·
[Android](#android-jetpack-compose) · [Flutter](#flutter) · [Desktop: Electron](#electron) ·
[Desktop: Tauri](#tauri) · [Windows native](#windows-native) · [Linux](#linux) ·
[Window material is not Liquid Glass](#window-material-is-not-liquid-glass)

---

## Choosing a route

| Platform | Backdrop + blur | Bend (refraction) | Route |
|---|---|---|---|
| iOS / iPadOS / macOS 26+ | system | system | Use the real material: SwiftUI `glassEffect`, UIKit `UIGlassEffect`. Never fake it there. |
| Android 12+ (Compose) | Haze | Haze `haze-glass`, needs Android 13+ | Spiralens recipe below |
| Web, Chromium (Chrome, Edge, Android WebView, Electron, Tauri on Windows) | `backdrop-filter` | `assets/refraction.js` | `assets/tokens.css` + `refraction.js` |
| Web, Safari / Firefox | `backdrop-filter` | mirror mode (`data-refract-source`) | Same files; glass with a source bends a copy of it, other glass stays blur |
| Flutter | `BackdropFilter` | `ImageFilter.shader`, Impeller only | shader route below |
| Linux desktop apps | in-app only | in-app only | draw the glass inside the app (web view or GPU shader) |

Rule of thumb: on Apple platforms use Apple's material; everywhere else build it
inside your own rendering, because only your app can see its own content.

## Apple native (SwiftUI, UIKit)

### Workflow Decision Tree
Choose the path that matches the request:

#### 1) Review an existing feature
- Inspect where Liquid Glass should be used and where it should not.
- Verify correct modifier order, shape usage, and container placement.
- Check for iOS 26+ availability handling and sensible fallbacks.

#### 2) Improve a feature using Liquid Glass
- Identify target components for glass treatment (surfaces, chips, buttons, cards).
- Refactor to use `GlassEffectContainer` where multiple glass elements appear.
- Introduce interactive glass only for tappable or focusable elements.

#### 3) Implement a new feature using Liquid Glass
- Design the glass surfaces and interactions first (shape, prominence, grouping).
- Add glass modifiers after layout/appearance modifiers.
- Add morphing transitions only when the view hierarchy changes with animation.

### Core Guidelines
- Prefer native Liquid Glass APIs over custom blurs.
- Use `GlassEffectContainer` when multiple glass elements coexist.
- Apply `.glassEffect(...)` after layout and visual modifiers.
- Use `.interactive()` for elements that respond to touch/pointer.
- Keep shapes consistent across related elements for a cohesive look.
- Gate with `#available(iOS 26, *)` and provide a non-glass fallback.

### Review Checklist
- **Availability**: `#available(iOS 26, *)` present with fallback UI.
- **Composition**: Multiple glass views wrapped in `GlassEffectContainer`.
- **Modifier order**: `glassEffect` applied after layout/appearance modifiers.
- **Interactivity**: `interactive()` only where user interaction exists.
- **Transitions**: `glassEffectID` used with `@Namespace` for morphing.
- **Consistency**: Shapes, tinting, and spacing align across the feature.

### Implementation Checklist
- Define target elements and desired glass prominence.
- Wrap grouped glass elements in `GlassEffectContainer` and tune spacing.
- Use `.glassEffect(.regular.tint(...).interactive(), in: .rect(cornerRadius: ...))` as needed.
- Use `.buttonStyle(.glass)` / `.buttonStyle(.glassProminent)` for actions.
- Add morphing transitions with `glassEffectID` when hierarchy changes.
- Provide fallback materials and visuals for earlier iOS versions.

### Quick Snippets
Use these patterns directly and tailor shapes/tints/spacing.

```swift
if #available(iOS 26, *) {
    Text("Hello")
        .padding()
        .glassEffect(.regular.interactive(), in: .rect(cornerRadius: 16))
} else {
    Text("Hello")
        .padding()
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 16))
}
```

```swift
GlassEffectContainer(spacing: 24) {
    HStack(spacing: 24) {
        Image(systemName: "scribble.variable")
            .frame(width: 72, height: 72)
            .font(.system(size: 32))
            .glassEffect()
        Image(systemName: "eraser.fill")
            .frame(width: 72, height: 72)
            .font(.system(size: 32))
            .glassEffect()
    }
}
```

```swift
Button("Confirm") { }
    .buttonStyle(.glassProminent)
```

### Implementing Liquid Glass Design in SwiftUI

#### Overview

Liquid Glass is a dynamic material introduced in iOS that combines the optical properties of glass with a sense of fluidity. It blurs content behind it, reflects color and light from surrounding content, and reacts to touch and pointer interactions in real time. This guide covers how to implement and customize Liquid Glass effects in SwiftUI applications. You should always seek guides on Liquid Glass when asked for help adopting new Apple design.

Key features of Liquid Glass:
- Blurs content behind the material
- Reflects color and light from surrounding content
- Reacts to touch and pointer interactions
- Can morph between shapes during transitions
- Available for standard and custom components

#### Basic Implementation

##### Adding Liquid Glass to a View

The simplest way to add Liquid Glass to a view is using the `glassEffect()` modifier:

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect()
```

By default, this applies the regular variant of Glass within a Capsule shape behind the view's content.

##### Customizing the Shape

You can specify a different shape for the Liquid Glass effect:

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect(in: .rect(cornerRadius: 16.0))
```

Common shape options:
- `.capsule` (default)
- `.rect(cornerRadius: CGFloat)`
- `.circle`

#### Customizing Liquid Glass Effects

##### Glass Variants and Properties

You can customize the Liquid Glass effect by configuring the `Glass` structure:

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect(.regular.tint(.orange).interactive())
```

Key customization options:
- `.regular` - Standard glass effect
- `.tint(Color)` - Add a color tint to suggest prominence
- `.interactive(Bool)` - Make the glass react to touch and pointer interactions

##### Making Interactive Glass

To make Liquid Glass react to touch and pointer interactions:

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect(.regular.interactive(true))
```

Or more concisely:

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect(.regular.interactive())
```

#### Working with Multiple Glass Effects

##### Using GlassEffectContainer

When applying Liquid Glass effects to multiple views, use `GlassEffectContainer` for better rendering performance and to enable blending and morphing effects:

```swift
GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()

        Image(systemName: "eraser.fill")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()
    }
}
```

The `spacing` parameter controls how the Liquid Glass effects interact with each other:
- Smaller spacing: Views need to be closer to merge effects
- Larger spacing: Effects merge at greater distances

##### Uniting Multiple Glass Effects

To combine multiple views into a single Liquid Glass effect, use the `glassEffectUnion` modifier:

```swift
@Namespace private var namespace

// Later in your view:
GlassEffectContainer(spacing: 20.0) {
    HStack(spacing: 20.0) {
        ForEach(symbolSet.indices, id: \.self) { item in
            Image(systemName: symbolSet[item])
                .frame(width: 80.0, height: 80.0)
                .font(.system(size: 36))
                .glassEffect()
                .glassEffectUnion(id: item < 2 ? "1" : "2", namespace: namespace)
        }
    }
}
```

This is useful when creating views dynamically or with views that live outside of an HStack or VStack.

#### Morphing Effects and Transitions

##### Creating Morphing Transitions

To create morphing effects during transitions between views with Liquid Glass:

1. Create a namespace using the `@Namespace` property wrapper
2. Associate each Liquid Glass effect with a unique identifier using `glassEffectID`
3. Use animations when changing the view hierarchy

```swift
@State private var isExpanded: Bool = false
@Namespace private var namespace

var body: some View {
    GlassEffectContainer(spacing: 40.0) {
        HStack(spacing: 40.0) {
            Image(systemName: "scribble.variable")
                .frame(width: 80.0, height: 80.0)
                .font(.system(size: 36))
                .glassEffect()
                .glassEffectID("pencil", in: namespace)

            if isExpanded {
                Image(systemName: "eraser.fill")
                    .frame(width: 80.0, height: 80.0)
                    .font(.system(size: 36))
                    .glassEffect()
                    .glassEffectID("eraser", in: namespace)
            }
        }
    }

    Button("Toggle") {
        withAnimation {
            isExpanded.toggle()
        }
    }
    .buttonStyle(.glass)
}
```

The morphing effect occurs when views with Liquid Glass appear or disappear due to view hierarchy changes.

#### Button Styling with Liquid Glass

##### Glass Button Style

SwiftUI provides built-in button styles for Liquid Glass:

```swift
Button("Click Me") {
    // Action
}
.buttonStyle(.glass)
```

##### Glass Prominent Button Style

For a more prominent glass button:

```swift
Button("Important Action") {
    // Action
}
.buttonStyle(.glassProminent)
```

#### Advanced Techniques

##### Background Extension Effect

To stretch content behind a sidebar or inspector with the background extension effect:

```swift
NavigationSplitView {
    // Sidebar content
} detail: {
    // Detail content
        .background {
            // Background content that extends under the sidebar
        }
}
```

##### Extending Horizontal Scrolling Under Sidebar

To extend horizontal scroll views under a sidebar or inspector:

```swift
ScrollView(.horizontal) {
    // Scrollable content
}
.scrollExtensionMode(.underSidebar)
```

#### Best Practices

1. **Container Usage**: Always use `GlassEffectContainer` when applying Liquid Glass to multiple views for better performance and morphing effects.

2. **Effect Order**: Apply the `.glassEffect()` modifier after other modifiers that affect the appearance of the view.

3. **Spacing Consideration**: Carefully choose spacing values in containers to control how and when glass effects merge.

4. **Animation**: Use animations when changing view hierarchies to enable smooth morphing transitions.

5. **Interactivity**: Add `.interactive()` to glass effects that should respond to user interaction.

6. **Consistent Design**: Maintain consistent shapes and styles across your app for a cohesive look and feel.

#### Example: Custom Badge with Liquid Glass

```swift
struct BadgeView: View {
    let symbol: String
    let color: Color

    var body: some View {
        ZStack {
            Image(systemName: "hexagon.fill")
                .foregroundColor(color)
                .font(.system(size: 50))

            Image(systemName: symbol)
                .foregroundColor(.white)
                .font(.system(size: 30))
        }
        .glassEffect(.regular, in: .rect(cornerRadius: 16))
    }
}

// Usage:
GlassEffectContainer(spacing: 20) {
    HStack(spacing: 20) {
        BadgeView(symbol: "star.fill", color: .blue)
        BadgeView(symbol: "heart.fill", color: .red)
        BadgeView(symbol: "leaf.fill", color: .green)
    }
}
```

#### References

- [Applying Liquid Glass to custom views](https://developer.apple.com/documentation/SwiftUI/Applying-Liquid-Glass-to-custom-views)
- [Landmarks: Building an app with Liquid Glass](https://developer.apple.com/documentation/SwiftUI/Landmarks-Building-an-app-with-Liquid-Glass)
- [SwiftUI View.glassEffect(_:in:isEnabled:)](https://developer.apple.com/documentation/SwiftUI/View/glassEffect(_:in:isEnabled:))
- [SwiftUI GlassEffectContainer](https://developer.apple.com/documentation/SwiftUI/GlassEffectContainer)
- [SwiftUI GlassEffectTransition](https://developer.apple.com/documentation/SwiftUI/GlassEffectTransition)
- [SwiftUI GlassButtonStyle](https://developer.apple.com/documentation/SwiftUI/GlassButtonStyle)

## Android (Jetpack Compose)

Proven in a shipped app (Spiralens, Haze 2.0.1). Libraries, from Maven Central:

```kotlin
implementation("dev.chrisbanes.haze:haze:2.0.1")
implementation("dev.chrisbanes.haze:haze-glass:2.0.1")   // experimental API
```

The pattern:

```kotlin
@OptIn(ExperimentalHazeApi::class)
fun Modifier.liquidGlass(
    backdrop: HazeState,
    shape: RoundedCornerShape = RoundedCornerShape(50),
    clear: Boolean = false,
): Modifier = this
    .hazeGlass(
        input = HazeInput.Sources(backdrop),
        performanceMode = HazePerformanceMode.Quality,   // keeps the blur while scrolling
        style = GlassStyle {
            backgroundColor(Page)            // opaque base: nothing sharp shows through
            tint(Cream.copy(alpha = if (clear) .04f else .07f))
            whitePoint(if (clear) 0f else .12f)
            chromaMultiplier(if (clear) 1.1f else 1.3f)
            optics(
                blurRadius = if (clear) 2.dp else 34.dp,
                refractionStrength = 1f,
                refractionHeightFraction = if (clear) .5f else .618f,
                refractionDisplacement = if (clear) 13.dp else 24.dp,
                refractionFoldStrength = .8f,
                depth = .9f,
            )
            specularIntensity(.7f)
            ambientResponse(.35f)
            edgeSoftness(1.dp)
            shape(shape)
            surfaceProfile(SurfaceProfile.Squircle)
            chromaticAberrationStrength(.12f)
        },
    )
    .border(1.dp, RimBrush, shape)          // the rim, brightest upper-left

val RimBrush = Brush.linearGradient(
    0f to Color.White.copy(alpha = .85f),
    .45f to Color.White.copy(alpha = .14f),
    1f to Color.White.copy(alpha = .50f),
)
```

`Page` and `Cream` are the app's own page and tint colours.

What the source needs:
- `val state = rememberHazeState()`, and the content behind the glass gets
  `Modifier.background(Page).hazeSource(state)`. **The source must be opaque**,
  and the style needs `backgroundColor(...)` too, or sharp text shows through
  the glass body and it looks broken.
- Bending needs Android 13+ (it runs as a runtime shader); blur needs 12+.
  Below that, and in battery saver, on low-RAM phones, or with animations
  turned off, fall back to a clipped dark veil + light gradient body + the same
  rim.
- Dialogs and bottom sheets are separate windows and cannot see the app behind
  them. For real glass there, draw them inside the app (an overlay host at the
  root) and make the page the haze source while one is open. Text-heavy
  overlays need a strong dark veil (about 60%).
- Live camera preview: use `PreviewView.ImplementationMode.COMPATIBLE`
  (TextureView) so it can be a haze source, clear glass with a 35% veil, and
  `HazePerformanceMode.Performance`. Check frame rate and heat on a real phone.

Sizes that worked well there: spacing and radii on Fibonacci steps (5, 8, 13,
21, 34, 55 dp), sheets no taller than 61.8% of the screen. That is a design
choice from that app, not an Apple rule.

If the bundled `liquid-glass-android` skill is installed, it holds the full
reference file this section was distilled from.

## Flutter

- Blur: `BackdropFilter(filter: ImageFilter.blur(...))` clipped to the shape.
- Bend: `ImageFilter.shader(FragmentShader)` lets a fragment shader displace
  the backdrop. Per the API docs it is **only supported on Impeller**, throws
  `UnsupportedError` elsewhere, and must flip the y-axis on the OpenGL ES
  backend. Check `ImageFilter.isShaderFilterSupported` and fall back to the
  blur-only glass when it is false.
- The shader does what `refraction.js` does: inside a rim band, sample the
  backdrop from further inward, strongest at the outline. Port the map logic
  from `assets/refraction.js` (`buildMap`) into GLSL.
- Several packages on pub.dev wrap this; most are early or experimental, so
  read their changelogs before depending on one.

## Electron

Electron renders with Chromium, so **the web recipe works in full, including
refraction**: `tokens.css` + `refraction.js` inside the window.

Optional window-level material (behind the whole window, not Liquid Glass):
- Windows: `backgroundMaterial: 'auto' | 'none' | 'mica' | 'acrylic' | 'tabbed'`
  in `BrowserWindow` options or `win.setBackgroundMaterial()`. Windows 11 22H2
  and up only.
- macOS: `vibrancy` / `win.setVibrancy()` with values such as `sidebar`,
  `menu`, `popover`, `hud`, `under-window`.
Source: electronjs.org/docs/latest/api/browser-window.

## Tauri

The web view differs by OS, and that decides refraction:
- Windows: WebView2 (Chromium) — refraction works.
- macOS: WKWebView (Safari engine) — live backdrop bend unavailable; use
  mirror mode (`data-refract-source`) for glass over known areas. Prefer the
  `window-vibrancy` crate's `apply_liquid_glass` (macOS 26+) for the window.
- Linux: WebKitGTK — same as macOS: mirror mode or blur.

The `window-vibrancy` crate (github.com/tauri-apps/window-vibrancy) also offers
`apply_mica` (Windows 11), `apply_acrylic` (Windows 10/11) and `apply_vibrancy`
(macOS 10.10+). It states Linux is unsupported: blur there is up to the user's
compositor.

## Windows native

WinUI / Windows App SDK ship Mica and Acrylic. They are Microsoft's materials:
they blur the desktop or window behind, with no rim and no bending. Use them
for the window background if you want, but build Liquid Glass controls inside
your own rendering (Win2D or a shader, or a WebView2 with the web recipe).

## Linux

There is no system glass to call. GTK and Qt apps draw their own: blur and
displace their own content in the toolkit's GPU path (a shader effect), or host
the UI in a Chromium-based web view (Electron, CEF) and use the web recipe.
Desktop-wide blur behind a window depends on the compositor (KWin, Hyprland and
others offer it; it is not a standard), so never rely on it for legibility.

## Window material is not Liquid Glass

Mica, Acrylic and macOS vibrancy blur **what is behind the window** (the
desktop, other apps). Liquid Glass is a control floating over **your own
content** and bending it. A window can use both, but one never replaces the
other, and window materials alone will not produce the look people mean when
they ask for Liquid Glass.
