---
name: apple-design
description: Apple's approach to interface design and fluid, physical motion, translated for the web. Use when building or reviewing gesture-driven UI, spring animations, drag/swipe/sheet interactions, momentum and interruptible transitions, translucent materials and depth, typography (optical sizing, tracking, leading, Dynamic Type and the Large Content Viewer), app icons (layers, appearances, system effects), launch screens (what they show, storyboard or information property list), widgets (appearances, rendering modes, the Edit Widget sheet), reduced-motion, or the design foundations (feedback, spatial consistency, restraint) behind Apple-style interfaces.
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

- **A haptic is a foreground feature on iOS.** "When external factors cause the haptic engine to stop, like audio interruptions from a phone call or because the user has put your app in the background, Core Haptics calls the `stoppedHandler`." (from [*Preparing your app to play haptics*, Core Haptics](https://developer.apple.com/documentation/corehaptics/preparing-your-app-to-play-haptics), read 2026-10-22.) The stop reason `CHHapticEngine.StoppedReason.applicationSuspended`: "This condition occurs if the system suspended your app while it was in the background. Restart the engine when your app returns to the foreground to ensure proper haptic playback." (from [*CHHapticEngine.StoppedReason.applicationSuspended*, Core Haptics](https://developer.apple.com/documentation/corehaptics/chhapticengine/stoppedreason/applicationsuspended), read 2026-10-22.) So treat a haptic as a foreground feature: an event that lands while the app is in the background gets no haptic. When the user must notice it, pair it with a sound or a notification. (Both sentences are the fork's conclusion from the two quotes above. Apple's pages do not say either one, and the `UIFeedbackGenerator` page says nothing about the background.)
- **Pick the feedback generator by what happened.** Apple names the meaning for each of these `UIFeedbackGenerator` subclasses: "Use impact feedback to indicate when an impact occurs." (`UIImpactFeedbackGenerator`) "Use selection feedback to indicate a change in selection." (`UISelectionFeedbackGenerator`) "Use notification feedback to indicate successes, failures, and warnings." (`UINotificationFeedbackGenerator`) (from [*UIFeedbackGenerator*, UIKit](https://developer.apple.com/documentation/uikit/uifeedbackgenerator), read 2026-10-22.)
- **Pick the haptic event type by the sound's attack.** "For a sharp transient, a chime with a sharp attack will probably work really well. But, if we have a sound that's much smoother, using those same haptics is probably not such a good idea. So, for something like this, a continuous haptic, ramping up and down probably works better. But, these are not hard rules." (from [*Designing Audio-Haptic Experiences*, WWDC19 session 810](https://developer.apple.com/videos/play/wwdc2019/810/), read 2026-10-22.) On the Apple Pay example, the talk says: "Our first idea was to mimic the waveform of the sound because it matches perfectly, but after some experimentation, we found that two simple taps actually did a better job." and "They don't always necessarily have to play the same thing, but they do have to play in the same tempo." (from [*Designing Audio-Haptic Experiences*, WWDC19 session 810](https://developer.apple.com/videos/play/wwdc2019/810/), read 2026-10-22.) The HIG: "For example, you generally want to match the intensity and sharpness of a haptic with the intensity and sharpness of the animation it accompanies. You can also synchronize sound with haptics" (from [*Playing haptics*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/playing-haptics), read 2026-10-22.) Core Haptics defines sharpness: "Haptic patterns with low sharpness have a round and organic feel, whereas haptic patterns with high sharpness feel more crisp and precise." (from [*hapticSharpness*, Core Haptics](https://developer.apple.com/documentation/corehaptics/chhapticevent/parameterid/hapticsharpness), read 2026-10-22.) So use a short, sharp transient haptic for a sound that starts abruptly and a continuous haptic for a smooth one. Match the sound's tempo and feel, not its waveform, and test by feel, because Apple says these are not hard rules. (Both sentences are the fork's conclusion from the quotes above. Apple's pages do not state either one.)

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

**VoiceOver announcements (iOS 17+).** Set a priority on each announcement and test its spoken output on a device.

- **Pick one of three priorities.** "You can specify the importance of this information using one of three announcement priorities: high, default, and low. High priority announcements can interrupt other speech and cannot be interrupted once started. Default priority announcements can interrupt existing speech but are interruptible if a new speech utterance is started. Low priority announcements are queued and spoken when other speech utterances have completed, if no new announcements have started." (from [*Build accessible apps with SwiftUI and UIKit*, WWDC 2023](https://developer.apple.com/videos/play/wwdc2023/10036/), read 2026-10-18.)
- **Set it on the string, then post.** In SwiftUI, "You specify an announcement priority using the accessibilitySpeechAnnouncementPriority property." (from [*AccessibilityNotification.Announcement*, Accessibility](https://developer.apple.com/documentation/accessibility/accessibilitynotification/announcement), read 2026-10-18.) In UIKit the same priority is the `UIAccessibilitySpeechAttributeAnnouncementPriority` key on an `NSAttributedString`, declared for iOS 17.0 and later (from [*UIAccessibilitySpeechAttributeAnnouncementPriority*, UIKit](https://developer.apple.com/documentation/uikit/uiaccessibilityspeechattributeannouncementpriority), read 2026-10-18).
- **A focus move is a second thing that can silence an announcement.** Apple's older guide says "When the layout of a screen changes, the VoiceOver focus ring, also known as the VoiceOver cursor, resets its position to the first element displayed on the screen from left to right and top to bottom." (from [*View Controller Programming Guide for iOS: Supporting Accessibility*, Apple archive](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/SupportingAccessibility.html), read 2026-10-18.) The Apple pages cited in this section do not say how a focus move affects a pending announcement, and the WWDC 2023 session covers only announcements against each other. Two forum reports describe the loss, so treat them as reports to test, not as rules. On iOS, an `.announcement` posted while changing screens "doesn't have time to finish its work and the focus is set to the first accessibility item"; the poster's fix was `.screenChanged` with the text as its argument (from [*VoiceOver focus is set to the first available element*, Apple Developer Forums, May 2022](https://developer.apple.com/forums/thread/705766), read 2026-10-18). On macOS, a high-priority announcement was never read when VoiceOver moved to the first element of a newly shown tab; the poster's workaround, reported for macOS 14 and later, posted `AccessibilityNotification.ScreenChanged` and then `AccessibilityNotification.Announcement` before switching the tab (from [*Setting accessibility focus when switching tabs*, Apple Developer Forums, Aug 2024](https://developer.apple.com/forums/thread/761728), read 2026-10-18).
- **When one action posts an announcement and moves focus, check what is spoken on a device.** The Simulator cannot do it: "Install your app on a physical device, since VoiceOver isn’t available on Simulator." (from [*Performing accessibility testing for your app*, Accessibility](https://developer.apple.com/documentation/accessibility/performing-accessibility-testing-for-your-app), read 2026-10-18.)

```swift
var done = AttributedString("Camera active")
done.accessibilitySpeechAnnouncementPriority = .high   // interrupts other speech, can't be interrupted
AccessibilityNotification.Announcement(done).post()
```

**VoiceOver's focus outline (iOS 17+).** Let the outline follow the shape the control shows. Override it only when the hit shape differs from the visible shape.

- **The outline follows the accessibility path, then the frame.** UIKit's `accessibilityPath` says "Setting the property, or overriding the method, will cause the assistive technology to prefer the path over the accessibility frame when highlighting the element." (from `accessibilityPath`, `UIAccessibility.h`, UIKit, iOS 27.0 SDK, read 2026-10-21.) Without a path, the outline uses the frame. The online page for `accessibilityPath` has no discussion text, so the header is the source.
- **In SwiftUI, the `.interaction` content shape covers accessibility.** Apple describes it as "The kind for hit-testing and accessibility." (from [*ContentShapeKinds.interaction*, SwiftUI](https://developer.apple.com/documentation/swiftui/contentshapekinds/interaction), read 2026-10-21.) The `accessibility` page repeats it: "To control the shape for accessibility and hit-testing, use the `interaction` kind." (from [*ContentShapeKinds.accessibility*, SwiftUI](https://developer.apple.com/documentation/swiftui/contentshapekinds/accessibility), read 2026-10-21.) Apple documents the plain `.contentShape(_:eoFill:)` only as "The hit testing shape for the view." (from [*contentShape(_:eoFill:)*, SwiftUI](https://developer.apple.com/documentation/swiftui/view/contentshape(_:eofill:)), read 2026-10-21.) Apple does not say which kind it sets, so check on a device whether a plain `.contentShape(Circle())` gives a circular outline.
- **Override the outline only when the hit shape differs.** `.contentShape(.accessibility, shape)` "causes the accessibility frame and path of the view’s underlying accessibility element to match the shape without adjusting the hit-testing shape, updating the visual focus ring that assistive apps, such as VoiceOver, draw". It adds "Updating the accessibility shape is only required if the shape or size used to hit-test significantly diverges from the visual shape of the view." (from [*ContentShapeKinds.accessibility*, SwiftUI](https://developer.apple.com/documentation/swiftui/contentshapekinds/accessibility), read 2026-10-21.) The kind is available from iOS 17.0.
- **No rule beyond Apple's text.** The Human Interface Guidelines Accessibility page says nothing about the shape of the outline.

```swift
Button(action: toggle) { LockIcon() }
    .contentShape(.interaction, Rectangle())   // wide hit area
    .contentShape(.accessibility, Circle())    // outline matches the visible circle
```

## 15. Typography — optical sizing, tracking, leading

Apple designs type to change shape with size; the same discipline applies on the web. (From *The Details of UI Typography*, WWDC 2020.)

- **Tracking (letter-spacing) is size-specific — never one value for all sizes.** "In a running app, the system font dynamically adjusts tracking at every point size." (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), read 2026-10-15.) The same page lists tracking in 1/1000 em by point size, so a value of -26 is `-0.026em`. Its table for SF Pro on iOS ("iOS, iPadOS, visionOS tracking values", tab "SF Pro") runs positive, then negative, then positive again: +41 at 6 pt, +6 at 11 pt, 0 at 12 pt, -6 at 13 pt, -26 at 17 pt, -4 at 23 pt, +3 at 24 pt, +14 at 28 to 30 pt, +10 at 36 to 40 pt, +6 at 52 to 56 pt, and 0 from 80 pt to 96 pt, the last row. Large text does not get negative tracking there: the negative values sit between 13 pt and 23 pt, and sizes from 24 pt up are positive until 0 at 80 pt. Those values are for Apple's system font. The page gives no tracking values for custom fonts.
- **Leading (line-height) tracks size inversely.** Tight on large headings, looser on body copy. Increase it for scripts with tall ascenders/descenders; tighten it for dense, information-heavy UI.
- **Build hierarchy from weight + size + leading as a set,** not size alone. Emphasize with weight — it adds presence without taking more space.
- **Respect the user's text-size setting** (Dynamic Type). Scale layout *with* the text — spacing in `rem`/`em`, not fixed px — so a larger font doesn't break the layout.
- **Need the actual point sizes or the larger-text layout rules?** Read `dynamic-type.md` in this skill folder: the iOS Dynamic Type table (Large and AX5, with the factor per style) and what a layout may change at larger sizes, each rule linked to its source.
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

## 18. App icons (iOS, iPadOS, macOS)

Apple's rules for a native app icon. The bullets down to the specification come from one page, the Human Interface Guidelines, App icons. The asset-catalog group after them comes from a second page, in Xcode's documentation. Every quote is word for word from that page's markdown copy, fetched 2026-10-24.

- **Provide layers, not a flattened image.** "Although you can provide a flattened image for your icon, layers give you the most control over how your icon design is represented. A layered app icon comes together to produce a sense of depth and vitality. On each platform, the system applies visual effects that respond to the environment and people’s interactions." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **An iOS icon is a background layer plus foreground layers, and the system adds Liquid Glass.** "iOS, iPadOS, macOS, and watchOS app icons include a background layer and one or more foreground layers that coalesce to create dimensionality. These icons take on Liquid Glass attributes like specular highlights, refraction, and translucency. These effects automatically adapt with the size of your icon, apply consistently across platforms, and can appear differently between system versions." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Build the layers in a design tool, then import them into Icon Composer.** "You use your favorite design tool to craft the individual foreground layers of your app icon. For iOS, iPadOS, macOS, and watchOS icons, you then import your icon layers into Icon Composer, a design tool included with Xcode and available from the Apple Developer website. In Icon Composer, you define the background layer for your icon, adjust your foreground layer placement, apply visual effects like specular highlights and refraction, annotate for default, dark, and mono appearance variants, test and preview your icon across system versions, and export your icon for use in Xcode." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.) The page says "mono" here and does not define it. Its other lists of appearances read "default, dark, clear, or tinted".
- **Give foreground layers clearly defined edges.** "Prefer clearly defined edges in foreground layers. To ensure system-drawn highlights and shadows look best, avoid soft and feathered edges on foreground layer shapes." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Bring layers into Icon Composer as vectors.** "Prefer vector graphics when bringing layers into Icon Composer. Unlike raster images, vector graphics (such as SVG or PDF) scale gracefully and appear crisp at any size. Outline artwork and convert text to outline in your design. For mesh gradients and raster artwork, prefer PNG format because it’s a lossless image format." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Hand over square, unmasked layers.** "Produce appropriately shaped, unmasked layers. The system masks all layer edges to produce an icon’s final shape. For iOS, iPadOS, and macOS icons, provide square layers so the system can apply rounded corners." The reason: "Providing layers with pre-defined masking negatively impacts specular highlight effects and makes edges look jagged." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Keep the design simple.** "Embrace simplicity in your icon design. Simple icons tend to be easiest for people to understand and recognize. An icon with fine visual features might look busy when rendered with system-provided shadows and highlights, and details may be hard to discern at smaller sizes." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Leave blurs and other effects to the system.** "Let the system handle blurring and other visual effects. The system dynamically applies visual effects to your app icon layers, so there’s no need to include specular highlights, drop shadows between layers, beveled edges, blurs, glows, and other effects. In addition to interfering with system-provided effects, custom effects are static, whereas the system supplies dynamic ones. If you do include custom visual effects on your icon layers, use them intentionally and test carefully with Icon Composer, on a simulated device in Device Hub, or on a physical device to make sure they appear as expected and don’t conflict with system effects." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Design for four appearances.** "In iOS, iPadOS, and macOS, people can choose whether their Home Screen app icons are default, dark, clear, or tinted in appearance." Apple adds: "You can design app icon variants for every appearance variant, and the system automatically generates variants you don’t provide." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Keep the features the same in every appearance.** "Keep your icon’s features consistent across appearances. To create a seamless experience, keep your icon’s core visual features the same in the default, dark, clear, and tinted appearances. Avoid creating custom icon variants that swap elements in and out with each variant, which may make it harder for people to find your app when they switch appearances." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Make dark and tinted icons sit beside system icons and widgets.** "Design dark and tinted icons that feel at home beside system app icons and widgets. You can preserve the color palette of your default icon, but be mindful that dark icons are more subdued, and clear and tinted icons are even more so. A great app icon is visible, legible, and recognizable, regardless of its appearance variant." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Derive the dark icon from the light one.** "Use your light app icon as the basis for your dark icon. Choose complementary colors that reflect the default design, and avoid excessively bright images. Color backgrounds generally offer the greatest contrast in dark icons." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **An alternate icon needs its own variants.** "Alternate app icons in iOS and iPadOS require their own dark, clear, and tinted variants." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)
- **Black backgrounds: Apple gives this rule under watchOS.** "Avoid using black for your icon’s background. Lighten a black background so the icon doesn’t blend into the display background." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.) The page puts it under its watchOS heading. Under iOS, iPadOS and macOS it reads "No additional considerations for iOS, iPadOS, or macOS." The page does not state this rule for iOS.
- **Specification.** In Apple's Specifications table, the row for iOS, iPadOS, macOS gives layout shape "Square", icon shape after system masking "Rounded rectangle (square)", layout size "1024x1024 px", style "Layered", and appearances "Default, dark, clear light, clear dark, tinted light, tinted dark". (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.)

**The asset-catalog route (Xcode).** Xcode's documentation describes a second way to supply an iOS or iPadOS icon, set beside the layered route above: image files in an asset catalog.

- **Apple points to Icon Composer for all variants.** The page opens with an "Important" note: "To create your app icon using Icon Composer for all variants, including different platforms and appearances, see" followed by a link to the Xcode article [`creating-your-app-icon-using-icon-composer`](https://developer.apple.com/documentation/xcode/creating-your-app-icon-using-icon-composer). (from [*Configuring your app icon using an asset catalog*, Xcode](https://developer.apple.com/documentation/xcode/configuring-your-app-icon), read 2026-10-24.)
- **The asset catalog offers three variations.** "iOS and iPadOS support three stylistic variations for app icons: Light, Dark, and Tinted." (from [*Configuring your app icon using an asset catalog*, Xcode](https://developer.apple.com/documentation/xcode/configuring-your-app-icon), read 2026-10-24.)
- **Supply the tinted icon as grayscale and the dark icon with a transparent background.** "Provide your tinted app icon as a grayscale image. Provide your dark app icon with a transparent background so the system-provided background can show through." (from [*Configuring your app icon using an asset catalog*, Xcode](https://developer.apple.com/documentation/xcode/configuring-your-app-icon), read 2026-10-24.)
- **The system can generate the variations.** "If you prefer, you can take advantage of the system’s automatically generated treatment that is applied to all app icons." (from [*Configuring your app icon using an asset catalog*, Xcode](https://developer.apple.com/documentation/xcode/configuring-your-app-icon), read 2026-10-24.)
- **The two Apple pages list different appearances.** This note is the fork's, not Apple's. The Xcode page names three variations, "Light, Dark, and Tinted", and its Appearance step reads "Then select Any, Dark, or Tinted from the Appearance pop-up menu." (from [*Configuring your app icon using an asset catalog*, Xcode](https://developer.apple.com/documentation/xcode/configuring-your-app-icon), read 2026-10-24.) The page does not mention clear. The Human Interface Guidelines say "default, dark, clear, or tinted" and list six appearances in the Specifications table. The Human Interface Guidelines change log has these entries. June 10, 2024: "Added guidance for creating dark and tinted app icon variants for iOS and iPadOS." June 9, 2025: "Updated guidance to reflect layered icons, consistency across platforms, and best practices for Liquid Glass." June 8, 2026: "Refined guidance for Liquid Glass." (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24.) The Xcode page's markdown copy carries no date. The fork cannot say which page is newer, and neither page explains the difference.

## 19. Launch screen

Apple's rules for the launch screen of an iOS app. The bullets come from three pages: the Human Interface Guidelines, Launching, then Xcode's documentation, Specifying your app’s launch screen, and the Bundle Resources documentation, UILaunchScreen. The two conclusions at the end also quote two subpages of the UILaunchScreen page, UIImageName and UIColorName. Every quote is word for word from the page text as developer.apple.com serves it, fetched 2026-10-25.

- **The system shows the launch screen first and swaps in the first screen.** "In iOS, iPadOS, and tvOS, the system displays your launch screen the moment your app or game starts and quickly replaces it with your first screen, giving people the impression that your experience is fast and responsive." (from [*Launching*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/launching), read 2026-10-25.)
- **Downplay it.** "Downplay the launch experience. A launch screen isn’t part of an onboarding experience or a splash screen, and it isn’t an opportunity for artistic expression." (from [*Launching*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/launching), read 2026-10-25.)
- **Make it nearly identical to the first screen.** "Design a launch screen that’s nearly identical to the first screen of your app or game. If you include elements that look different when launching completes, people may experience an unpleasant flash between the launch screen and your first screen. If your app or game displays a solid color before transitioning to the first screen, create a launch screen that displays only that solid color. Also make sure that your launch screen matches the device’s current orientation and appearance mode." (from [*Launching*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/launching), read 2026-10-25.)
- **Leave text off.** "Avoid including text on your launch screen, even if your first screen displays text. Because the content in a launch screen doesn’t change, any text you display won’t be localized." (from [*Launching*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/launching), read 2026-10-25.)
- **Do not advertise.** "Don’t advertise. The launch screen isn’t a branding opportunity. Avoid creating a screen that looks like a splash screen or an “About” window, and don’t include logos or other branding elements unless they’re a fixed part of your app’s first screen." (from [*Launching*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/launching), read 2026-10-25.)
- **Put a splash screen at the start of onboarding.** "If you need a splash screen, consider displaying it at the beginning of your onboarding flow." (from [*Launching*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/launching), read 2026-10-25.)
- **Every iOS app provides one.** "Every iOS app must provide a launch screen, a screen that displays while your app launches." (from [*Specifying your app’s launch screen*, Xcode](https://developer.apple.com/documentation/xcode/specifying-your-apps-launch-screen), read 2026-10-25.)
- **There are two ways to create it.** "You create a launch screen for your app in your Xcode project in one of two ways:" The page then lists "Information property list" and "User interface file". (from [*Specifying your app’s launch screen*, Xcode](https://developer.apple.com/documentation/xcode/specifying-your-apps-launch-screen), read 2026-10-25.)
- **A storyboard launch screen follows a short list of rules.** Under "Follow these guidelines when creating a launch screen storyboard:" the page lists, among others, "Use only UIKit classes." and "Don’t make any connections to your code, for example, don’t add actions or outlets." and "Don’t use any custom classes." and "Don’t use runtime attributes." (from [*Specifying your app’s launch screen*, Xcode](https://developer.apple.com/documentation/xcode/specifying-your-apps-launch-screen), read 2026-10-25.) The same list also asks for a single root view and no deprecated views such as `UIWebView`.
- **The information property list route is the `UILaunchScreen` dictionary.** Its keys are `UIColorName` ("The name of a color to use as the background color on the launch screen."), `UIImageName` ("The name of an image to display during app launch."), `UIImageRespectsSafeAreaInsets`, `UINavigationBar`, `UITabBar` and `UIToolbar`. (from [*UILaunchScreen*, Bundle Resources](https://developer.apple.com/documentation/bundleresources/information-property-list/uilaunchscreen), read 2026-10-25.) The page also says: "Use this key to configure the user interface during app launch in a way that doesn’t rely on storyboards. If you prefer to use storyboards, use `UILaunchStoryboardName` instead." (from [*UILaunchScreen*, Bundle Resources](https://developer.apple.com/documentation/bundleresources/information-property-list/uilaunchscreen), read 2026-10-25.)
- **A gradient background works through either route.** The Xcode page says a launch screen storyboard "contains basic UIKit views, and uses size classes and Auto Layout constraints to support different device sizes and resolutions." (from [*Specifying your app’s launch screen*, Xcode](https://developer.apple.com/documentation/xcode/specifying-your-apps-launch-screen), read 2026-10-25.) The `UIImageName` page says only this about sizing: "Because the image comes from your asset catalog, you can use slicing to provide a small image that works on many different platforms." (from [*UIImageName*, Bundle Resources](https://developer.apple.com/documentation/bundleresources/information-property-list/uilaunchscreen/uiimagename), read 2026-10-25.) The fork tested the information property list route. The `UILaunchScreen` dictionary held `UIColorName` (a named color), `UIImageName` (an image set whose light and dark images are each a 1 × 1024 px column with catalog slicing, mode 3-part-vertical, 1 px caps at the top and bottom, the center stretched) and `UIImageRespectsSafeAreaInsets` set to false. On the iOS 27.0 simulator the launch screen filled the whole screen with the image's vertical gradient, edge to edge behind the status bar and the home indicator. It had the same pixel values as a storyboard launch screen whose image view is pinned to the root view's four edges. The iOS 18.6 simulator gave the same result, read from a 60 fps screen recording. Both were read 2026-10-26. So a sliced `UIImageName` image stretches over the whole screen when `UIImageRespectsSafeAreaInsets` is false, and a gradient needs no storyboard. A storyboard with an image view pinned to its root view's edges draws the same. (This bullet is the fork's conclusion from that test, not from Apple's text. Apple's pages do not say it and do not mention gradients, and none of the six `UILaunchScreen` keys is documented as scaling or stretching the image.)
- **Take the launch screen's colors and images from the asset catalog, with a dark variant.** For `UIColorName`: "Provide a value for this key that’s the name of a color in your asset catalog." (from [*UIColorName*, Bundle Resources](https://developer.apple.com/documentation/bundleresources/information-property-list/uilaunchscreen/uicolorname), read 2026-10-25.) For `UIImageName`: "Provide a value for this key that’s the name of an image in your asset catalog." (from [*UIImageName*, Bundle Resources](https://developer.apple.com/documentation/bundleresources/information-property-list/uilaunchscreen/uiimagename), read 2026-10-25.) The Human Interface Guidelines ask that the launch screen "matches the device’s current orientation and appearance mode". (from [*Launching*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/launching), read 2026-10-25.) So build the launch screen from named colors and image sets that have a dark variant, and it follows the appearance mode. (This bullet is the fork's conclusion from the quotes above. Apple's pages do not say that a dark variant is what makes a launch screen follow the appearance mode.)

## 20. Widgets: appearances and the Edit Widget sheet

Apple's rules for an iOS or iPadOS Home Screen widget: how it looks under the four appearances, and what the app controls in the sheet where a person edits the widget. The fork calls that sheet the Edit Widget sheet. Apple's pages call it "the user interface for editing the widget" and "your widget’s editing-mode user interface". The appearance bullets come from the Human Interface Guidelines, Widgets, and from four WidgetKit and SwiftUI pages. The Edit Widget bullets come from the Human Interface Guidelines, from WidgetKit's Making a configurable widget, and from Xcode's Build Settings Reference. Every Apple quote is word for word from the page text as developer.apple.com serves it, fetched 2026-10-26, with the line wraps of the markdown copies joined and link markup dropped. A sentence that starts with "Observed" is not Apple's words. It records what was seen on an iOS 26 simulator on 2026-10-26, and Apple's pages do not state it. A sentence marked as the fork's conclusion is not Apple's words either.

- **A Home Screen widget has four appearances.** "On the Home Screen of iPad, iPhone, and Apple HomeView, people choose from different appearances for widgets: light, dark, clear, and tinted. In light and dark appearances, widgets have a full-color design. In a clear appearance, the system desaturates the widget and adds translucency, highlights, and the Liquid Glass material. In a tinted appearance, the system desaturates the widget and its content, then applies a person’s selected tint color." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.)
- **The full color rendering mode leaves your colors alone.** "Each appearance described above includes a rendering mode that depends on the platform and a person’s appearance settings:" The first item reads "The system uses the full color rendering mode for system family widgets across all platforms to display your widget in full color. It doesn’t change the color of your views." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.) The same list names a third mode for the Lock Screen: "The system uses the vibrant rendering mode for widgets on the Lock Screen of iPhone and iPad, and on iPhone in StandBy in low-light conditions." (from the same page, read 2026-10-26.)
- **Clear and tinted render the widget in the accented rendering mode.** When a person chooses a tinted or clear appearance for their Home Screen, the system "Renders your widget in the `accented` rendering mode" (from [*Optimizing your widget for accented rendering mode and Liquid Glass*, WidgetKit](https://developer.apple.com/documentation/widgetkit/optimizing-your-widget-for-accented-rendering-mode-and-liquid-glass), read 2026-10-26.) The Human Interface Guidelines describe the mode: "The system uses the accented rendering mode for system family widgets across all platforms and for accessory widgets on Apple Watch. In the accented rendering mode, the system removes the background and replaces it with a tinted color effect for a tinted appearance and a Liquid Glass background for a clear appearance. Additionally, it divides the widget’s views into an accent group and a primary group, and then applies a solid color to each group." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.)
- **Group the components into an accented and a primary group.** "Group widget components into an accented and a primary group. The accented rendering mode divides the widget’s view hierarchy into an accent group and a primary group. On iPhone, iPad, and Mac, the system tints primary and accented content white. On Apple Watch, the system tints primary content white and accented content in the color of the watch face." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.) On iPhone and iPad both groups are white, so moving a view into the accent group does not by itself make it look different from the primary group. (This sentence is the fork's conclusion from Apple's text. It was not tested, and Apple's pages do not say it outright.)
- **`widgetAccentable` marks the accent group, and the system keeps only the alpha channel.** "If the `accentable` parameter is `true`, the system adds that view and all of its subviews to the accent group. It puts all other views in the default group." "When applying the colors, the system treats the widget’s views as if they were template images. It ignores the view’s color — rendering the new colors based on the view’s alpha channel." (from [*widgetAccentable(_:)*, SwiftUI](https://developer.apple.com/documentation/swiftui/view/widgetaccentable(_:)), read 2026-10-26.) The WidgetKit article calls the second group the primary group: "Views you don’t mark as accentable are part of the primary group." (from [*Optimizing your widget for accented rendering mode and Liquid Glass*, WidgetKit](https://developer.apple.com/documentation/widgetkit/optimizing-your-widget-for-accented-rendering-mode-and-liquid-glass), read 2026-10-26.) The SwiftUI page says default group. (That the two names mean the same group is the fork's reading.)
- **Opacity survives the accented mode and color does not.** The same article says "When a person chooses a tinted or clear appearance for their Home Screen, the system:" and two items on its list read "Tints opaque images with a single white color" and "Maintains opacity for transparent content and gradients, and tints them white". (from [*Optimizing your widget for accented rendering mode and Liquid Glass*, WidgetKit](https://developer.apple.com/documentation/widgetkit/optimizing-your-widget-for-accented-rendering-mode-and-liquid-glass), read 2026-10-26.) Observed: opacity steps of one color survived the accented mode as grey and white stripes. So to tell two things apart in the accented mode, use opacity, shape or text, and do not use hue. (The last sentence is the fork's conclusion from Apple's text and the observation. Apple's pages do not state it.)
- **Do not carry meaning in color alone.** "Convey meaning without relying on specific colors to represent information. Widgets can appear monochromatic (with or without a custom tint color), and in watchOS, the system may invert colors depending on the watch face a person chooses. Use text and iconography in addition to color to express meaning." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.)
- **Use full-color images judiciously.** "Use full-color images judiciously. When a person chooses a tinted or clear appearance for their widgets, the system by default desaturates full-color images. You can choose to render images in full-color, even when a person chooses a tinted or clear widget appearance. However, full-color images in these appearances draw special attention to the widget, which might make it feel as if the widget doesn’t belong to the platform. For example, a full-color image in a widget might appear out of place when a person chooses a clear widget appearance. Consider reserving full-color images to represent media content, such as album art for a music app’s widget, and use full-color images with smaller dimensions than the size of the widget." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.) WidgetKit gives matching advice: "Reserve the `fullColor` rendering mode for images that represent media content, such as album artwork or a book cover." (from [*Optimizing your widget for accented rendering mode and Liquid Glass*, WidgetKit](https://developer.apple.com/documentation/widgetkit/optimizing-your-widget-for-accented-rendering-mode-and-liquid-glass), read 2026-10-26.)
- **An `Image` has four accented rendering modes.** The page opens "Constants that indicate the rendering mode for an `Image` in when displayed in a widget in `accented` mode." (the page prints "in when"). `accented`: "Specifies that the `Image` should be included as part of the accented widget group." `accentedDesaturated`: "Maps the luminance of the `Image` in to the alpha channel, replacing color channels with the color applied to the accent group." `desaturated`: "Maps the luminance of the `Image` in to the alpha channel, replacing color channels with the color applied to the default group." `fullColor`: "Specifies that the `Image` should be rendered at full color with no other color modifications. Only applies to iOS." (from [*WidgetAccentedRenderingMode*, WidgetKit](https://developer.apple.com/documentation/widgetkit/widgetaccentedrenderingmode), read 2026-10-26.) The modifier is `widgetAccentedRenderingMode(_:)` on `Image`, and the page warns: "If the `Image` is a subview for a group that has `widgetAccentable(true)` applied, this modifier may conflict." (from [*widgetAccentedRenderingMode(_:)*, SwiftUI](https://developer.apple.com/documentation/swiftui/image/widgetaccentedrenderingmode(_:)), read 2026-10-26.)
- **The parameter types build the Edit Widget sheet.** "WidgetKit uses the parameter type information to automatically create the user interface for editing the widget. For example, if the type is String, the person enters a string value. If the type is an Int, they use a number pad. For a parameter that is a predefined, static, list of values, define a custom type that conforms to AppEnum." (from [*Making a configurable widget*, WidgetKit](https://developer.apple.com/documentation/widgetkit/making-a-configurable-widget), read 2026-10-26.)
- **The order of the parameters is the order of the rows.** "The order of the parameters in the intent determines the order in which they appear when a person edits your widget." (from [*Making a configurable widget*, WidgetKit](https://developer.apple.com/documentation/widgetkit/making-a-configurable-widget), read 2026-10-26.)
- **A query supplies the choices and a default.** "When a person edits a widget with a custom intent that provides dynamic values, the system invokes the query object’s suggestedEntities() method to get the list of possible choices." Among the tasks of an `EntityQuery` the page lists "Providing a list of suggested values when a person edits a widget." and "Specifying a default value for a parameter." It adds "If your widget includes nonoptional parameters, you must supply a default value." and "A second option is to use a query type that implements `defaultResult()`, as shown in the next section." (from [*Making a configurable widget*, WidgetKit](https://developer.apple.com/documentation/widgetkit/making-a-configurable-widget), read 2026-10-26.)
- **Observed: an empty optional entity parameter shows its own title as its value, and `defaultResult()` fills the row.** On the iOS 26 simulator, an optional `AppEntity` parameter with no value showed the parameter's own `title` as its placeholder value in the Edit Widget sheet. A parameter titled "Item" would read "Item | Item". An `EntityQuery.defaultResult()` filled the row when the widget was added. Apple's page lists `defaultResult()` only as a way to supply a default and says nothing about the placeholder. So to make the row start with a real value, implement `defaultResult()` on the query. (The last sentence is the fork's conclusion from the observation.)
- **Choose between a fixed widget and a configurable one.** "Choose between automatically displaying content and letting people customize displayed information. In some cases, people need to configure a widget to ensure it displays the information that’s most useful for them." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.)
- **Asset-catalog colors style the editing interface.** "In your asset catalog, you can also specify the colors you want the system to use as it generates your widget’s editing-mode user interface." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.) The sentence names no build setting. Xcode's reference lists `ASSETCATALOG_COMPILER_WIDGET_BACKGROUND_COLOR_NAME` as "The name of a color resource to use as the background color for a widget." (from [*Build settings reference*, Xcode](https://developer.apple.com/documentation/xcode/build-settings-reference), read 2026-10-26.) Observed: setting that build setting to a color asset recolored the background of the Edit Widget sheet. (Linking the HIG sentence to this build setting is the fork's conclusion from that observation. Apple's pages do not state it.)
- **Write a succinct widget description that begins with an action verb.** "Write a succinct widget description. The widget gallery displays descriptions that help people understand what each widget does. Begin a description with an action verb — for example, “See the current weather conditions and forecast for a location” or “Keep track of your upcoming events and meetings.” Avoid including unnecessary phrases that reference the widget itself, like “This widget shows…,” “Use this widget to…,” or “Add this widget.” Use approachable language and sentence-style capitalization." (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26.) Apple names the widget gallery here. Observed: the widget's `.description(_:)` text also appeared under the title and the app name in the Edit Widget sheet.

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
| VoiceOver announcement that must be heard (iOS 17+) | Set its priority. If the same action moves focus, check the spoken output on a device, since the Simulator has no VoiceOver (from [*Build accessible apps with SwiftUI and UIKit*, WWDC 2023](https://developer.apple.com/videos/play/wwdc2023/10036/) and [*Performing accessibility testing for your app*, Accessibility](https://developer.apple.com/documentation/accessibility/performing-accessibility-testing-for-your-app), read 2026-10-18) | `accessibilitySpeechAnnouncementPriority` (`.high`, default, `.low`) then `AccessibilityNotification.Announcement(_:).post()`; UIKit `UIAccessibilitySpeechAttributeAnnouncementPriority` |
| VoiceOver outline on a non-rectangular control (iOS 17+) | Follows the content shape. Override only when the hit shape differs from the visible shape (from `accessibilityPath`, `UIAccessibility.h`, iOS 27.0 SDK, and [*ContentShapeKinds.accessibility*, SwiftUI](https://developer.apple.com/documentation/swiftui/contentshapekinds/accessibility), read 2026-10-21) | `.contentShape(.accessibility, shape)` (iOS 17+, from [*ContentShapeKinds.accessibility*, SwiftUI](https://developer.apple.com/documentation/swiftui/contentshapekinds/accessibility)); `.contentShape(.interaction, shape)` for hit-testing and accessibility together (from [*ContentShapeKinds.interaction*, SwiftUI](https://developer.apple.com/documentation/swiftui/contentshapekinds/interaction)); UIKit `accessibilityPath` |
| App icon (iOS, iPadOS, macOS) | Foreground layers with clearly defined edges over a background layer, square and unmasked, vector where possible, imported into Icon Composer. The system draws highlights, shadows and blurs. Keep the features the same across appearances (from [*App icons*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/app-icons), read 2026-10-24) | 1024x1024 px, layered. Appearances: default, dark, clear light, clear dark, tinted light, tinted dark |
| App icon from images in an asset catalog (iOS, iPadOS) | Tinted icon as a grayscale image, dark icon with a transparent background, or the system's automatically generated treatment. Apple points to Icon Composer for all variants (from [*Configuring your app icon using an asset catalog*, Xcode](https://developer.apple.com/documentation/xcode/configuring-your-app-icon), read 2026-10-24) | Appearance pop-up: Any, Dark, Tinted. The Human Interface Guidelines list default, dark, clear and tinted, so the counts differ between the two pages |
| Launch screen | The first screen without its text and data: its background only, following the appearance mode, no logo, no text (from [*Launching*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/launching), read 2026-10-25) | `UILaunchScreen` with `UIColorName` and a sliced `UIImageName` for a flat color or a gradient, or `UILaunchStoryboardName` with a storyboard of UIKit views (from [*Specifying your app’s launch screen*, Xcode](https://developer.apple.com/documentation/xcode/specifying-your-apps-launch-screen) and [*UILaunchScreen*, Bundle Resources](https://developer.apple.com/documentation/bundleresources/information-property-list/uilaunchscreen), read 2026-10-25). The gradient case is the fork's tested conclusion, section 19 |
| Home Screen widget in the tinted and clear appearances (iOS, iPadOS) | Design for the accented rendering mode. The system removes the background, divides the views into an accent group and a primary group, and colors each group with one solid color, white on iPhone and iPad (from [*Widgets*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/widgets), read 2026-10-26) | `.widgetAccentable()` on the accent group (from [*widgetAccentable(_:)*, SwiftUI](https://developer.apple.com/documentation/swiftui/view/widgetaccentable(_:)), read 2026-10-26). `Image` cases `accented`, `accentedDesaturated`, `desaturated` and `fullColor`, with `fullColor` only for media content (from [*WidgetAccentedRenderingMode*, WidgetKit](https://developer.apple.com/documentation/widgetkit/widgetaccentedrenderingmode) and [*Optimizing your widget for accented rendering mode and Liquid Glass*, WidgetKit](https://developer.apple.com/documentation/widgetkit/optimizing-your-widget-for-accented-rendering-mode-and-liquid-glass), read 2026-10-26). Telling things apart by text, shape or opacity instead of hue is the fork's conclusion, section 20 |
| Edit Widget sheet of a widget with a `WidgetConfigurationIntent` (iOS, iPadOS) | The `@Parameter` types build the rows and their order sets the row order. A query supplies the choices and a default (from [*Making a configurable widget*, WidgetKit](https://developer.apple.com/documentation/widgetkit/making-a-configurable-widget), read 2026-10-26) | `@Parameter(title:)` in row order, `EntityQuery.suggestedEntities()` and `defaultResult()`. The build setting `ASSETCATALOG_COMPILER_WIDGET_BACKGROUND_COLOR_NAME` exists (from [*Build settings reference*, Xcode](https://developer.apple.com/documentation/xcode/build-settings-reference), read 2026-10-26). Observed on an iOS 26 simulator, not Apple's words: that setting recolors the sheet's background, the widget `.description(_:)` text shows in the sheet, and an empty optional entity parameter shows its own title as its value. Section 20 |
