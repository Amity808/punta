# 03 — Database Schema & Drizzle ORM Models

This document defines the PostgreSQL database schema and the equivalent **Drizzle ORM** TypeScript models.

---

## 1. Raw PostgreSQL DDL

The following SQL queries define the table structures, indexes, foreign keys, and constraints in PostgreSQL:

```sql
-- Enums for categorization
CREATE TYPE user_career_enum AS ENUM ('frontend', 'backend', 'pm', 'hr', 'analyst');
CREATE TYPE user_experience_enum AS ENUM ('entry', 'mid', 'senior', 'lead');
CREATE TYPE user_goal_enum AS ENUM ('new_job', 'promotion', 'growth', 'leadership');
CREATE TYPE question_category_enum AS ENUM ('technical', 'behavioral', 'scenario', 'system_design', 'whiteboard', 'leadership');
CREATE TYPE question_difficulty_enum AS ENUM ('easy', 'medium', 'hard');
CREATE TYPE interview_stage_enum AS ENUM ('recruiter', 'technical', 'system_design', 'final');
CREATE TYPE interview_outcome_enum AS ENUM ('offer', 'rejected', 'pending');
CREATE TYPE response_status_enum AS ENUM ('draft', 'submitted');

-- Users Table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Onboarding Profiles Table
CREATE TABLE onboarding_profiles (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    career user_career_enum NOT NULL,
    experience_level user_experience_enum NOT NULL,
    goal user_goal_enum NOT NULL,
    daily_frequency INT NOT NULL DEFAULT 5,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Questions Table
CREATE TABLE questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    content_markdown TEXT NOT NULL,
    category question_category_enum NOT NULL,
    difficulty question_difficulty_enum NOT NULL,
    estimated_time_mins INT NOT NULL,
    sample_answer TEXT NOT NULL,
    key_concepts JSONB NOT NULL DEFAULT '[]'::jsonb,
    common_mistakes JSONB NOT NULL DEFAULT '[]'::jsonb,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- User Responses Table (Answers & Drafts)
CREATE TABLE user_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    user_text TEXT NOT NULL,
    status response_status_enum NOT NULL DEFAULT 'draft',
    submitted_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    CONSTRAINT unique_user_question_draft UNIQUE(user_id, question_id)
);

-- Bookmarks Table (Many-to-Many join)
CREATE TABLE bookmarks (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    PRIMARY KEY (user_id, question_id)
);

-- Companies Table
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) UNIQUE NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    industry VARCHAR(255) NOT NULL,
    logo_url VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Interview Reports Table (Contributions)
CREATE TABLE interview_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL, -- Nullable to allow anonymous contributions
    company_id UUID NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    role VARCHAR(255) NOT NULL,
    stage interview_stage_enum NOT NULL,
    question_category VARCHAR(255) NOT NULL,
    question_text TEXT NOT NULL,
    difficulty question_difficulty_enum NOT NULL,
    outcome interview_outcome_enum NOT NULL,
    is_anonymous BOOLEAN DEFAULT FALSE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Streak Histories Table
CREATE TABLE streak_histories (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    current_streak INT NOT NULL DEFAULT 0,
    longest_streak INT NOT NULL DEFAULT 0,
    last_activity_date DATE,
    questions_completed_today INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- User Badges Table
CREATE TABLE user_badges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    badge_type VARCHAR(100) NOT NULL,
    awarded_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Indexes for performance
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_questions_category_difficulty ON questions(category, difficulty);
CREATE INDEX idx_user_responses_lookup ON user_responses(user_id, status);
CREATE INDEX idx_interview_reports_company ON interview_reports(company_id);
CREATE INDEX idx_user_badges_user ON user_badges(user_id);
```

---

## 2. Drizzle ORM TypeScript Definitions

The equivalent models configured for a Drizzle/Postgres setup:

```typescript
import { 
  pgTable, 
  uuid, 
  varchar, 
  text, 
  timestamp, 
  integer, 
  boolean, 
  date, 
  jsonb, 
  pgEnum, 
  unique,
  primaryKey
} from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

// 1. Define Enums
export const careerEnum = pgEnum('user_career_enum', ['frontend', 'backend', 'pm', 'hr', 'analyst']);
export const experienceEnum = pgEnum('user_experience_enum', ['entry', 'mid', 'senior', 'lead']);
export const goalEnum = pgEnum('user_goal_enum', ['new_job', 'promotion', 'growth', 'leadership']);
export const categoryEnum = pgEnum('question_category_enum', ['technical', 'behavioral', 'scenario', 'system_design', 'whiteboard', 'leadership']);
export const difficultyEnum = pgEnum('question_difficulty_enum', ['easy', 'medium', 'hard']);
export const stageEnum = pgEnum('interview_stage_enum', ['recruiter', 'technical', 'system_design', 'final']);
export const outcomeEnum = pgEnum('interview_outcome_enum', ['offer', 'rejected', 'pending']);
export const responseStatusEnum = pgEnum('response_status_enum', ['draft', 'submitted']);

// 2. Define Tables
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  fullName: varchar('full_name', { length: 255 }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});

export const onboardingProfiles = pgTable('onboarding_profiles', {
  userId: uuid('user_id').primaryKey().references(() => users.id, { onDelete: 'cascade' }),
  career: careerEnum('career').notNull(),
  experienceLevel: experienceEnum('experience_level').notNull(),
  goal: goalEnum('goal').notNull(),
  dailyFrequency: integer('daily_frequency').default(5).notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});

export const questions = pgTable('questions', {
  id: uuid('id').primaryKey().defaultRandom(),
  title: varchar('title', { length: 255 }).notNull(),
  contentMarkdown: text('content_markdown').notNull(),
  category: categoryEnum('category').notNull(),
  difficulty: difficultyEnum('difficulty').notNull(),
  estimatedTimeMins: integer('estimated_time_mins').notNull(),
  sampleAnswer: text('sample_answer').notNull(),
  keyConcepts: jsonb('key_concepts').default([]).notNull(),
  commonMistakes: jsonb('common_mistakes').default([]).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

export const userResponses = pgTable('user_responses', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  questionId: uuid('question_id').notNull().references(() => questions.id, { onDelete: 'cascade' }),
  userText: text('user_text').notNull(),
  status: responseStatusEnum('status').default('draft').notNull(),
  submittedAt: timestamp('submitted_at', { withTimezone: true }),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
}, (t) => [
  unique('unique_user_question_draft').on(t.userId, t.questionId)
]);

export const bookmarks = pgTable('bookmarks', {
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  questionId: uuid('question_id').notNull().references(() => questions.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (t) => [
  primaryKey({ name: 'bookmarks_pk', columns: [t.userId, t.questionId] })
]);

export const companies = pgTable('companies', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).unique().notNull(),
  slug: varchar('slug', { length: 255 }).unique().notNull(),
  industry: varchar('industry', { length: 255 }).notNull(),
  logoUrl: varchar('logo_url', { length: 255 }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

export const interviewReports = pgTable('interview_reports', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id, { onDelete: 'set null' }),
  companyId: uuid('company_id').notNull().references(() => companies.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 255 }).notNull(),
  stage: stageEnum('stage').notNull(),
  questionCategory: varchar('question_category', { length: 255 }).notNull(),
  questionText: text('question_text').notNull(),
  difficulty: difficultyEnum('difficulty').notNull(),
  outcome: outcomeEnum('outcome').notNull(),
  isAnonymous: boolean('is_anonymous').default(false).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

export const streakHistories = pgTable('streak_histories', {
  userId: uuid('user_id').primaryKey().references(() => users.id, { onDelete: 'cascade' }),
  currentStreak: integer('current_streak').default(0).notNull(),
  longestStreak: integer('longest_streak').default(0).notNull(),
  lastActivityDate: date('last_activity_date'),
  questionsCompletedToday: integer('questions_completed_today').default(0).notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
});

export const userBadges = pgTable('user_badges', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  badgeType: varchar('badge_type', { length: 100 }).notNull(),
  awardedAt: timestamp('awarded_at', { withTimezone: true }).defaultNow().notNull(),
});
```
