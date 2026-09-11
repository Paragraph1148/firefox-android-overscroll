# Is anyone already implementing elastic overscroll in Firefox for Android?

**Short answer: no.** Researched 2026-09-11 against `mozilla-firefox/firefox` `main`
and bugzilla.mozilla.org.

There is no open bug, no assignee, and no Phabricator revision proposing elastic
(rubber-band / stretch) overscroll for web content on Firefox for Android. The idea
is unclaimed.

---

## 1. What Firefox actually does today

The premise "desktop has it, Android doesn't" is correct, but the reason is more
specific than "it hasn't been ported yet". Android has overscroll *code*; it just
takes a completely different path from desktop.

The fork is a single `#ifdef` in APZ — `gfx/layers/apz/src/AsyncPanZoomController.cpp:130-138`:

```cpp
// Choose between platform-specific implementations.
#ifdef MOZ_WIDGET_ANDROID
typedef WidgetOverscrollEffect OverscrollEffect;
typedef AndroidSpecificState PlatformSpecificState;
#else
typedef GenericOverscrollEffect OverscrollEffect;
typedef PlatformSpecificStateBase PlatformSpecificState;
#endif
```

Both effects are defined in `gfx/layers/apz/src/Overscroll.h`.

### Desktop — `GenericOverscrollEffect` (the animation we want)

APZ *physically* overscrolls the scroll frame:

```cpp
mApzc.mX.OverscrollBy(aOverscroll.x);
mApzc.mY.OverscrollBy(aOverscroll.y);
```

`AsyncPanZoomController::GetOverscrollTransform()` (`AsyncPanZoomController.cpp:5049`)
then translates the composited content by that offset, and `RelieveOverscroll()`
starts a spring-back `OverscrollAnimation`. The spring is tunable via prefs
(`StaticPrefList.yaml:769-782`): `apz.overscroll.spring_stiffness = 200`,
`apz.overscroll.damping = 1.1`, `apz.overscroll.max_velocity = 10`.

That is the elastic bounce shipping on macOS since Firefox 89, and on
Windows/Linux since `apz.overscroll.enabled` became `true` unconditionally
(`StaticPrefList.yaml:746-749` — no platform gate).

### Android — `WidgetOverscrollEffect` (a glow, not a bounce)

APZ deliberately does **not** move the content. It zeroes the overscroll and hands
the delta to the widget:

```cpp
controller->UpdateOverscrollOffset(mApzc.GetGuid(), aOverscroll.x,
                                   aOverscroll.y, mApzc.IsRootContent());
aOverscroll = ParentLayerPoint();
```

The delta then travels:

```
WidgetOverscrollEffect::ConsumeOverscroll   (Overscroll.h:208)
  -> GeckoSession.updateOverscrollOffset    (GeckoSession.java:8093)
    -> OverscrollEdgeEffect.setDistance     (OverscrollEdgeEffect.java:185)
      -> android.widget.EdgeEffect.onPull
        -> drawn in GeckoView.onDraw         (GeckoView.java:929)
```

So Android gets the **legacy Android edge glow** — that shaded arc at the screen
edge. The page itself never moves.

## 2. Why it looks broken on Android 12+, not just dated

`OverscrollEdgeEffect.java` contains no reference to `getDistance()`,
`RenderEffect`, or `setRenderEffect` — it is written purely against the glow model.

On Android 12 (API 31) `EdgeEffect` switched its default to `TYPE_STRETCH`, and its
`draw(Canvas)` no longer paints a glow. Instead it reaches into the canvas and
stretches the RenderNode being recorded
(AOSP `EdgeEffect.java:606-660`):

```java
} else if (edgeEffectBehavior == TYPE_STRETCH && canvas instanceof RecordingCanvas) {
    ...
    RenderNode renderNode = recordingCanvas.mNode;
    ...
    renderNode.stretch(vecX, vecY, mWidth, mHeight);
}
```

**`GeckoView` is backed by a `SurfaceView` by default** (`GeckoView.java:392`). Web
content is composited by Gecko into that Surface, so it is not part of the
RenderNode that `renderNode.stretch()` operates on. The stretch is applied to a node
that holds no web content, and the SurfaceView punches through it.

The net effect on a modern Android device is that the glow is gone and the stretch
has nothing to stretch. Mozilla's own bugs describe exactly these symptoms:

| Bug | Summary | State |
| --- | --- | --- |
| [1892102](https://bugzilla.mozilla.org/show_bug.cgi?id=1892102) | "Overscroll at the top goes sideways on Android" — filed by a Mozilla graphics engineer: *"The page should stretch vertically. Actual results: The page does not stretch vertically."* | NEW, unassigned |
| [1894112](https://bugzilla.mozilla.org/show_bug.cgi?id=1894112) | "Can't overscroll on the left, only on the right on Android 12 and laters" | NEW, unassigned |
| [1809969](https://bugzilla.mozilla.org/show_bug.cgi?id=1809969) | "Overscroll in web content" (Firefox for Android), from Fenix issue #1055 | NEW, unassigned, last touched 2026-09-03 |
| [1895538](https://bugzilla.mozilla.org/show_bug.cgi?id=1895538) | "[meta] Wonky scrolling issues on Android" — 24 dependencies | NEW meta |

## 3. Who *is* working in this area

Checked every open Bugzilla bug matching `overscroll` (77 open) for an assignee:

| Bug | Assignee | Overlaps our idea? |
| --- | --- | --- |
| [2062598](https://bugzilla.mozilla.org/show_bug.cgi?id=2062598) `overscroll-behavior` doesn't work on Android with fling scrolling | i.am.kanaru.sato@gmail.com | No — CSS property propagation, not the visual effect |
| [1704080](https://bugzilla.mozilla.org/show_bug.cgi?id=1704080) Elastic overscroll fails without scrollbars | hikezoe.birchill@mozilla.com | No — desktop |
| [1860662](https://bugzilla.mozilla.org/show_bug.cgi?id=1860662) Nested scroll frames + `overscroll-behavior:none` | i.am.kanaru.sato@gmail.com | No |
| [2039210](https://bugzilla.mozilla.org/show_bug.cgi?id=2039210) Don't fire `scrollend` after overscroll animation | ajakobi@mozilla.com | No |

Nobody is switching Android to an elastic effect. **The idea is free.**

## 4. Precedent: this team mentors volunteers here

[Bug 1892177](https://bugzilla.mozilla.org/show_bug.cgi?id=1892177) — "Need zeroing
overscroll vector components where overscrolling should not happen on axis" — was
filed in this exact code, tagged `good-first-bug`, mentored by `dan@dlrobertson.com`,
implemented by a volunteer, and landed in June 2025. Comment #2:

> "I think this would be a good first issue for someone looking to get started
> contributing to Gecko. If someone is looking to pick this up, please leave a
> comment, and we can provide some good next steps."

That is a strong signal that a well-argued proposal in this area will get a
reviewer.

## 5. Honest scoping

This is a real, unclaimed, wanted improvement. It is **not** a beginner-sized first
patch:

- It spans C++ APZ, GeckoView's Java layer, and Android rendering internals.
- Building Firefox for Android needs a full Gecko build plus the Android SDK/NDK
  (hours for the first build).
- Flipping Android to `GenericOverscrollEffect` is an architectural change, not a
  bug fix. Mozilla chose `WidgetOverscrollEffect` on purpose, to match platform
  convention. Reversing that needs a rationale, not just a patch.
- Contributions go through **Phabricator / `moz-phab`**, not GitHub pull requests.

## 6. Recommended plan

1. **Land something small in APZ first**, to get a working build, a reviewer contact
   and the `moz-phab` workflow under your belt.
   [Bug 2070702](https://bugzilla.mozilla.org/show_bug.cgi?id=2070702) — "Rename
   `PAPZInputBridge` to `PAPZBridge`" — is an open, unassigned `good-first-bug` in
   Core :: Panning and Zooming, touched 2026-09-09.
2. **Comment on [bug 1892102](https://bugzilla.mozilla.org/show_bug.cgi?id=1892102)**
   with the SurfaceView/RenderNode diagnosis from section 2. That bug has been open
   since 2024 with no root cause written down. A correct diagnosis is a genuine
   contribution even before any code.
3. **Then propose the real change**: make Android use the desktop
   `GenericOverscrollEffect` so the content stretches in the compositor, where Gecko
   already controls the pixels — sidestepping the SurfaceView problem entirely, and
   reusing the spring physics that already ship on desktop.

## Sources

- `gfx/layers/apz/src/AsyncPanZoomController.cpp`, `gfx/layers/apz/src/Overscroll.h`,
  `modules/libpref/init/StaticPrefList.yaml`,
  `mobile/android/geckoview/src/main/java/org/mozilla/geckoview/{GeckoSession,GeckoView,OverscrollEdgeEffect}.java`
  — all from `mozilla-firefox/firefox` `main`
- AOSP `core/java/android/widget/EdgeEffect.java`
- Bugzilla REST API, all open bugs matching `overscroll`
- https://firefox-source-docs.mozilla.org/contributing/contribution_quickref.html

---

# Addendum: stretch vs. rubber-band, and why Chrome settles the argument

Follow-up question: Android 12's own overscroll *stretches* the content rather than
rubber-banding it. Shouldn't Firefox match the platform instead of copying its own
desktop behaviour?

**Yes — and it costs no more than the translate.** Chromium has already made exactly
this decision, and its implementation is the design document we need.

## Chromium runs both effects through one pipeline

In `cc/trees/property_tree.cc` Chromium keeps two tiny functions side by side, fed by
the same `ElasticOverscrollController` that drives macOS rubber-banding:

```cpp
// Desktop
void ApplyElasticOverscrollTranslate(..., gfx::Transform* transform) {
  transform->Translate(-elastic_overscroll.second.x(),
                       -elastic_overscroll.second.y());
}

// Android
void ApplyElasticOverscrollStretch(..., gfx::Transform* transform) {
  const gfx::Vector2dF scale_factor(
      1.f + std::abs(elastic_overscroll.second.x()) / scroller_size.width(),
      1.f + std::abs(elastic_overscroll.second.y()) / scroller_size.height());

  gfx::PointF pivot;                                  // stretch away from the far edge
  if (elastic_overscroll.second.x() > 0.f) pivot.set_x(scroller_size.width());
  if (elastic_overscroll.second.y() > 0.f) pivot.set_y(scroller_size.height());

  // Translate(Pivot) -> Scale -> Translate(-Pivot)
  transform->Translate(pivot.OffsetFromOrigin());
  transform->Scale(scale_factor.x(), scale_factor.y());
  transform->Translate(-pivot.OffsetFromOrigin());
}
```

Selected by `#if BUILDFLAG(IS_ANDROID)` in `TransformTree::UpdateLocalTransform`.

The critical point: **the Android stretch is an affine transform** — a scale anchored
at the opposite edge — not a non-linear distortion shader. Content nearest the
dragged edge moves most because it is furthest from the pivot. No new rendering
primitive is required.

## This maps one-to-one onto Gecko

`AsyncPanZoomController::GetOverscrollTransform()`
(`AsyncPanZoomController.cpp:5049`) is already Chrome's *translate* variant:

```cpp
// The overscroll effect is a simple translation by the overscroll offset.
ParentLayerPoint overscrollOffset(-mX.GetOverscroll(), -mY.GetOverscroll());
return AsyncTransformComponentMatrix().PostTranslate(overscrollOffset.x,
                                                     overscrollOffset.y, 0);
```

It returns an `AsyncTransformComponentMatrix` (a 4x4 matrix), and everything the
stretch needs is already in scope in that class:

- `Metrics().GetCompositionBounds()` — the scroller size for the scale factor.
- `AsyncTransformComponentMatrix::PostScale` — already used at
  `AsyncPanZoomController.cpp:5468` for zoom.

So the stretch is a second branch in one existing function, not a new pipeline.

## Why "just let Android draw it" is the path that does not work

The intuitive shortcut — call `RenderNode.stretch()` / `View.setRenderEffect()` and
let the platform render the effect — is the one genuinely blocked route, for the
reason in section 2: those APIs deform a **RenderNode**, and GeckoView's web content
lives in a **SurfaceView** composited outside the View hierarchy.

The only way to put web content inside a RenderNode is to switch GeckoView to
`TextureView` (`GeckoView.java:377` — supported, but not the default because it costs
latency, memory and power). Trading that for an animation would be rejected, and
rightly.

Chrome hit the same wall and drew the same conclusion: reimplement the stretch in
your own compositor, where you already own the pixels.

## Jetpack Compose / Kotlin are not involved

- GeckoView is **Java**, not Kotlin — `GeckoView.java`, `GeckoSession.java`,
  `OverscrollEdgeEffect.java`.
- Compose's `androidx.compose.foundation.OverscrollEffect` only affects
  Compose-drawn content. It cannot reach pixels Gecko composites into a Surface, for
  exactly the same reason `RenderNode.stretch()` cannot.
- The change is C++ in APZ, plus *deleting* Java. There is no port and no new UI
  framework.

## Known consequence to be ready to defend

A translate can be undone for `position: fixed` content; a stretch cannot be undone
by a simple inverse. Chromium therefore skips that correction entirely on Android
(`property_tree.cc`, `UpdateLocalTransform`):

```cpp
// Android does a stretch effect instead of translation - since we cannot do
// a simple translation to undo the root elastic overscroll effect -
// on Android we simply skip this.
#if !BUILDFLAG(IS_ANDROID)
  if (node->should_undo_overscroll) {
    UndoOverscroll(*node, position_adjustment, viewport_property_ids);
  }
#endif
```

So fixed headers stretch along with the page. That is the accepted trade, and it is
an open spec question in Gecko too —
[bug 1761049](https://bugzilla.mozilla.org/show_bug.cgi?id=1761049)
"[css-overscroll] Whether to move position:fixed elements during overscrolling".
Expect a reviewer to raise it.

## Revised shape of the change

1. Android stops using `WidgetOverscrollEffect` and uses `GenericOverscrollEffect`,
   so APZ holds real overscroll state and runs the existing spring animation.
2. `GetOverscrollTransform()` gains a `MOZ_WIDGET_ANDROID` branch producing the
   pivot-anchored scale instead of the translate.
3. The `EdgeEffect` glow path is retired — `OverscrollEdgeEffect.java`,
   `GeckoSession.updateOverscrollOffset` / `updateOverscrollVelocity`, and the
   `GeckoView.onDraw` call site.
4. Resolve the interaction with the dynamic toolbar and with `position: fixed`
   content.

Steps 1 and 2 are small. Steps 3 and 4 are where the review effort lives.

## Additional sources

- Chromium `cc/trees/property_tree.cc`, `cc/input/scroll_elasticity_helper.{h,cc}`
  (`chromium/src` `main`)
- AOSP `core/java/android/widget/EdgeEffect.java`, `draw(Canvas)` stretch branch
