# 05 — Frontend Architecture & State Management

This document defines the client-side design for the **CareerLift** single-page application built on React 19, TypeScript, and Vite.

---

## 1. Directory Structure Organization

CareerLift uses a modular, domain-driven architecture to keep code decoupled and maintainable:

```text
src/
├── main.tsx                # App bootstrap (QueryClientProvider, AuthProvider)
├── index.css               # Global tailwind imports & CSS variables
├── routes.tsx              # React Router v7 routes layout structure
├── lib/
│   ├── api.ts              # Axios interceptors (attaches JWT automatically)
│   └── utils.ts            # CN (tailwind-merge) helper
├── shared/                 # Global UI & Helpers
│   ├── components/         # Reusable design system cards, badges, inputs
│   └── hooks/              # Global hooks (e.g. useMediaQuery)
└── modules/                # Feature-specific domains
    ├── auth/
    ├── onboarding/
    ├── dashboard/
    ├── questions/
    ├── companies/
    ├── contribute/
    ├── streaks/
    └── profile/
```

---

## 2. Global State Management (Zustand)

Zustand is utilized for local-first, low-frequency reactive values. All store states are typed strictly.

### A. Auth Store (`src/modules/auth/store.ts`)
Tracks authentication tokens and primary profile properties. Saves session data to `localStorage` automatically via Zustand's `persist` middleware.

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export interface User {
  id: string;
  fullName: string;
  email: string;
  career?: string;
  experienceLevel?: string;
  goal?: string;
  dailyFrequency?: number;
}

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  setAuth: (user: User, token: string) => void;
  clearAuth: () => void;
  updateUserPreferences: (profile: Partial<User>) => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      isAuthenticated: false,
      setAuth: (user, token) => set({ user, token, isAuthenticated: true }),
      clearAuth: () => set({ user: null, token: null, isAuthenticated: false }),
      updateUserPreferences: (profile) =>
        set((state) => ({
          user: state.user ? { ...state.user, ...profile } : null,
        })),
    }),
    {
      name: 'careerlift-auth-storage',
    }
  )
);
```

### B. Question & Draft Store (`src/modules/questions/store.ts`)
Caches active drafts locally on the user's browser, preventing loss of progress if they accidentally close a tab.

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface QuestionFilters {
  role: string;
  difficulty: string;
  category: string;
  experienceLevel: string;
  company: string;
}

interface QuestionState {
  filters: QuestionFilters;
  activeDrafts: Record<string, string>; // Maps questionId -> response text draft
  setFilter: (key: keyof QuestionFilters, value: string) => void;
  resetFilters: () => void;
  saveDraftLocal: (questionId: string, text: string) => void;
  clearDraftLocal: (questionId: string) => void;
}

const initialFilters = {
  role: '',
  difficulty: '',
  category: '',
  experienceLevel: '',
  company: '',
};

export const useQuestionStore = create<QuestionState>()(
  persist(
    (set) => ({
      filters: initialFilters,
      activeDrafts: {},
      setFilter: (key, value) =>
        set((state) => ({
          filters: { ...state.filters, [key]: value },
        })),
      resetFilters: () => set({ filters: initialFilters }),
      saveDraftLocal: (questionId, text) =>
        set((state) => ({
          activeDrafts: { ...state.activeDrafts, [questionId]: text },
        })),
      clearDraftLocal: (questionId) =>
        set((state) => {
          const newDrafts = { ...state.activeDrafts };
          delete newDrafts[questionId];
          return { activeDrafts: newDrafts };
        }),
    }),
    {
      name: 'careerlift-question-drafts',
    }
  )
);
```

---

## 3. Server State Management (TanStack Query)

TanStack Query manages all API server state cache cycles. 

### Key Design Rules:
* **Query Keys:** Defined in static config files to prevent typos (e.g. `export const questionKeys = { list: ['questions'] as const, detail: (id: string) => ['questions', id] as const }`).
* **Stale Time:** Queries have a default `staleTime` of 5 minutes to prevent unnecessary requests. Daily questions (`/questions/daily`) have a `staleTime` of 1 hour since they do not change frequently during a single session.

### Mutation and Cache Invalidation Example:
When a user submits a question response, the streak counters and daily totals need to update instantly on the dashboard:

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useQuestionStore } from '../store';
import axios from 'axios';

export function useSubmitResponse(questionId: string) {
  const queryClient = useQueryClient();
  const { clearDraftLocal } = useQuestionStore();

  return useMutation({
    mutationFn: async (responseText: string) => {
      const response = await axios.post(`/api/v1/questions/${questionId}/responses`, {
        responseText,
        status: 'submitted',
      });
      return response.data;
    },
    onSuccess: (data) => {
      // Clear the local draft on successful submission
      clearDraftLocal(questionId);

      // Invalidate the cache to trigger a background fetch for updated stats
      queryClient.invalidateQueries({ queryKey: ['dashboard', 'summary'] });
      queryClient.invalidateQueries({ queryKey: ['questions', questionId] });
      queryClient.invalidateQueries({ queryKey: ['streaks'] });
    },
  });
}
```

---

## 4. CSS & Design System Integration

Styling uses Tailwind CSS utility classes customized through `tailwind.config.js` to ensure visual consistency:
* **Theme Styling:** A clean, professional, Notion/Linear style using custom CSS variables (e.g., `--background`, `--foreground`, `--border`, `--ring`).
* **Dark Mode:** Supports a sleek dark mode out-of-the-box (`.dark` class applied to `<html>`).
* **Mobile-First Responsive Modifiers:** Default styles target mobile layouts; breakpoints (`sm:`, `md:`, `lg:`) are used to adapt layouts for tablet and desktop views.
