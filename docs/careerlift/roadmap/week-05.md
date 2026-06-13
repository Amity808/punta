# Week 5 — Streaks, Badges, Profile & Settings

This week focuses on implementing gamification logic on the frontend, including the streaks page, unlockable badges, user profiles, and application settings.

---

## Daily Schedule

### Day 1 — Streaks Page & Activity Grid
* **Focus:** Build the streaks visualization dashboard and activity calendar.
* **Tasks:**
  1. Set up route `/streaks` page layout.
  2. Build statistics counters showing: Current Streak, Longest Streak, Questions Completed, and Contributions Count.
  3. Create the `ActivityCalendar` grid component (a GitHub-like contribution grid showing practice activity over the last 12 months).
* **Duration:** 8 hours
* **Verification:** Verify that active practice days render with a green background accent on the calendar grid.

### Day 2 — Badges Grid & Unlock Animations
* **Focus:** Build the achievements grid and unlock confirmation modals.
* **Tasks:**
  1. Create the `BadgeGrid` section under the streaks page to display earned and locked badges.
  2. Implement `BadgeCard` components with different opacity states depending on whether the badge is unlocked.
  3. Build the `BadgeUnlockModal` popup overlay, adding micro-animations (e.g., confetti) using `canvas-confetti` when a badge is unlocked.
* **Duration:** 7 hours
* **Verification:** Trigger a mock badge unlock event and verify that the confirmation modal and confetti animation render correctly.

### Day 3 — User Profile Stats Grid & Edit Mode
* **Focus:** Create the user profile overview and profile editor pages.
* **Tasks:**
  1. Set up route `/profile` page layout.
  2. Build the statistics grid showing total answers, bookmarks count, and contributions count.
  3. Create the `EditProfile` view containing input fields for Name, Career Track, Experience Level, and Goals.
  4. Connect the form submission to the backend API via TanStack Query mutation to update the user's profile.
* **Duration:** 8 hours
* **Verification:** Update your name in the profile editor, save changes, and verify that the updated name is saved in the database.

### Day 4 — Settings Preferences Panel
* **Focus:** Build the application settings page and preference toggle switches.
* **Tasks:**
  1. Set up route `/settings` page layout.
  2. Implement preference form sections:
     * Question Frequency options (5, 10, or 20 questions).
     * Question Category filters.
     * Notification options (Email, Push notifications).
     * Daily Reminder Time pickers.
  3. Build toggles and selectors using Radix/Shadcn primitives.
* **Duration:** 8 hours
* **Verification:** Update settings preferences, refresh the page, and verify that the updated selections persist.

### Day 5 — Timezone Streak Logic Integration
* **Focus:** Integrate client timezones with backend controllers to verify streak updates.
* **Tasks:**
  1. Update the frontend axios instance to detect the user's local timezone (via `Intl.DateTimeFormat().resolvedOptions().timeZone`) and attach it as a header to api calls.
  2. Integrate the Express streak controller on the backend to dynamically evaluate user streaks on question submissions.
  3. Write integration tests to verify that consecutive question completions correctly increment user streaks.
* **Duration:** 8 hours
* **Verification:** Complete a daily practice question and verify that the backend returns `dailyTargetMet: true` and updates the streak count.
