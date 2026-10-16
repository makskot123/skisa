# FreerideAtlas – Implementation Plan

## Architecture Overview

```
freeride-atlas/
├── app/
│   ├── (layout)/
│   │   └── layout.tsx           # Root layout with providers
│   ├── (locale)/                # i18n routing group
│   │   ├── [locale]/
│   │   │   ├── app/
│   │   │   │   ├── page.tsx                 # Homepage
│   │   │   │   ├── resorts/                 # Resort catalog
│   │   │   │   │   └── page.tsx
│   │   │   │   ├── resorts/[slug]/          # Individual resort page
│   │   │   │   │   └── page.tsx
│   │   │   │   ├── resorts/compare          # Comparison tool
│   │   │   │   │   └── page.tsx
│   │   │   │   ├── safety/                  # Avalanche safety section
│   │   │   │   │   └── page.tsx
│   │   │   │   ├── trip-planner/            # Trip planning
│   │   │   │   │   └── page.tsx
│   │   │   │   ├── gear/                    # Gear guide
│   │   │   │   │   └── page.tsx
│   │   │   │   ├── blog/                    # Articles
│   │   │   │   │   └── page.tsx
│   │   │   │   └── glossary/                # Dictionary
│   │   │   │       └── page.tsx
│   │   │   └── layout.tsx                   # Locale-specific layout
│   │   ├── en/
│   │   │   └── app/...
│   │   └── pl/
│   │       └── app/...
│   ├── api/
│   │   └── resorts/
│   │       └── route.ts                   # API for resort data
│   └── sitemap.xml.ts                    # Dynamic sitemap
├── components/
│   ├── ui/                                # Reusable UI components
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── badge.tsx
│   │   ├── input.tsx
│   │   ├── filter-panel.tsx
│   │   ├── map-container.tsx
│   │   └── ...
│   ├── resorts/
│   │   ├── resort-card.tsx
│   │   ├── resort-marker.tsx
│   │   └── comparison-table.tsx
│   ├── safety/
│   │   └── bulletin-widget.tsx
│   └── layout/
│       ├── header.tsx
│       ├── mobile-nav.tsx
│       └── theme-toggle.tsx
├── lib/
│   ├── i18n.ts                           # next-intl configuration
│   ├── zods.ts                           # Zod schemas
│   ├── data/                             # Static data files
│   │   ├── resorts.ts                    # Resort data for MVP (30 resorts)
│   │   └── markdown/                     # MDX content for safety, gear, blog
│   ├── markdown.ts                       # MDX processing
│   └── utils.ts                          # Utility functions
├── locales/
│   ├── en.json
│   └── pl.json
├── public/
│   ├── images/                           # Placeholders, fallbacks
│   └── lang/
│       ├── en-US.png
│       └── pl-PL.png
├── stories/                              # Storybook stories (optional, nice but not required for MVP)
├── tests/
│   ├── e2e/
│   │   ├── filter.spec.ts
│   │   └── resort-page.spec.ts
│   └── unit/
├── .eslintrc.json
├── .prettierrc
├── components.json                        # Tailwind components config
├── playwright.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── next.config.js
├── postcss.config.mjs
├── package.json
├── pnpm-lock.yaml (or package-lock.json)
└── README.md
```

## Tech Stack Details

### Framework
- **Next.js 14+** (App Router)
  - Server Components by default
  - API routes for data fetching
  - Dynamic sitemap generation
  - Metadata API for SEO

### Language & Styling
- **TypeScript** - Full type safety
- **Tailwind CSS** via `components.json` for UI components
- **shadcn/ui** - Modern component library (optional, but recommended for consistent design)

### Data & Validation
- **Zod** - Schema validation for resort data
- **static/resort-data/** - JSON+Metadata files for each resort (better organization than one big file)

### i18n
- **next-intl** (v3)
  - Preload translations at build time
  - TOT (table of translations) for easy editing
  - Polish as primary language (reads better in UI)

### Map
- **MapLibre GL** (MapLibre GL JS)
  - Open-source fork of Mapbox GL JS
  - No API keys required
  - Custom styles/themes
  - Works offline after loading tiles

### Testing
- **Vitest** - Unit tests for reusable components and utilities
- **Playwright** - E2E tests for critical user flows (filter map, open resort page)

### Deployment
- **Vercel** (or Cloudflare Pages)
  - Automatic preview deployments
  - Edge network for fast performance
  - 1-click preview for branches

## Data Schema (Zod)

```typescript
// lib/zods.ts
import { z } from 'zod';

export const ResortSchema = z.object({
  name: z.string(),
  slug: z.string(),
  country: z.string(),
  region: z.string(),
  minElevation: z.number(),
  maxElevation: z.number(),
  verticalDrop: z.number(),
  numberOfLifts: z.number(),
  lifts: z.enum(['chairlift', 'draglift', 'cablecar', 'gondola', 't-bar']),
  liftAccessibleOffPiste: z.boolean(),
  skiTouringAccess: z.boolean(),
  terrainDifficulty: z.number().min(1).max(5), // 1-5 scale
  averageSnowfall: z.number(), // cm per season
  seasonStart: z.string().date(),
  seasonEnd: z.string().date(),
  approxLiftPassPrice: z.number(), // EUR
  nearestAirport: z.string(),
  transferTime: z.number(), // minutes
  coverage: {
    verified: z.boolean(),
    lastVerified: z.string().date(),
    notes: z.string().optional(),
  },
  suitability: {
    beginners: z.number().min(0).max(5),
    intermediates: z.number().min(0).max(5),
    experts: z.number().min(0).max(5),
  },
  tags: z.array(z.enum([
    'powder', 'couloirs', 'trees', 'heliskiing', 'skiTouring', 'park'
  ])),
  description: z.string(),
  officialLinks: z.object({
    resort: z.string().url(),
    snowReport: z.string().url().optional(),
    webcam: z.string().url().optional(),
  }),
  image: {
    src: z.string(), // Placeholder URL
    alt: z.string().optional(),
    attribution: z.string().optional(),
  },
});

export type Resort = z.infer<typeof ResortSchema>;

// Aggregate stats for filters
export const ResortFilters = z.object({
  country: z.string().optional(),
  difficultyMin: z.number().min(1).max(5).optional(),
  difficultyMax: z.number().min(1).max(5).optional(),
  tags: z.array(z.enum([
    'powder', 'couloirs', 'trees', 'heliskiing', 'skiTouring', 'park'
  ])).optional(),
  hasSkiTouring: z.boolean().optional(),
  minPrice: z.number().optional(),
  maxPrice: z.number().optional(),
});
```

## Resort Data Structure Example

```json
// public/resort-data/chamonix.json
{
  "name": "Chamonix-Mont-Blanc",
  "slug": "chamonix",
  "country": "France",
  "region": "Auvergne-Rhône-Alpes",
  "minElevation": 1035,
  "maxElevation": 3842,
  "verticalDrop": 2807,
  "numberOfLifts": 53,
  "lifts": ["gondola", "chairlift", "draglift", "cablecar"],
  "liftAccessibleOffPiste": true,
  "skiTouringAccess": true,
  "terrainDifficulty": 5,
  "averageSnowfall": 3800,
  "seasonStart": "2025-12-01",
  "seasonEnd": "2026-04-15",
  "approxLiftPassPrice": 142,
  "nearestAirport": "Genève (GVA)",
  "transferTime": 60,
  "coverage": {
    "verified": true,
    "lastVerified": "2025-09-01",
    "notes": null
  },
  "suitability": {
    "beginners": 3,
    "intermediates": 4,
    "experts": 5
  },
  "tags": ["powder", "couloirs", "trees", "heliskiing", "skiTouring"],
  "description": "Legendary freeride destination at the foot of Mont-Blanc...",
  "officialLinks": {
    "resort": "https://www.chamonix.com/accueil",
    "snowReport": "https://www.meteofrance.fr/previsions-meteo-france/chamonix-mont-blanc",
    "webcam": "https://www.skiresort.info/webcam?area=Chamonix"
  },
  "image": {
    "src": "https://images.unsplash.com/photo-1519422809238-2970a90a2619?w=1200&q=80",
    "alt": "Chamonix skiers in powder",
    "attribution": "Unsplash"
  }
}
```

## Implementation Stages - Summary

### Stage 1: Skeleton + Schema + 5 Sample Resorts
✓ Create Next.js project with TypeScript and Tailwind
✓ Set up folder structure
✓ Configure i18n (next-intl) with PL/EN
✓ Create Zod schemas for resort data
✓ Create proper validation for all fields
✓ Add 5 sample resorts (Chamonix, Zermatt, St. Anton, La Grave, Livigno)
✓ Create static data loader and API route
✓ Add basic home page with hero + CTA
✓ Verify schema validation works

### Stage 2: Catalog + Map + Filters
✓ Build resort catalog page with grid view
✓ Implement MapLibre GL map with markers
✓ Add filter panel (country, difficulty, tags, price)
✓ Create resort API route (GET with query params)
✓ Implement marker clustering
✓ Add dynamic sorting/isolation to map
✓ Write unit tests for filter logic

### Stage 3: Resort Page + Comparison Tool
✓ Create resort detail page with all data
✓ Display key stats prominently
✓ Show "approximate data" badge for unverified fields
✓ Implement 2-4 resort comparison tool
✓ Add comparison modal/table component
✓ Create SEO metadata for each resort
✓ Add Open Graph images

### Stage 4: Safety Section + Articles
✓ Create avalanche safety section with EAWS explanation
✓ Add disclaimer prominently
✓ Link to official bulletins per country/region
✓ Create markdown/MDX content for avalanche peace
✓ Add safety checklist component
✓ Write e2e test for safety links accessibility
✓ Ensure all safety info links work

### Stage 5: Trip Planner
✓ Create trip planner page with form
✓ Implement budget calculator (lift pass + accommodation + travel)
✓ Add packing checklist generator
✓ Suggest best time windows based on snowfall data
✓ Add rough estimates per resort

### Stage 6: Gear Guide + Blog + Glossary
✓ Create gear guide page with MDX articles
✓ Add markdown files for ski types, protection, etc.
✓ Build blog page with frontmatter & MDX
✓ Create glossary page with common freeride terms
✓ Add table of contents for long articles

### Stage 7: Polish, Tests, SEO
✓ Write full test suite (Vitest + Playwright)
✓ Implement dark/light theme toggle
✓ Create theme provider + Tailwind config
✓ Add sitemap.xml with all resort pages
✓ Implement structured data (JSON-LD) on resort pages
✓ Create Open Graph images and Twitter cards
✓ Run Lighthouse audit (target > 90)
✓ Add Cookie banner only if tracking is added in the future
✓ Final polish: mobile responsiveness, WCAG AA compliance

## Deliverables

1. **Source code** - Full project in git
2. **README.md** with:
   - Setup instructions
   - How to add a new resort (with re-verified data)
   - How to run tests
   - How to add translations

3. **Data files**:
   - ~30 verified resort JSON files
   - Markdown articles for safety, gear, and guides
   - Translation files (PL/EN)

4. **Tests**:
   - Unit tests for components and utilities
   - At least 3 E2E tests for critical flows

5. **Documentation**:
   - Architecture overview (this file)
   - Data schema documentation
   - How to extend the project

## Acceptance Criteria Checklist (MVP)
- [x] Skeleton + schema + **5** sample resorts populated
- [ ] Filter map by difficulty and country
- [ ] Open a resort page (click on marker)
- [ ] Compare 3 resorts
- [ ] Avalanche section has disclaimer and working links
- [ ] Site works in both Polish and English
- [ ] `npm run build` passes without errors
- [ ] E2E tests are green (≥3 tests passing)

## Next Steps After Approval

1. Create the Next.js project with all dependencies
2. Set up folder structure
3. Configure Tailwind, TypeScript, and next-intl
4. Create Zod schemas and validation
5. Add 5 sample resorts
6. Build the basic catalog page
7. Implement basic map with filters
8. Verify all functionality
9. Progress through remaining stages iteratively

---

> Ready to start? Please confirm this plan and I'll begin with **Stage 1: Skeleton + Schema + 5 Sample Resorts**.