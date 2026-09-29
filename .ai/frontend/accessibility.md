# Frontend Accessibility Standards

## 1. Purpose

This document defines the accessibility standards, implementation requirements, testing practices, and design principles for all KAMPYN frontend applications.

KAMPYN serves students, faculty, university administrators, vendors, staff, and guests with diverse abilities, devices, assistive technologies, and levels of digital literacy. Accessibility MUST be considered a core product requirement, not an optional enhancement.

All frontend interfaces MUST aim to conform to **WCAG 2.2 Level AA**, unless a stricter legal, contractual, or institutional requirement applies.

These standards apply to:

- Student-facing web applications.
- University administration dashboards.
- Vendor and food court dashboards.
- Marketing websites and landing pages.
- Booking, ordering, and payment workflows.
- Community and messaging interfaces.
- Forms, dialogs, notifications, and interactive components.
- Responsive and mobile web interfaces.
- Shared component libraries and SDK-provided UI components.

Accessibility MUST be considered throughout design, development, testing, and release.

---

## 2. Core Principles

All KAMPYN interfaces MUST follow these principles.

### 2.1 Perceivable

- Information MUST be presented in ways users can perceive.
- Meaningful images MUST have appropriate text alternatives.
- Color MUST NOT be the only means of conveying information.
- Text and interactive elements MUST have sufficient contrast.
- Dynamic content MUST be accessible to assistive technologies.
- Content MUST remain usable when text is enlarged or spacing is adjusted.

### 2.2 Operable

- All essential functionality MUST be usable with a keyboard.
- Interactive controls MUST have visible focus indicators.
- Users MUST have sufficient time to complete tasks.
- Motion, animations, and transitions MUST respect reduced-motion preferences.
- Navigation MUST be predictable and consistent.
- Interactive targets MUST be sufficiently large and appropriately spaced.

### 2.3 Understandable

- Interfaces MUST use clear, consistent language.
- Navigation and interaction patterns MUST be predictable.
- Forms MUST provide meaningful labels, instructions, and error feedback.
- Validation errors MUST explain what went wrong and how to correct it.
- Important actions MUST communicate their consequences clearly.

### 2.4 Robust

- Interfaces MUST use semantic HTML and valid accessible patterns.
- Components MUST expose roles, names, states, and values correctly.
- Interfaces SHOULD work with commonly used assistive technologies.
- Custom widgets MUST follow established accessibility interaction patterns.
- Accessibility MUST NOT depend on a specific browser, device, or input method without a documented product requirement.

### 2.5 Inclusive by Default

- Accessibility MUST be included in the initial design and implementation.
- Shared components MUST provide accessible defaults.
- Developers MUST NOT rely on users to discover hidden interaction conventions.
- Accessibility fixes MUST address underlying component or design problems rather than applying isolated visual workarounds.

---

## 3. Standards and Compliance

### 3.1 Target Standard

KAMPYN frontend applications MUST target:

- WCAG 2.2 Level AA.
- Semantic HTML and appropriate native browser behavior.
- WAI-ARIA Authoring Practices for custom interactive widgets.
- Accessible interaction patterns consistent with established platform conventions.

WCAG conformance MUST be assessed against the applicable success criteria for the actual interface and its user workflows.

### 3.2 Compliance Scope

Accessibility reviews MUST include:

- Primary user journeys.
- Authentication and account recovery.
- Search and filtering.
- Food ordering and checkout.
- Payments and transaction status.
- Hostel, guest house, and resource bookings.
- Complaint submission and tracking.
- Community and messaging.
- Administrative workflows.
- Error, empty, loading, and success states.
- Responsive layouts and supported input methods.

### 3.3 No Accessibility by ARIA Alone

- Native HTML elements MUST be preferred over custom elements when they provide the required functionality.
- ARIA MUST be used only when necessary to expose semantics, relationships, or states not adequately provided by native HTML.
- ARIA MUST NOT be used to compensate for incorrect HTML structure.
- Custom controls MUST implement their full keyboard and assistive-technology behavior.
- Incorrect ARIA MUST be treated as an accessibility defect.

---

## 4. Semantic HTML

Semantic HTML MUST be the default for all frontend development.

### 4.1 Document Structure

Use appropriate structural elements:

- `<header>` for introductory or navigational content.
- `<nav>` for navigation regions.
- `<main>` for the primary page content.
- `<section>` for meaningful thematic groups.
- `<article>` for self-contained content.
- `<aside>` for complementary information.
- `<footer>` for footer content.
- `<form>` for user input workflows.
- `<button>` for actions.
- `<a>` for navigation to a destination.

Rules:

- Every page MUST have a clear primary content region.
- Pages SHOULD have a logical heading hierarchy.
- Landmark regions MUST have accessible names when multiple regions of the same type exist.
- Generic containers MUST NOT replace semantic elements without justification.
- Document structure MUST reflect the meaning and relationships of the content.

### 4.2 Headings

- Use headings in a logical hierarchy.
- Every major page region SHOULD have a descriptive heading.
- Heading levels MUST NOT be selected solely for visual styling.
- Do not skip heading levels without a valid structural reason.
- Use CSS classes or design-system typography tokens to control visual appearance independently of heading semantics.

Example:

```tsx
<main>
  <h1>Food Court</h1>

  <section aria-labelledby="popular-items-heading">
    <h2 id="popular-items-heading">Popular Items</h2>
  </section>

  <section aria-labelledby="all-items-heading">
    <h2 id="all-items-heading">All Items</h2>
  </section>
</main>
```

### 4.3 Buttons and Links

- Buttons MUST perform actions.
- Links MUST navigate to destinations.
- A button MUST NOT be used as a substitute for a link when navigation is the intended behavior.
- An anchor MUST NOT be used as a substitute for a button when performing an action.
- Every interactive element MUST have a meaningful accessible name.
- Icon-only controls MUST have an accessible label.
- Disabled controls MUST expose their disabled state correctly.

Prohibited:

```tsx
<div onClick={handleSubmit}>
  Submit
</div>
```

Preferred:

```tsx
<button type="button" onClick={handleSubmit}>
  Submit
</button>
```

### 4.4 Native Controls

Prefer native controls for common interactions:

- `<button>`
- `<input>`
- `<select>`
- `<textarea>`
- `<fieldset>`
- `<legend>`
- `<details>`
- `<summary>`

Native controls provide built-in keyboard interaction, focus behavior, and accessibility semantics. Custom replacements MUST justify why native controls are insufficient.

---

## 5. Keyboard Accessibility

Every essential feature MUST be accessible through a keyboard without requiring a mouse, touch, or other pointing device.

### 5.1 Keyboard Support

- All interactive controls MUST be reachable through keyboard navigation.
- Keyboard focus MUST follow a logical sequence.
- Interactive elements MUST support their expected keyboard behavior.
- Keyboard users MUST be able to complete all essential workflows.
- Users MUST be able to escape from modal or temporary interaction contexts where appropriate.
- Keyboard traps MUST NOT occur, except for intentionally modal contexts that provide a reliable escape mechanism.

### 5.2 Standard Keyboard Behavior

| Key | Expected behavior |
|---|---|
| Tab | Move focus to the next focusable element |
| Shift + Tab | Move focus to the previous focusable element |
| Enter | Activate links and applicable controls |
| Space | Activate buttons and toggle applicable controls |
| Arrow keys | Navigate within widgets that define arrow-key interaction |
| Escape | Dismiss dismissible dialogs, popovers, and menus where appropriate |
| Home / End | Move to the first or last item in supported composite widgets |

Custom widgets MUST document and implement their expected keyboard behavior.

### 5.3 Focus Order

- Focus order MUST match the meaningful reading and interaction order.
- CSS visual rearrangement MUST NOT create confusing keyboard navigation.
- Positive `tabindex` values MUST NOT be used to manually construct page focus order.
- Elements that are not interactive MUST NOT be made keyboard-focusable without a valid reason.
- Hidden and inactive content MUST NOT remain unexpectedly focusable.

### 5.4 Focus Visibility

- Every keyboard-focusable control MUST have a clearly visible focus indicator.
- Focus indicators MUST have sufficient contrast against adjacent colors.
- Focus indicators MUST NOT be obscured by sticky headers, overlays, or clipped containers.
- Focus styles MUST NOT be removed without an accessible replacement.

Example:

```scss
:focus-visible {
  outline: 3px solid var(--color-focus);
  outline-offset: 3px;
}
```

The focus token MUST be defined and validated against the relevant design-system backgrounds.

### 5.5 Skip Navigation

- Pages with repeated navigation MUST provide a mechanism to skip directly to the main content.
- The skip link MUST become visible when focused.
- The target MUST be focusable or otherwise reliably reached.
- The skip link MUST remain functional across responsive layouts.

Example:

```tsx
<a className="skip-link" href="#main-content">
  Skip to main content
</a>

<main id="main-content" tabIndex={-1}>
  {/* Page content */}
</main>
```

---

## 6. Screen Reader Accessibility

KAMPYN MUST provide meaningful and predictable experiences for screen-reader users.

### 6.1 Accessible Names

- Every control MUST have a meaningful accessible name.
- Names MUST communicate the purpose of the control.
- Accessible names MUST remain concise and distinguishable in repeated interfaces.
- Decorative icons MUST NOT produce redundant announcements.
- Accessible names MUST NOT be generated from ambiguous visual context alone.

### 6.2 Accessible Descriptions

Use descriptions when a control requires additional context.

- Use `aria-describedby` to associate instructions, hints, or relevant error messages.
- Descriptions MUST be programmatically connected to the relevant control.
- Avoid repeating the same information in the accessible name and description unless necessary.

Example:

```tsx
<label htmlFor="student-email">
  University Email
</label>

<input
  id="student-email"
  type="email"
  aria-describedby="email-hint"
/>

<p id="email-hint">
  Use the email address registered with your university.
</p>
```

### 6.3 Live Regions

Dynamic content that needs to be announced MUST use appropriate accessible mechanisms.

Examples include:

- Form submission status.
- Search result counts.
- Cart updates.
- Booking confirmation.
- Payment status.
- Validation errors.
- Important asynchronous notifications.

Use live regions deliberately:

```tsx
<div role="status" aria-live="polite">
  {statusMessage}
</div>
```

Rules:

- Use `role="status"` or `aria-live="polite"` for non-urgent updates.
- Use `role="alert"` or `aria-live="assertive"` only for urgent information requiring immediate attention.
- Avoid announcing every minor UI update.
- Live-region content MUST be concise and meaningful.
- A live region SHOULD exist before its content changes when reliable announcements are required.
- Do not move keyboard focus to every status message.

### 6.4 Dynamic Content

- Changes to expanded or collapsed content MUST expose the correct state.
- Loading states MUST communicate progress or status when meaningful.
- Updates to asynchronously loaded content MUST be announced when users need to know they occurred.
- Content inserted into the DOM MUST be accessible in a logical reading order.
- Removed or replaced content MUST NOT leave keyboard focus in an invalid or confusing position.

---

## 7. Color and Contrast

Color MUST support visual clarity and MUST NOT be the sole way information is conveyed.

### 7.1 Text Contrast

KAMPYN interfaces MUST meet WCAG 2.2 AA contrast requirements:

| Content | Minimum contrast ratio |
|---|---:|
| Normal text | 4.5:1 |
| Large text | 3:1 |
| User-interface components and meaningful graphical objects | 3:1 |

Large text generally means at least 18pt regular or 14pt bold, subject to WCAG definitions.

### 7.2 Non-Color Indicators

- Errors MUST use text, icons, or other non-color indicators in addition to color.
- Status badges MUST include meaningful labels.
- Required fields MUST be identified through accessible text or semantics.
- Charts MUST use distinguishable patterns, labels, or other non-color cues where needed.
- Success, warning, and error states MUST remain understandable in grayscale or under color-vision deficiencies.

### 7.3 Design Tokens

- Colors MUST be defined through centralized design tokens.
- Accessibility contrast MUST be evaluated for all supported themes.
- Theme changes MUST NOT introduce inaccessible text or control combinations.
- Component authors MUST NOT assume that a token is accessible without checking its actual foreground and background pairing.

### 7.4 Dark Mode

- Dark mode MUST receive the same accessibility validation as light mode.
- Text MUST remain readable without excessive glare or insufficient contrast.
- Focus indicators MUST remain visible.
- Disabled, selected, invalid, and interactive states MUST remain distinguishable.
- User theme preferences SHOULD be respected.

---

## 8. Forms and Validation

Forms MUST be understandable, keyboard-accessible, and compatible with assistive technologies.

### 8.1 Labels

- Every form control MUST have a programmatically associated label.
- Placeholder text MUST NOT be the only label.
- Labels MUST clearly identify the expected input.
- Visually hidden labels MAY be used when visible labels are not appropriate, provided they remain available to assistive technologies.
- Related controls SHOULD be grouped using `<fieldset>` and `<legend>` where appropriate.

### 8.2 Instructions

- Required fields MUST be clearly identified.
- Input format requirements MUST be explained before users submit where practical.
- Instructions MUST be programmatically associated with the relevant field when needed.
- Users MUST NOT be expected to infer validation rules from placeholder examples alone.

### 8.3 Validation Errors

- Errors MUST be identified in text.
- Errors MUST explain how to correct the input when possible.
- Invalid controls MUST expose their state using `aria-invalid`.
- Error messages MUST be associated using `aria-describedby` or an equivalent appropriate mechanism.
- Form submission failures MUST be communicated without relying exclusively on color.
- Error messages MUST remain available until the error is corrected or the form state changes appropriately.

Example:

```tsx
<label htmlFor="phone">
  Phone Number
</label>

<input
  id="phone"
  type="tel"
  aria-invalid={hasError}
  aria-describedby={hasError ? "phone-error" : undefined}
/>

{hasError && (
  <p id="phone-error">
    Enter a valid phone number.
  </p>
)}
```

### 8.4 Error Summary

For complex forms:

- Provide a summary of validation errors when appropriate.
- Make the summary easy to discover.
- Link summary entries to the corresponding fields where practical.
- After a failed submission, focus management SHOULD guide users to the summary or first invalid field without disorienting them.
- Preserve valid user input when validation fails.

### 8.5 Form Submission

- Submission controls MUST communicate their purpose.
- Submission state MUST be exposed to assistive technologies where meaningful.
- Duplicate submission MUST be prevented when it could create duplicate transactions.
- Users MUST receive a clear success or failure message.
- A form MUST NOT silently clear important entered data after an error.
- Async validation MUST communicate relevant pending and result states.

### 8.6 Authentication

Login, signup, verification, and recovery workflows MUST:

- Use clearly labelled controls.
- Explain required credentials and formats.
- Provide accessible error feedback.
- Avoid unnecessary cognitive or memory burdens.
- Support password managers and browser autofill where practical.
- Make verification and recovery instructions accessible.
- Ensure time-limited verification flows communicate their remaining time and available recovery options where applicable.

---

## 9. Images, Icons, and Media

### 9.1 Images

Every image MUST be classified as informative, functional, decorative, or complex.

- Informative images MUST have useful alternative text.
- Functional images MUST have alternative text describing the action or destination.
- Decorative images MUST use empty alternative text (`alt=""`) or be hidden from assistive technologies when appropriate.
- Complex images MUST have an accessible text equivalent or a nearby detailed description.
- Image alternative text MUST describe purpose rather than appearance alone.

Examples:

```tsx
<img
  src="/food-item.jpg"
  alt="Paneer butter masala with naan"
/>

<img
  src="/decorative-pattern.svg"
  alt=""
/>
```

### 9.2 Icons

- Icon-only buttons MUST have accessible names.
- Decorative icons MUST be hidden from assistive technologies where appropriate.
- Icons MUST NOT be the only means of communicating an important state unless their meaning is programmatically exposed and clear.
- Icons with ambiguous meanings MUST be accompanied by text.
- Icon fonts and SVGs MUST NOT create unintended accessible names.

Example:

```tsx
<button type="button" aria-label="Remove item from cart">
  <TrashIcon aria-hidden="true" />
</button>
```

### 9.3 Video and Audio

- Videos containing meaningful speech MUST provide captions.
- Prerecorded video with relevant visual information SHOULD include audio description or an equivalent accessible alternative where required.
- Audio-only content MUST provide an appropriate transcript.
- Media controls MUST be keyboard-accessible and labelled.
- Autoplaying audio MUST NOT be used.
- Automatically playing video MUST respect applicable accessibility requirements and provide user control.

### 9.4 Charts and Data Visualizations

- Charts MUST have accessible titles and summaries.
- Important trends and values MUST be available in text or an equivalent data representation.
- Color MUST NOT be the only way to distinguish data series.
- Interactive chart controls MUST be keyboard-accessible.
- Complex dashboards SHOULD provide accessible tables or downloadable data alternatives.

---

## 10. Interactive Components

Custom interactive components MUST implement complete accessible semantics and interaction behavior.

### 10.1 Modals and Dialogs

- Use native `<dialog>` where suitable or a well-tested accessible dialog implementation.
- Dialogs MUST have an accessible name.
- Modal dialogs MUST manage focus appropriately.
- Focus MUST move into the dialog when opened.
- Keyboard focus MUST remain within the modal while it is active.
- Escape SHOULD close dismissible dialogs.
- Closing a dialog MUST return focus to the invoking control when it still exists and is appropriate.
- Background content MUST not remain interactable when a modal is active.
- Important information MUST NOT be available only through a transient dialog without a suitable alternative.

### 10.2 Dropdowns and Menus

- Use native `<select>` for simple selection wherever suitable.
- Custom menus MUST follow a consistent keyboard interaction model.
- Menu triggers MUST expose their expanded state.
- Menu items MUST have meaningful accessible names.
- Focus behavior MUST be predictable.
- Menus MUST be dismissible using expected keyboard interactions.
- A menu MUST NOT be implemented as a generic list of clickable `<div>` elements.

### 10.3 Tabs

- Tabs MUST expose their role, selected state, and relationship to their associated panels.
- Keyboard navigation MUST be implemented consistently.
- The active tab MUST be distinguishable visually and programmatically.
- Tab panels MUST have appropriate accessible relationships.
- Switching tabs MUST NOT unexpectedly lose user input or context.

### 10.4 Accordions

- Accordion triggers MUST be keyboard-accessible buttons.
- Expanded and collapsed states MUST be programmatically exposed.
- Triggers MUST have meaningful names.
- Expanded content MUST be reachable in a logical focus order.
- Visual state MUST match the programmatic state.

### 10.5 Tooltips and Popovers

- Tooltips MUST NOT contain essential information that is unavailable elsewhere.
- Tooltips MUST be available through appropriate pointer and keyboard interactions.
- Content MUST remain available long enough to be read and interacted with when applicable.
- Popovers containing interactive controls MUST implement appropriate focus management.
- Hover-only content MUST NOT be the sole source of critical instructions.

### 10.6 Toasts and Notifications

- Notifications MUST expose appropriate accessible roles and announcement priority.
- Important information MUST remain available beyond a short-lived toast when users may need to refer to it.
- Non-urgent messages SHOULD avoid interrupting the current screen-reader task.
- Dismissal controls MUST be keyboard-accessible and labelled.
- Toasts MUST NOT steal focus unexpectedly.

### 10.7 Autocomplete and Comboboxes

- Input and suggestion relationships MUST be programmatically exposed.
- Suggestions MUST be keyboard-accessible.
- Active suggestion state MUST be communicated.
- Users MUST be able to accept, dismiss, and navigate suggestions predictably.
- The typed value MUST remain understandable when no suggestions are available.
- Asynchronous suggestion loading MUST communicate relevant status.

### 10.8 Carousels

- Carousel controls MUST have meaningful accessible names.
- Previous and next controls MUST be keyboard-accessible.
- The current slide or item position SHOULD be communicated where useful.
- Automatic rotation SHOULD be avoided.
- If automatic rotation exists, users MUST be able to pause, stop, or control it.
- Focus MUST NOT be moved unexpectedly when the carousel changes.

---

## 11. Responsive and Touch Accessibility

KAMPYN MUST support accessible interaction across desktop, tablet, and mobile layouts.

### 11.1 Reflow

- Content MUST reflow at narrow viewport widths without requiring unnecessary horizontal scrolling.
- Essential content MUST remain available when zoomed.
- Responsive layouts MUST preserve semantic order.
- Content MUST NOT be hidden solely because a viewport is small if it is necessary for completing an essential task.
- Tables and complex data views MUST provide a usable strategy for narrow screens.

### 11.2 Zoom and Text Resizing

- Users MUST be able to zoom text and page content without loss of essential functionality.
- Layouts MUST accommodate text resizing.
- Fixed-height containers MUST NOT clip text at increased zoom levels.
- Controls MUST remain reachable when users enlarge content.
- Responsive testing SHOULD include 200% zoom and applicable WCAG reflow conditions.

### 11.3 Touch Targets

- Interactive targets SHOULD be at least 44 × 44 CSS pixels where practical.
- The implementation MUST meet WCAG 2.2 AA target-size requirements or their applicable exceptions.
- Closely spaced controls MUST reduce the risk of accidental activation.
- Touch interactions MUST NOT require precise gestures when a simpler accessible alternative is practical.
- Drag-and-drop interactions MUST provide a single-pointer alternative that does not require dragging.

### 11.4 Orientation

- Content MUST support both portrait and landscape orientation unless a specific orientation is essential to the task.
- Layouts MUST not unnecessarily restrict device orientation.
- Orientation changes MUST preserve context and input where possible.

---

## 12. Motion and Animation

Motion MUST be purposeful, restrained, and accessible.

- Respect `prefers-reduced-motion`.
- Avoid unnecessary parallax, continuous movement, and large animated transitions.
- Avoid flashing content that may trigger seizures.
- Automatically moving content MUST provide suitable user control when required.
- Motion MUST NOT be the only way to communicate state or meaning.
- Animation MUST NOT obscure content or make interaction difficult.
- Loading animations MUST have accessible status text where relevant.

Example:

```scss
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

Reduced-motion handling MUST be tested to ensure the interface remains understandable and usable when animations are removed.

---

## 13. Content and Readability

### 13.1 Language

- Page language MUST be declared using the appropriate `lang` attribute.
- Changes in language within a page SHOULD be identified programmatically.
- Content MUST use clear, consistent terminology.
- Acronyms and specialized terminology SHOULD be explained where users may not understand them.
- Instructions MUST describe what users need to do, not merely what the system expects internally.

### 13.2 Readability

- Text SHOULD be concise and organized into meaningful sections.
- Important actions SHOULD use clear, descriptive labels.
- Avoid vague control labels such as "Click here", "More", or "Submit" when more meaningful text is practical.
- Instructions MUST not rely on visual position alone, such as "select the green button on the right".
- Error messages MUST be actionable and respectful.

### 13.3 Content Consistency

- Repeated controls MUST use consistent terminology.
- Navigation labels MUST remain consistent across applications.
- Status names MUST communicate clear business meaning.
- Time, date, currency, and number formats SHOULD follow the user's configured locale where supported.
- Content MUST remain understandable without relying on color, icons, or visual arrangement alone.

---

## 14. Accessibility in KAMPYN Workflows

Accessibility MUST be validated against complete real-world user journeys, not only individual components.

### 14.1 Food Ordering

- Users MUST be able to discover food items through keyboard and assistive technology.
- Item names, prices, availability, dietary information, and customization options MUST be accessible.
- Add-to-cart controls MUST communicate the item and action.
- Cart updates MUST be announced when meaningful.
- Quantity controls MUST have clear names and values.
- Checkout MUST be fully keyboard-accessible.
- Order totals, fees, and important conditions MUST be clearly presented.
- Order submission and confirmation MUST provide accessible status feedback.

### 14.2 Search and Discovery

- Search fields MUST have accessible labels.
- Search suggestions MUST be accessible to keyboard and screen-reader users.
- Filters MUST expose selected and expanded states.
- Result counts and empty states MUST be communicated.
- Sorting controls MUST identify the active selection.
- Search results MUST have a meaningful structure and predictable reading order.
- Updating search results MUST not unexpectedly move focus.

### 14.3 Booking

- Resource type, location, date, time, availability, and booking status MUST be accessible.
- Calendars MUST provide a keyboard-accessible way to select dates.
- Availability MUST NOT be conveyed through color alone.
- Conflicting and unavailable slots MUST have clear descriptions.
- Booking confirmation and cancellation MUST communicate their outcome.
- Time-limited holds MUST be explained accessibly.
- Booking workflows MUST remain usable without drag-and-drop or complex gestures.

### 14.4 Payments

- Payment forms and status messages MUST be accessible.
- Amount, currency, payment method, and order context MUST be clear.
- Pending, successful, failed, and uncertain payment states MUST be distinguishable.
- Errors MUST provide actionable guidance.
- Users MUST be able to understand whether a payment action has completed or requires follow-up.
- Third-party payment interfaces SHOULD be evaluated for accessibility before integration.

### 14.5 Community and Messaging

- Message composition MUST be keyboard-accessible.
- Message content MUST have a logical reading order.
- Sender identity and relevant message metadata MUST be available to assistive technologies.
- New-message announcements MUST avoid overwhelming users.
- Message status MUST be communicated without color alone.
- Moderation, reporting, and blocking actions MUST have accessible names and confirmation behavior.
- Keyboard navigation MUST remain predictable when messages arrive dynamically.

### 14.6 Administrative Dashboards

- Data tables MUST have appropriate headers and accessible relationships.
- Filters and bulk actions MUST be keyboard-accessible.
- Charts MUST have text alternatives.
- Important status changes MUST be accessible.
- Destructive actions MUST communicate their consequences.
- Permission-related errors MUST be clear without disclosing sensitive system information.
- Dense interfaces MUST preserve usable reading order and focus visibility.

---

## 15. React and Next.js Implementation Standards

Accessibility MUST be incorporated into the component architecture and implementation patterns.

### 15.1 Component Design

- Shared components MUST expose accessible properties and sensible defaults.
- Components MUST use semantic elements whenever possible.
- Components MUST support keyboard interaction when interactive.
- Components MUST expose required states such as expanded, selected, checked, disabled, and invalid.
- Component APIs MUST discourage inaccessible configurations.
- Accessibility behavior MUST be tested at the shared-component level.

### 15.2 TypeScript

- Component props SHOULD model accessibility-relevant states explicitly.
- Required labels MUST be enforced in component APIs where practical.
- Reusable controls SHOULD accept typed descriptions, labels, and state properties.
- Type definitions MUST NOT encourage invalid combinations of roles, states, and controls.
- Custom ARIA attributes MUST be reviewed for correctness and necessity.

### 15.3 React State

- Visual state and accessibility state MUST remain synchronized.
- Expanded controls MUST update `aria-expanded`.
- Selected options MUST expose their selected state.
- Invalid fields MUST expose validation state.
- Async loading and submission MUST communicate meaningful progress.
- State updates MUST NOT unexpectedly disrupt focus or reading order.

### 15.4 Next.js

- Each page MUST have an appropriate document title.
- Page metadata MUST clearly describe its purpose.
- Route changes MUST provide a usable focus and announcement experience.
- Server-rendered and client-rendered content MUST preserve consistent semantic structure.
- Hydration MUST NOT produce duplicate or conflicting accessibility behavior.
- Dynamic route content MUST expose a meaningful heading and context after navigation.

### 15.5 Third-Party Components

- Third-party UI components MUST be evaluated for accessibility before adoption.
- Component libraries MUST NOT be assumed accessible solely because they advertise accessibility support.
- Known limitations MUST be documented with a mitigation plan.
- Critical workflows MUST NOT depend on inaccessible third-party components without an approved alternative.
- Upgrades MUST be tested for accessibility regressions.

---

## 16. Design System Requirements

Accessibility MUST be built into the KAMPYN design system.

### 16.1 Shared Tokens

The design system SHOULD define centralized tokens for:

- Text and background contrast pairings.
- Focus indicators.
- Error, warning, success, and informational states.
- Disabled and selected states.
- Typography scale and line height.
- Spacing and interactive target dimensions.
- Reduced-motion behavior.
- Responsive breakpoints.

### 16.2 Shared Components

The following components MUST have documented accessibility behavior:

- Buttons and icon buttons.
- Text fields and text areas.
- Selects and comboboxes.
- Checkboxes and radio groups.
- Switches.
- Dialogs and drawers.
- Menus and popovers.
- Tabs and accordions.
- Tooltips.
- Toasts and alerts.
- Data tables.
- Pagination.
- Date and time pickers.
- Search and filter controls.
- File upload controls.
- Navigation and breadcrumbs.
- Loading indicators and progress bars.

### 16.3 Component Documentation

Each interactive shared component MUST document:

- Semantic element or role.
- Required accessible name.
- Keyboard interactions.
- Focus behavior.
- Accessible state attributes.
- Validation and error behavior.
- Screen-reader expectations.
- Known accessibility limitations.
- Automated and manual test coverage.

### 16.4 Reuse

- Accessible behavior MUST be implemented centrally wherever components are shared.
- Application teams SHOULD use the shared accessible components instead of creating duplicate custom controls.
- Accessibility fixes SHOULD be applied at the source component when the defect is shared.
- Exceptions MUST document the reason for bypassing the design system.

---

## 17. Accessibility Testing

Accessibility MUST be verified through a combination of automated testing, manual inspection, assistive-technology testing, and user feedback.

### 17.1 Automated Testing

Use appropriate tools within the frontend testing workflow, such as:

- axe-core.
- `jest-axe` or equivalent test integration.
- Playwright accessibility checks where supported.
- ESLint accessibility rules, including suitable JSX accessibility checks.
- Lighthouse accessibility audits as a supplementary signal.

Rules:

- Automated checks MUST run on critical shared components and representative pages.
- Critical user journeys SHOULD have automated accessibility coverage.
- Automated test failures MUST be reviewed and resolved or formally documented.
- Automated tools MUST NOT be treated as proof of full WCAG conformance.

### 17.2 Component Testing

Shared interactive components MUST be tested for:

- Accessible names and roles.
- Keyboard navigation.
- Focus behavior.
- State changes.
- Validation messages.
- Disabled and selected states.
- Live-region announcements where applicable.
- Responsive behavior.
- Reduced-motion preferences where applicable.

### 17.3 Manual Testing

Manual testing MUST include:

- Keyboard-only navigation.
- Visible focus verification.
- Logical focus order.
- Zoom and reflow checks.
- Contrast inspection.
- Form error handling.
- Dialog and menu behavior.
- Dynamic content announcements.
- Responsive interaction checks.

### 17.4 Assistive Technology Testing

Critical workflows SHOULD be tested with representative combinations of:

- NVDA with a supported desktop browser.
- VoiceOver with a supported Apple browser and operating system.
- TalkBack with a supported Android browser.
- Other assistive technologies required by institutional or contractual accessibility commitments.

Testing MUST verify actual usability rather than only the presence of ARIA attributes.

### 17.5 End-to-End Testing

End-to-end accessibility testing SHOULD cover:

- Authentication.
- Search and discovery.
- Ordering and checkout.
- Booking and cancellation.
- Payment status.
- Complaint submission.
- Messaging.
- Administrative actions.

Tests MUST verify both successful and failure states.

---

## 18. Accessibility in CI/CD

Accessibility checks MUST be integrated into the development lifecycle.

- Linting SHOULD include JSX accessibility rules.
- Automated accessibility tests MUST run in CI for designated critical components and workflows.
- Pull requests MUST identify accessibility-impacting changes.
- Accessibility failures MUST be addressed before production release when they affect essential workflows or violate required standards.
- Exceptions MUST include documented risk, scope, mitigation, owner, and remediation plan.
- Shared component changes MUST trigger relevant regression tests.
- Dependency upgrades MUST be checked for changes to accessibility behavior.

Accessibility MUST be included in code review and release readiness rather than postponed to a separate audit.

---

## 19. Accessibility Defect Severity

Accessibility defects MUST be prioritized according to their impact on users and essential tasks.

| Severity | Description | Expected response |
|---|---|---|
| Critical | Blocks an essential workflow for a user group, creates a serious accessibility barrier, or prevents completion of a core task | Resolve before release or apply an approved emergency mitigation |
| High | Substantially impairs access to important content or functionality | Prioritize for immediate remediation |
| Medium | Creates friction, confusion, or partial barriers but a reasonable alternative exists | Schedule remediation based on user impact |
| Low | Minor usability or semantic issue with limited impact | Address through normal maintenance |

Severity MUST be based on actual impact, not merely on the number of affected elements.

A defect that blocks a critical workflow MUST NOT be downgraded solely because an alternate workflow exists if that alternative is itself inaccessible, burdensome, or not reasonably discoverable.

---

## 20. Accessibility and Localization

KAMPYN MAY be used by institutions serving multilingual populations.

- Text expansion MUST be accommodated by layouts.
- Language-specific content MUST use the appropriate language metadata.
- Mixed-language content SHOULD be marked appropriately.
- Date, time, and number formats SHOULD respect locale settings.
- Right-to-left content MUST be supported where required by target institutions.
- Localized labels MUST retain sufficient space and meaningful accessible names.
- Icons and symbols MUST be reviewed for cultural ambiguity.
- Accessibility MUST be retested after localization because translated text can change labels, layout, and interaction context.

---

## 21. Accessibility and Self-Hosted Universities

Universities deploying KAMPYN through self-hosted installations MUST retain access to the shared accessibility features.

- Core accessible components MUST be included in supported distributions.
- University-specific branding MUST NOT compromise contrast, focus visibility, or text readability.
- Theme customization MUST use validated design tokens or pass accessibility checks.
- Custom modules MUST follow the same accessibility standards.
- Configuration MUST NOT silently disable keyboard support, labels, or required status announcements.
- Accessibility-related settings and limitations MUST be documented for administrators.
- SDKs and extension points SHOULD provide accessible defaults and clear implementation guidance.

Institutional customization MUST NOT bypass essential accessibility safeguards.

---

## 22. Accessibility Documentation

Accessibility documentation MUST be maintained alongside the frontend implementation.

Relevant documentation SHOULD include:

- Design-system accessibility guidelines.
- Component accessibility contracts.
- Keyboard interaction references.
- Form and validation patterns.
- Modal and focus-management patterns.
- Screen-reader announcement conventions.
- Testing procedures.
- Known limitations and remediation plans.
- Accessibility review results for major releases.

Documentation MUST remain aligned with the implemented behavior.

---

## 23. Prohibited Anti-Patterns

The following practices are prohibited unless a documented exception is approved and an accessible alternative is provided:

- Using clickable `<div>` or `<span>` elements instead of semantic controls.
- Removing visible focus indicators without a replacement.
- Using placeholder text as the only form label.
- Communicating errors or statuses through color alone.
- Using `tabindex` values greater than zero to control navigation order.
- Adding ARIA roles that conflict with native HTML semantics.
- Creating custom widgets without keyboard interaction support.
- Making essential information available only on hover.
- Hiding focusable content without controlling its focusability.
- Automatically moving focus on routine state changes.
- Using inaccessible third-party controls in essential workflows.
- Relying solely on automated accessibility audits.
- Using animations that ignore reduced-motion preferences.
- Requiring drag-and-drop when a practical accessible alternative is possible.
- Creating keyboard traps.
- Making essential workflow actions inaccessible at mobile viewport sizes.
- Using icon-only controls without accessible names.
- Treating an accessibility overlay or toolbar as a replacement for accessible implementation.
- Shipping known critical accessibility blockers without an approved mitigation.

---

## 24. Accessibility Review Checklist

### Structure and Semantics

- [ ] Semantic HTML is used appropriately.
- [ ] Pages have a clear heading hierarchy.
- [ ] Landmarks are meaningful and identifiable.
- [ ] Buttons and links have correct semantics.
- [ ] Interactive controls have accessible names.

### Keyboard and Focus

- [ ] All essential functionality is keyboard-accessible.
- [ ] Focus order is logical.
- [ ] Focus indicators are visible.
- [ ] No unintended keyboard traps exist.
- [ ] Dialogs and popovers manage focus correctly.
- [ ] Skip navigation is available where required.

### Screen Readers

- [ ] Dynamic states are exposed correctly.
- [ ] Form labels and descriptions are associated.
- [ ] Important asynchronous updates are announced appropriately.
- [ ] Decorative content is hidden when appropriate.
- [ ] Custom controls expose correct roles, names, and states.

### Visual Accessibility

- [ ] Text contrast meets WCAG 2.2 AA.
- [ ] Meaningful graphical elements meet applicable contrast requirements.
- [ ] Color is not the sole indicator of meaning.
- [ ] Text resizing and zoom work without loss of essential functionality.
- [ ] Focus indicators remain visible in all themes.
- [ ] Reduced-motion preferences are respected.

### Forms and Workflows

- [ ] Inputs have associated labels.
- [ ] Instructions and required states are clear.
- [ ] Errors are descriptive and actionable.
- [ ] Validation states are programmatically exposed.
- [ ] Essential workflows can be completed without a pointer.
- [ ] Success, failure, and pending states are communicated.

### Responsive Design

- [ ] Content reflows at supported narrow widths.
- [ ] Essential content is not clipped.
- [ ] Touch targets are sufficiently large.
- [ ] Orientation is not unnecessarily restricted.
- [ ] Responsive layouts preserve logical reading and focus order.

### Testing and Delivery

- [ ] Automated accessibility tests pass.
- [ ] Manual keyboard testing is completed.
- [ ] Critical workflows receive assistive-technology testing.
- [ ] Known defects are documented and prioritized.
- [ ] Accessibility-impacting changes are included in code review.
- [ ] Documentation and component contracts are updated.

---

## 25. Definition of Done

A frontend feature is considered accessibility-complete only when:

- It follows semantic HTML and appropriate accessibility patterns.
- All essential functionality is operable using a keyboard.
- Accessible names, roles, states, and descriptions are correctly implemented.
- Visual content meets applicable contrast and readability requirements.
- Forms provide accessible labels, instructions, and validation feedback.
- Dynamic updates are exposed appropriately to assistive technologies.
- Responsive layouts preserve access to essential content and controls.
- Motion respects reduced-motion preferences.
- Shared components meet their documented accessibility contracts.
- Automated and relevant manual tests have been completed.
- Critical accessibility defects have been resolved or have an approved mitigation.
- Relevant documentation has been updated.

**Final rule:** Accessibility is a fundamental requirement of KAMPYN's user experience. Every feature, shared component, and workflow MUST be designed and implemented so that users with different abilities and interaction methods can perceive, understand, navigate, and operate it with dignity and independence.