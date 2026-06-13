# Week 2 — Dashboard & Shared UI Components

This week focuses on creating the dashboard layouts, implementing shared UI components, and integrating initial mock data sources for user stats and feeds.

---

## Daily Schedule

### Day 1 — Shared Design Primitives
* **Focus:** Build global UI primitives (Cards, Badges, Loaders, and State layouts) to establish the design system.
* **Tasks:**
  1. Add Radix UI primitive modules for Dialogs, Popovers, and Dropdown Menus.
  2. Implement global layouts: `Header`, `Sidebar` (Notion/Linear style navigation panel), and responsive `PageContainer`.
  3. Code reusable helper cards: `Card`, `Badge` (used for difficulty levels: Easy, Medium, Hard), and status indicators (`Loader`, `EmptyState`, `ErrorState`).
* **Duration:** 8 hours
* **Verification:** Run Storybook or load a temporary workspace components sandbox page to verify style guides and visual integrity.

### Day 2 — Dashboard Shell & Welcome Card
* **Focus:** Build dashboard shell layout and the main user Welcome Card.
* **Tasks:**
  1. Create the dashboard layout under `/dashboard`.
  2. Implement the `WelcomeCard` component to show user details:
     * Localized greetings ("Hello, Abdullahi 👋").
     * Current daily streak indicator.
     * Daily target progress bar ("Questions Completed Today").
     * Overall "Readiness Score" tracker.
  3. Connect the component to Zustand's `useAuthStore` to fetch the logged-in user's profile details.
* **Duration:** 7 hours
* **Verification:** Log in with a test user and verify that the Welcome Card renders with the user's correct name and daily question target.

### Day 3 — Today's Questions Card Grid
* **Focus:** Build question feed cards for today's tasks.
* **Tasks:**
  1. Create the `TodayQuestions` section list.
  2. Implement `QuestionFeedCard` components, displaying:
     * Question title & short summary.
     * Category badge (e.g., Technical, System Design) and difficulty indicator.
     * Estimated time to complete.
  3. Add interaction triggers to the cards: "Start" (links to `/questions/:id`) and "Bookmark".
* **Duration:** 8 hours
* **Verification:** Render five mock question cards on the dashboard and verify the links to `/questions/:id` route correctly.

### Day 4 — Sidebar Feed Panels
* **Focus:** Build panels for secondary content: "Continue Learning", "Recommended Topics", and "Recent Company Reports".
* **Tasks:**
  1. Create the `ContinueLearning` sidebar module to list unanswered questions the user previously started.
  2. Add the `RecommendedTopics` tags list, generating recommendation tags based on the user's selected career track.
  3. Create the `RecentCompanyReports` feed component, listing the latest community-contributed interview questions.
* **Duration:** 7 hours
* **Verification:** Verify that all layout components fit cleanly into the grid structure on desktop, tablet, and mobile viewports.

### Day 5 — Backend Summary Aggregator & Cache Layer
* **Focus:** Build API endpoints for the dashboard feed and set up server caching.
* **Tasks:**
  1. Code Express handler `/api/v1/dashboard/summary`.
  2. Write database queries to compile:
     * The user's active streak stats.
     * Unfinished question drafts.
     * 5 daily practice questions matching the user's career and experience settings.
     * The 5 most recent company reports.
  3. Connect the API to the React client using TanStack Query, replacing frontend mock feeds with live database queries.
* **Duration:** 8 hours
* **Verification:** Log in, check the browser's Network Inspector, and verify that `/api/v1/dashboard/summary` resolves in under 150ms.
