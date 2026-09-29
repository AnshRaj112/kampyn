# Frontend Styling Standards

## Purpose

This document defines the styling architecture, design system, CSS conventions, responsive design, theming, and visual consistency standards for the KAMPYN frontend.

KAMPYN uses Next.js, React, TypeScript, and a combination of Tailwind CSS and CSS Modules/SCSS where appropriate. Styling must remain modular, reusable, accessible, responsive, maintainable, and consistent across all platform modules.

The goal is to:
- Establish a consistent visual identity across KAMPYN.
- Build reusable design tokens and components.
- Avoid duplicated styles and conflicting CSS.
- Support responsive layouts across mobile, tablet, desktop, and large displays.
- Maintain accessibility and usability.
- Support light and dark themes where required.
- Keep styling predictable across independently developed features.
- Minimize unnecessary CSS and runtime overhead.
- Support university-specific branding and self-hosted deployments without rewriting feature styles.

Styling decisions must follow the design system rather than introducing isolated visual conventions in individual components.

---

## 1. Core Principles

### 1.1 Design-System First

All production UI must use the shared design system wherever a suitable token, component, or pattern exists.

The design system defines:
- Color tokens.
- Typography.
- Spacing.
- Sizing.
- Layout constraints.
- Borders and radii.
- Shadows and elevation.
- Breakpoints.
- Motion and transitions.
- Component variants.
- Theme behavior.
- Accessibility requirements.

Do not introduce arbitrary values when an appropriate design token already exists.

Feature-specific styles are permitted when they represent genuinely unique requirements and cannot reasonably be expressed through existing tokens or component variants.

### 1.2 Consistency Over Individual Preference

Visual consistency must be maintained across:
- Food ordering.
- Food court management.
- Hostel and guest house booking.
- Washing machine scheduling.
- Library vacancy.
- Shuttle booking.
- Community and chat.
- Notifications.
- Inventory.
- Complaints.
- HR management.
- Administrative dashboards.
- Marketing and onboarding pages.

Components that perform similar functions should share common styling patterns, interaction states, and accessibility behavior.

### 1.3 Separation of Structure and Presentation

- React components define UI structure and behavior.
- Design tokens define reusable visual values.
- Tailwind CSS handles utility-driven layout and common styling.
- CSS Modules or SCSS handle complex, component-specific styles where appropriate.
- Global styles are limited to application-wide foundations.
- Theme configuration controls visual variation.

Do not mix business logic into styling definitions.

### 1.4 Avoid Unnecessary Complexity

Prefer:
- Existing design tokens over custom values.
- Shared components over repeated styling.
- Simple layout primitives over deeply nested wrappers.
- CSS layout features over JavaScript-based layout calculations.
- CSS transitions over JavaScript animation for simple state changes.

Do not introduce additional styling libraries, runtime CSS engines, or abstraction layers without a clear technical justification.

### 1.5 Accessibility Is Mandatory

Styling must not compromise accessibility.

All UI must account for:
- Sufficient color contrast.
- Visible keyboard focus.
- Readable typography.
- Clear interaction states.
- Touch target sizing.
- Reduced-motion preferences.
- Zoom and text scaling.
- High-contrast and forced-color environments where applicable.
- Responsive reflow.

A visually correct design that cannot be used accessibly is not considered complete.

---

## 2. Styling Technology Standards

### 2.1 Tailwind CSS

Tailwind CSS is the preferred tool for:
- Flexbox and grid layouts.
- Spacing.
- Responsive utilities.
- Typography utilities.
- Common visual properties.
- Simple hover, focus, active, and disabled states.
- Standard component composition.

Use Tailwind utilities for straightforward styling that remains readable and maintainable.

Example:

```tsx
export function FoodItemCard() {
  return (
    <article className="flex flex-col gap-3 rounded-xl border p-4">
      <h3 className="text-base font-semibold">
        Vegetable Biryani
      </h3>
      <p className="text-sm text-muted-foreground">
        Freshly prepared meal
      </p>
    </article>
  );
}
```

Do not use long, unreadable utility strings to implement complex visual behavior that would be clearer in a reusable component or CSS Module.

### 2.2 CSS Modules and SCSS

CSS Modules or SCSS may be used for:
- Complex component-specific layouts.
- Sophisticated pseudo-element styling.
- Complex animations.
- Advanced selectors.
- Reusable local style structures.
- Third-party library overrides that cannot be handled cleanly through utilities.
- Visualizations or components requiring detailed CSS control.

Styles must remain scoped to the owning component or feature wherever practical.

Example:

```text
components/
└── food-item-card/
    ├── food-item-card.tsx
    ├── food-item-card.module.scss
    └── index.ts
```

Use descriptive, component-specific class names.

```scss
.card {
  display: flex;
  flex-direction: column;
  gap: var(--space-3);
}

.title {
  color: var(--color-text-primary);
  font-weight: var(--font-weight-semibold);
}
```

Avoid global selectors that unintentionally affect unrelated components.

### 2.3 Choosing Between Tailwind and CSS Modules

Use Tailwind when:
- The styling is straightforward.
- The layout can be expressed clearly with utilities.
- The component follows established design-system patterns.

Use CSS Modules or SCSS when:
- The styling is complex or highly specialized.
- The stylesheet improves readability.
- Advanced CSS features are needed.
- A component has substantial styling logic that would otherwise produce difficult-to-maintain JSX.

Do not duplicate the same styling responsibility across Tailwind and CSS Modules without a clear reason.

### 2.4 Avoid Inline Styles

Inline styles should not be the default styling mechanism.

Avoid:

```tsx
<div style={{ color: "#4ea199", padding: "16px" }}>
  Content
</div>
```

Prefer design tokens and reusable styling:

```tsx
<div className="text-primary p-4">
  Content
</div>
```

Inline styles are acceptable for narrowly scoped dynamic values that cannot be expressed cleanly through classes, such as values computed for a chart or a controlled visual property.

Dynamic values must be validated and must not be used to inject untrusted CSS.

---

## 3. Design Tokens

Design tokens are the single source of truth for reusable visual values.

Tokens must be centralized and consumed through the approved styling system.

### 3.1 Token Categories

KAMPYN must define tokens for:

| Category | Examples |
|---|---|
| Colors | Primary, secondary, background, surface, border, text |
| Semantic colors | Success, warning, error, information |
| Typography | Font family, size, weight, line height, letter spacing |
| Spacing | Small, medium, large, section spacing |
| Sizing | Control heights, icon sizes, content widths |
| Radius | Small, medium, large, pill |
| Elevation | Surface levels, shadows |
| Borders | Widths, styles, semantic border colors |
| Breakpoints | Mobile, tablet, desktop, wide |
| Motion | Duration, easing, transition behavior |
| Layers | Dropdown, sticky header, modal, toast |
| Layout | Page width, sidebar width, content spacing |

Tokens should be semantic where possible rather than tied only to their visual appearance.

Prefer `--color-text-secondary` over `--gray-500` when the token's role is secondary text.

### 3.2 Color Tokens

Example foundation:

```css
:root {
  --color-primary: #4ea199;
  --color-primary-hover: #438f88;
  --color-primary-active: #397d77;

  --color-secondary: #6fc3bd;

  --color-background: #ffffff;
  --color-surface: #f8fafc;
  --color-surface-raised: #ffffff;

  --color-text-primary: #17212b;
  --color-text-secondary: #5d6875;
  --color-text-muted: #778391;

  --color-border: #dce3e8;

  --color-success: #23834b;
  --color-warning: #a66a12;
  --color-error: #c43b3b;
  --color-info: #2563a6;
}
```

These are illustrative starting tokens. The canonical palette must be validated against the approved KAMPYN brand system and accessibility requirements.

### 3.3 Semantic Color Usage

Use semantic tokens for interface meaning.

Examples:
- Primary actions use the primary action token.
- Destructive actions use the error token.
- Completed states use the success token.
- Warnings use the warning token.
- Secondary descriptions use the secondary text token.
- Disabled controls use disabled tokens.

Do not use color alone to communicate status. Pair color with text, icons, patterns, or another accessible cue.

### 3.4 Dark Theme

Dark theme must use a distinct semantic palette rather than simply inverting every color.

Example:

```css
[data-theme="dark"] {
  --color-background: #101719;
  --color-surface: #172124;
  --color-surface-raised: #1e2a2e;

  --color-text-primary: #f2f6f6;
  --color-text-secondary: #bac7c9;
  --color-text-muted: #93a2a5;

  --color-border: #304044;

  --color-primary: #6fc3bd;
  --color-primary-hover: #83d0ca;
  --color-primary-active: #58aaa4;
}
```

Dark theme must be tested independently for:
- Contrast.
- Borders and separators.
- Shadows and elevation.
- Disabled states.
- Form controls.
- Overlays.
- Charts and data visualizations.
- Third-party widgets.
- Focus visibility.

Do not rely on browser automatic color inversion as the theme implementation.

### 3.5 Token Naming

Use consistent, semantic names.

Preferred:

```css
--color-background
--color-surface
--color-text-primary
--color-text-secondary
--color-border
--space-4
--radius-lg
--shadow-overlay
--duration-normal
```

Avoid:
- Unclear abbreviations.
- Duplicate tokens for identical values without a reason.
- Component-specific tokens in global foundations.
- Names that encode a temporary implementation detail.

### 3.6 Token Ownership

- Global tokens belong in the shared design system.
- Component tokens belong to reusable component styles.
- Feature-specific tokens belong to the owning feature only when the values are genuinely unique.
- Tenant branding overrides must use documented theme variables.
- One-off overrides must not silently redefine global tokens.

---

## 4. Typography

Typography must support readability, visual hierarchy, and accessibility.

### 4.1 Font Families

Use approved, consistent font families across the application.

- Prefer a small number of font families.
- Provide sensible system fallbacks.
- Avoid loading unnecessary font weights.
- Use optimized font loading in Next.js.
- Avoid blocking initial rendering with excessive font requests.
- Support appropriate language-specific glyph coverage.

### 4.2 Typography Scale

Define a consistent typography scale for:
- Display headings.
- Page titles.
- Section headings.
- Card headings.
- Body text.
- Secondary text.
- Labels.
- Captions.
- Metadata.
- Buttons and controls.

Example:

```css
:root {
  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;
  --font-size-2xl: 1.5rem;
  --font-size-3xl: 1.875rem;
  --font-size-4xl: 2.25rem;

  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;
}
```

The final scale should be defined in the shared theme and adjusted for responsive needs.

### 4.3 Typography Rules

- Use headings in a logical hierarchy.
- Do not select heading levels based solely on visual size.
- Keep body text readable at normal zoom.
- Use suitable line heights for paragraphs and dense information.
- Avoid excessive uppercase text.
- Avoid long paragraphs with overly wide line lengths.
- Prevent text clipping in cards, buttons, forms, and tables.
- Support browser text scaling.
- Avoid fixed-height containers that cut off translated or enlarged text.

### 4.4 Dynamic and Multilingual Content

KAMPYN may display names, descriptions, messages, and university-specific content that vary in length and language.

Layouts must tolerate:
- Long names.
- Multi-line titles.
- Different character widths.
- Mixed scripts.
- Right-to-left content where supported.
- User-generated text.
- Unexpected line breaks.

Use logical CSS properties such as `margin-inline`, `padding-inline`, `inset-inline`, and `text-align: start` where appropriate.

---

## 5. Spacing and Layout

### 5.1 Spacing Scale

Use a consistent spacing scale rather than arbitrary margins and padding.

Example:

```css
:root {
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-5: 1.25rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-10: 2.5rem;
  --space-12: 3rem;
  --space-16: 4rem;
}
```

Use spacing tokens for:
- Component gaps.
- Card padding.
- Form field spacing.
- Page gutters.
- Section separation.
- Navigation layout.

Avoid arbitrary spacing values unless the layout requires a specific exception.

### 5.2 Layout Systems

Prefer CSS Flexbox and Grid for layout.

Use:
- Flexbox for one-dimensional alignment.
- Grid for two-dimensional layout.
- Container constraints for readable content width.
- CSS positioning for overlays and anchored elements.
- Sticky positioning for appropriate persistent navigation or context.

Avoid using absolute positioning to construct primary page layouts.

### 5.3 Content Width

Page layouts must define readable content boundaries.

Use:
- Full-width layouts for dashboards and data-heavy workspaces where appropriate.
- Constrained widths for long-form text and forms.
- Responsive gutters for different viewport sizes.
- Consistent alignment across related pages.

Avoid forcing all pages into the same fixed width regardless of their content.

### 5.4 Spacing Consistency

Related components must use consistent:
- Internal padding.
- Inter-component gaps.
- Section spacing.
- Alignment.
- Grid gaps.
- Page gutters.

Do not use repeated manual margin adjustments to compensate for inconsistent component design.

---

## 6. Responsive Design

All KAMPYN interfaces must support mobile, tablet, desktop, and wide screens.

### 6.1 Mobile-First

Start with the smallest practical viewport and progressively enhance the layout.

- Avoid desktop-only assumptions.
- Avoid fixed widths that cause horizontal overflow.
- Use flexible layouts.
- Scale typography and spacing appropriately.
- Ensure primary actions remain accessible.
- Adapt navigation and complex controls for touch input.

### 6.2 Breakpoints

Use centralized breakpoints consistent with the design system.

Example:

```css
:root {
  --breakpoint-sm: 40rem;
  --breakpoint-md: 48rem;
  --breakpoint-lg: 64rem;
  --breakpoint-xl: 80rem;
}
```

When using Tailwind, keep its breakpoint configuration aligned with the design-system definitions.

Do not create arbitrary breakpoints for each component without a documented need.

### 6.3 Responsive Layout Rules

- Use flexible widths such as `minmax()`, `min()`, `max()`, and `clamp()` where appropriate.
- Avoid unnecessary fixed heights.
- Ensure grid columns collapse gracefully.
- Allow tables and data-heavy content to use suitable responsive patterns.
- Keep important actions visible and reachable.
- Avoid layouts that depend on hover.
- Support touch input without relying on precise pointer movement.

### 6.4 Responsive Data-Dense Interfaces

For dashboards, inventory, orders, and administrative interfaces:
- Preserve critical information.
- Use appropriate table overflow or alternative card layouts.
- Avoid shrinking text until it becomes unreadable.
- Keep essential actions available.
- Provide a clear strategy for narrow screens.
- Avoid hiding important operational information without an alternative.

### 6.5 Testing

Test representative viewport sizes, including:
- Narrow mobile.
- Large mobile.
- Tablet portrait and landscape.
- Laptop.
- Desktop.
- Wide desktop.

Verify actual content behavior, not only the appearance of empty placeholder layouts.

---

## 7. Component Styling

### 7.1 Shared Components

Reusable components must expose intentional visual variants rather than relying on consumers to override internal CSS.

Examples:
- Buttons.
- Inputs.
- Selects.
- Cards.
- Dialogs.
- Badges.
- Tabs.
- Navigation items.
- Tooltips.
- Tables.
- Dropdown menus.
- Toasts.
- Empty states.

Variants should be explicit, typed, and limited to supported use cases.

Example:

```ts
type ButtonVariant =
  | "primary"
  | "secondary"
  | "outline"
  | "ghost"
  | "destructive";
```

Avoid creating a new visual variant for every feature-specific preference.

### 7.2 Component Encapsulation

A component should own its internal visual structure and state presentation.

- Avoid styling internal elements through fragile descendant selectors.
- Avoid depending on generated class names from external libraries.
- Expose stable styling hooks only when needed.
- Use composition for complex layouts.
- Keep variants consistent with the design system.

### 7.3 Class Composition

When dynamic classes are needed:
- Use the project's approved class composition utility.
- Keep conditional styling readable.
- Avoid repeated complex ternary expressions.
- Centralize shared variants when appropriate.
- Avoid arbitrary string concatenation that can produce invalid or conflicting classes.

### 7.4 Avoid Style Leakage

- Prefer CSS Modules for scoped styles.
- Avoid broad selectors such as `div`, `section`, or `button` in feature styles.
- Limit global resets and typography rules to the design-system foundation.
- Avoid `!important` except for documented third-party integration constraints.
- Do not override unrelated components from a feature stylesheet.

---

## 8. Theme Architecture

KAMPYN must support a consistent theme architecture that can accommodate platform-wide design and university-specific branding.

### 8.1 Theme Sources

The design system should support:
- Default platform theme.
- Light and dark appearance.
- Approved tenant branding overrides.
- System appearance preference, where enabled.
- User-selected appearance, where supported.

The precedence rules must be explicit and consistent.

### 8.2 CSS Variables

Use CSS custom properties for runtime theme values.

```css
:root {
  --color-primary: #4ea199;
  --color-background: #ffffff;
  --color-text-primary: #17212b;
}

[data-theme="dark"] {
  --color-primary: #6fc3bd;
  --color-background: #101719;
  --color-text-primary: #f2f6f6;
}
```

Components should consume semantic variables rather than embedding theme-specific hex values.

### 8.3 Tenant Branding

Tenant branding may customize approved values such as:
- Primary and secondary brand colors.
- Logo and favicon.
- Approved typography options.
- Limited theme preferences.

Tenant customization must not:
- Break accessibility contrast requirements.
- Override critical semantic states without validation.
- Inject arbitrary CSS or scripts.
- Alter security-sensitive interface behavior.
- Require feature-specific styles to be rewritten.

Validate tenant-provided theme values and constrain them to approved configuration fields.

### 8.4 Theme Switching

Theme switching must:
- Avoid noticeable flashes where practical.
- Avoid hydration mismatches.
- Respect configured preference precedence.
- Persist only approved preferences.
- Remain accessible to keyboard and assistive technology.
- Apply consistently to dialogs, portals, charts, and third-party components.

---

## 9. Interaction States

Every interactive component must define its visual interaction states.

At minimum, consider:
- Default.
- Hover.
- Focus-visible.
- Active or pressed.
- Disabled.
- Loading.
- Selected.
- Expanded or collapsed.
- Invalid.
- Success, where relevant.

### 9.1 Focus

Keyboard focus must remain clearly visible.

- Use `:focus-visible` where appropriate.
- Never remove outlines without providing a visible replacement.
- Ensure focus indicators contrast with adjacent colors.
- Prevent overlays and sticky elements from obscuring focused controls.
- Preserve logical focus order.

### 9.2 Hover

Hover styling must be supplementary.

- Do not make essential actions available only on hover.
- Ensure touch devices retain full functionality.
- Avoid hover effects that create layout shifts.
- Keep hover behavior consistent across related components.

### 9.3 Disabled and Loading States

Disabled and loading controls must:
- Communicate their current state.
- Avoid ambiguous appearance.
- Prevent unintended duplicate actions.
- Maintain sufficient contrast where required.
- Provide accessible state information.
- Avoid hiding errors or status messages.

A disabled visual style must not be used as a replacement for backend authorization.

### 9.4 Validation States

Form controls must clearly communicate:
- Valid input where feedback is necessary.
- Invalid input.
- Required status.
- Help text.
- Server-side validation failures.
- Disabled or read-only state.

Do not rely on color alone to communicate an error.

---

## 10. Accessibility in Styling

Accessibility is a required design-system property.

### 10.1 Contrast

Text, icons, boundaries, and interaction states must meet the project's accessibility requirements, targeting WCAG 2.2 AA.

Validate contrast for:
- Body text.
- Large text.
- Button labels.
- Placeholder text where used.
- Form borders and control boundaries.
- Focus indicators.
- Status indicators.
- Dark and light themes.
- Tenant-specific palettes.

Do not assume a color is accessible because it is part of the brand palette.

### 10.2 Touch Targets

Interactive controls must provide suitable target size and spacing for touch users.

Small icons must have sufficiently large interactive hit areas, even when the visible icon is compact.

### 10.3 Zoom and Reflow

Layouts must remain usable under:
- Browser zoom.
- Increased text size.
- Narrow viewports.
- Orientation changes.
- Text wrapping and localization.

Avoid fixed heights and clipping that prevent users from accessing content.

### 10.4 Motion

Respect `prefers-reduced-motion`.

- Avoid unnecessary motion.
- Keep transitions short and purposeful.
- Do not use flashing effects.
- Avoid parallax or motion that interferes with reading.
- Provide reduced-motion alternatives for meaningful animation.

Example:

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    scroll-behavior: auto !important;
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

This is an illustrative baseline. Prefer targeted overrides where possible, and verify that reducing motion does not break functional transitions.

### 10.5 Forced Colors

Where appropriate, ensure controls remain usable in forced-color and high-contrast environments.

Do not depend entirely on background images, shadows, or subtle color differences to communicate component boundaries or state.

---

## 11. Forms and Inputs

Forms must be visually consistent, readable, and accessible.

- Use shared form components.
- Keep labels visible and associated with inputs.
- Distinguish placeholders from labels.
- Provide consistent field spacing.
- Make errors visually and semantically clear.
- Ensure required and optional fields are distinguishable.
- Maintain readable control heights and spacing.
- Support keyboard navigation and touch input.
- Ensure autofill styling remains legible.
- Support long validation messages without clipping.

Inputs must support appropriate states for:
- Default.
- Focus.
- Invalid.
- Disabled.
- Read-only.
- Loading, where relevant.

Do not implement custom controls when native HTML controls can provide the required experience accessibly.

---

## 12. Navigation and Layout Patterns

### 12.1 Navigation

Navigation styling must communicate:
- Current location.
- Available actions.
- Hierarchy.
- Expanded or collapsed state.
- Disabled or restricted options, where appropriate.

Use consistent navigation patterns across the user application, vendor interfaces, and administrative workspaces.

### 12.2 Cards

Cards must have a consistent visual system for:
- Padding.
- Border.
- Radius.
- Elevation.
- Heading.
- Supporting content.
- Actions.
- Loading and empty states.

Avoid nesting too many cards, which can make interfaces visually heavy and obscure information hierarchy.

### 12.3 Dialogs and Overlays

Dialogs, drawers, popovers, and menus must:
- Use consistent elevation and layering.
- Remain within the viewport.
- Support keyboard interaction.
- Avoid clipping important content.
- Work at narrow screen sizes.
- Preserve readable content spacing.
- Use the shared overlay and focus-management patterns.

Styling alone does not provide accessible dialog behavior; component semantics and interaction handling are also required.

### 12.4 Tables

Data tables must support:
- Readable column spacing.
- Clear headers.
- Consistent alignment.
- Long-value wrapping or truncation with access to full content.
- Responsive behavior.
- Loading and empty states.
- Clear row and action affordances.

Do not rely on color alone for selected, flagged, or status rows.

---

## 13. Motion and Animation

Motion should communicate state changes and provide feedback without distracting users.

Suitable use cases:
- Small hover or press feedback.
- Dialog and drawer transitions.
- Expand and collapse interactions.
- Loading indicators.
- Subtle state transitions.
- Meaningful onboarding guidance.

Standards:
- Prefer CSS transitions for simple state changes.
- Use transform and opacity for lightweight animation where appropriate.
- Avoid animating layout-heavy properties when a more efficient alternative exists.
- Respect reduced-motion preferences.
- Avoid excessive animation on data-heavy dashboards.
- Avoid continuous animation without functional purpose.
- Ensure animations do not delay access to critical controls.

Motion durations and easing should use centralized design tokens.

---

## 14. Icons and Imagery

### 14.1 Icons

- Use the approved icon library and shared icon components.
- Keep icon sizes consistent.
- Align icons correctly with text.
- Use accessible labels for icon-only controls.
- Mark decorative icons appropriately.
- Avoid mixing icon styles without a documented reason.

Do not use emoji as a replacement for core interface icons.

### 14.2 Images

Images must:
- Have appropriate dimensions and aspect ratios.
- Use responsive sizing.
- Avoid layout shifts.
- Use suitable optimization and loading strategies.
- Have meaningful alternative text when informative.
- Use empty alternative text when decorative.
- Avoid embedding text in images when actual text is possible.

### 14.3 Image Containers

Define consistent image behavior for:
- Food item thumbnails.
- Vendor logos.
- User avatars.
- Campus images.
- Promotional banners.
- Community media.
- Empty-state illustrations.

Use appropriate `object-fit` behavior and avoid distorted images.

---

## 15. Data-Dense and Operational Interfaces

KAMPYN includes interfaces where clarity and speed are more important than decoration.

Examples:
- Vendor order dashboards.
- Inventory reporting.
- Food court administration.
- Booking management.
- HR workflows.
- Complaint tracking.
- Analytics and operational reporting.

These interfaces must:
- Prioritize information hierarchy.
- Use consistent status presentation.
- Avoid excessive visual ornamentation.
- Make primary actions easy to locate.
- Support dense information without reducing readability.
- Provide clear loading and error states.
- Preserve responsive access to essential information.
- Maintain consistent alignment and table patterns.

Use charts, badges, icons, and color only when they improve understanding.

---

## 16. Marketing Website Styling

The KAMPYN marketing website may use more expressive visual styling than operational application interfaces, but it must still follow the same brand foundations.

Marketing pages may use:
- Larger typography.
- More generous section spacing.
- Illustrative graphics.
- Brand gradients.
- Promotional layouts.
- Richer transitions.
- Responsive storytelling sections.

Marketing styles must not leak into the authenticated application or alter shared component behavior.

Keep marketing-specific compositions and styles isolated in their own route or feature modules while reusing brand tokens and shared primitives where appropriate.

SEO, readable content, accessibility, and performance remain mandatory.

---

## 17. Third-Party Styling

Third-party libraries must be integrated without compromising the design system.

- Use supported theme and styling APIs.
- Prefer wrapper components for repeated configuration.
- Scope overrides carefully.
- Avoid fragile selectors based on undocumented internal markup.
- Avoid global overrides that affect unrelated pages.
- Verify dark theme and responsive behavior.
- Review bundle and runtime costs before adding a library.

If a third-party component cannot meet accessibility or theming requirements, assess whether it should be replaced or wrapped with an accessible alternative.

---

## 18. Performance

Styling must not introduce unnecessary rendering, layout, or loading costs.

### 18.1 CSS Performance

- Avoid excessively broad selectors.
- Avoid deeply nested SCSS.
- Avoid repeated style definitions.
- Avoid unnecessary runtime style generation.
- Avoid excessive use of `!important`.
- Prefer CSS-native layout and responsive behavior.
- Keep animations lightweight.
- Remove unused styles where tooling supports it.

### 18.2 Layout Stability

Prevent layout shifts by:
- Defining image aspect ratios or dimensions.
- Reserving space for loading content where practical.
- Avoiding late layout changes caused by font loading.
- Avoiding unexpected content injection.
- Using stable component dimensions where appropriate.

### 18.3 Bundle Size

- Avoid unnecessary styling dependencies.
- Use the project's approved Tailwind build configuration.
- Avoid shipping unused CSS where tooling can remove it safely.
- Keep component styles close to their owners.
- Review large third-party stylesheets.

### 18.4 Rendering

Avoid styling approaches that force excessive JavaScript-driven layout updates.

Prefer CSS media queries, container queries, grid, flexbox, and custom properties for responsive and theme behavior.

---

## 19. Naming and File Organization

Suggested structure:

```text
src/
├── app/
│   └── globals.css
├── design-system/
│   ├── tokens/
│   │   ├── colors.css
│   │   ├── typography.css
│   │   ├── spacing.css
│   │   ├── motion.css
│   │   └── themes.css
│   ├── components/
│   └── utilities/
├── components/
│   ├── button/
│   │   ├── button.tsx
│   │   └── button.module.scss
│   └── ...
├── features/
│   ├── food/
│   │   ├── components/
│   │   └── styles/
│   └── ...
└── styles/
    ├── reset.css
    ├── globals.css
    └── utilities.css
```

This structure is illustrative. Follow the established repository architecture and avoid creating unused directories.

### 19.1 Global Styles

Global styles should be limited to:
- CSS reset or normalization.
- Base typography.
- Design tokens.
- Theme definitions.
- Global accessibility behavior.
- Essential application-wide defaults.

Do not put feature-specific component styling into global stylesheets.

### 19.2 Naming Conventions

Use descriptive names that reflect visual or semantic responsibility.

For CSS Modules and SCSS, prefer names such as:

```scss
.card {}
.cardHeader {}
.cardBody {}
.cardActions {}
```

Use the project's established naming convention consistently.

Avoid:
- Cryptic abbreviations.
- Generic names such as `.box1`.
- Unrelated selectors grouped into one class.
- Names coupled to temporary content.
- Duplicate classes for identical styling without a reason.

### 19.3 File Size

Production source files should normally remain at or below 200 lines unless there is explicit architectural justification.

Large stylesheets should be split by cohesive responsibility, not arbitrarily.

Avoid creating many tiny style files that make a component difficult to understand.

---

## 20. Styling Governance

### 20.1 Introducing New Tokens

A new design token should be introduced only when:
- It represents a reusable design decision.
- Existing tokens cannot express the requirement clearly.
- It improves consistency or maintainability.
- Its usage and ownership are understood.

Do not create tokens for every individual CSS value.

### 20.2 Changing Existing Tokens

Changes to shared tokens may affect many features.

Before modifying them:
- Identify impacted components and routes.
- Check both light and dark themes.
- Validate contrast and accessibility.
- Review responsive behavior.
- Check tenant branding overrides.
- Update relevant visual tests or documentation.

### 20.3 Exceptions

A feature may deviate from shared styling only when there is a clear design or functional requirement.

Exceptions must:
- Be limited in scope.
- Avoid breaking accessibility.
- Avoid introducing conflicting global behavior.
- Be documented when they affect shared patterns.
- Be reviewed for possible future design-system support.

---

## 21. Testing Requirements

Styling must be tested through a combination of automated and manual validation.

### 21.1 Visual Testing

Use visual regression testing where available for:
- Shared components.
- Key workflows.
- Light and dark themes.
- Responsive breakpoints.
- Important loading and error states.
- Tenant branding variants.

Visual tests should use stable data and deterministic rendering wherever practical.

### 21.2 Accessibility Testing

Test:
- Color contrast.
- Keyboard focus visibility.
- Text scaling and zoom.
- Reduced-motion behavior.
- Forced-color support where relevant.
- Responsive reflow.
- Form error presentation.
- Screen-reader interaction in conjunction with semantic markup.

Automated accessibility tools must supplement, not replace, manual testing.

### 21.3 Responsive Testing

Verify:
- No unintended horizontal overflow.
- Content remains readable.
- Primary actions remain reachable.
- Dialogs and menus fit within the viewport.
- Tables and dense content have usable narrow-screen behavior.
- Touch interactions work without hover.

### 21.4 Cross-Browser Testing

Test supported browsers and devices according to the project's browser support policy.

Pay particular attention to:
- CSS feature support.
- Form control rendering.
- Font behavior.
- Sticky and fixed elements.
- Scroll containers.
- Focus behavior.
- Native control accessibility.

### 21.5 Component State Testing

Test visual behavior for:
- Default.
- Hover.
- Focus.
- Active.
- Disabled.
- Loading.
- Selected.
- Invalid.
- Empty.
- Error.
- Expanded or collapsed.

---

## 22. Anti-Patterns

The following are prohibited unless an explicit, reviewed exception exists:

- Arbitrary values where design tokens already exist.
- Duplicating shared styles across components.
- Styling all components through global selectors.
- Excessive inline styles.
- Overusing `!important`.
- Deeply nested SCSS.
- Using absolute positioning for primary page layout.
- Fixed dimensions that break responsive behavior.
- Fixed heights that clip dynamic or translated content.
- Using color alone to communicate meaning.
- Removing focus outlines without accessible replacements.
- Hover-only interactions.
- Unnecessary animation.
- Runtime CSS generation without justification.
- Introducing a new styling library without review.
- Overriding third-party internals with fragile selectors.
- Creating component variants for one-off visual preferences.
- Allowing tenant branding to bypass accessibility or inject arbitrary CSS.
- Using visual styling as a substitute for semantic HTML or accessible interaction behavior.
- Mixing marketing styles into operational application components.
- Building separate theme implementations for each feature without a shared token system.

---

## 23. Code Review Checklist

Before approving styling changes, verify:

- [ ] Existing design tokens and components were reused where appropriate.
- [ ] New tokens are justified and documented.
- [ ] Tailwind and CSS Modules are used consistently.
- [ ] Styles are scoped to the correct component or feature.
- [ ] Global styles remain limited to shared foundations.
- [ ] Layouts use appropriate CSS Grid or Flexbox.
- [ ] Responsive behavior is defined and tested.
- [ ] Content does not clip at narrow widths or increased text sizes.
- [ ] Light and dark themes are both considered.
- [ ] Tenant branding cannot bypass theme constraints.
- [ ] Interaction states are defined.
- [ ] Keyboard focus is clearly visible.
- [ ] Color contrast meets accessibility requirements.
- [ ] Motion respects reduced-motion preferences.
- [ ] Touch interactions do not depend on hover.
- [ ] Images and icons have appropriate behavior.
- [ ] Third-party styles are scoped and maintainable.
- [ ] Layout stability and CSS performance are considered.
- [ ] Visual and accessibility tests cover relevant states.
- [ ] File size and naming standards are followed.
- [ ] No unnecessary styling dependency or abstraction was introduced.

---

## 24. Definition of Done

A styling implementation is complete when:

- It follows the shared KAMPYN design system.
- Existing tokens and components are reused where appropriate.
- Styles are modular, scoped, and maintainable.
- Responsive behavior is implemented and verified.
- Light and dark themes work where supported.
- Interaction states are clear and consistent.
- Accessibility requirements are met.
- Content remains usable under zoom, text scaling, and localization.
- Styling does not compromise application performance.
- Third-party integrations are appropriately scoped.
- Relevant visual, responsive, and accessibility tests pass.
- Documentation is updated where shared patterns or tokens change.

**Final principle:** KAMPYN styling must be consistent by design, responsive by default, accessible in every interaction state, and flexible enough to support multiple universities without fragmenting the design system.