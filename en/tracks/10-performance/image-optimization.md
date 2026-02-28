# Image Optimization

## Overview

Images account for 50–70% of page weight on most websites. An unoptimized image library can easily add 5–10 seconds to page load time on a mobile connection. Image optimization encompasses choosing the right format, sizing images correctly, compressing efficiently, serving responsively, and lazy loading below-the-fold images. This chapter covers the full pipeline: from upload processing with Sharp to responsive serving and format negotiation.

---

## Prerequisites

- HTML `<img>` and `<picture>` elements
- Basic understanding of CDNs
- Node.js for Sharp-based processing

---

## Core Concepts

### Modern image formats

| Format | Browser support | Use case | Typical size vs JPEG |
|--------|----------------|----------|---------------------|
| JPEG | Universal | Photos | Baseline |
| PNG | Universal | Transparency, screenshots | 2–3× larger than JPEG |
| WebP | 97%+ | Photos + transparency | 25–35% smaller than JPEG |
| AVIF | 90%+ | Photos | 50% smaller than JPEG |
| SVG | Universal | Icons, logos | Vector — scales perfectly |

**Strategy:** serve AVIF to browsers that support it, WebP as fallback, JPEG/PNG for older browsers.

### The image optimization checklist

- Correct dimensions (don't serve 2000px image for a 400px slot)
- Right format (AVIF/WebP for photos, SVG for icons)
- Proper compression (quality 75–85 for photos)
- Responsive sizes (`srcset` for different viewport sizes)
- Lazy loading for below-the-fold images
- CDN delivery (geographically close to users)

---

## Hands-On Examples

### Server-side processing with Sharp

```bash
npm install sharp
```

```typescript
// src/lib/image-processor.ts
import sharp from 'sharp';
import path from 'path';

interface ProcessOptions {
  width?: number;
  height?: number;
  quality?: number;
}

interface ProcessedImage {
  webp: Buffer;
  avif: Buffer;
  jpeg: Buffer;
  width: number;
  height: number;
}

export async function processUploadedImage(
  inputBuffer: Buffer,
  options: ProcessOptions = {}
): Promise<ProcessedImage> {
  const { width = 1200, quality = 80 } = options;

  const base = sharp(inputBuffer)
    .resize(width, undefined, {
      withoutEnlargement: true,  // don't upscale small images
      fit: 'inside',
    });

  const [webp, avif, jpeg, metadata] = await Promise.all([
    base.clone().webp({ quality }).toBuffer(),
    base.clone().avif({ quality: quality - 5 }).toBuffer(), // AVIF uses slightly lower quality for same perceptual quality
    base.clone().jpeg({ quality, progressive: true }).toBuffer(),
    base.metadata(),
  ]);

  return {
    webp,
    avif,
    jpeg,
    width: metadata.width ?? width,
    height: metadata.height ?? 0,
  };
}

// Generate multiple sizes for responsive images
export async function generateResponsiveSizes(
  inputBuffer: Buffer,
  sizes = [400, 800, 1200, 1920]
): Promise<Map<number, ProcessedImage>> {
  const results = new Map<number, ProcessedImage>();

  await Promise.all(
    sizes.map(async (size) => {
      const processed = await processUploadedImage(inputBuffer, { width: size });
      results.set(size, processed);
    })
  );

  return results;
}
```

### Upload endpoint with image processing

```typescript
// src/routes/uploads.ts
import { pipeline } from 'stream/promises';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { processUploadedImage } from '../lib/image-processor.js';

const s3 = new S3Client({ region: process.env.AWS_REGION });
const CDN_BASE = process.env.CDN_BASE_URL!;

fastify.post('/api/uploads/image', { preHandler: [requireAuth] }, async (request, reply) => {
  const data = await request.file();
  if (!data) return reply.status(400).send({ error: 'No file uploaded' });

  // Validate file type
  if (!data.mimetype.startsWith('image/')) {
    return reply.status(400).send({ error: 'File must be an image' });
  }

  const inputBuffer = await data.toBuffer();

  // Limit size (5MB)
  if (inputBuffer.length > 5 * 1024 * 1024) {
    return reply.status(400).send({ error: 'Image too large (max 5MB)' });
  }

  const id = crypto.randomUUID();
  const processed = await processUploadedImage(inputBuffer);

  // Upload all formats to S3 in parallel
  await Promise.all([
    s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: `images/${id}.webp`,
      Body: processed.webp,
      ContentType: 'image/webp',
      CacheControl: 'public, max-age=31536000, immutable',
    })),
    s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: `images/${id}.avif`,
      Body: processed.avif,
      ContentType: 'image/avif',
      CacheControl: 'public, max-age=31536000, immutable',
    })),
    s3.send(new PutObjectCommand({
      Bucket: process.env.S3_BUCKET!,
      Key: `images/${id}.jpg`,
      Body: processed.jpeg,
      ContentType: 'image/jpeg',
      CacheControl: 'public, max-age=31536000, immutable',
    })),
  ]);

  return {
    id,
    urls: {
      avif: `${CDN_BASE}/images/${id}.avif`,
      webp: `${CDN_BASE}/images/${id}.webp`,
      jpeg: `${CDN_BASE}/images/${id}.jpg`,
    },
    width: processed.width,
    height: processed.height,
  };
});
```

### Responsive images in HTML

```html
<!-- picture element — browser selects best format + size -->
<picture>
  <!-- AVIF for supporting browsers (smallest) -->
  <source
    srcset="
      /images/hero-400.avif  400w,
      /images/hero-800.avif  800w,
      /images/hero-1200.avif 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
    type="image/avif"
  />

  <!-- WebP fallback -->
  <source
    srcset="
      /images/hero-400.webp  400w,
      /images/hero-800.webp  800w,
      /images/hero-1200.webp 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
    type="image/webp"
  />

  <!-- JPEG final fallback -->
  <img
    src="/images/hero-800.jpg"
    srcset="
      /images/hero-400.jpg  400w,
      /images/hero-800.jpg  800w,
      /images/hero-1200.jpg 1200w
    "
    sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
    alt="Hero image"
    width="1200"
    height="600"
    loading="eager"
    fetchpriority="high"
  />
</picture>
```

### Responsive images in React

```tsx
// src/components/OptimizedImage.tsx
interface OptimizedImageProps {
  id: string;
  alt: string;
  width: number;
  height: number;
  sizes?: string;
  priority?: boolean;
}

const CDN = process.env.NEXT_PUBLIC_CDN_URL ?? '';
const RESPONSIVE_WIDTHS = [400, 800, 1200, 1920];

function buildSrcSet(id: string, format: string): string {
  return RESPONSIVE_WIDTHS
    .map((w) => `${CDN}/images/${id}-${w}.${format} ${w}w`)
    .join(', ');
}

export function OptimizedImage({
  id,
  alt,
  width,
  height,
  sizes = '(max-width: 768px) 100vw, 50vw',
  priority = false,
}: OptimizedImageProps) {
  return (
    <picture>
      <source srcSet={buildSrcSet(id, 'avif')} sizes={sizes} type="image/avif" />
      <source srcSet={buildSrcSet(id, 'webp')} sizes={sizes} type="image/webp" />
      <img
        src={`${CDN}/images/${id}-800.jpg`}
        srcSet={buildSrcSet(id, 'jpg')}
        sizes={sizes}
        alt={alt}
        width={width}
        height={height}
        loading={priority ? 'eager' : 'lazy'}
        decoding={priority ? 'sync' : 'async'}
        fetchPriority={priority ? 'high' : 'auto'}
      />
    </picture>
  );
}
```

### Next.js Image component

Next.js handles optimization automatically:

```tsx
import Image from 'next/image';

// next.config.js
module.exports = {
  images: {
    domains: ['cdn.example.com'],
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
};

// Usage
function ProductCard({ product }: { product: Product }) {
  return (
    <Image
      src={product.imageUrl}
      alt={product.name}
      width={400}
      height={300}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      // priority for above-the-fold images
      placeholder="blur"
      blurDataURL={product.imagePlaceholder} // low-quality placeholder
    />
  );
}
```

### Lazy loading

```html
<!-- Native lazy loading — browser handles it automatically -->
<img
  src="/images/below-fold.jpg"
  alt="..."
  loading="lazy"     <!-- defer loading until near viewport -->
  decoding="async"   <!-- don't block main thread for decode -->
  width="800"
  height="600"
/>
```

```typescript
// Intersection Observer for JavaScript-driven lazy loading
function lazyLoadImages() {
  const images = document.querySelectorAll('img[data-src]');

  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          const img = entry.target as HTMLImageElement;
          img.src = img.dataset.src!;
          img.removeAttribute('data-src');
          observer.unobserve(img);
        }
      });
    },
    { rootMargin: '200px' } // load 200px before entering viewport
  );

  images.forEach((img) => observer.observe(img));
}
```

---

## Common Patterns & Best Practices

- **Always set `width` and `height`** — prevents CLS (layout shift) as images load
- **Use `loading="lazy"` for below-the-fold images** — native, zero JavaScript
- **Use `fetchpriority="high"` on LCP images** — tells the browser to prioritize the largest visible image
- **Process images at upload time** — not at serve time; pre-generate all required sizes and formats
- **Use a CDN with image optimization** (Cloudflare Images, Imgix) — they transform on the fly
- **Include a low-quality placeholder** (LQIP) — a 20px blurred version while the real image loads

---

## Anti-Patterns to Avoid

- Serving unoptimized originals from the origin (100MB PSD exports)
- Single image URL with no srcset — always 1200px on mobile
- `width="100%"` without `aspect-ratio` or explicit height — causes layout shift
- Not lazy loading a page with 50 product images — all load on initial render
- JPEG for logos and icons — use SVG; they scale without quality loss

---

## Debugging & Troubleshooting

**"Images are causing layout shift (CLS)"**
Missing `width` and `height` attributes. The browser does not know the image dimensions until it downloads it. Set explicit dimensions or use `aspect-ratio` in CSS.

**"LCP is slow"**
Check if the LCP image has `loading="lazy"` — remove it. Add `fetchpriority="high"`. Add a `<link rel="preload">` in the `<head>`.

**"WebP is not being served to Chrome"**
Check that the `<picture>` element's `<source>` has `type="image/webp"`. Without the `type` attribute, the browser doesn't know it's WebP and may not use it.

---

## Further Reading

- [web.dev: Use modern image formats](https://web.dev/uses-webp-images/)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [Responsive images guide — MDN](https://developer.mozilla.org/en-US/docs/Learn/HTML/Multimedia_and_embedding/Responsive_images)
- [Squoosh — browser image compression tool](https://squoosh.app/)
- Track 10: [Web Performance Metrics](web-performance-metrics.md)

---

## Summary

Image optimization is the highest-impact optimization for most content-heavy websites. The pipeline is: upload → Sharp processing (multiple sizes × multiple formats) → S3/storage → CDN delivery. In HTML, use `<picture>` with AVIF, WebP, and JPEG sources plus `srcset` for responsive sizes. Always set `width` and `height` attributes to prevent CLS. Use `loading="lazy"` for below-the-fold images and `fetchpriority="high"` for the LCP image. If you use Next.js, the built-in `<Image>` component handles the entire pipeline automatically.
