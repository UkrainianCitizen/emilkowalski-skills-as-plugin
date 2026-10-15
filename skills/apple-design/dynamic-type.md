# Dynamic Type: point sizes and layout rules

Reference for `SKILL.md`. Read it when a question needs the actual point sizes at an accessibility text setting, or the layout rules for larger text.

## Point sizes at the accessibility settings

Dynamic Type grows small text styles by a larger factor than large ones, so text sizes converge at the accessibility sizes. A role tied to a smaller style ends up close to a larger role. This is the platform's design, not a bug in a role system. (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), tables "iOS, iPadOS Dynamic Type sizes" and "iOS, iPadOS larger accessibility type sizes", read 2026-10-14.)

iOS point sizes at Large (the default) and AX5 (the largest accessibility size). Factor = AX5 / Large.

| Style | Large | AX5 | Factor |
|---|---|---|---|
| Large Title | 34 | 60 | 1.76 |
| Title 2 | 22 | 56 | 2.55 |
| Headline, Body | 17 | 53 | 3.12 |
| Footnote | 13 | 44 | 3.38 |
| Caption 1 | 12 | 43 | 3.58 |

The Large and AX5 columns are from the HIG tables. The factor column is computed: 60/34 = 1.76, 56/22 = 2.55, 53/17 = 3.12, 44/13 = 3.38, 43/12 = 3.58.

Caveat: single source. The HIG table is the authority for these values, and no second source confirms them.

## What a larger text size may change

A larger text size makes text bigger. The layout stays the same screen with the same order and hierarchy. It changes only where bigger text would crowd or truncate: lines wrap, side-by-side items stack, columns drop. The relative hierarchy between text elements holds at every size. An element sized to the screen rather than to a text style may give up height to the text. Not every element has to grow: text the person does not need to read larger, such as tab titles, may hold its size.

Apple's wording:

- "Be sure to maintain the relative hierarchy and visual distinction of text elements when people adjust text sizes." (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), read 2026-10-14.)
- "When someone chooses a larger text size, they typically want to make the content they care about easier to read; they don't always want to increase the size of every word on the screen." (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), read 2026-10-14.)
- "Make sure your app's layout adapts to all font sizes." (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), read 2026-10-14.)
- "Keep text truncation to a minimum as font size increases." (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), read 2026-10-14.)
- "Consider adjusting your layout at large font sizes … consider using a stacked layout where text appears above secondary items." (from [*Typography*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/typography), read 2026-10-14.)
- "Support larger text sizes." (from [*Accessibility*, Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/accessibility), read 2026-10-14.)

Caveat: single source. Both pages are the HIG. The summary paragraph above, including the rule that an element sized to the screen may give up height to the text, is a reading of these quotes, not an HIG sentence.
