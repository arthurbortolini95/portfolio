# Cross-Platform Tooltip Interaction Strategy for Mobile Users

> **TLDR**: Enhanced accessibility for +43,000 students by implementing a hybrid hover-and-click tooltip trigger system to overcome mobile browser interaction limitations.

> **Domain:** Frontend | UI/UX Component Library

> **Technologies:** React, TypeScript, SCSS Modules, Vitest, Storybook

## Context

As a top contributor to a greenfield classes scheduling application, I delivered a localized platform for 32,000 users in China and 11,000 users in Indonesia. The application relies heavily on informational tooltips to guide students through complex scheduling workflows.

## Problem/Issue/Bottleneck

The original `TooltipWrapper` relied exclusively on CSS hover states for visibility. While this provided a clean desktop experience, it created a significant functional bottleneck for the majority of our users who accessed the platform via mobile web browsers. Mobile browsers lack native hover listeners, rendering critical contextual information inaccessible to touch-device users.

## Solution

I refactored the `TooltipWrapper` component to transition from a hover-only model to a hybrid interaction strategy:

1.  **Hybrid Trigger Logic:** I extended the component props to include `infoIconVisibility`, allowing the UI to conditionally render an informational trigger icon on mobile devices.
2.  **Stateful Toggle System:** Implemented a React `useState` hook to manage visibility explicitly, enabling tooltips to be toggled via click/tap events rather than relying solely on pseudo-classes.
3.  **Outside-Click Management:** Added an automated listener that closes open tooltips when a user clicks elsewhere on the document, ensuring the mobile UI remains uncluttered.
4.  **Device-Specific Rendering:** Leveraged SCSS media queries to show the click-trigger icon only on mobile viewports, maintaining the original hover behavior for desktop users to avoid UI regression.

## Impact

- **Accessibility:** Restored access to vital information for **+43,000 mobile students**, ensuring feature parity across all device types.
- **User Engagement:** Qualitative feedback indicated reduced friction in the scheduling process, as users no longer had to switch to desktop to view class details.
- **Maintainability:** By centralizing this logic in a reusable `TooltipWrapper`, the mobile-friendly behavior was automatically propagated across all 20+ scheduling sub-projects.
- **Reliability:** Achieved high test coverage with Vitest, specifically validating toggle behaviors and device-conditional visibility.

## Diff

The core component was updated to include state management and a toggleable "Info" trigger.

```typescript
// examples/efd-react-library/lib/components/TooltipWrapper/index.tsx

export const TooltipWrapper: React.FC<TooltipWrapperProps> = ({
  text,
  fixedPosition,
  contentCustomClassname = "",
  showTooltip = true,
  showOnHover = true,
  infoIconVisibility = "never",
  infoIconPosition = "left",
  children,
}) => {
  // Manual visibility state added to support click triggers on mobile
  const [isTooltipVisible, setIsTooltipVisible] = useState(false);
  const id = useId();
  const autoPosition = useTooltipPosition(id);
  const childrenWithProps = cloneElement(children, {
    "aria-describedby": id,
  });
  const position = fixedPosition ?? autoPosition;

  // Toggle function prevents event bubbling and flips visibility state
  const toggleTooltipVisibility = (e?: React.MouseEvent<HTMLButtonElement, MouseEvent> | MouseEvent) => {
    e?.stopPropagation();
    setIsTooltipVisible((curr) => !curr);
  };

  // Logic to handle closing the tooltip when clicking outside on mobile
  useEffect(() => {
    if (infoIconVisibility === "never") return;

    if (isTooltipVisible) {
      document.addEventListener("click", toggleTooltipVisibility);
    } else {
      document.removeEventListener("click", toggleTooltipVisibility);
    }

    return () => document.removeEventListener("click", toggleTooltipVisibility);
  }, [infoIconVisibility, isTooltipVisible]);

  if (!showTooltip) {
    return children;
  }

  return (
    <div
      className={clsx(styles.tooltipWrapper, showOnHover && styles.showOnHover)}
      role="tooltip"
      data-testid={TOOLTIP_WRAPPER_TEST_ID}
    >
      {childrenWithProps}
      <span
        className={clsx(
          styles.tooltipContent,
          styles[position],
          contentCustomClassname,
          isTooltipVisible ? styles.visible : styles.hidden
        )}
        id={id}
        data-testid={TOOLTIP_CONTENT_TEST_ID}
      >
        {text}
      </span>

      {/* IconButton acts as theTap-to-show trigger for touch devices */}
      <IconButton
        onClick={toggleTooltipVisibility}
        className={clsx(styles.infoIcon, styles[infoIconVisibility], styles[infoIconPosition])}
        variant="info"
        bgColor="white"
        iconColor="black"
        iconWeight="bold"
        data-testid={TOOLTIP_INFO_BTN_TEST_ID}
      />
    </div>
  );
};
```

The styling ensures that the click trigger is only visible on viewports where hover is unsupported or restricted.

```scss
// examples/efd-react-library/lib/components/TooltipWrapper/styles.module.scss

.infoIcon {
  position: absolute;

  // Mobile-only visibility logic implemented via media queries
  &.mobileOnly {
    @media screen and (min-width: variables.$breakpoint-s) {
      display: none; // Hide on larger screens where hover is preferred
    }
  }

  // Ensure tooltips triggered by click transition smoothly
  &.visible {
    visibility: visible;
    opacity: 1;
  }

  &.hidden {
    visibility: hidden;
    opacity: 0;
  }
}
```
