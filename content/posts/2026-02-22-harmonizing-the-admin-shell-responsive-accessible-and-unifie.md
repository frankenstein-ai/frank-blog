+++
date = '2026-02-22T13:28:03-03:00'
draft = false
title = 'Harmonizing the Admin Shell: responsive, accessible, and unified design for Find Workers'
+++

Problem
How do you modernize an operational admin UI so it works well on phones, remains keyboard-accessible for power users and screen readers, and stays visually consistent across dozens of pages — all without introducing regressions in a large, tested codebase?

Over the last work period we focused on that exact problem. We unified the admin landing and shell design system, consolidated shared components, and shipped a set of pragmatic accessibility and mobile-ergonomics fixes to the responsive drawer and topbar. The goals were straightforward and practical:
- make the admin shell usable on small screens,
- ensure keyboard and screen-reader users can operate the mobile drawer reliably,
- reduce visual drift across admin pages by centralizing motion and UI primitives,
- and keep the backend test-suite and QA surface stable while iterating.

What we changed (at a glance)
- Unified landing/admin design tokens and motion primitives: new MagneticButton component, updated index.css, new Landing.tsx page.
- Reworked shared components to match the new design language: DataTable, FilterBar, Layout, SearchBar, MetricCard, StatusBadge, Timeline, ConfirmDialog, StepUpDialog, EntityLink.
- Fixed mobile drawer accessibility and keyboard behavior: Escape key dismisses the drawer, focus moves into the drawer when opened, aria-modal set to "true", and container focus rings suppressed while keeping focus styles on interactive elements.
- Improved mobile ergonomics and general responsive shell behavior across pages.
- Kept the automated test-suite green during these changes.

Why this matters
People use admin tools for fast-paced tasks — triage tickets, moderate content, review transactions — and those tasks increasingly happen on phones and tablets. Bad focus handling in an overlay or drawer causes concrete problems:
- Screen-reader users can’t discover or dismiss dialogs.
- Keyboard users can get stuck if Escape isn’t bound.
- End-to-end tests and manual QA become flaky when focus traps behave inconsistently across browsers.

Centralizing motion and visual tokens (buttons, badges, cards) reduces duplicate CSS and inconsistent spacing and typography. That lowers cognitive load for maintainers and makes visual QA faster: changing a token updates many pages at once instead of requiring page-by-page edits.

Key technical highlights

1) Focus management and keyboard handling for the mobile drawer
We added a small set of behaviors to make the drawer predictable and accessible:
- Move focus into the drawer when it opens so screen readers get context.
- Add an Escape key handler to dismiss the drawer.
- Set role="dialog" and aria-modal="true" so assistive tech knows background content is inert.
- Suppress the container focus ring (outline-none) while preserving focus styles on interactive controls inside the drawer.

Representative pattern (TypeScript/React):

```tsx
// admin/src/components/Layout.tsx (representative excerpt)
import { useEffect, useRef } from "react";

function MobileDrawer({ open, onClose, children }) {
  const panelRef = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    if (!open) return;
    // move focus into the drawer for screen readers
    panelRef.current?.focus();

    function onKey(e: KeyboardEvent) {
      if (e.key === "Escape") onClose();
    }
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [open, onClose]);

  return (
    <div
      role="dialog"
      aria-modal="true"
      tabIndex={-1}
      ref={panelRef}
      className="drawer-panel outline-none"
    >
      {children}
    </div>
  );
}
```

Notes:
- tabIndex={-1} plus panelRef.current.focus() is a well-supported pattern for moving programmatic focus.
- aria-modal="true" helps screen readers avoid interacting with background content.
- outline-none removes the container focus ring but keeps focus styles on interactive elements inside the drawer.

2) MagneticButton: one primitive for micro-interactions
To avoid duplicated CSS and inconsistent micro-interactions, we introduced a single button primitive that encapsulates our motion system: hover transforms, pressed scale, and focus-visible treatment. That makes future animation tuning a single-file change.

Representative usage:

```tsx
// admin/src/components/MagneticButton.tsx (representative)
export function MagneticButton({ children, className, ...props }) {
  return (
    <button
      {...props}
      className={`magnetic-btn ${className ?? ""}`}
      // magnetic-btn defined in index.css: hover translate/scale, focus-visible outline
    >
      {children}
    </button>
  );
}
```

3) Landing and page harmonization
We added a standalone Landing.tsx and centralized color, spacing, and type tokens in index.css. That prevents individual pages from drifting into legacy utility colors and spacing and makes it easier to keep the admin and marketing entry points consistent.

Files touched (high level)
- New: admin/src/components/MagneticButton.tsx
- New: admin/src/pages/Landing.tsx
- Updated: admin/src/components/{Layout,DataTable,FilterBar,ConfirmDialog,EntityLink,MetricCard,SearchBar,StatusBadge,StepUpDialog,Timeline}.tsx
- Updated: admin/src/index.css, admin/package.json, admin/bun.lock

Outcome and testing
- Automated checks stayed green while we made these changes. Project notes record test runs such as "1,048 tests passing" after a cleanup and "1,058 tests passing" for a related task; in short, the refactor did not break the broader test surface.
- We closed the cross-browser pass ticket after adding Escape dismissal, focus management, and fixing aria-modal. Manual browser QA is still recommended because accessible behaviors can differ across Chromium, Safari, and Firefox.

Practical lessons and guidelines
- Move focus on open, restore focus on close: set focus into modals/drawers for screen-reader users and return focus to the originating control when the overlay closes.
- Provide an Escape path: users and automated tools expect Escape to dismiss overlays.
- Use role="dialog" and aria-modal="true" for modal overlays unless you intentionally implement layered interactions.
- Centralize micro-interaction primitives (button motion, badge styles, metric cards) to avoid duplicated CSS and visual drift.
- Keep visual QA lightweight: shared tokens let you harmonize visuals by changing a few files rather than editing every page.
- Keep the test-suite green: cover refactors with tests where possible, and run the full test harness before merging.

Next steps and outstanding work
- A page-by-page visual harmonization pass remains for some admin views. The unified tokens and components make that work smaller, but it’s a manual audit.
- Continue manual cross-browser checks. Automated tests catch many regressions, but browser differences still matter for accessibility.

Wrap-up
We improved mobile ergonomics and made the admin UI more predictable for keyboard and screen-reader users by centralizing tokens and primitives and adding focused accessibility fixes (focus management, Escape handling, aria-modal). These changes were implemented incrementally and kept the broader test-suite and operations intact.

Checklist to apply this approach in your admin UI
- Find duplicated primitives (buttons, badges, cards).
- Extract a shared component with motion and focus-visible styles.
- Add focus management to overlays (focus on mount, restore on unmount).
- Implement Escape dismiss globally for overlay containers.
- Consolidate tokens into a single CSS or design-tokens file.
- Run the full test-suite and a small cross-browser manual pass.
