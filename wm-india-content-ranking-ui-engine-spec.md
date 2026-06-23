# WM India Content Ranking UI Engine

## Why

WM India needs a discovery system that keeps nationally significant posts visible while still giving every new post an equal launch window. The implementation must combine lifecycle-based ranking, cached scoring infrastructure, anti-spam vote handling, and UI rendering that matches the provided desktop and mobile reference screens.

## What

Build the first working WM India product surface: a feed and post-detail experience backed by PostgreSQL/Prisma, Redis impression buffering, velocity scoring, lifecycle-based feed routing, and UI components matching `/Users/mbodapud/Downloads/stitch_what_matter_wm_india_ui`.

The feed must present:
- Incubator Spotlight for new posts during the 6-hour launch window.
- One unified percentile-ranked feed for all ranking-eligible posts.

The post-detail views must support original, video, hyperlink, and discourse layouts using the reference PNG/HTML exports as the visual source of truth.

## Constraints

### Must
- Keep every implementation runnable locally with documented commands and local `.env` values.
- Keep the application compatible with Vercel's free plan deployment model: serverless-safe routes, no required always-on custom server process for web requests, and build/runtime configuration that works with standard Vercel project settings.
- Store persistent production data only in a managed cloud PostgreSQL-compatible database. Local development may use local PostgreSQL, but the schema, Prisma configuration, SQL, and environment variables must remain compatible with cloud Postgres providers.
- Use PostgreSQL as the primary database and Prisma as the ORM.
- Cache `upvotesCount`, `impressionsCount`, and `velocityScore` on `Post` for fast feed reads.
- Use the exact velocity formula `V = U^2 / (I + 50)`.
- Treat post age `0-6 hours` as the incubator window for ranking bypass and top spotlight placement.
- Rank all ranking-eligible posts by score percentile, not by a fixed absolute velocity cutoff. Compute each post's percentile rank from `velocityScore` against the current ranking-eligible post population.
- Return one unified ranked list ordered by `scorePercentile` descending, then `velocityScore` descending, then newer `createdAt`.
- Preserve the brief's Spotlight behavior, but remove all named feed streams and category lanes. Desktop and mobile must both show a single ranked feed after Spotlight.
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
- Do not introduce features that require a paid Vercel plan unless the spec is explicitly updated.
- Do not persist production application data in local files, SQLite, browser storage, or any non-PostgreSQL primary store.
- Do not count shadowbanned activity in global rankings.
- Do not update PostgreSQL directly on every viewport impression.
- Do not route feed lanes with a fixed threshold such as `velocityScore >= 5.0`.
- Do not create named percentile/category feed lanes.
- Do not hand-roll broad visual deviations from the reference screens during the first implementation pass.
- Do not add unrelated social features, recommendation algorithms, or moderation tooling beyond the shadowban gate.

### Out of Scope
- Admin UI for creating or managing shadowbans.
- Personalized ranking.
- Full production observability and analytics dashboards.
- Native mobile apps.

## Current State

The repository at `/Users/mbodapud/Desktop/WM` currently contains this specification but is not yet initialized as an application. The user-provided product brief is in `/Users/mbodapud/.codex/attachments/1741e7a4-8af9-46d1-9b94-51f49a807f4d/pasted-text.txt`.

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
**What:** Build server-side ranking functions that return a single feed payload. New posts age `<= 6 hours` must bypass ranking and appear in Spotlight. All ranking-eligible posts must be ranked by `velocityScore`, assigned a `scorePercentile` against the current ranking-eligible post population, and returned in one ordered list. Include deterministic sort rules for ties: higher `scorePercentile`, then higher `velocityScore`, then newer `createdAt`.
**Files:** `src/lib/ranking.ts`, `src/server/feed.ts`, `app/api/feed/route.ts`, `src/lib/ranking.test.ts`
**Verify:** Unit tests covering 6-hour incubator posts, top velocity spotlight placement, percentile calculation, unified ranked-list ordering, small ranking-eligible post samples, and tie ordering.

### T4: Add Upvote API with Silent Shadowban Handling
**What:** Implement the upvote endpoint or server action. For active users, insert one `Upvote` row idempotently and return success. For users with `shadowbannedUntil` in the future, return the same success shape without writing an `Upvote` row or changing counters. Preserve duplicate-upvote behavior as a successful no-op.
**Files:** `app/api/posts/[postId]/upvote/route.ts`, `src/server/upvotes.ts`, `src/server/upvotes.test.ts`
**Verify:** Tests for active user upvote, duplicate active user upvote, expired shadowban, active shadowban mock success, and no ranking mutation from shadowbanned users.

### T5: Add Redis Impression Buffer and Flush Worker
**What:** Track viewport impressions through a lightweight endpoint that increments Redis keys with `INCRBY` or equivalent atomic increments. Add a scheduled worker that runs every 5 minutes, reads buffered post impression counts, bulk-updates `Post.impressionsCount`, recalculates `velocityScore = POWER(upvotesCount, 2) / (impressionsCount + 50)`, and clears only successfully flushed Redis counters.
**Files:** `src/lib/redis.ts`, `app/api/impressions/route.ts`, `src/workers/flush-impressions.ts`, `src/workers/flush-impressions.test.ts`
**Verify:** Tests proving buffered impressions do not touch PostgreSQL until flush, flush recalculates score, and failed writes do not lose Redis counts.

### T6: Build Feed UI Components from References
**What:** Implement `SpotlightCard`, a reusable `RankedPostCard`, and a single responsive ranked feed layout. Match the screenshots and style guide closely for spacing, typography, borders, card hierarchy, media placement, and interaction panels while removing category-lane navigation. Use the provided PNGs as visual acceptance references for visual style, not for named stream behavior.
**Files:** `src/components/feed/SpotlightCard.tsx`, `src/components/feed/RankedPostCard.tsx`, `src/components/feed/FeedLayout.tsx`, `app/page.tsx`, `app/globals.css`
**Verify:** Run the app and compare desktop and mobile styling against `mobile_home_feed_video_below_title_1..3/screen.png`; confirm Spotlight remains anchored and all remaining posts render in one percentile-ranked list without segmented stream controls.

### T7: Build Post Detail Layouts
**What:** Implement post-detail pages for video-below-title, external-link, sticky discourse, and refined discourse variants. Use the HTML exports in the reference package for structural guidance, but adapt them into project components and real data boundaries.
**Files:** `app/posts/[postId]/page.tsx`, `src/components/post/PostHeader.tsx`, `src/components/post/PostMedia.tsx`, `src/components/post/DiscoursePanel.tsx`, `src/components/post/ExternalLinkPreview.tsx`, `src/components/post/PostActions.tsx`
**Verify:** Compare desktop screenshots against all `desktop_post_detail_*` reference folders. Confirm sticky discourse stays visible on desktop, post content remains readable, and mobile stacks without overlap.

### T8: End-to-End Seed and QA Flow
**What:** Add seed data that exercises all lifecycle states and post types: 6-hour incubator spotlight, top/middle/bottom percentile ranked posts, video, hyperlink, original, and discourse-heavy posts. Add a README section documenting local setup and the exact visual reference folder.
**Files:** `prisma/seed.ts`, `README.md`, `package.json`
**Verify:** `npm run seed`; open the feed and at least one post-detail page for every post type; confirm unified percentile ordering, counters, upvote behavior, and visual layouts work together.

## Validation

- `npm install`
- `npx prisma validate`
- `npm run lint`
- `npm test`
- `npm run dev`
- Manual API check: create a post with `U=300`, `I=800`, flush impressions, and confirm `velocityScore` is approximately `105.88`.
- Manual ranking check: seed at least eight ranking-eligible posts with different `velocityScore` values and confirm the feed returns one list ordered by `scorePercentile` descending.
- Manual API check: shadowban a user until a future timestamp, call the upvote endpoint, confirm HTTP success and no new `Upvote` row.
- Manual UI check: desktop feed shows one ranked list after Spotlight and post-detail matches the provided desktop PNG references.
- Manual UI check: mobile feed keeps Spotlight at top and shows one ranked list without category segmented controls or layout overlap.
