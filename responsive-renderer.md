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
- **Inconsistent Breakpoint Logic:** Without a centralized standard, different teams implemented responsive behavior differently, leading to UX inconsistencies across the platform.

## Solution

### Initial Implementation

I designed and implemented a `ResponsiveRenderer` component to centralize all device-detection logic. The component was a simple wrapper that:

- Received children and rendered them based on window size detected via a `resize` event listener
- Used **conditional rendering** (not CSS `display: none`) to completely remove elements from the DOM when not needed, improving performance for our mobile-heavy user base who rarely switched devices mid-session
- Provided a clean, reusable API with type-safe responsive keys (e.g., `mobile-only`, `tablet-and-desktop`)

**Benefits of this approach:**

- Centralized the logic in a single, testable component
- Eliminated code duplication across dozens of components
- Improved performance by keeping the DOM tree shallow
- Enhanced maintainability and consistency across teams

### Problem Discovery

After deploying the initial version, I discovered a critical performance issue: **each `ResponsiveRenderer` instance was creating its own window `resize` event listener**. In pages with dozens of responsive components, this meant dozens of redundant listeners, negatively impacting performance and memory usage—especially problematic for our mobile users.

### Final Solution

To address this, I refactored the architecture using React's "lifting state up" pattern:

1.  **Context-Driven Detection:** Created a custom `useDeviceType` hook that encapsulates the device identification logic, then lifted it into a dedicated React context (`DeviceTypeContext`). This ensures only **one resize listener exists** at the context level, regardless of how many `ResponsiveRenderer` components consume it. We traded some architectural complexity for significant performance gains.

2.  **Conditional Rendering (Default Strategy):** Maintained the original approach of completely unmounting components that don't match the current `deviceType`, keeping the DOM lightweight.

3.  **CSS-Based Visibility (Optional Strategy):** Extended the component to support a `css` rendering strategy for scenarios where component state must be preserved or where quick viewport transitions are expected. This leverages SCSS media queries to toggle visibility without unmounting the subtree.

4.  **Flexible API:** Preserved the type-safe responsive keys to ensure consistent breakpoint behavior across the monorepo.

## Impact

- **Performance:** Significantly reduced DOM complexity for 43,000 students by conditionally unmounting off-screen components by default.
- **Maintainability:** Centralized the responsive logic into a single, highly tested component, reducing the potential for "responsive bugs" in the scheduling app.
- **Scalability:** The component was adopted across the repo, providing a unified standard for frontend engineers to apply it wherever needed.
- **Developer Experience:** Provided a clear "Why" vs "How" documentation, allowing developers to choose between `conditional-rendering` and `css` strategies based on specific technical trade-offs.
- **Testing:** Consolidated responsive logic into a single, heavily tested component, eliminating the need to test device-detection logic across dozens of components and reducing test maintenance overhead.

## Diff

The below snippet demonstrates the core of the centralized device type detection hook, which ensures only one resize listener is ever active:

```typescript
import { useEffect, useState } from "react";
import { Breakpoints } from "./constants";
import { DeviceType } from "./types";

/**
 * React hook to determine the current device type (mobile, tablet, desktop) based on window width.
 *
 * ⚠️ This hook adds a window resize event listener every time it is used. To avoid performance issues,
 * use this hook only ONCE in your app, typically inside a context provider (e.g., DeviceTypeContextProvider).
 *
 * @returns The current device type as 'mobile', 'tablet', or 'desktop'.
 */
export function useDeviceType(): DeviceType {
  const [device, setDevice] = useState<DeviceType>(() => {
    if (typeof window === "undefined") return "desktop";
    const width = window.innerWidth;
    if (width < Breakpoints.md) return "mobile";
    if (width < Breakpoints.lg) return "tablet";
    return "desktop";
  });

  useEffect(() => {
    function handleResize() {
      const width = window.innerWidth;
      if (width < Breakpoints.md) setDevice("mobile");
      else if (width < Breakpoints.lg) setDevice("tablet");
      else setDevice("desktop");
    }
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return device;
}
```

The component leverages a clean conditional pattern to switch between the DOM-heavy and performance-optimized rendering strategies.

```typescript
// responsive-renderer/component.tsx

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
// responsive-renderer/styles.module.scss

@use "@styles/breakpoints";

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
