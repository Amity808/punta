# Week 4 — Company Database & Contributions

This week focuses on building the crowd-sourced interview database, including searchable company directories, stage timelines, and report submission forms.

---

## Daily Schedule

### Day 1 — Company Search Directory
* **Focus:** Build the company directory list page with text search filters.
* **Tasks:**
  1. Set up route `/companies` page template.
  2. Implement search inputs to filter companies by name, role, or location.
  3. Create `CompanyCard` components displaying:
     * Company name and logo.
     * Total questions and reports count.
     * Average difficulty level.
     * Roles available.
  4. Write debounced query hooks in TanStack Query to update lists as users type.
* **Duration:** 8 hours
* **Verification:** Type "Flutterwave" in the search box and verify that the API request is sent after a 300ms debounce delay.

### Day 2 — Company Details Profile Header & Stages
* **Focus:** Build company detail views and interview stage timeline visualizations.
* **Tasks:**
  1. Set up route `/companies/:companyId` page layout.
  2. Build the company profile hero panel displaying: name, industry, and total submissions.
  3. Create `InterviewStagesTimeline` component to visualize the hiring process:
     * Stage 1: Recruiter Screen.
     * Stage 2: Technical Interview.
     * Stage 3: System Design.
     * Stage 4: Final Round.
  4. Add frequency percentage indicators to each stage based on community reports.
* **Duration:** 7 hours
* **Verification:** Open `/companies/flutterwave` and verify that the stage timeline displays the correct sequence of steps.

### Day 3 — Questions Feed & Community Tips
* **Focus:** Build the questions lists and community tips sections on the company detail page.
* **Tasks:**
  1. Create the `MostReportedQuestions` section on the company details page.
  2. Code simple card list displaying question titles, categories, and report frequencies.
  3. Build the `CandidateTips` block to display community-contributed advice.
  4. Create tab controllers to switch between the questions list and reports history views.
* **Duration:** 8 hours
* **Verification:** Toggle between tabs on the company page to verify that query filters apply correctly.

### Day 4 — Interview Submission Form UI
* **Focus:** Create the interview report submission form with validation.
* **Tasks:**
  1. Set up route `/contribute` page layout.
  2. Build the `SubmissionForm` using React Hook Form:
     * Fields: Company Name, Role, Stage, Question Category, Question Text, Difficulty, Outcome (Offer/Rejected/Pending).
     * Anonymous Toggle switch.
  3. Write client-side Zod validation rules (all fields required; minimum question length of 30 characters).
* **Duration:** 8 hours
* **Verification:** Submit an empty form and verify that validation error messages display below all required fields.

### Day 5 — Backend Contribution Endpoints & DB Triggers
* **Focus:** Connect forms to the database and write logic to auto-create companies.
* **Tasks:**
  1. Code Express API endpoints:
     * `GET /api/v1/companies` (lists companies with counts).
     * `GET /api/v1/companies/:id` (returns company profile, stages, and reports).
     * `POST /api/v1/contributions` (creates a report).
  2. Write backend trigger logic: If a user submits a report for a company that does not exist in the database, automatically create a new company record.
  3. Connect the frontend form to the API using TanStack Query, and show a confirmation modal on success.
* **Duration:** 8 hours
* **Verification:** Submit a report, verify that the database records the entry, and confirm that the user's points total increases on successful submission.
