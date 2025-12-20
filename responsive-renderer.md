# Architecting a DOM-Optimized Multi-Strategy Responsive Renderer Component

> **TLDR**: Engineered a reusable `ResponsiveRenderer` component impacting 43,000 students by optimizing DOM weight through conditional rendering and versatile CSS-based visibility.

> **Domain:** Frontend | UI Component Library

> **Technologies:** React, TypeScript, SCSS Modules, Vitest, Monorepo Architecture

## Context

As a key developer and top-contributor to a greenfield classes scheduling application , I delivered a platform serving over 43,000 students across the Chinese and Indonesian markets. During development, a recurring architectural need emerged: a standardized way to handle device-specific UI components (mobile vs. desktop) without duplicating logic across the codebase.

## Problem/Issue/Bottleneck

The application required a highly responsive UI, but implementing localized visibility checks in every component led to several issues:

- **Code Duplication:** Device viewport logic was scattered across dozens of components, making maintenance difficult.
- **DOM Bloat:** Traditional CSS-based hiding (`display: none`) keeps components in the DOM, which increased the memory footprint and initial render time for our mobile-heavy user base.
- **Inflexible Strategies:** Different use cases required different technical approaches—some needed components completely unmounted for performance, while others required persistent state that only CSS hiding could provide.

## Solution

I designed and implemented a `ResponsiveRenderer` component that encapsulates device-detection logic and offers two distinct rendering strategies:

1.  **Conditional Rendering (Default):** This strategy utilizes React's lifecycle to completely unmount components that do not match the current `deviceType`. I chose this as the default because users on our platform rarely switch between devices mid-session, and removing hidden elements results in a much lighter DOM.
2.  **CSS-Based Visibility:** For scenarios where component state must be preserved or where quick viewport transitions are expected, I extended the component to support a `css` strategy. This leverages SCSS media queries to toggle visibility without unmounting the subtree.
3.  **Context-Driven Detection:** Integrated the component with a custom `useDeviceTypeContext` hook to provide accurate, real-time device categorization (mobile, tablet, desktop) across the monorepo.
4.  **Flexible API:** Defined a preset of responsive keys (e.g., `mobile-only`, `tablet-and-desktop`) to ensure type safety and consistent breakpoint behavior across teams.

## Impact

- **Performance:** Significantly reduced DOM complexity for 43,000 students by conditionally unmounting off-screen components by default.
- **Maintainability:** Centralized the responsive logic into a single, highly tested component, reducing the potential for "responsive bugs" in the scheduling app.
- **Scalability:** The component was adopted across the repo, providing a unified standard for frontend engineers to apply it wherever needed.
- **Developer Experience:** Provided a clear "Why" vs "How" documentation, allowing developers to choose between `conditional-rendering` and `css` strategies based on specific technical trade-offs.
- **Testing:** Consolidated responsive logic into a single, heavily tested component, eliminating the need to test device-detection logic across dozens of components and reducing test maintenance overhead.

## Diff

The component leverages a clean conditional pattern to switch between the DOM-heavy and performance-optimized rendering strategies.

```typescript
// packages/react-components/src/components/responsive-renderer/component.tsx

export const ResponsiveRenderer = ({
  children,
  renderOn,
  renderStrategy = "conditional-rendering",
}: ResponsiveRendererProps) => {
  // Access centralized device context provided by the monorepo
  const { deviceType } = useDeviceTypeContext();

  // CSS Strategy: Keeps components in DOM but hides via media queries
  if (renderStrategy === "css")
    return <div className={clsx(styles.responsiveRenderer, styles[renderOn])}>{children}</div>;

  // Conditional Rendering Strategy: Completely removes subtree from DOM if mismatch
  // This is preferred for mobile performance to keep the DOM tree shallow
  return <>{renderOn.includes(deviceType as string) ? children : null}</>;
};
```

The underlying SCSS ensures that when the `css` strategy is used, breakpoints are applied consistently using monorepo-standard mixins.

```scss
// packages/react-components/src/components/responsive-renderer/styles.module.scss

@use "@monorepo/styles/breakpoints";

.responsive-renderer {
  // Strategy for components that should only exist on tablet/desktop viewports
  &.tablet-and-desktop {
    @include breakpoints.mq("mobile", max) {
      display: none; // Hides content on mobile viewports
    }
  }

  // Strategy for components that are exclusive to mobile devices
  &.mobile-only {
    @include breakpoints.mq("tablet", min) {
      display: none; // Hides content as soon as screen size reaches tablet width
    }
  }
}
```
