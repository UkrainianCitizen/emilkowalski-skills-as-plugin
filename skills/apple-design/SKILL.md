---
name: apple-design
description: Apple's approach to interface design and fluid, physical motion, translated for the web. Use when building or reviewing gesture-driven UI, spring animations, drag/swipe/sheet interactions, momentum and interruptible transitions, translucent materials and depth, typography (optical sizing, tracking, leading, Dynamic Type and the Large Content Viewer), reduced-motion, or the design foundations (feedback, spatial consistency, restraint) behind Apple-style interfaces.
---

# Apple Design

## Initial Response

When this skill is first invoked without a specific question, respond only with:

> I'm ready to help you build fluid, Apple-style interfaces on the web, my knowledge comes from Apple's WWDC design talks, translated for the web.

Do not provide any other information until the user asks a question.

How Apple builds interfaces that stop feeling like a computer and start feeling like an extension of you. This knowledge comes from Apple's WWDC design talks — chiefly *Designing Fluid Interfaces* (WWDC 2018) — distilled and translated into the web platform (CSS, Pointer Events, `requestAnimationFrame`, spring libraries like Motion/Framer Motion).

The through-line: **an interface feels alive when motion starts from the current on-screen value, inherits the user's velocity, projects momentum forward, and can be grabbed and reversed at any instant.** Springs are the tool that makes all of this natural, because they are inherently interruptible and velocity-aware.

## The Core Idea

> "When we align the interface to the way we think and move, something magical happens — it stops feeling like a computer and starts feeling like a seamless extension of us."

An interface is fluid when it behaves like the physical world: things respond instantly, move continuously, carry momentum, resist at boundaries, and can be redirected mid-motion. Everything below is a way to get closer to that.

Apple frames design as serving four human needs: **safety/predictability, understanding, achievement, and joy.** Every rule here serves one of them.

## 1. Response — kill latency

The moment lag appears, the feeling of directness "falls off a cliff." Response is the foundation everything else is built on.

- **Respond on pointer-down, not on release.** Highlight a button the instant it's pressed. Waiting for `click`/touch-up to show feedback feels dead.
- **Be vigilant about every latency.** Audit debounces, artificial timers, transition waits, and the ~300ms tap delay. Anything on the input path that isn't essential is a regression.
- **Feedback must be continuous *during* the interaction, not just at the end.** For a drag, slider, or drawer, update the UI 1:1 with the pointer the whole way through — never animate only when the gesture completes.

```css
/* Feedback lives on the press, and it's instant */
.button:active {
  transform: scale(0.97);
  transition: transform 100ms ease-out;
}
```

## 2. Direct manipulation — 1:1 tracking

> "Touch and content should move together."

When the user drags something, it must stay glued to the finger — and respect the offset from *where they grabbed it*. Snapping to the element's center on grab breaks the illusion immediately.

- Use Pointer Events with `setPointerCapture` so tracking continues even when the pointer leaves the element's bounds.
- Track a short **velocity/position history** (last few `pointermove` events), not just the current point — you'll need velocity at release.

```js
el.addEventListener('pointerdown', (e) => {
  el.setPointerCapture(e.pointerId);
  const grabOffset = e.clientY - el.getBoundingClientRect().top; // respect where they grabbed
  // ...track position + timestamp history for velocity
});
```

## 3. Interruptibility — the single most important principle

> "The thought and the gesture happen in parallel."

Every animation must be interruptible and redirectable at any moment. A user must be able to grab a moving element mid-flight and reverse it without waiting for the animation to finish. A closing modal the user grabs again should follow the finger — not finish closing first, then reopen.

- **Never lock out input during a transition.**
- **Always animate from the *presentation* (current) value, never the target value.** On interrupt, read the element's live on-screen transform and start the new animation from there. Starting from the logical/target value causes a visible jump.
- **Avoid CSS transitions and `@keyframes` for anything gesture-driven** — they can't be smoothly grabbed and reversed mid-flight. Springs animate from the current value by default, which is exactly what interruption needs.
- **When a gesture reverses, blend velocity — don't hard-cut it.** Replacing one animation with another at a reversal creates a velocity discontinuity, a "brick wall." Spring libraries that carry velocity through a re-target avoid it. (This is what iOS's *additive animations* do natively; on the web, choose a spring library that re-targets from the current velocity.)
- **Decompose 2D motion into independent X and Y springs.** A single spring on a 2D distance desyncs when X and Y have different velocities.

## 4. Behavior over animation — use springs

> "Think of animation as a conversation between you and the object, not something prescribed by the interface."

A pre-scripted, fixed-duration animation can't respond to new input. A spring can — new input just changes the target, and the motion stays continuous. Reach for springs for anything a user can touch.

Apple deliberately replaced the physics triplet (mass/stiffness/damping) with two designer-friendly parameters. Think in these:

- **Damping ratio** — controls overshoot. `1.0` = critically damped, no bounce, smooth settle. `< 1.0` = overshoots and oscillates. Lower = bouncier.
- **Response** — how quickly the value reaches the target, in seconds. Lower = snappier. **This is not "duration"** — a spring has no fixed duration; its settle time emerges from the parameters.

**Defaults:**
- Start most UI at **damping `1.0`** (critically damped) — graceful and non-distracting.
- Add bounce (**damping ~`0.8`**) **only when the gesture itself carried momentum** (a flick, a throw, a drag release). Overshoot on a menu that just faded in feels wrong; overshoot on a card you flicked feels right.

**Concrete values Apple ships:**

| Interaction | Damping | Response |
| --- | --- | --- |
| Move / reposition (e.g. PiP) | `1.0` | `0.4` |
| Rotation | `0.8` | `0.4` |
| Drawer / sheet | `0.8` | `0.3` |

**Web mapping (Motion / Framer Motion):** the `bounce` + `duration` spring API maps closely to Apple's damping + response. A safe house style is `damping: 1.0` springs everywhere by default; reserve bounce for momentum-driven, physical interactions.

```js
import { animate } from 'motion';

// Critically damped default (no overshoot)
animate(el, { y: 0 }, { type: 'spring', bounce: 0, duration: 0.4 });

// Momentum interaction — a little bounce, only because a flick preceded it
animate(el, { y: target }, { type: 'spring', bounce: 0.2, duration: 0.4 });
```

## 5. Velocity handoff — the seam between drag and animation

When a gesture ends, the animation must **continue at the finger's exact velocity**, so there's no visible seam between dragging and animating. This is the detail that most separates "fluid" from "fine."

Pass the pointer's release velocity as the spring's initial velocity. Some spring APIs want **relative** velocity — normalize it by the remaining distance to the target:

```
relativeVelocity = gestureVelocity / (targetValue − currentValue)
```

Example: element at `y=50`, target `y=150` (100px to go), finger moving 50px/s → initial spring velocity = `50 / 100 = 0.5`. Framer Motion / Motion take absolute px/s velocity directly (`velocity` option), so you usually hand it the raw value.

## 6. Momentum projection — animate to where the gesture is *going*

> "Take a small input and make a big output."

Don't snap to the nearest boundary from the *release point*. Use velocity to **project the resting position** — exactly like scroll deceleration — then snap to the target nearest that projected point. This is what makes a flick feel like it throws the element.

Apple's exact projection function (from the *Designing Fluid Interfaces* sample code):

```js
// decelerationRate ≈ 0.998 for normal scroll feel; 0.99 for snappier
function project(initialVelocity /* px/s */, decelerationRate = 0.998) {
  return (initialVelocity / 1000) * decelerationRate / (1 - decelerationRate);
}

const projectedEndpoint = currentPosition + project(releaseVelocity);
const target = nearestSnapPoint(projectedEndpoint);   // choose target from the projection
animateSpringTo(target, { velocity: releaseVelocity }); // then hand off velocity (§5)
```

Note: the physics-textbook `v²/(2·decel)` is *not* what Apple ships — use the exponential-decay form above. This is the standard behavior in good bottom-sheets and carousels (Vaul, Embla).

## 7. Spatial consistency — symmetric paths, anchored origins

> "If something disappears one way, we expect it to emerge from where it came."

- **Enter and exit along the same path.** A panel that slides in from the right must dismiss to the right. In-from-right / out-the-bottom feels disconnected and confusing.
- **Anchor interactions to their source.** A menu, popover, or sheet should originate from the element that triggered it — set `transform-origin` to the trigger, so the spatial relationship between button and content is obvious. (This is the same origin-awareness point as popovers scaling from their trigger, not their center.)
- **Mirror the easing on reversible transitions** so the outbound path matches the return path (use inverse cubic-bézier control points for the two directions).

## 8. Hint in the direction of the gesture

Humans predict a final state from a trajectory. Intermediate motion should telegraph where things are going — Control Center modules "grow up and out toward your finger." Make the in-between frames point at the outcome, not just interpolate blindly to it.

## 9. Rubber-banding — soft boundaries

At an edge, resist progressively instead of stopping hard. A hard stop reads as "frozen"; continuous resistance reads as "responsive, but there's nothing more here." Apply damping that increases the further past the boundary the user drags.

```js
// The further past the bound, the less the element follows — real things slow before they stop
function rubberband(overshoot, dimension, constant = 0.55) {
  return (overshoot * dimension * constant) / (dimension + constant * Math.abs(overshoot));
}
```

## 10. Gesture design details (the "feel" checklist)

- **Tap:** highlight on touch-*down* (instant), commit on touch-*up*. Add ~10px of hysteresis/hit padding around the target, and allow cancel-by-dragging-away and back.
- **Hit region:** "As a general rule, a button needs a hit region of at least 44x44 pt — in visionOS, 60x60 pt — to ensure that people can select it easily, whether they use a fingertip, a pointer, their eyes, or a remote." (from [*Buttons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/buttons), read 2026-10-15.) On the web, "The size of the target for pointer inputs is at least 24 by 24 CSS pixels" is the AA floor (from [*Understanding Success Criterion 2.5.8: Target Size (Minimum)*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html), read 2026-10-15, with exceptions for spacing, an equivalent control, inline targets, user-agent-sized targets and essential presentation), and "at least 44 by 44 CSS pixels" is the AAA level (from [*Understanding Success Criterion 2.5.5: Target Size (Enhanced)*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/target-size-enhanced.html), read 2026-10-15).
- **Drag/swipe:** require a small movement threshold (hysteresis, ~10px) before committing to a direction, then track 1:1.
- **Detect all plausible gestures in parallel from the first move**, then confidently cancel the losers once intent is clear. Avoid recognizers that only report a *final* state (`swipeleft`-type events) — they throw away the continuous tracking you need for feedback.
- **Minimize disambiguation delays.** Double-tap detection unavoidably delays single taps; only pay that cost where double-tap truly exists.

## 11. Frame-level smoothness

Smoothness is about *what's in the frames*, not just the frame rate.

- Keep the per-frame positional change below the perception threshold to avoid strobing.
- For very fast motion, a subtle **motion blur / stretch** encodes speed and reads better than a hard sharp streak.
- `requestAnimationFrame` is the web's display-synced clock (Apple uses `CADisplayLink`). Animate only compositor-friendly properties — `transform` and `opacity` — and hint with `will-change` where motion is imminent.

## 12. Materials & depth — translucency conveys hierarchy

Apple uses translucent materials as a floating functional layer that brings structure without stealing focus. On the web, approximate with `backdrop-filter`.

- **Build nav/toolbars/sheets as translucent layers** (`backdrop-filter: blur()` + a semi-transparent background) with content scrolling underneath — not opaque bars that consume a fixed strip.
- **Material weight encodes hierarchy:** darker/heavier materials separate structural regions (sidebars); lighter materials draw attention to interactive elements (buttons). **Never stack a light translucent surface on another** — legibility collapses.
- **Bigger surfaces should read as thicker:** stronger blur + a deeper shadow than small chips. Consider context-aware shadow — heavier over busy/text content for separation, lighter over plain backgrounds.
- **Dim to focus, separate to keep flow.** A modal task pairs the surface with a dimming scrim and pushes the background back/down. A parallel, non-blocking panel uses translucency and offset *without* a scrim so the flow isn't broken. For stacked sheets, progressively dim and push back each parent layer.
- **Vibrancy keeps text legible over changing backgrounds.** Over blurred/translucent surfaces, don't use flat gray text — use higher-contrast, slightly heavier weight, and a small letter-spacing bump. Put color on a solid layer, not the translucent foreground.
- **Scroll edge effects, not hard dividers.** Instead of a 1px border under a sticky header, fade a small blur/gradient mask where content meets floating chrome — only where floating UI actually overlaps content.
- **Materialize, don't just fade.** For glass/blur surfaces, animate blur radius and scale together on enter/exit, so the surface reads as a real material arriving rather than a plain opacity fade.

```css
.toolbar {
  background: rgba(255, 255, 255, 0.6);
  backdrop-filter: blur(20px) saturate(180%);
  border-top: 1px solid rgba(255, 255, 255, 0.4); /* bright top edge = light catching the material */
}
```

## 13. Multimodal feedback — motion + sound + haptics

Three rules for combining senses (from *Designing Audio-Haptic Experiences*):

1. **Causality** — it must be obvious what caused the feedback. Trigger it on the actual causal event (the toggle flipping, the item snapping home), and match its character to the action's physicality.
2. **Harmony** — the visual, the sound, and the haptic must fire on the **same frame**. Latency between them destroys the illusion. Don't let a CSS transition lag the audio/haptic (Vibration API).
3. **Utility** — add feedback only where it earns its place. Reserve haptics/sound for meaningful moments (success, error, commit, snap). Over-feedback trains users to ignore all of it.

## 14. Reduced motion & accessibility

Reduced motion doesn't mean *no* feedback — it means a gentler, non-vestibular equivalent. Respond to three independent signals and bake them into your components:

- **`prefers-reduced-motion: reduce`** — replace slides/springs/parallax with short opacity **cross-fades or static transitions**. Drop elastic/overshoot. Keep opacity/color changes that aid comprehension.
- **`prefers-reduced-transparency: reduce`** — make translucent surfaces frostier/solid: raise background opacity, drop the blur.
- **`prefers-contrast: more`** — near-solid backgrounds with a defined, contrasting border.

Also: avoid full-viewport moving backgrounds, slow looping oscillations (near 0.2 Hz / one cycle per 5s), and abrupt brightness jumps (ease dark↔light theme changes). Make large moving objects semi-transparent while they travel, and fade big surfaces out during a large reposition and back in once settled.

```css
@media (prefers-reduced-motion: reduce) {
  .sheet { transition: opacity 200ms ease; transform: none !important; }
}
@media (prefers-reduced-transparency: reduce) {
  .toolbar { background: white; backdrop-filter: none; }
}
```

## 15. Typography — optical sizing, tracking, leading

Apple designs type to change shape with size; the same discipline applies on the web. (From *The Details of UI Typography*, WWDC 2020.)

- **Tracking (letter-spacing) is size-specific — never one value for all sizes.** "In a running app, the system font dynamically adjusts tracking at every point size." (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), read 2026-10-15.) The same page lists tracking in 1/1000 em by point size, so a value of -26 is `-0.026em`. Its table for SF Pro on iOS ("iOS, iPadOS, visionOS tracking values", tab "SF Pro") runs positive, then negative, then positive again: +41 at 6 pt, +6 at 11 pt, 0 at 12 pt, -6 at 13 pt, -26 at 17 pt, -4 at 23 pt, +3 at 24 pt, +14 at 28 to 30 pt, +10 at 36 to 40 pt, +6 at 52 to 56 pt, and 0 from 80 pt to 96 pt, the last row. Large text does not get negative tracking there: the negative values sit between 13 pt and 23 pt, and sizes from 24 pt up are positive until 0 at 80 pt. Those values are for Apple's system font. The page gives no tracking values for custom fonts.
- **Leading (line-height) tracks size inversely.** Tight on large headings, looser on body copy. Increase it for scripts with tall ascenders/descenders; tighten it for dense, information-heavy UI.
- **Build hierarchy from weight + size + leading as a set,** not size alone. Emphasize with weight — it adds presence without taking more space.
- **Respect the user's text-size setting** (Dynamic Type). Scale layout *with* the text — spacing in `rem`/`em`, not fixed px — so a larger font doesn't break the layout.
- **Grow first.** "Scaling with dynamic type is always preferred to showing the Large Content Viewer." (from [*Large Content Viewer*, WWDC 2019](https://developer.apple.com/videos/play/wwdc2019/261/), read 2026-10-15.) A hard-coded size is what breaks it: "Text may also appear clipped, because its container has a fixed frame." (from [*Get started with Dynamic Type*, WWDC 2024](https://developer.apple.com/videos/play/wwdc2024/10074/), read 2026-10-15.) On the web, "the author's responsibility is to create web content that does not prevent the user agent from scaling": text must resize up to 200 percent "without loss of content or functionality" (from [*Understanding Success Criterion 1.4.4: Resize Text*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/resize-text.html), read 2026-10-15), and content must work at a width equivalent to 320 CSS pixels "without requiring scrolling in two dimensions" (from [*Understanding Success Criterion 1.4.10: Reflow*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/reflow.html), read 2026-10-15).
- **Scale essential icons, hold decorative ones back.** "When working with images and icons in large text, you'll want to achieve a balance between scaling an icon for larger text while keeping it from taking too much space from the text." Apple's example is the Settings app on iPhone: each row has a text label and a decorative image, and "With larger text sizes, the decorative image does not grow in size, even though the text label gets larger. This is because apps should prioritize scaling essential content over decorative views." An image should scale "if the image has text or key iconography that is relevant to the rest of the content." (from [*Get started with Dynamic Type*, WWDC 2024](https://developer.apple.com/videos/play/wwdc2024/10074/), read 2026-10-15.)
- **Wrap text under an image that doesn't scale.** "When images don't scale, ensure that text wraps under the unscaled image, to use the most available space." In rare cases a purely decorative view can be removed at the largest text size, as long as "functionality and essential content are not lost by doing this." (from [*Get started with Dynamic Type*, WWDC 2024](https://developer.apple.com/videos/play/wwdc2024/10074/), read 2026-10-15.)
- **Let an icon that should scale follow the text.** "If the image is an SF Symbol, the SF Symbol will resize automatically!" For "an image or a PDF in your assets, you can use the ScaledMetric API to let the image resize based on the selected text size." In UIKit, create the image with a symbol configuration for a text style (`UIImage.SymbolConfiguration(textStyle: .body)`). (from [*Get started with Dynamic Type*, WWDC 2024](https://developer.apple.com/videos/play/wwdc2024/10074/), read 2026-10-15.)
- **The viewer is the exception.** Use it only for UI that "must remain small due to unavoidable design constraints" (from [*UILargeContentViewerInteraction*, UIKit](https://developer.apple.com/documentation/uikit/uilargecontentviewerinteraction), read 2026-10-15), and "only for the case when your custom UI cannot grow in size." (from [*Large Content Viewer*, WWDC 2019](https://developer.apple.com/videos/play/wwdc2019/261/), read 2026-10-15.) Apple's example is the tab bar: its buttons stay small because growing them "would leave very little room for the main content." Standard UIKit bars show the viewer by default, at the Accessibility text sizes only. (from [*Large Content Viewer*, WWDC 2019](https://developer.apple.com/videos/play/wwdc2019/261/), read 2026-10-15.)
- **The Large Content Viewer is the fallback for UI that can't grow.** At the Accessibility text sizes, a long press on such an item shows a larger version of its icon and title, and the user can drag across the bar and release to activate a button (from [*Large Content Viewer*, WWDC 2019](https://developer.apple.com/videos/play/wwdc2019/261/), read 2026-10-15). A custom control opts in with `showsLargeContentViewer`, `largeContentTitle` and `largeContentImage`, plus a `UILargeContentViewerInteraction` on the bar, in UIKit (from [*UILargeContentViewerInteraction*, UIKit](https://developer.apple.com/documentation/uikit/uilargecontentviewerinteraction), read 2026-10-15), or with `.accessibilityShowsLargeContentViewer()` in SwiftUI (from [*accessibilityShowsLargeContentViewer()*, SwiftUI](https://developer.apple.com/documentation/swiftui/view/accessibilityshowslargecontentviewer()), read 2026-10-15). On the web, a control that stays small still needs a programmatically determinable name (from [*Understanding Success Criterion 4.1.2: Name, Role, Value*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/name-role-value.html), read 2026-10-15). `aria-label` provides one "where a visible label cannot be used" (from [*ARIA14: Using aria-label to provide an accessible name where a visible label cannot be used*, WCAG 2.2 techniques](https://www.w3.org/WAI/WCAG22/Techniques/aria/ARIA14.html), read 2026-10-15), and a visible label's text should be contained in the accessible name (from [*Understanding Success Criterion 2.5.3: Label in Name*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/label-in-name.html), read 2026-10-15).
- **Default to the platform's system font** before a custom face; it already ships optical sizing, tracking tables, and legibility tuning. Override only with a reason.

```css
:root { font: 100%/1.5 system-ui, sans-serif; } /* body: system font, comfortable leading */

.display {
  font-size: clamp(2rem, 5vw, 4rem);
  line-height: 1.05;        /* tight leading for large text */
  /* no fixed letter-spacing: Apple's SF Pro table gives a different value at every size */
  font-optical-sizing: auto;
}
```

```swift
// A custom bar button that can't grow with Dynamic Type opts in to the Large Content Viewer
button.showsLargeContentViewer = true
button.largeContentTitle = "Contacts"
customBar.addInteraction(UILargeContentViewerInteraction())
```

## 16. Design foundations — the eight principles

The motion and craft above serve Apple's eight design principles (*Principles of Great Design*, WWDC 2026). Use these as the names you reason with:

1. **Purpose.** Make with intention; decide what *not* to build. Every feature asks for the user's time, attention, and trust — spend that budget only where it pays off.
2. **Agency.** Keep people in control: offer choices, don't force a single path. Back it with forgiveness — easy undo for slips, a confirmation dialog only for genuinely destructive, irreversible actions (use sparingly; overusing it trains people to click through).
3. **Responsibility.** Act in the user's interest. Privacy: ask at the right moment, only for what's needed, transparently. Safety: anticipate misuse and harm — especially with AI (an allergy-aware recipe app must not suggest a harmful ingredient). Add previews, confirmations, disclaimers; cut a feature whose risk outweighs its value.
4. **Familiarity.** Build on what people already know. Use metaphors that are neither too literal nor too abstract (a trash can means delete), and honor their physics. Be consistent: things that look the same must behave the same and live in the same place (close is always top-left on macOS) so people can predict what happens next. Only break a familiar pattern if you can prove it's better — then test it, don't assume.
5. **Flexibility.** Design for different contexts, devices, and the full range of abilities. Adapt to the platform (iPhone = quick touch; desktop = deep workflows with precise pointer control) and to the situation. Design inclusively (age, language, expertise, accessibility). When no single layout fits everyone, let people personalize — rearrange controls, hide what they don't use.
6. **Simplicity — not minimalism.** Strip the unnecessary so the core purpose shines; burying everything in one place looks minimal but isn't simple. Be concise (plain language, no jargon, fewer steps) and clear (use hierarchy — order, spacing, contrast — so the most important thing is the most obvious). Every element earns its place; sometimes *adding* context simplifies (a video scrubber that shows time remaining). Show the common path first, advanced options one level deeper.
7. **Craft.** Uncompromising attention to detail builds trust. Beautiful typography, colors that adapt to light/dark, clear iconography, and responsive animations that give immediate, natural feedback. Nothing is random — every spacing, timing, and alignment value is a deliberate choice you can defend. Jittery scroll, misaligned icons, and layouts that break on rotation read as carelessness. Craft needs iteration and longevity — keep evolving the design as features and hardware change.
8. **Delight.** The result of getting the other seven right, not confetti tacked on top. Decide the emotion you want people to feel (calm, confident, excited) and reinforce it in every decision.

Tactical rules that serve these:

- **Feedback comes in four kinds:** status, completion, warning, error. Confirm meaningful actions, expose ongoing status, warn before problems, validate inline (not on submit).
- **Wayfinding.** Every screen should answer: Where am I? Where can I go? What's there? How do I get out? Never trap the user.
- **Grouping & mapping.** Proximity implies relationship; place a control near what it affects and arrange controls to mirror what they change. If you need a label to explain a control, the mapping is weak.
- **Direct, specific labels beat safe generic ones.** Name nav items for their contents ("Progress", "Library"), not vague umbrellas ("Home"). Specificity creates predictability.

## 17. Process

- **Prototype interactively — an interactive demo is worth "a million static designs."** You discover the interface by building and playing with it; a working prototype also sets a concrete bar that prevents a mediocre final implementation.
- **Design interaction and visuals together.** "You shouldn't be able to tell where one ends and the other begins." Motion is not a layer added after the pixels.
- **Test with real people in real context**, and review motion with fresh eyes — play it in slow motion / frame-by-frame to catch what's invisible at full speed.

## Quick Reference

| Need | Technique | Concrete value |
| --- | --- | --- |
| Default UI spring | Critically damped, no overshoot | `damping 1.0`, `response 0.3–0.4` |
| Momentum / flick spring | Under-damped, slight bounce | `damping ~0.8`, `response 0.3–0.4` |
| Gesture → spring velocity | Hand off release velocity | `gestureVelocity / (target − current)` if normalized |
| Flick landing point | Project momentum | `current + (v/1000)·d/(1−d)`, `d ≈ 0.998` |
| Interrupt cleanly | Start from presentation (live) value | read the on-screen transform |
| Avoid reversal "brick wall" | Carry velocity through re-target | spring that blends velocity |
| Reversible transition | Mirror the easing curve | inverse cubic-bézier |
| Decide reverse vs. commit | Use velocity **sign**, not position | at release |
| 1:1 drag | Pointer Events + capture | respect the grab offset |
| Feedback | On pointer-down, continuous | never only at the end |
| Boundary | Rubber-band, don't hard-stop | progressive resistance |
| Translucent chrome | `backdrop-filter` layer | content scrolls under |
| Type tracking | Size-specific, never fixed (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), read 2026-10-15) | Apple's SF Pro iOS table: `-0.026em` at 17 pt, `+0.014em` at 28 to 30 pt, `0` at 80 pt and up. No values for custom fonts |
| Control that can't grow with text size | Prefer scaling. Only a control that must stay small opts in to the Large Content Viewer (from [*Large Content Viewer*, WWDC 2019](https://developer.apple.com/videos/play/wwdc2019/261/), read 2026-10-15) | `showsLargeContentViewer` (from [*UILargeContentViewerInteraction*, UIKit](https://developer.apple.com/documentation/uikit/uilargecontentviewerinteraction), read 2026-10-15) / `.accessibilityShowsLargeContentViewer()` (from [*accessibilityShowsLargeContentViewer()*, SwiftUI](https://developer.apple.com/documentation/swiftui/view/accessibilityshowslargecontentviewer()), read 2026-10-15); on the web, [*Understanding Success Criterion 1.4.4: Resize Text*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/resize-text.html) and [*Understanding Success Criterion 1.4.10: Reflow*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/reflow.html), read 2026-10-15 |
| Icon beside text at large sizes | Essential icons scale, decorative ones hold their size, text wraps under the unscaled image (from [*Get started with Dynamic Type*, WWDC 2024](https://developer.apple.com/videos/play/wwdc2024/10074/), read 2026-10-15) | SF Symbol scales automatically; `ScaledMetric`; `UIImage.SymbolConfiguration(textStyle:)` |
| Button hit region | At least 44x44 pt (60x60 pt in visionOS) (from [*Buttons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/buttons), read 2026-10-15) | web: 24x24 CSS px (AA, from [*Understanding Success Criterion 2.5.8: Target Size (Minimum)*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html), read 2026-10-15), 44x44 (AAA, from [*Understanding Success Criterion 2.5.5: Target Size (Enhanced)*, WCAG 2.2](https://www.w3.org/WAI/WCAG22/Understanding/target-size-enhanced.html), read 2026-10-15) |
| Reduced motion | Cross-fade, not slide/spring | `@media (prefers-reduced-motion)` |
