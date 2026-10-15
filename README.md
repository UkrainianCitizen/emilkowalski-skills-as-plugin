> **Fork note:** This is a fork of [emilkowalski/skills](https://github.com/emilkowalski/skills) that adds Claude Code plugin support (`.claude-plugin/plugin.json` and `marketplace.json`) and extends some skills with material upstream lacks. Upstream does not accept outside pull requests, so the plugin layer and the extensions live only here. Skills not listed below are unchanged.
>
> Changes to skills in this fork:
>
> - **apple-design** — Dynamic Type and the Large Content Viewer: grow first (scaling is always preferred), the viewer is the exception (only for UI that must stay small, with the tab bar as Apple's example), how a fixed-size control opts in (UIKit and SwiftUI), and the web translation, plus how icons scale beside larger text (essential scales, decorative holds back, text wraps under the unscaled image) and the 44x44 pt button hit region with its WCAG target-size counterparts. Sources: *Large Content Viewer: Ensuring Readability for Everyone* (WWDC 2019, session 261), Apple's `UILargeContentViewerInteraction` and SwiftUI `accessibilityShowsLargeContentViewer()` documentation, *Get started with Dynamic Type* (WWDC 2024, session 10074), Apple's Human Interface Guidelines, Buttons, WCAG 2.2 Understanding SC 2.5.8 Target Size (Minimum) and SC 2.5.5 Target Size (Enhanced).
> - **apple-design** — tracking correction: upstream says large display text wants negative tracking, tighten headings, body near `0`, and shows `letter-spacing: -0.02em`. Apple's Human Interface Guidelines, Typography, says the system font "dynamically adjusts tracking at every point size" and lists tracking values per size. In its table for SF Pro on iOS ("iOS, iPadOS, visionOS tracking values", tab "SF Pro") the values are negative only from 13 pt to 23 pt, positive from 24 pt up (+14 at 28 to 30 pt) and 0 at 80 pt. The tracking rule, its Quick Reference row and the `.display` CSS example now follow that table. The page gives no tracking values for custom fonts.
> - **apple-design** — `dynamic-type.md`, a reference file next to `SKILL.md`: the iOS Dynamic Type point-size table (Large and AX5, with the growth factor per style, small styles grow by a larger factor so sizes converge) and the layout rules for larger text (layout keeps order and hierarchy, text wraps, items stack, columns drop, not every element has to grow). Source: Apple's Human Interface Guidelines, Typography and Accessibility. `SKILL.md` points to it from the Dynamic Type bullets.
> - **Source links:** every rule this fork adds or corrects in the skills names its source with an inline link and the date it was read, so each claim is one fetch to verify. Upstream lines the fork has not changed keep their original form.
>
> ```bash
> /plugin marketplace add UkrainianCitizen/emilkowalski-skills-as-plugin
> /plugin install emilkowalski-skills@emilkowalski-skills
> ```


<a href="http://aiforui.dev/">
<img width="360" height="202" alt="opengraph-image 2" src="https://github.com/user-attachments/assets/84fca9a6-0b2b-4927-8f48-f3ce194a5c47" />
</a>

# Skills For Designers and Engineers

[![skills.sh](https://skills.sh/b/emilkowalski/skills)](https://skills.sh/emilkowalski/skills)

For designers and engineers to help them build better user interfaces.

Knowing whether you made a right choice when it comes to animations, or design in general, is hard. These skills aim to help you get to those right decisions faster.

They are based on my years of experience working at companies like Vercel and Linear.

All the skills here are a side-effect of domain-expertise. AI doesn’t replace such expertise, it amplifies what you can get out of it and makes you way better relative to others.

So learn to code, design, or develop expertise in any other field. It’s extremely valuable.

You can stay up to date with my skills here:

[Sign Up To The Newsletter](https://aiforui.dev/skills)

## Install

```bash
npx skills@latest add emilkowalski/skills
```

Or, in Claude Code, as a plugin:

```bash
/plugin marketplace add emilkowalski/skills
/plugin install emilkowalski-skills@emilkowalski-skills
```

## Why use it?

Agents don’t have great taste

I have seen plenty of times that agents don’t pick the right ingredients for an animation. An `ease-in` easing for an enter animation when it’s supposed to be `ease-out` ([here’s why](https://emilkowal.ski/ui/7-practical-animation-tips#4.-choose-the-right-easing)). Or they choose a solid border instead of a semi-transparent shadow for your UIs.

All these small things compound and make your interface either amazing, or just... not that great.

As explained in [Agents with Taste](https://emilkowal.ski/ui/agents-with-taste), these skills list all the little mistakes agents can potentially make and explain how to fix them.

This is your shortcut to great interfaces. A shortcut to stand out in a sea of slop.

## Reference

- **[emil-design-eng](./skills/emil-design-eng/SKILL.md)** — The main skill that consists of mostly animation, but also some design advice.
- **[animate](./skills/animate/SKILL.md)** — Builds an animation from scratch while choosing the correct curve, duration, properties, and so on.
- **[animate-expo](./skills/animate-expo/SKILL.md)** — The same bar, for React Native and Expo: gestures, sheets, haptics, screen transitions, and keeping motion off the JS thread.
- **[review-animations](./skills/review-animations/SKILL.md)** — Review your animations in a strict way, based on my rules.
- **[improve-animations](./skills/improve-animations/SKILL.md)** — Audit all the animations in your codebase and get prioritized, self-contained plans that any agent can execute.
- **[find-animation-opportunities](./skills/find-animation-opportunities/SKILL.md)** — Search your UI for places that would genuinely benefit from motion, while also telling you what not to animate.
- **[animation-vocabulary](./skills/animation-vocabulary/SKILL.md)** — Get better animations from an AI by telling it exactly what you want by using the right words.
- **[apple-design](./skills/apple-design/SKILL.md)** — Apple’s principles for interface design and fluid motion, distilled from their WWDC design talks and translated for the web.
- **[write-swift](./skills/write-swift/SKILL.md)** — Write modern Swift. Includes: value types, Swift 6 concurrency, generics, performance, and Swift Testing.
- **[pick-ui-library](./skills/pick-ui-library/SKILL.md)** — Have your agent pick the right library for the task based on libraries I use and trust, instead of letting AI hand-roll a toast component or install an abandoned package.
- **[prototype](./skills/prototype/SKILL.md)** — Build multiple different versions of a UI piece you describe and go through them using a switcher.
- **[mobile-native](./skills/mobile-native/SKILL.md)** — Make your web app feel native on a phone: sticky hover states, tap highlight flashes, the 100vh bug, inputs that zoom the page, laggy taps, safe areas, and the rest of the small fixes that separate a website from an app.
- **[break-ui](./skills/break-ui/SKILL.md)** — Try to break the UI you built with worst-case data: long names, unusual emails, one-letter names, huge counts, empty lists, long labels.
- **[ask-sonner](./skills/ask-sonner/SKILL.md)** — Your guide to working with [Sonner](https://sonner.emilkowal.ski), my toast library. Contains setup, styling, recipes, and fixes for the most common issues.
