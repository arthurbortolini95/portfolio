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
- **Performance Overhead from Multiple Listeners:** Without a centralized device type context, each usage of device-detection logic (e.g., in every `ResponsiveRenderer` instance) would add its own window resize event listener. This could lead to dozens of redundant listeners, negatively impacting performance and memory usage, especially in large applications.

## Solution

I designed and implemented a `ResponsiveRenderer` component that encapsulates device-detection logic and offers two distinct rendering strategies, while also centralizing device type detection for optimal performance:

1.  **Conditional Rendering (Default):** This strategy utilizes React's lifecycle to completely unmount components that do not match the current `deviceType`. I chose this as the default because users on our platform rarely switch between devices mid-session, and removing hidden elements results in a much lighter DOM.
2.  **CSS-Based Visibility:** For scenarios where component state must be preserved or where quick viewport transitions are expected, I extended the component to support a `css` strategy. This leverages SCSS media queries to toggle visibility without unmounting the subtree.
3.  **Context-Driven Detection & Performance:** Integrated the component with a custom `useDeviceTypeContext` hook to provide accurate, real-time device categorization (mobile, tablet, desktop) across the monorepo. This context is powered by a single, centralized hook that manages the window resize event listener. By doing so, we avoid the performance and memory overhead of having multiple listeners attached by each component instance, ensuring that only one listener is ever active regardless of how many times the device type is consumed throughout the app.
4.  **Flexible API:** Defined a preset of responsive keys (e.g., `mobile-only`, `tablet-and-desktop`) to ensure type safety and consistent breakpoint behavior across teams.

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
