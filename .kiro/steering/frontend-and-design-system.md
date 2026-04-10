---
inclusion: fileMatch
fileMatchPattern: "app/views/**,app/components/**,app/helpers/**,app/javascript/**,app/assets/tailwind/**"
---

# Frontend & Design System

## TailwindCSS v4.x Design System

The design system is defined in `app/assets/tailwind/maybe-design-system.css`. Always reference this file before writing styles.

### Functional Tokens (REQUIRED)

Always use functional tokens instead of raw Tailwind classes:

```
text-primary       NOT  text-white / text-gray-900
bg-container       NOT  bg-white / bg-gray-900
border-primary     NOT  border-gray-200
text-secondary     NOT  text-gray-500
bg-surface         NOT  bg-gray-50
```

### Rules

- NEVER create new styles in `maybe-design-system.css` or `application.css` without explicit permission
- Always generate semantic HTML
- Use the `icon` helper from `application_helper.rb` — NEVER use `lucide_icon` directly
- Design system supports light/dark mode via `data-theme` attribute

### Icon Helper

```ruby
icon("icon-name", size: "md", color: "default")
# Sizes: xs (w-3), sm (w-4), md (w-5), lg (w-6), xl (w-7), 2xl (w-8)
# Colors: default, white, success, warning, destructive, current
# Options: custom: true (for SVG), as_button: true (wraps in DS::Button)
```

## ViewComponents

Two base classes:

- `ApplicationComponent < ViewComponent::Base` — includes Turbo helpers
- `DesignSystemComponent < ViewComponent::Base` — for DS/ components

### Component vs Partial Decision

Use ViewComponents when:

- Complex logic or styling with variants/sizes
- Reused across multiple views
- Interactive behavior or Stimulus controllers
- Configurable slots or complex APIs
- Accessibility features or ARIA support

Use Partials when:

- Primarily static HTML with minimal logic
- Used in one or few specific contexts
- Simple template content
- No variants, sizes, or complex configuration

Prefer components over partials when available.

### Design System Components (DS/)

Located in `app/components/DS/`:

- `DS::Button` — with variants (primary, icon, etc.) and sizes
- `DS::Dialog` — modal/drawer with header, body, footer slots
- `DS::Menu` — dropdown menus with menu items
- `DS::Tabs` — tabbed navigation
- `DS::Disclosure` — expandable sections
- `DS::Alert` — alert messages
- `DS::Toggle` — toggle switches
- `DS::Tooltip` — hover tooltips
- `DS::FilledIcon` — icons with background fills
- `DS::Link` — styled links

Each may have an associated Stimulus controller (e.g., `dialog_controller.js`, `menu_controller.js`).

### Domain UI Components (UI/)

Located in `app/components/UI/` — domain-specific components like `AccountPage`.

## Stimulus Controllers

### Declarative Actions (Required)

```erb
<!-- GOOD: HTML declares what happens -->
<div data-controller="toggle">
  <button data-action="click->toggle#toggle" data-toggle-target="button">Show</button>
  <div data-toggle-target="content" class="hidden">Content</div>
</div>
```

```javascript
// GOOD: Controller just responds
export default class extends Controller {
  static targets = ["button", "content"];
  toggle() {
    this.contentTarget.classList.toggle("hidden");
  }
}
```

### Rules

- Keep controllers lightweight (<7 targets)
- Single responsibility or highly related responsibilities
- Use private methods, expose clear public API
- Component controllers (in `app/components/`) stay within their component — not used in `app/views/`
- Global controllers (in `app/javascript/controllers/`) can be used anywhere
- Pass data via `data-*-value` attributes, never inline JS
- No domain logic in Stimulus controllers
- Use Stimulus callbacks, actions, targets, values, and classes

## View Templates

- Keep domain logic OUT of ERB templates — logic belongs in component/model files
- Use `styled_form_with` helper for forms (wraps `form_with` with `StyledFormBuilder`)
- Use `format_money`, `format_date` helpers for display formatting
- Pagination via Pagy (`include Pagy::Backend` in controllers, `include Pagy::Frontend` in helpers)
- Markdown rendering via `markdown(text)` helper (Redcarpet)

## Lookbook

Component development and preview available at `/design-system` (mounted as Lookbook engine).
