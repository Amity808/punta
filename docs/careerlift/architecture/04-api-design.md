# 04 — REST API Design & Contracts

This document details the RESTful API endpoints exposed by the Express backend to serve the CareerLift client SPA.

---

## Global Routing Scheme

All paths are prefixed with `/api/v1`. The API utilizes standard HTTP status codes:
* `200 OK` - Request succeeded.
* `201 Created` - Resource created successfully.
* `400 Bad Request` - Client side validation failed (e.g., Zod inputs mismatch).
* `401 Unauthorized` - Token missing or expired.
* `403 Forbidden` - Insufficient permissions.
* `404 Not Found` - Resource does not exist.
* `429 Too Many Requests` - Rate limit exceeded.
* `500 Internal Server Error` - Database down or unexpected error.

---

## Endpoint List

### 1. Authentication
#### `POST /auth/register`
* **Purpose:** Registers a new candidate.
* **Request Body:**
  ```json
  {
    "fullName": "Abdullahi",
    "email": "abdullahi@example.com",
    "password": "strongPassword123!"
  }
  ```
* **Success Response (201):**
  ```json
  {
    "status": "success",
    "token": "eyJhbGciOi...",
    "user": {
      "id": "7bf3b946-b333-4f1b-bc69-1c9dc86ad5bb",
      "email": "abdullahi@example.com",
      "fullName": "Abdullahi"
    }
  }
  ```

#### `POST /auth/login`
* **Purpose:** Authenticates returning user.
* **Request Body:**
  ```json
  {
    "email": "abdullahi@example.com",
    "password": "strongPassword123!"
  }
  ```
* **Success Response (200):**
  ```json
  {
    "status": "success",
    "token": "eyJhbGciOi...",
    "user": {
      "id": "7bf3b946-b333-4f1b-bc69-1c9dc86ad5bb",
      "email": "abdullahi@example.com",
      "fullName": "Abdullahi",
      "isOnboarded": true
    }
  }
  ```

---

### 2. Onboarding Flow
#### `POST /onboarding`
* **Purpose:** Save onboarding step selections.
* **Headers:** `Authorization: Bearer <token>`
* **Request Body:**
  ```json
  {
    "career": "frontend",
    "experienceLevel": "mid",
    "goal": "new_job",
    "dailyFrequency": 5
  }
  ```
* **Success Response (200):**
  ```json
  {
    "status": "success",
    "message": "Onboarding complete."
  }
  ```

---

### 3. Dashboard Summaries
#### `GET /dashboard/summary`
* **Purpose:** Pulls primary feed values.
* **Headers:** `Authorization: Bearer <token>`
* **Success Response (200):**
  ```json
  {
    "user": {
      "fullName": "Abdullahi",
      "currentStreak": 14,
      "questionsCompletedToday": 3,
      "targetQuestionsToday": 5,
      "readinessScore": 72
    },
    "continueLearning": [
      {
        "id": "4a737f1e-f3b3-469b-8e2b-23214b7ac812",
        "title": "Design a Distributed Rate Limiter",
        "category": "system_design",
        "difficulty": "hard",
        "timeLeftMins": 12
      }
    ],
    "recommendedTopics": ["React hooks state management", "System design scalability patterns"],
    "recentCompanyReports": [
      {
        "id": "dbf2c1a8-c2b3-4e4b-97aa-36421a8d021f",
        "companyName": "Flutterwave",
        "role": "Backend Engineer",
        "stage": "technical",
        "createdAt": "2026-06-12T15:20:00Z"
      }
    ]
  }
  ```

---

### 4. Questions Module
#### `GET /questions`
* **Purpose:** Paginated query lists. Supports filters.
* **Query Params:** `category`, `difficulty`, `role`, `experienceLevel`, `company`, `page`, `limit`
* **Success Response (200):**
  ```json
  {
    "data": [
      {
        "id": "a90bb7cb-92c1-460d-85fa-b6a9082a6d71",
        "title": "Explain React Reconciliation",
        "category": "technical",
        "difficulty": "medium",
        "estimatedTimeMins": 10,
        "isBookmarked": false
      }
    ],
    "meta": {
      "total": 45,
      "page": 1,
      "limit": 10
    }
  }
  ```

#### `GET /questions/daily`
* **Purpose:** Generates today's target card set matching user configurations.
* **Success Response (200):**
  ```json
  {
    "count": 5,
    "questions": [
      {
        "id": "a90bb7cb-92c1-460d-85fa-b6a9082a6d71",
        "title": "Explain React Reconciliation",
        "category": "technical",
        "difficulty": "medium",
        "estimatedTimeMins": 10,
        "isBookmarked": false
      }
    ]
  }
  ```

#### `GET /questions/:id`
* **Purpose:** Specific question layout specs.
* **Success Response (200):**
  ```json
  {
    "id": "a90bb7cb-92c1-460d-85fa-b6a9082a6d71",
    "title": "Explain React Reconciliation",
    "contentMarkdown": "How does React update the DOM in modern versions? Write down your understanding of Fibers, the Commit phase, and Diffing.",
    "category": "technical",
    "difficulty": "medium",
    "estimatedTimeMins": 10,
    "isBookmarked": false,
    "sampleAnswer": "React uses a virtual representation of the DOM...",
    "keyConcepts": ["Virtual DOM", "Fibers", "Reconciliation Algorithm"],
    "commonMistakes": ["Assuming the entire DOM is re-rendered on every state change"],
    "draftResponse": "My current draft response text...",
    "relatedQuestions": [
      {
        "id": "e3012921-27aa-4fbe-ae61-0f727abac211",
        "title": "Explain useMemo vs useCallback",
        "difficulty": "easy"
      }
    ]
  }
  ```

#### `POST /questions/:id/responses`
* **Purpose:** Save response as draft or final submission.
* **Request Body:**
  ```json
  {
    "responseText": "Fibers are lightweight execution threads used in React 16+ reconciliation...",
    "status": "submitted" // OR "draft"
  }
  ```
* **Success Response (200):**
  ```json
  {
    "status": "success",
    "message": "Response submitted.",
    "dailyTargetMet": true,
    "streakUpdated": {
      "currentStreak": 15,
      "questionsCompletedToday": 4
    }
  }
  ```

#### `POST /questions/:id/bookmark`
* **Purpose:** Bookmark a question for future review.
* **Success Response (200):**
  ```json
  {
    "status": "success",
    "isBookmarked": true
  }
  ```

#### `DELETE /questions/:id/bookmark`
* **Purpose:** Remove bookmark.
* **Success Response (200):**
  ```json
  {
    "status": "success",
    "isBookmarked": false
  }
  ```

---

### 5. Company Database & Contributions
#### `GET /companies`
* **Purpose:** List companies.
* **Query Params:** `search`, `role`, `location`
* **Success Response (200):**
  ```json
  {
    "data": [
      {
        "id": "d1c25141-f761-46ab-8c90-959f636a0bf2",
        "name": "Flutterwave",
        "slug": "flutterwave",
        "questionsCount": 42,
        "reportsCount": 18,
        "averageDifficulty": "medium",
        "rolesAvailable": ["Frontend Engineer", "Backend Engineer"]
      }
    ]
  }
  ```

#### `GET /companies/:companyId`
* **Purpose:** High-fidelity overview of a company's interview statistics.
* **Success Response (200):**
  ```json
  {
    "company": {
      "id": "d1c25141-f761-46ab-8c90-959f636a0bf2",
      "name": "Flutterwave",
      "industry": "Fintech",
      "totalReports": 18,
      "totalQuestions": 42
    },
    "interviewStages": [
      { "stage": "recruiter", "percentage": 100 },
      { "stage": "technical", "percentage": 85 },
      { "stage": "system_design", "percentage": 40 },
      { "stage": "final", "percentage": 30 }
    ],
    "mostReportedQuestions": [
      {
        "id": "a90bb7cb-92c1-460d-85fa-b6a9082a6d71",
        "title": "Explain React Reconciliation",
        "frequency": 7
      }
    ],
    "reports": [
      {
        "id": "dbf2c1a8-c2b3-4e4b-97aa-36421a8d021f",
        "role": "Frontend Engineer",
        "stage": "technical",
        "difficulty": "medium",
        "outcome": "offer",
        "questionText": "Build a multi-currency payment checkout UI mock in 30 minutes.",
        "createdAt": "2026-06-11T09:00:00Z"
      }
    ],
    "candidateTips": [
      "Be prepared for scenario questions on handling slow API responses."
    ]
  }
  ```

#### `POST /contributions`
* **Purpose:** Submit a crowd-sourced interview report.
* **Request Body:**
  ```json
  {
    "companyName": "Flutterwave",
    "role": "Frontend Engineer",
    "stage": "technical",
    "questionCategory": "technical",
    "questionText": "Explain JS closures and prototype inheritances.",
    "difficulty": "medium",
    "outcome": "offer",
    "isAnonymous": true
  }
  ```
* **Success Response (201):**
  ```json
  {
    "status": "success",
    "message": "Thank you for contributing!",
    "pointsAwarded": 100
  }
  ```

---

### 6. Streaks & Gamification
#### `GET /streaks`
* **Purpose:** Get full details on streaks, badge achievements, and calendars.
* **Success Response (200):**
  ```json
  {
    "streak": {
      "currentStreak": 14,
      "longestStreak": 32,
      "questionsCompleted": 156,
      "contributionsCount": 5
    },
    "recentActivityCalendar": [
      { "date": "2026-06-12", "completedCount": 5 },
      { "date": "2026-06-13", "completedCount": 3 }
    ],
    "badges": [
      {
        "id": "badge_7_day",
        "name": "7-Day Streak",
        "description": "Completed daily questions for 7 consecutive days.",
        "awardedAt": "2026-06-05T12:00:00Z"
      }
    ]
  }
  ```
