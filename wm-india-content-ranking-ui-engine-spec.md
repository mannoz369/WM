# WM India Content Ranking UI Engine

## Why

WM India needs a discovery system that keeps nationally significant posts visible while still giving every new post an equal launch window. The implementation must combine lifecycle-based ranking, cached scoring infrastructure, anti-spam vote handling, and UI rendering that matches the provided desktop and mobile reference screens.

## What

Build the first working WM India product surface: a feed and post-detail experience backed by PostgreSQL/Prisma, Redis impression buffering, velocity scoring, lifecycle-based feed routing, and UI components matching `/Users/mbodapud/Downloads/stitch_what_matter_wm_india_ui`.

The feed must route posts into:
- Incubator Spotlight for new posts.
- National Focus for high-velocity graduated posts.
- Daily Pulse for lower-velocity graduated posts.

The post-detail views must support original, video, hyperlink, and discourse layouts using the reference PNG/HTML exports as the visual source of truth.

## Constraints

### Must
- Use PostgreSQL as the primary database and Prisma as the ORM.
- Cache `upvotesCount`, `impressionsCount`, and `velocityScore` on `Post` for fast feed reads.
- Use the exact velocity formula `V = U^2 / (I + 50)`.
- Treat post age `0-6 hours` as the incubator window for ranking bypass and top spotlight placement.
- Preserve the brief's UI behavior: Spotlight anchored first, desktop split feed, mobile sticky segmented control for `National Focus` and `Daily Pulse`, and swipe-friendly mobile streams.
- Match the visual references in `/Users/mbodapud/Downloads/stitch_what_matter_wm_india_ui`, especially:
  - `intellectual_minimalist/DESIGN.md`
  - `mobile_home_feed_video_below_title_1/screen.png`
  - `mobile_home_feed_video_below_title_2/screen.png`
  - `mobile_home_feed_video_below_title_3/screen.png`
  - `desktop_post_detail_video_below_title_1/screen.png`
  - `desktop_post_detail_video_below_title_2/screen.png`
  - `desktop_post_detail_external_link_view/screen.png`
  - `desktop_post_detail_sticky_discourse_view/screen.png`
  - `desktop_post_detail_refined_discourse_view_1..4/screen.png`
- Use the design tokens from the reference: off-white background, Playfair Display headlines, Inter UI/body text, royal indigo accent, 1px structural borders, no shadows, sharp corners.
- Implement one-upvote-per-user idempotency with a unique database constraint.
- Short-circuit shadowbanned upvotes with mock HTTP 200 success while making no database write.
- Track home-feed viewport impressions through Redis counters, then flush to PostgreSQL in a scheduled worker.

### Must Not
- Do not count shadowbanned activity in global rankings.
- Do not update PostgreSQL directly on every viewport impression.
- Do not hand-roll broad visual deviations from the reference screens during the first implementation pass.
- Do not add unrelated social features, recommendation algorithms, or moderation tooling beyond the shadowban gate.

### Out of Scope
- Admin UI for creating or managing shadowbans.
- Personalized ranking.
- Full production observability and analytics dashboards.
- Native mobile apps.

## Current State

The repository at `/Users/mbodapud/Desktop/WM` is currently empty and not yet initialized as an application. The user-provided product brief is in `/Users/mbodapud/.codex/attachments/1741e7a4-8af9-46d1-9b94-51f49a807f4d/pasted-text.txt`.

The UI reference package exists outside the repo at `/Users/mbodapud/Downloads/stitch_what_matter_wm_india_ui`. It contains PNG screenshots for mobile home feed and desktop post-detail states, plus HTML exports for several desktop post-detail layouts. The style guide in `intellectual_minimalist/DESIGN.md` defines the visual system.

Suggested implementation stack, unless the user chooses otherwise before coding:
- Next.js app with TypeScript.
- Prisma schema and migrations for PostgreSQL.
- Redis client for impression buffering.
- Background worker or scheduled route/job for Redis-to-Postgres counter flushing.
- Componentized feed cards and post-detail layouts.

## Tasks

### T1: Scaffold Application Foundation
**What:** Create the app structure, package scripts, TypeScript config, styling setup, environment examples, and a minimal home route. Configure the UI theme from the reference design tokens: Playfair Display, Inter, off-white surfaces, royal indigo accents, 1px borders, no shadows, and sharp primary surfaces.
**Files:** `package.json`, `next.config.*`, `tsconfig.json`, `app/layout.tsx`, `app/page.tsx`, `app/globals.css`, `.env.example`
**Verify:** `npm run lint` and `npm run dev`; manually open the home route and confirm typography, spacing, and color tokens match `intellectual_minimalist/DESIGN.md`.

### T2: Define Data Model and Score Persistence
**What:** Add Prisma models for `Post`, `Upvote`, and `User` with the fields from the brief, including `PostType`, cached counters, ranking indexes, and `@@unique([userId, postId])`. Add a migration containing the PostgreSQL trigger `sync_post_velocity_on_upvote()` so every successful upvote increments `upvotesCount` and recalculates `velocityScore`.
**Files:** `prisma/schema.prisma`, `prisma/migrations/*/migration.sql`, `src/lib/db.ts`
**Verify:** `npx prisma validate`; apply migrations against a local PostgreSQL database and confirm an inserted `Upvote` updates the related `Post.velocityScore`.

### T3: Implement Ranking and Feed Routing
**What:** Build server-side ranking functions that return a single feed payload plus derived UI lanes. New posts age `<= 6 hours` must bypass ranking and appear in Spotlight. Graduated posts route to `National Focus` when `velocityScore >= 5.0`, otherwise to `Daily Pulse`. Include deterministic sort rules for ties, using higher `velocityScore` then newer `createdAt`.
**Files:** `src/lib/ranking.ts`, `src/server/feed.ts`, `app/api/feed/route.ts`, `src/lib/ranking.test.ts`
**Verify:** Unit tests covering incubator posts, top velocity spotlight selection, `V >= 5.0`, `V < 5.0`, and tie ordering.

### T4: Add Upvote API with Silent Shadowban Handling
**What:** Implement the upvote endpoint or server action. For active users, insert one `Upvote` row idempotently and return success. For users with `shadowbannedUntil` in the future, return the same success shape without writing an `Upvote` row or changing counters. Preserve duplicate-upvote behavior as a successful no-op.
**Files:** `app/api/posts/[postId]/upvote/route.ts`, `src/server/upvotes.ts`, `src/server/upvotes.test.ts`
**Verify:** Tests for active user upvote, duplicate active user upvote, expired shadowban, active shadowban mock success, and no ranking mutation from shadowbanned users.

### T5: Add Redis Impression Buffer and Flush Worker
**What:** Track viewport impressions through a lightweight endpoint that increments Redis keys with `INCRBY` or equivalent atomic increments. Add a scheduled worker that runs every 5 minutes, reads buffered post impression counts, bulk-updates `Post.impressionsCount`, recalculates `velocityScore = POWER(upvotesCount, 2) / (impressionsCount + 50)`, and clears only successfully flushed Redis counters.
**Files:** `src/lib/redis.ts`, `app/api/impressions/route.ts`, `src/workers/flush-impressions.ts`, `src/workers/flush-impressions.test.ts`
**Verify:** Tests proving buffered impressions do not touch PostgreSQL until flush, flush recalculates score, and failed writes do not lose Redis counts.

### T6: Build Feed UI Components from References
**What:** Implement `SpotlightCard`, `NationalFocusCard`, `DailyPulseCard`, desktop feed lanes, and mobile dual-stream segmented/swipe layout. Match the screenshots and style guide closely for spacing, typography, borders, card hierarchy, media placement, and interaction panels. Use the provided PNGs as visual acceptance references.
**Files:** `src/components/feed/SpotlightCard.tsx`, `src/components/feed/NationalFocusCard.tsx`, `src/components/feed/DailyPulseCard.tsx`, `src/components/feed/FeedLayout.tsx`, `app/page.tsx`, `app/globals.css`
**Verify:** Run the app and compare desktop and mobile screenshots against `mobile_home_feed_video_below_title_1..3/screen.png`; confirm Spotlight remains anchored and mobile toggles/swipes between `National Focus` and `Daily Pulse`.

### T7: Build Post Detail Layouts
**What:** Implement post-detail pages for video-below-title, external-link, sticky discourse, and refined discourse variants. Use the HTML exports in the reference package for structural guidance, but adapt them into project components and real data boundaries.
**Files:** `app/posts/[postId]/page.tsx`, `src/components/post/PostHeader.tsx`, `src/components/post/PostMedia.tsx`, `src/components/post/DiscoursePanel.tsx`, `src/components/post/ExternalLinkPreview.tsx`, `src/components/post/PostActions.tsx`
**Verify:** Compare desktop screenshots against all `desktop_post_detail_*` reference folders. Confirm sticky discourse stays visible on desktop, post content remains readable, and mobile stacks without overlap.

### T8: End-to-End Seed and QA Flow
**What:** Add seed data that exercises all lifecycle routes and post types: incubator spotlight, high-velocity national focus, low-velocity daily pulse, video, hyperlink, original, and discourse-heavy posts. Add a README section documenting local setup and the exact visual reference folder.
**Files:** `prisma/seed.ts`, `README.md`, `package.json`
**Verify:** `npm run seed`; open the feed and at least one post-detail page for every post type; confirm ranking lanes, counters, upvote behavior, and visual layouts work together.

## Validation

- `npm install`
- `npx prisma validate`
- `npm run lint`
- `npm test`
- `npm run dev`
- Manual API check: create a post with `U=300`, `I=800`, flush impressions, and confirm `velocityScore` is approximately `105.88`.
- Manual API check: shadowban a user until a future timestamp, call the upvote endpoint, confirm HTTP success and no new `Upvote` row.
- Manual UI check: desktop feed uses parallel lanes and post-detail matches the provided desktop PNG references.
- Manual UI check: mobile feed keeps Spotlight at top, shows sticky segmented controls, and allows switching between `National Focus` and `Daily Pulse` without layout overlap.
