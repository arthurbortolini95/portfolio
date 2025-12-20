# Responsive Image Optimization & Edge Caching

> **TLDR**: Optimized frontend performance by implementing responsive WebP delivery and immutable CloudFront caching, resulting in a 70% reduction in total request latency.

> **Domain:** Full Stack | Infrastructure

> **Technologies:** React, TypeScript, AWS CDK, CloudFront, Zustand, WebP

## Context

The classes scheduling application serves over 43,000 users across international markets including China and Indonesia. As a media-heavy interface, the platform relies on visual "track" images to help users identify class categories quickly.

## Problem/Issue/Bottleneck

- **High Payload Size:** Original images (PNG/JPG) ranged from 250KB to 700KB, creating significant overhead on mobile networks.
- **Latency:** Total request time averaged 275ms, with image downloads alone taking up to 20ms.
- **Inefficient Caching:** Global cache policies were too permissive for dynamic content while failing to leverage immutability for static assets.
- **Unoptimized Delivery:** The same high-resolution image was served regardless of the user's viewport or device capability.

## Solution

We adopted a multi-layered optimization strategy covering asset generation, component architecture, and infrastructure configuration:

1. **Responsive Component Architecture:** Replaced the legacy `TrackImage` with a `ResponsiveImage` component utilizing the `<picture>` tag. This allows the browser to select the most appropriate asset based on resolution (250w vs 500w) and viewport media queries.
2. **Modern Format Migration:** Converted all track assets to WebP format. We chose WebP over maintaining legacy formats to achieve superior compression ratios (up to 95% reduction) without perceptible quality loss.
3. **Client-Side Preloading Store:** Implemented a global `preloadedImgsStore` using `Zustand` and a custom hook `usePreloadedImgConfigs`. By pre-instantiating `Image` objects in the background, we eliminated "pop-in" effects during navigation.
4. **Infrastructure-as-Code (IaC) Cache Tuning:** Leveraging AWS CDK, we bifurcated the CloudFront distribution logic. We shifted from a generic 1-day TTL to a tiered approach:

- **Static Assets (`/assets/*`):** 1-year TTL with `immutable` headers.
- **Application Logic:** Reduced to 10-minute TTL with `no-cache, must-revalidate` to ensure immediate delivery of updates while maintaining edge efficiency.

## Impact

- **Payload Reduction:** Image sizes dropped from ~700KB to as low as 20KB (approx. 95% reduction for small variants).
- **Latency Improvement:** Total request time per image reduced from ~275ms to an average of 80ms (70% faster).
- **Download Efficiency:** On-wire download time improved by 90%, dropping from 10ms to 1ms on average.
- **Infrastructure Cost:** Drastic reduction in data transfer out (DTO) costs due to smaller file sizes and high cache hit ratios on immutable assets.
- **User Experience:** Faster Largest Contentful Paint (LCP) and elimination of layout shifts by providing explicit `sizes` and `srcSet`.

## Diff

The `ResponsiveImage` component leverages the HTML5 `<picture>` element to delegate resolution switching to the browser.

```typescript
// src/components/ResponsiveImage/types.ts
interface ResponsiveImageConfig {
  src: string;
  sources: { media?: string; srcSet: string; sizes: string }[];
}

interface ResponsiveImageProps {
  className: string;
  alt?: string;
  imgConfig: ResponsiveImageConfig;
}
```

```typescript
// src/components/ResponsiveImage/ResponsiveImage.tsx
const ResponsiveImage = ({ imgConfig, className, alt = "responsive-image", ...rest }: ResponsiveImageProps) => {
  return (
    <picture>
      {imgConfig.sources.map((source) => (
        <source {...source} key={source.srcSet} />
      ))}
      <img src={imgConfig.src} className={className} alt={alt} {...rest} />
    </picture>
  );
};
```

We implemented a centralized utility to standardize how responsive configurations are generated for the various application tracks.

```typescript
// src/lib/utils/buildImageConfig.ts
const SMALL_SCREEN_MEDIA_QUERY = "(max-width: 1024px)";

export const buildImageConfig = (imgKey: AppImageKeys): ResponsiveImageConfig => {
  const src250 = IMGS_MAP_250W[imgKey];
  const src500 = IMGS_MAP_500W[imgKey];

  return {
    src: src500,
    sources: [
      { media: SMALL_SCREEN_MEDIA_QUERY, srcSet: `${src250} 250w`, sizes: "250px" },
      { srcSet: `${src500} 500w`, sizes: "500px" },
    ],
  };
};
```

Using AWS CDK, we defined a specific `ResponseHeadersPolicy` to enforce long-term caching for static assets, significantly improving subsequent page load speeds.

```typescript
// cdk/lib/constructs/cloudfront.ts
const assetCacheResponseHeadersPolicy = new ResponseHeadersPolicy(this, "asset-cache-policy-headers", {
  responseHeadersPolicyName: "asset-cache-policy-headers",
  customHeadersBehavior: {
    customHeaders: [
      {
        header: "Cache-Control",
        override: true,
        value: "public, max-age=31536000, immutable", // 1 year cache
      },
    ],
  },
});

this.distribution.addBehavior("/assets/*", new S3Origin(bucket), {
  allowedMethods: AllowedMethods.ALLOW_GET_HEAD,
  viewerProtocolPolicy: ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
  responseHeadersPolicy: assetCacheResponseHeadersPolicy,
});
```
