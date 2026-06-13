# Week 3 — Questions Module & Interactive Detail Panel

This week focuses on building the core practice interface, including the search and filter questions library and the interactive split-panel practice workspace.

---

## Daily Schedule

### Day 1 — Questions Library Grid & Filters
* **Focus:** Build the main questions library route with multi-option search and filter headers.
* **Tasks:**
  1. Set up route `/questions` page shell.
  2. Implement `FilterPanel` header supporting filters for Category, Difficulty, Career Role, Experience Level, and Company.
  3. Code Zustand state filters modifiers to store selected states.
  4. Build the paginated `QuestionGrid` feed to list questions matching the active filters.
* **Duration:** 8 hours
* **Verification:** Interact with filter dropdowns on the frontend and verify that query params update correctly in the URL.

### Day 2 — Split-Screen Detail Workspace Layout
* **Focus:** Create the split-panel workspace workspace layout for active question sessions.
* **Tasks:**
  1. Setup route `/questions/:id` split page template:
     * Left Panel: Question description (scrollable markdown).
     * Right Panel: User workspace response editor and hidden feedback details.
  2. Integrate a markdown parser (e.g. `react-markdown`) to render questions with code snippets.
  3. Create header panels displaying the question's metadata (category badge, difficulty, estimated time).
* **Duration:** 7 hours
* **Verification:** Verify that both panels scroll independently on desktop and stack cleanly into a single-column layout on mobile viewports.

### Day 3 — User Response Editor & Autosave
* **Focus:** Build the practice response editor with autosaving capabilities.
* **Tasks:**
  1. Build `ResponseArea` using a styled `<textarea>` with line counters and word counts.
  2. Setup Zustand action `saveDraftLocal` to save text changes to local storage.
  3. Implement a debounce utility (300ms) to prevent excessive writes to local storage during fast typing.
  4. Write "Save Draft" and "Submit Response" button handlers.
* **Duration:** 8 hours
* **Verification:** Type an answer, refresh the page, and verify that your typed draft is restored from local storage.

### Day 4 — Hidden Answer Reveal Section
* **Focus:** Build components for displaying hidden solutions and common mistakes.
* **Tasks:**
  1. Build the collapsable `HiddenAnswerPanel` component.
  2. Code the "Reveal Answer" click trigger. When clicked, it calls the backend API to record the action and reveals:
     * High-quality sample answers.
     * Core checklist concepts (bullet list).
     * Common mistakes to avoid.
  3. Create the `RelatedQuestions` list to recommend similar questions based on the current question's tags.
* **Duration:** 8 hours
* **Verification:** Click "Reveal Answer" and verify that the layout expands smoothly with micro-animations.

### Day 5 — Backend Controller Integration
* **Focus:** Integrate client workspaces with backend controllers and query handlers.
* **Tasks:**
  1. Implement Express router handlers:
     * `GET /api/v1/questions` (queries the database with filters).
     * `GET /api/v1/questions/:id` (returns complete details, including sample answers).
     * `POST /api/v1/questions/:id/responses` (saves the response and updates the user's daily progress).
  2. Integrate endpoints on the frontend using TanStack Query, replacing local mock variables.
* **Duration:** 8 hours
* **Verification:** Submit an answer, verify that the database records the response, and confirm that the local draft is automatically deleted.
