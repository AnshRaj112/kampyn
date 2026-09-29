# Responsive Design Standards

## 1. Purpose

This document defines the responsive design standards for KAMPYN's web interfaces, ensuring a consistent, accessible, and usable experience across mobile phones, tablets, laptops, desktops, and large displays.

KAMPYN is a multi-feature campus platform used by students, faculty, administrators, vendors, and university staff. Its interfaces must accommodate different user roles, devices, screen sizes, input methods, and usage environments.

All responsive interfaces must be:

- **Mobile-first:** Designed for small screens before being progressively enhanced for larger screens.
- **Adaptive:** Able to accommodate different viewport sizes, orientations, and input methods.
- **Accessible:** Usable with keyboard, touch, assistive technologies, zoom, and text resizing.
- **Consistent:** Aligned with KAMPYN's design system, layout conventions, and reusable components.
- **Performant:** Efficient across different devices, network conditions, and hardware capabilities.
- **Content-driven:** Structured around the importance and hierarchy of content rather than device-specific assumptions.
- **Maintainable:** Built using shared layout primitives, design tokens, and reusable responsive patterns.

This document applies to responsive layouts, component behavior, navigation, typography, media, forms, tables, dashboards, and interactive interfaces. Refer to `frontend/react.md`, `frontend/nextjs.md`, `frontend/components.md`, and `frontend/accessibility.md` for related implementation standards.

---

## 2. Core Principles

All responsive design implementations must follow these principles:

1. Design for the smallest practical viewport first.
2. Progressively enhance layouts as available space increases.
3. Use content-driven breakpoints rather than targeting specific devices.
4. Prefer fluid layouts over fixed dimensions.
5. Preserve content hierarchy across viewport sizes.
6. Keep critical actions discoverable and accessible.
7. Avoid horizontal scrolling for ordinary page content.
8. Support touch, keyboard, pointer, and assistive input.
9. Avoid hiding essential information solely to simplify mobile layouts.
10. Use shared design tokens for spacing, typography, sizing, and breakpoints.
11. Design for variable content, including long names, translations, and empty states.
12. Ensure layouts remain usable under browser zoom and text enlargement.
13. Avoid maintaining separate desktop and mobile implementations unless the interaction model genuinely differs.
14. Test actual responsive behavior instead of relying only on visual assumptions.
15. Treat responsiveness as a system-wide quality requirement, not a final styling task.

---

## 3. Mobile-First Design

### 3.1 Mobile-First Implementation

Begin with a layout that works at narrow viewport widths. Add enhancements for larger screens using responsive CSS.

Mobile-first design must not mean simply shrinking a desktop layout. Reconsider:
- Content hierarchy.
- Navigation structure.
- Information density.
- Interaction patterns.
- Input methods.
- Component arrangement.
- Visibility and prioritization of actions.

A narrow-screen layout must preserve the core task and necessary information, even when secondary elements are reorganized or progressively disclosed.

### 3.2 Progressive Enhancement

As screen space increases:
- Introduce additional columns where they improve comprehension.
- Expand content regions while preserving readable line lengths.
- Display secondary navigation where appropriate.
- Use additional context and metadata when useful.
- Increase information density only when the layout remains understandable.
- Avoid stretching content across the full viewport without a reason.

Larger screens should enhance the experience rather than introduce a fundamentally different or unnecessarily complex one.

### 3.3 Mobile as a First-Class Experience

The mobile experience must support important KAMPYN workflows, including:
- Browsing food courts and menus.
- Placing and tracking food orders.
- Managing bookings and reservations.
- Viewing shuttle information.
- Scheduling shared campus resources.
- Submitting and tracking complaints.
- Receiving notifications.
- Accessing relevant community features.

Critical workflows must not require a desktop viewport unless the workflow has a documented operational requirement.

---

## 4. Responsive Breakpoints

### 4.1 Breakpoint Strategy

Breakpoints must be selected according to when content or layout needs to change, not based on named devices.

Use the following initial breakpoint scale as a shared baseline. Adjust it when actual content requirements justify a change.

| Breakpoint | Minimum width | Intended layout |
|---|---:|---|
| Base | 0px | Narrow phones and compact viewports |
| `sm` | 640px | Large phones and compact tablets |
| `md` | 768px | Tablets and narrow split-screen layouts |
| `lg` | 1024px | Laptops and expanded tablet layouts |
| `xl` | 1280px | Desktop and wide application layouts |
| `2xl` | 1536px | Large desktop and wide dashboards |

These values are design-system defaults, not device classifications. Components may require their own content-driven adjustments within this scale.

### 4.2 Breakpoint Rules

- Use a consistent breakpoint scale across the application.
- Avoid creating many one-off breakpoints.
- Add custom breakpoints only when a genuine layout constraint requires them.
- Prefer CSS media queries and responsive layout primitives over JavaScript viewport checks.
- Avoid device detection for layout decisions.
- Ensure content remains functional between defined breakpoints.
- Test layouts just below and above each breakpoint.

### 4.3 Container Queries

Use CSS container queries when a component's layout should depend on the size of its parent rather than the viewport.

Appropriate use cases include:
- Reusable cards in different grid configurations.
- Dashboard widgets.
- Embedded panels.
- Reusable feature components.
- Components placed in sidebars or split layouts.

Prefer container queries over viewport media queries when component-level adaptability is the actual requirement.

---

## 5. Fluid Layouts and Sizing

### 5.1 Fluid Dimensions

Prefer flexible sizing techniques:
- CSS Grid.
- Flexbox.
- Percentages.
- `minmax()`.
- `min()`, `max()`, and `clamp()`.
- Content-based sizing.
- Intrinsic layout sizing.

Avoid unnecessary fixed widths and heights that break when content or viewport dimensions change.

### 5.2 Fixed Dimensions

Fixed dimensions may be used when they represent a real constraint, such as:
- Icon sizes.
- Small control dimensions.
- Consistent navigation affordances.
- Known media aspect ratios.
- Carefully bounded sidebar widths.

Fixed sizing must not cause:
- Content clipping.
- Overlapping elements.
- Unusable forms.
- Horizontal overflow.
- Text truncation that hides important information.

### 5.3 Maximum Content Width

Use maximum widths to keep content readable on wide screens.

Examples:
- Article and explanatory content should have comfortable line lengths.
- Forms should avoid unnecessarily wide input fields.
- Dashboards may use wider containers to support meaningful data layouts.
- Full-width interfaces should still have consistent horizontal spacing.

### 5.4 Minimum Sizing

Use `min-width: 0` where necessary in flex and grid children to allow content to shrink correctly.

Use `minmax(0, 1fr)` when grid tracks need to shrink without overflowing.

Avoid forcing components to maintain desktop minimum widths inside narrow layouts unless horizontal scrolling is a deliberate and accessible interaction.

### 5.5 Dynamic Viewport Units

When designing full-height layouts:
- Prefer dynamic viewport units such as `dvh` when appropriate.
- Account for mobile browser interface changes.
- Avoid assuming that `100vh` always equals the visible mobile viewport.
- Ensure content remains reachable when browser chrome expands or collapses.
- Do not trap users in fixed-height containers without an accessible scrolling path.

---

## 6. Layout Systems

### 6.1 CSS Grid

Use CSS Grid for two-dimensional layouts, such as:
- Product and menu grids.
- Dashboard card arrangements.
- Admin data overviews.
- Multi-column forms.
- Content and sidebar compositions.

Use flexible tracks and explicit minimum sizing to prevent overflow.

```css
.item-grid {
  display: grid;
  grid-template-columns: repeat(
    auto-fit,
    minmax(min(100%, 16rem), 1fr)
  );
  gap: 1rem;
}
```

### 6.2 Flexbox

Use Flexbox for one-dimensional layouts, such as:
- Navigation rows.
- Toolbars.
- Button groups.
- Card content alignment.
- Form action rows.
- Responsive header sections.

Allow items to wrap when necessary and define behavior for long labels and variable content.

### 6.3 Responsive Layout Primitives

Prefer shared layout components and design-system utilities for:
- Page containers.
- Stacks.
- Grids.
- Responsive columns.
- Section spacing.
- Content alignment.
- Sidebar layouts.

Do not recreate slightly different versions of the same responsive layout in multiple features.

### 6.4 Layout Reorganization

A layout may reorganize between viewport sizes when that improves usability.

For example:
- Desktop sidebars may become mobile drawers.
- Multi-column forms may become single-column forms.
- Desktop toolbars may wrap or collapse.
- Dense data tables may become cards or structured horizontal scrollers.
- Multi-panel workflows may become sequential screens.

Reorganization must preserve meaning, reading order, focus order, and access to required functionality.

### 6.5 Reading Order

The visual order must not contradict the logical DOM order in a way that confuses keyboard, screen-reader, or sequential navigation.

Avoid relying on CSS `order` or grid placement to create a substantially different visual sequence from the underlying document order.

---

## 7. Responsive Navigation

### 7.1 Navigation Principles

Navigation must remain:
- Discoverable.
- Consistent.
- Keyboard-accessible.
- Touch-friendly.
- Understandable across screen sizes.
- Predictable when switching between layouts.

Navigation changes must preserve access to the same essential destinations, even when their presentation changes.

### 7.2 Desktop Navigation

Desktop navigation may use:
- Persistent side navigation.
- Top navigation.
- Expanded menus.
- Multi-column navigation groups.

Choose the pattern according to the application's information architecture and user workflow.

Avoid excessive navigation density and ensure that long labels do not overlap adjacent controls.

### 7.3 Mobile Navigation

Mobile navigation may use:
- Collapsible menus.
- Drawers.
- Bottom navigation for a small set of primary destinations.
- Contextual feature navigation.
- Expandable navigation groups.

Use the simplest pattern that supports the user's common tasks.

Mobile navigation must:
- Provide a clear open and close mechanism.
- Support keyboard interaction.
- Manage focus appropriately when modal.
- Avoid obscuring important content.
- Prevent background interaction when a modal drawer requires it.
- Expose meaningful labels and states to assistive technologies.

### 7.4 Primary Actions

Important actions must remain easy to find on mobile.

For example:
- Food ordering should make the cart or checkout path discoverable.
- Booking workflows should make the next step clear.
- Complaint submission should provide a visible path to submit or save.
- Critical administrative actions must remain accessible without relying on hover.

Do not hide important actions inside an overflow menu solely to save space.

### 7.5 Navigation State

When changing responsive navigation modes:
- Avoid losing the user's current route.
- Keep active destination indicators accurate.
- Avoid accidental state resets.
- Preserve keyboard focus when practical.
- Ensure expanded and collapsed states remain synchronized with accessible attributes.

---

## 8. Responsive Typography

### 8.1 Fluid Typography

Use a consistent typography scale and fluid sizing where it improves readability.

CSS `clamp()` may be used to scale typography between sensible minimum and maximum values.

```css
.page-title {
  font-size: clamp(1.75rem, 1.2rem + 2vw, 3rem);
  line-height: 1.15;
}
```

Avoid extreme fluid scaling that makes text too small on mobile or oversized on large screens.

### 8.2 Readability

- Use readable font sizes and line heights.
- Maintain appropriate line lengths.
- Avoid overly dense paragraphs.
- Preserve sufficient spacing between headings, text, and controls.
- Ensure long words and identifiers do not break layouts.
- Avoid clipping text to force a fixed-height design.

### 8.3 Text Scaling

Interfaces must remain usable when:
- Browser zoom is increased.
- Text size is enlarged.
- System accessibility text settings are applied where supported.
- Content is translated into longer strings.

Do not disable user zoom or rely on fixed text dimensions that prevent enlargement.

### 8.4 Truncation

Truncate text only when the complete value remains available through an accessible method or is not essential to the task.

Do not truncate:
- Critical status information.
- Payment totals.
- Essential booking details.
- Important error messages.
- User input required for decision-making.

For identifiers or long labels, prefer wrapping, expandable content, or an explicit details view.

---

## 9. Responsive Spacing

### 9.1 Spacing Tokens

Use shared spacing tokens for:
- Page padding.
- Section gaps.
- Card padding.
- Form spacing.
- Navigation spacing.
- Grid gaps.
- Inline control spacing.

Avoid arbitrary spacing values repeated across components.

### 9.2 Adaptive Spacing

Spacing may scale based on viewport size, but must remain consistent with the design system.

- Reduce unnecessary whitespace on compact viewports.
- Preserve enough space for touch interaction.
- Avoid crowding dense information.
- Use consistent vertical rhythm.
- Avoid excessive padding that pushes important content below the fold.

### 9.3 Safe Areas

For edge-to-edge mobile layouts, respect device safe areas where relevant.

Use appropriate CSS environment variables, such as `env(safe-area-inset-bottom)`, for fixed controls and bottom navigation.

Ensure fixed elements do not cover page content or keyboard-accessible controls.

---

## 10. Responsive Components

### 10.1 Component Adaptability

Reusable components must adapt to their available space rather than assuming a specific page or device.

Components should define:
- Minimum practical width.
- Wrapping behavior.
- Content overflow behavior.
- Responsive spacing.
- Responsive alignment.
- Visibility rules for secondary content.
- Interaction changes, if required.

### 10.2 Cards

Responsive cards must:
- Maintain consistent internal spacing.
- Allow variable content length.
- Avoid fixed heights that clip information.
- Reflow content and actions as needed.
- Keep primary actions visible.
- Preserve clear information hierarchy.

Cards in a grid must remain understandable when arranged in a single column.

### 10.3 Buttons

Buttons must:
- Remain readable at narrow widths.
- Support touch interaction.
- Avoid text clipping.
- Wrap or stack appropriately in button groups.
- Preserve visual hierarchy between primary and secondary actions.

Do not make every mobile button full width by default. Choose sizing based on task importance, context, and available space.

### 10.4 Inputs

Responsive form controls must:
- Fit their container.
- Remain readable at zoomed sizes.
- Avoid horizontal overflow.
- Support appropriate input types and mobile keyboards.
- Maintain visible labels and validation feedback.
- Avoid overly small touch targets.

### 10.5 Dialogs and Drawers

Dialogs must adapt to viewport constraints.

On narrow viewports:
- Use appropriate width and height limits.
- Keep content scrollable.
- Ensure action buttons remain reachable.
- Prevent background content from being unintentionally interacted with.
- Respect safe areas.
- Keep close and dismissal controls accessible.

For complex mobile workflows, a full-screen dialog or dedicated route may be more usable than a small centered modal.

### 10.6 Tooltips and Hover Interactions

Never make essential information or functionality available only through hover.

Provide alternatives for:
- Touch input.
- Keyboard focus.
- Screen readers.
- Devices without hover capability.

### 10.7 Dropdowns and Menus

Menus must:
- Fit within the visible viewport.
- Avoid clipping behind parent containers.
- Support keyboard interaction.
- Handle long labels.
- Position correctly when the viewport changes.
- Avoid requiring hover to remain open.

Use established accessible menu components rather than creating custom behavior without a strong reason.

---

## 11. Responsive Data Tables

### 11.1 General Rules

Data tables are common in administrative and operational interfaces. They must remain usable on narrow screens.

Avoid making all tables horizontally scrollable by default. Select a presentation appropriate to the data and workflow.

### 11.2 Responsive Strategies

Possible approaches include:
- Prioritizing essential columns.
- Providing optional column visibility.
- Using compact table layouts.
- Converting rows into structured cards.
- Enabling horizontal scrolling for genuinely tabular data.
- Using dedicated detail views for complete records.

### 11.3 Horizontal Scrolling

When horizontal scrolling is necessary:
- Restrict scrolling to the table region rather than the whole page.
- Make the scroll affordance discoverable.
- Preserve headers and column meaning.
- Support keyboard access.
- Ensure important actions are not hidden without an alternative.
- Avoid excessive nested scrolling.

### 11.4 Data Integrity

Do not silently remove important data from a mobile view.

If columns are hidden or summarized:
- Preserve access to the complete record.
- Make omitted information discoverable.
- Maintain appropriate context.
- Ensure sorting and filtering behavior remains understandable.

### 11.5 Administrative Workflows

Administrative interfaces must support meaningful operation on smaller screens where mobile support is required.

For dense or complex tasks, provide an appropriate compact layout or an explicit detail workflow rather than squeezing a desktop table into an unusable viewport.

---

## 12. Responsive Dashboards

### 12.1 Dashboard Structure

Dashboards must prioritize:
- Critical metrics.
- Active tasks.
- Alerts and exceptions.
- Frequently used actions.
- Supporting trends and reports.

Use a clear hierarchy that adapts to available width.

### 12.2 Widget Layout

Dashboard widgets must:
- Reflow naturally.
- Avoid clipping charts and labels.
- Support variable content.
- Preserve meaningful relationships between metrics.
- Avoid excessively small cards.
- Adapt to the width of their container where appropriate.

### 12.3 Charts

Responsive charts must:
- Remain legible at small sizes.
- Avoid overlapping labels.
- Use appropriate responsive axes and legends.
- Provide accessible textual summaries or equivalent information.
- Avoid communicating critical information through color alone.
- Handle dense data through filtering, zooming, pagination, or aggregation as appropriate.

### 12.4 Role-Specific Dashboards

Student, vendor, faculty, and administrator dashboards may have different information priorities.

Responsive layouts must preserve each role's core workflow without assuming that every user has a large display.

Do not duplicate an entire dashboard implementation for each viewport. Prefer shared feature components and responsive composition.

---

## 13. Responsive Forms and Multi-Step Workflows

### 13.1 Form Layout

Forms must adapt from multi-column layouts to single-column layouts when space becomes constrained.

- Keep related fields grouped.
- Preserve logical reading order.
- Avoid overly wide input fields.
- Maintain visible labels.
- Ensure errors remain associated with the relevant input.
- Keep primary submission actions discoverable.

### 13.2 Multi-Step Workflows

For workflows such as booking, checkout, and registration:
- Clearly identify the current step.
- Preserve entered information when navigating between steps.
- Keep progress indicators legible.
- Avoid requiring unnecessary scrolling to locate primary actions.
- Make back and continue controls easy to reach.
- Prevent responsive changes from disrupting workflow state.

### 13.3 Mobile Input

Use input types and autocomplete attributes that support mobile input methods.

Examples include:
- Email input for email addresses.
- Numeric input patterns where appropriate.
- Date and time controls suitable for the supported browser environment.
- Appropriate autocomplete hints for common form fields.

Do not use a numeric keyboard for values that may contain leading zeros or non-numeric characters unless the data contract supports it.

### 13.4 Validation Feedback

Validation messages must:
- Wrap without clipping.
- Remain visible near the relevant field.
- Avoid shifting the interface in confusing ways.
- Be announced appropriately to assistive technology.
- Remain available when users zoom or rotate their devices.

---

## 14. Touch and Pointer Interaction

### 14.1 Touch Targets

Interactive controls must have touch-friendly target areas.

- Aim for at least 44 × 44 CSS pixels for common touch controls where practical.
- Follow applicable accessibility requirements when larger targets are necessary.
- Avoid placing destructive actions immediately beside common actions without sufficient separation.
- Provide adequate spacing between adjacent interactive elements.

### 14.2 Input Modality

Support:
- Touch.
- Mouse and trackpad.
- Keyboard.
- Assistive technologies.
- Voice input where supported by the platform.

Do not assume that a mobile viewport means touch is available or that a desktop viewport means only a mouse is available.

### 14.3 Gestures

Gestures may supplement standard controls but must not be the only way to complete an important action.

For example:
- Swiping may supplement visible carousel controls.
- Dragging may supplement keyboard-accessible reordering.
- Pinch gestures must not be required to access content.
- Swipe-to-delete must have a discoverable alternative.

### 14.4 Pointer Precision

Do not require precise pointer movement for common actions.

Avoid:
- Tiny icon-only controls without adequate hit areas.
- Closely spaced actions.
- Hover-only controls.
- Drag-only workflows without an alternative.

---

## 15. Responsive Media and Assets

### 15.1 Images

Images must:
- Scale within their containers.
- Preserve appropriate aspect ratios.
- Avoid layout shifts.
- Use suitable responsive sources where supported.
- Have meaningful alternative text when informative.
- Use empty alternative text when purely decorative.

### 15.2 Video and Embedded Content

Embedded media must:
- Respect container width.
- Maintain appropriate aspect ratios.
- Avoid causing page overflow.
- Provide accessible controls.
- Avoid autoplay with sound.
- Support appropriate captions or transcripts where needed.

### 15.3 Icons

- Use consistent icon sizes.
- Keep icons aligned with text.
- Provide accessible names for meaningful icon-only controls.
- Avoid using icons as the only indicator of status or meaning.
- Ensure icons do not become distorted when scaled.

### 15.4 Maps and Complex Visualizations

Maps and complex visualizations must have:
- Responsive containers.
- Usable controls at narrow widths.
- Appropriate loading and error states.
- Alternative representations of essential information.
- No dependency on hover-only interaction.

---

## 16. Orientation and Viewport Changes

Interfaces must remain usable when users rotate devices or resize browser windows.

- Support portrait and landscape layouts where practical.
- Do not lock orientation unless a specific application requirement justifies it.
- Recalculate responsive layouts without losing user input.
- Preserve relevant component state when viewport dimensions change.
- Ensure modals, menus, and fixed controls remain reachable after rotation.
- Avoid assumptions about a device's orientation or physical dimensions.

Test important workflows in both orientations on supported mobile devices.

---

## 17. Zoom, Reflow, and Accessibility

Responsive design must support accessibility requirements described in `frontend/accessibility.md`.

At minimum:
- Support browser zoom without loss of essential functionality.
- Allow text enlargement without clipping critical content.
- Reflow ordinary content at narrow effective viewport widths.
- Preserve keyboard navigation and visible focus.
- Ensure fixed or sticky elements do not obscure focused controls.
- Respect reduced-motion preferences.
- Ensure status, validation, and dynamic content remain perceivable.
- Avoid conveying essential information exclusively through color, position, or animation.

Do not disable pinch-to-zoom or override user accessibility settings without a justified and approved requirement.

---

## 18. Responsive Behavior and State

Responsive changes must not unexpectedly reset user state.

Preserve where appropriate:
- Form values.
- Current route.
- Selected filters.
- Pagination state.
- Open workflow steps.
- Draft content.
- Relevant scroll position.
- Selection state.

When a component changes its presentation mode, ensure that the underlying interaction state remains coherent.

Do not use viewport-specific conditional rendering in ways that create duplicate interactive elements, conflicting IDs, or inconsistent accessibility trees.

---

## 19. Responsive Performance

### 19.1 General Performance

Responsive interfaces must account for lower-powered devices and constrained network conditions.

- Avoid rendering unnecessary components.
- Use appropriately sized images and media.
- Defer non-critical content where suitable.
- Avoid expensive layout recalculations.
- Avoid unnecessary event listeners and resize observers.
- Prefer CSS-based responsive layout over JavaScript-driven layout when practical.

### 19.2 JavaScript Viewport Logic

Use JavaScript viewport logic only when behavior genuinely depends on viewport or media capability.

When required:
- Prefer established hooks or platform APIs.
- Avoid repeated independent resize listeners.
- Clean up subscriptions.
- Avoid hydration mismatches in server-rendered applications.
- Ensure initial rendering remains stable.
- Keep behavioral differences documented and tested.

### 19.3 Animation

- Use animations sparingly.
- Avoid large layout-triggering animations.
- Prefer efficient properties such as `transform` and `opacity` where appropriate.
- Respect reduced-motion settings.
- Avoid animation that interferes with touch interaction or content readability.

### 19.4 Layout Stability

Prevent unexpected layout shifts caused by:
- Images without reserved dimensions.
- Late-loading fonts.
- Dynamic banners.
- Asynchronously loaded content.
- Expanding validation messages.
- Unstable navigation and toolbar dimensions.

Use suitable layout reservation and loading patterns.

---

## 20. Responsive Design Tokens

Responsive behavior must use the shared KAMPYN design system.

Centralize tokens for:
- Breakpoints.
- Page gutters.
- Content widths.
- Grid gaps.
- Typography scale.
- Spacing scale.
- Control heights.
- Navigation dimensions.
- Layering.
- Safe-area spacing.

Example conceptual token structure:

```css
:root {
  --breakpoint-sm: 40rem;
  --breakpoint-md: 48rem;
  --breakpoint-lg: 64rem;
  --breakpoint-xl: 80rem;
  --breakpoint-2xl: 96rem;

  --page-gutter: 1rem;
  --content-max-width: 80rem;
  --section-gap: 2rem;
}

@media (min-width: 48rem) {
  :root {
    --page-gutter: 1.5rem;
    --section-gap: 2.5rem;
  }
}

@media (min-width: 80rem) {
  :root {
    --page-gutter: 2rem;
    --section-gap: 3rem;
  }
}
```

This example is illustrative. Actual tokens must be defined and maintained through the project's approved styling architecture. CSS custom properties do not function as media-query breakpoint values in all CSS contexts; use the supported build-system configuration or explicit media-query values for breakpoint declarations.

Avoid scattering unrelated responsive values throughout feature stylesheets.

---

## 21. Testing Responsive Interfaces

### 21.1 Testing Strategy

Responsive testing must combine:
- Automated tests.
- Browser viewport testing.
- Real-device testing where available.
- Keyboard and accessibility testing.
- Workflow-based verification.
- Performance checks.

A layout that appears correct at one viewport is not considered verified.

### 21.2 Viewport Coverage

Test representative widths around the shared breakpoints:

| Viewport width | Purpose |
|---:|---|
| 320px | Very narrow mobile layout |
| 360px | Common compact mobile layout |
| 390px | Typical modern mobile layout |
| 640px | Small breakpoint transition |
| 768px | Tablet layout |
| 1024px | Compact desktop or tablet landscape |
| 1280px | Standard desktop |
| 1536px | Wide desktop |

These widths are a baseline. Add specific viewports when a component or workflow has a known constraint.

### 21.3 Boundary Testing

Test immediately below and above relevant breakpoints.

Check for:
- Overflow.
- Unexpected wrapping.
- Navigation collisions.
- Clipped content.
- Broken grid tracks.
- Missing actions.
- Unusable controls.
- Incorrect component visibility.

### 21.4 Workflow Testing

Verify important KAMPYN workflows at supported viewport sizes, including:
- Browse and search.
- Add to cart and checkout.
- Booking and cancellation.
- Complaint submission.
- Resource scheduling.
- Notifications and status updates.
- Role-specific administrative tasks.

### 21.5 Accessibility Testing

Test:
- Keyboard navigation.
- Visible focus.
- Screen-reader labels and status announcements.
- Browser zoom.
- Text enlargement.
- Touch target usability.
- Orientation changes.
- Reduced motion.

### 21.6 Browser and Device Coverage

Use the project's supported browser matrix.

Where practical, test on:
- Chromium-based browsers.
- Firefox.
- Safari and iOS Safari.
- Android browsers.

Emulators and browser developer tools are useful, but they do not fully replace real-device testing for touch, safe areas, virtual keyboards, and browser chrome behavior.

### 21.7 Automated Checks

Use automated tools where appropriate to detect:
- Horizontal overflow.
- Missing accessible names.
- Invalid markup.
- Layout regressions.
- Visual differences at key viewports.

Automated checks supplement, but do not replace, manual workflow testing.

---

## 22. Responsive Design for KAMPYN Workflows

### 22.1 Food Ordering

Food ordering interfaces must:
- Keep menu browsing comfortable on narrow screens.
- Maintain clear item names, prices, and availability.
- Keep add-to-cart actions accessible.
- Make cart contents and totals easy to review.
- Ensure customization options remain usable.
- Keep checkout and order status information discoverable.

Avoid dense desktop-style menu grids on small screens where they compromise readability or interaction.

### 22.2 Bookings

Booking interfaces must:
- Present date, time, location, and booking status clearly.
- Adapt calendars and selection controls to narrow screens.
- Keep availability and confirmation information readable.
- Preserve entered details while navigating.
- Make cancellation and modification workflows understandable.

### 22.3 Shared Resource Scheduling

Resource scheduling interfaces must:
- Provide a usable compact view of availability.
- Avoid relying exclusively on wide timeline grids.
- Offer accessible alternatives to drag-based scheduling.
- Preserve time and resource context.
- Clearly indicate selected and unavailable slots.

### 22.4 Complaints and Support

Complaint interfaces must:
- Support comfortable text entry on mobile.
- Make attachment and submission actions discoverable.
- Present status and response history clearly.
- Keep long conversations readable.
- Ensure important updates are not hidden by responsive truncation.

### 22.5 Administration and Operations

Administrative interfaces must:
- Prioritize critical operational information.
- Adapt dense data to the available viewport.
- Keep common actions accessible.
- Support detailed inspection of records.
- Prevent destructive controls from being triggered accidentally.
- Provide suitable mobile layouts for tasks expected to be performed on mobile devices.

---

## 23. Common Anti-Patterns

The following patterns are prohibited unless an explicit exception is documented:

- Designing desktop layouts first and compressing them onto mobile.
- Using fixed widths for entire page layouts.
- Using device detection to determine layout.
- Adding excessive one-off breakpoints.
- Hiding essential actions on smaller screens.
- Making ordinary page content horizontally scrollable.
- Forcing desktop data tables into narrow viewports without a usable strategy.
- Using hover as the only way to access controls.
- Using fixed-height cards that clip variable content.
- Disabling browser zoom.
- Using tiny touch targets.
- Depending on drag or gesture interactions without alternatives.
- Reordering content visually in a way that contradicts logical reading order.
- Using JavaScript for layout that CSS can handle.
- Creating separate desktop and mobile implementations without a clear need.
- Resetting user input during responsive transitions.
- Allowing dialogs or fixed navigation to cover essential content.
- Ignoring safe areas and virtual keyboards.
- Testing only one viewport size.
- Using screenshots as the only evidence of responsive correctness.
- Optimizing for device names instead of actual content constraints.

---

## 24. Code Review Checklist

### Layout
- [ ] Does the layout work on narrow viewports?
- [ ] Is the design mobile-first?
- [ ] Are breakpoints based on content needs?
- [ ] Are fluid sizing and layout primitives used appropriately?
- [ ] Is horizontal overflow prevented for ordinary content?
- [ ] Are wide-screen content widths controlled?

### Navigation
- [ ] Is navigation discoverable across screen sizes?
- [ ] Are primary actions accessible?
- [ ] Are menus and drawers keyboard- and touch-friendly?
- [ ] Is focus handled correctly when navigation opens or closes?

### Components
- [ ] Do components adapt to their available space?
- [ ] Are cards and forms resilient to variable content?
- [ ] Are dialogs usable at narrow heights and widths?
- [ ] Are hover-only interactions avoided?
- [ ] Are data tables usable without losing essential information?

### Typography and Spacing
- [ ] Is text readable at small widths?
- [ ] Does content tolerate zoom and enlargement?
- [ ] Are spacing tokens reused?
- [ ] Are long labels and translated strings handled?
- [ ] Is truncation used responsibly?

### Accessibility
- [ ] Are controls touch-friendly?
- [ ] Is keyboard interaction supported?
- [ ] Is the logical reading order preserved?
- [ ] Are focus indicators visible?
- [ ] Are fixed elements prevented from obscuring content?
- [ ] Are reduced-motion preferences respected?

### Performance
- [ ] Are images and media appropriately sized?
- [ ] Is unnecessary JavaScript viewport logic avoided?
- [ ] Are layout shifts minimized?
- [ ] Are animations efficient and restrained?
- [ ] Is performance acceptable on lower-powered devices?

### Verification
- [ ] Have representative viewport widths been tested?
- [ ] Have breakpoint boundaries been tested?
- [ ] Have important workflows been verified?
- [ ] Have orientation changes been considered?
- [ ] Have supported browsers and real devices been tested where practical?
- [ ] Are accessibility and automated checks passing?

---

## 25. Definition of Done

A responsive design change is complete only when:

- [ ] The layout follows the mobile-first approach.
- [ ] Breakpoints are based on actual content and layout requirements.
- [ ] The interface adapts across supported viewport sizes.
- [ ] Essential information and actions remain accessible.
- [ ] No unintended horizontal overflow or content clipping exists.
- [ ] Navigation and interactions support touch, keyboard, and pointer input.
- [ ] Typography, spacing, and sizing use the approved design system.
- [ ] Forms, dialogs, tables, and dashboards remain usable at relevant sizes.
- [ ] Zoom, text enlargement, and orientation changes have been considered.
- [ ] Responsive transitions preserve relevant user state.
- [ ] Performance and layout stability have been checked.
- [ ] Automated and manual responsive tests have been completed.
- [ ] Accessibility requirements have been verified.
- [ ] Documentation and shared tokens are updated where necessary.
- [ ] Any exception to these standards is documented and approved.

**The objective is to make KAMPYN usable and consistent wherever users access it, without sacrificing accessibility, clarity, functionality, or maintainability to fit a particular screen size.**