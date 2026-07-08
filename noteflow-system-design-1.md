# NoteFlow AI — System Design Document
**Phase 1 · Full-Stack Architecture**
*Frontend: Next.js + TypeScript + Tailwind + shadcn/ui*
*Backend: Java Spring Boot · PostgreSQL · AWS S3 · Clerk Auth*

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Tech Stack Summary](#3-tech-stack-summary)
4. [Authentication — Clerk](#4-authentication--clerk)
5. [Feature 1 — Smart Notes & File Upload](#5-feature-1--smart-notes--file-upload)
6. [Feature 2 — AI Note Refactor (Gemini Flash + GPT-4o Mini)](#6-feature-2--ai-note-refactor-gemini-flash--gpt-4o-mini)
7. [Feature 3 — Task Scheduler & Calendar](#7-feature-3--task-scheduler--calendar)
8. [Feature 4 — Interview Ready Questions + Voice Practice](#8-feature-4--interview-ready-questions--voice-practice)
9. [Feature 5 — Profile, Settings & GitHub-style Streaks](#9-feature-5--profile-settings--github-style-streaks)
10. [Database Schema (PostgreSQL)](#10-database-schema-postgresql)
11. [AWS S3 Architecture](#11-aws-s3-architecture)
12. [API Design (Spring Boot REST)](#12-api-design-spring-boot-rest)
13. [Frontend Architecture (Next.js)](#13-frontend-architecture-nextjs)
14. [Data Flow Diagrams](#14-data-flow-diagrams)
15. [Security Considerations](#15-security-considerations)
16. [Deployment Architecture](#16-deployment-architecture)
17. [Phase 1 Development Roadmap](#17-phase-1-development-roadmap)

---

## 1. Project Overview

**NoteFlow AI** is an intelligent, all-in-one productivity platform that combines note-taking, AI-powered content enhancement, task management, interview preparation, and voice-based practice into a single cohesive experience.

### Core Pillars
| Pillar | Description |
|--------|-------------|
| **Capture** | Rich notes with file/image/code attachments |
| **Enhance** | AI refactoring via Gemini Flash & GPT-4o Mini |
| **Organize** | Tasks, calendar, routines |
| **Prepare** | Interview Q&A generation + Voice practice sessions |
| **Track** | Streaks, activity heatmaps, profile |

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│   Next.js 14 (App Router) · TypeScript · Tailwind · shadcn/ui   │
│   Clerk SDK (Frontend Auth)                                      │
└─────────────────┬───────────────────────────────────────────────┘
                  │ HTTPS / REST / WebSocket
┌─────────────────▼───────────────────────────────────────────────┐
│                    API GATEWAY / REVERSE PROXY                   │
│          Nginx  (or AWS API Gateway for managed option)          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│                  BACKEND LAYER (Java Spring Boot)                │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │  Notes       │  │  AI Service  │  │  Task/Calendar        │ │
│  │  Controller  │  │  Controller  │  │  Controller           │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬────────────┘ │
│         │                 │                      │               │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────────▼────────────┐ │
│  │  Notes       │  │  AI Proxy    │  │  Scheduler            │ │
│  │  Service     │  │  Service     │  │  Service              │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬────────────┘ │
│         │                 │                      │               │
│  ┌──────▼─────────────────▼──────────────────────▼────────────┐ │
│  │                   Repository Layer (JPA/Hibernate)          │ │
│  └─────────────────────────┬───────────────────────────────────┘ │
└────────────────────────────┼────────────────────────────────────┘
                             │
         ┌───────────────────┼─────────────────────┐
         │                   │                      │
┌────────▼───────┐  ┌────────▼───────┐  ┌──────────▼──────────┐
│  PostgreSQL    │  │   AWS S3       │  │  External AI APIs   │
│  (Primary DB)  │  │  (File Store)  │  │  Gemini Flash       │
│                │  │                │  │  GPT-4o Mini        │
└────────────────┘  └────────────────┘  └─────────────────────┘
         │
┌────────▼───────┐
│  Clerk Auth    │
│  (JWT/Webhook) │
└────────────────┘
```

---

## 3. Tech Stack Summary

### Frontend
| Layer | Technology |
|-------|-----------|
| Framework | Next.js 14 (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS + shadcn/ui |
| Auth Client | Clerk Next.js SDK |
| Rich Text | TipTap or Lexical Editor |
| File Upload | react-dropzone + AWS S3 pre-signed URLs |
| Voice | Web Speech API + MediaRecorder API |
| Calendar | FullCalendar or shadcn custom calendar |
| State Management | Zustand + React Query (TanStack) |
| Code Highlighting | Shiki or Prism.js |

### Backend
| Layer | Technology |
|-------|-----------|
| Framework | Spring Boot 3.x |
| Language | Java 21 (LTS) |
| ORM | Spring Data JPA + Hibernate |
| Database | PostgreSQL 16 |
| File Storage | AWS S3 SDK v2 |
| Auth Validation | Clerk JWT + Spring Security |
| AI Integration | OpenAI Java SDK + Gemini REST client |
| Build | Maven / Gradle |
| API Style | RESTful JSON |

### Infrastructure
| Service | Purpose |
|---------|---------|
| AWS S3 | File/image/document storage |
| AWS CloudFront | CDN for S3 assets |
| AWS RDS | Managed PostgreSQL |
| Clerk | Authentication & user management |
| Vercel | Frontend hosting (Next.js) |
| AWS EC2 / ECS | Spring Boot hosting |

---

## 4. Authentication — Clerk

### Flow
```
User visits app
     │
     ▼
Clerk Middleware (Next.js)
     │ unauthenticated
     ▼
Clerk Sign-in / Sign-up Page
     │ success
     ▼
Clerk issues JWT (session token)
     │
     ▼
Frontend sends JWT in Authorization header
     │
     ▼
Spring Boot Security Filter
     │ validates JWT via Clerk JWKS endpoint
     ▼
Extract clerkUserId → look up/create User in PostgreSQL
     │
     ▼
Request proceeds to controller
```

### Spring Boot Clerk JWT Filter
```java
// ClerkJwtFilter.java
// 1. Extract Bearer token from Authorization header
// 2. Fetch Clerk JWKS from: https://api.clerk.com/v1/jwks
// 3. Verify JWT signature and expiry
// 4. Extract sub (clerkUserId) and set SecurityContext
// 5. Auto-create User row on first login via Clerk webhook
```

### Clerk Webhooks (user.created, user.updated)
- Spring Boot exposes `/api/webhooks/clerk` endpoint
- Syncs user profile data to PostgreSQL `users` table
- Svix signature validation required

---

## 5. Feature 1 — Smart Notes & File Upload

### Capabilities
- Rich text editor (headings, bold, italic, lists, blockquotes)
- Code blocks with syntax highlighting (50+ languages)
- Inline image upload
- File attachments: PDF, DOCX, PPTX, TXT, CSV
- Note tagging & folder organization
- Note search (full-text PostgreSQL `tsvector`)
- Pinning, archiving, soft-delete

### File Upload Flow
```
User drops/selects file in Editor
          │
          ▼
Frontend calls POST /api/notes/attachments/presign
  { filename, contentType, noteId }
          │
          ▼
Spring Boot generates S3 pre-signed PUT URL (15-min TTL)
Returns: { uploadUrl, fileKey, attachmentId }
          │
          ▼
Frontend PUTs file directly to S3 pre-signed URL
(never passes through Spring Boot — saves bandwidth)
          │
          ▼
Frontend calls POST /api/notes/attachments/confirm
  { attachmentId, fileKey }
          │
          ▼
Spring Boot marks attachment as CONFIRMED in DB
Stores CloudFront CDN URL for serving
          │
          ▼
Attachment rendered inside note editor
```

### S3 Bucket Structure
```
noteflow-assets/
├── users/{clerkUserId}/
│   ├── notes/{noteId}/
│   │   ├── images/
│   │   │   └── {uuid}.{ext}
│   │   └── attachments/
│   │       └── {uuid}.{ext}
│   └── profile/
│       └── avatar.{ext}
```

### Note Data Model (simplified)
```
Note
├── id (UUID)
├── clerkUserId
├── title
├── content (JSON - TipTap/Lexical format)
├── contentText (plain text for full-text search)
├── tags[]
├── folderId (nullable)
├── isPinned
├── isArchived
├── createdAt / updatedAt
└── Attachments[]
    ├── id (UUID)
    ├── noteId
    ├── fileName
    ├── fileType
    ├── s3Key
    ├── cdnUrl
    └── sizeBytes
```

---

## 6. Feature 2 — AI Note Refactor (Gemini Flash + GPT-4o Mini)

### Capabilities
- Refactor & enhance raw notes into structured, readable notes
- Add key points / TL;DR summary
- Generate interview questions from note content
- Add suggestions, explanations, and related concepts
- Side-by-side original vs. enhanced view
- User selects AI provider: **Gemini Flash** or **GPT-4o Mini**
- Save enhanced version as new note or overwrite

### AI Refactor Flow
```
User opens Note → clicks "AI Enhance"
          │
          ▼
Select AI Provider (Gemini Flash | GPT-4o Mini)
Select Enhancement Type:
  [✓] Refactor & Structure
  [✓] Key Points
  [✓] Interview Questions
  [✓] Suggestions
          │
          ▼
POST /api/ai/refactor
{
  noteId,
  provider: "GEMINI_FLASH" | "GPT4O_MINI",
  options: { refactor, keyPoints, questions, suggestions }
}
          │
          ▼
Spring Boot AI Service:
  1. Fetch note content from DB
  2. Build structured prompt (see below)
  3. Route to correct provider
  4. Stream response back via SSE
          │
          ▼
Frontend renders streaming response
in side-by-side split view
          │
          ▼
User clicks "Save Enhanced Note"
POST /api/ai/refactor/{jobId}/save
```

### Prompt Template (System Design)
```
SYSTEM:
You are an expert note-taking assistant. Enhance the given raw note.
Return a JSON object with these fields:
{
  "refactoredNote": "...",    // Structured, well-formatted version
  "keyPoints": ["..."],       // 5-8 bullet key takeaways
  "interviewQuestions": [     // 5-10 interview questions
    { "question": "...", "difficulty": "easy|medium|hard" }
  ],
  "suggestions": ["..."]      // 3-5 improvements or related topics
}

USER:
[Raw note content here]
```

### AI Provider Abstraction (Spring Boot)
```java
// AIProviderService.java
interface AIProviderService {
    Flux<String> streamRefactor(String prompt, AIOptions options);
}

// GeminiFlashService.java implements AIProviderService
// - Uses Google Generative AI REST API
// - Model: gemini-1.5-flash (free tier)

// GPT4oMiniService.java implements AIProviderService
// - Uses OpenAI Java SDK
// - Model: gpt-4o-mini

// AIServiceFactory.java
// Routes to correct implementation based on provider enum
```

### AI Job Tracking (DB)
```
AIRefactorJob
├── id (UUID)
├── noteId
├── clerkUserId
├── provider (GEMINI_FLASH | GPT4O_MINI)
├── status (PENDING | PROCESSING | COMPLETED | FAILED)
├── inputTokens
├── outputTokens
├── resultJson (JSONB)
└── createdAt / completedAt
```

---

## 7. Feature 3 — Task Scheduler & Calendar

### Capabilities
- To-do list with priorities (High/Medium/Low), due dates, subtasks
- Recurring tasks (daily, weekly, monthly, custom)
- Calendar view: Month / Week / Day
- Drag-and-drop rescheduling
- Routine builder (e.g., "Morning Routine: 7AM daily")
- Link tasks to notes
- Status: TODO → IN_PROGRESS → DONE → ARCHIVED
- Notifications/reminders (browser push via Web Push API)

### Data Model
```
Task
├── id (UUID)
├── clerkUserId
├── title
├── description
├── priority (HIGH | MEDIUM | LOW)
├── status (TODO | IN_PROGRESS | DONE | ARCHIVED)
├── dueDate (timestamp with timezone)
├── dueTime (optional)
├── isRecurring (boolean)
├── recurrenceRule (iCal RRULE string, e.g. "FREQ=DAILY;INTERVAL=1")
├── linkedNoteId (nullable FK)
├── tags[]
├── subtasks[]
├── createdAt / updatedAt
└── completedAt

Routine
├── id (UUID)
├── clerkUserId
├── name (e.g., "Morning Routine")
├── startTime (time)
├── daysOfWeek (array: [MON, TUE, ...])
├── tasks[] (ordered list of Task references)
└── isActive
```

### Calendar API Endpoints
```
GET  /api/tasks?start={date}&end={date}&view={month|week|day}
POST /api/tasks
PUT  /api/tasks/{id}
DELETE /api/tasks/{id}
PATCH /api/tasks/{id}/status
POST /api/tasks/{id}/reschedule { newDate, newTime }
GET  /api/routines
POST /api/routines
```

### Frontend Calendar Architecture
```
CalendarPage
├── CalendarHeader (view switcher: Month/Week/Day)
├── CalendarGrid (FullCalendar or custom)
│   └── TaskEventCard (draggable)
├── TaskSidebar
│   ├── TodoList (today's tasks)
│   ├── UpcomingTasks
│   └── RoutineList
└── TaskCreateModal
    ├── BasicInfo (title, priority, date)
    ├── RecurrenceBuilder
    └── LinkToNote (search & select)
```

---

## 8. Feature 4 — Interview Ready Questions + Voice Practice

### Sub-Feature 4A: Interview Question Search

#### Capabilities
- Search across all user notes by topic/keyword
- AI extracts and consolidates all interview questions from matching notes
- Grouped by topic/difficulty
- Bookmark/save questions to a "Question Bank"
- Export as PDF or flashcard set

#### Flow
```
User types topic (e.g., "React hooks")
          │
          ▼
POST /api/interview/search { query, filters }
          │
          ▼
Spring Boot:
  1. Full-text search notes (PostgreSQL tsvector)
  2. Fetch all AIRefactorJob results for matched notes
  3. Aggregate interviewQuestions arrays
  4. Optionally call AI to generate NEW questions
     from raw note content if no prior AI job
          │
          ▼
Return: {
  topicSummary: "...",
  questions: [
    { question, difficulty, sourceNoteTitle, sourceNoteId }
  ],
  relatedTopics: ["..."]
}
          │
          ▼
Frontend renders QuestionCard grid
  ├── Filter by difficulty
  ├── Bookmark toggle
  └── "Practice This" button → opens Voice Session
```

### Sub-Feature 4B: Voice Chat Practice Session

#### Capabilities
- Two modes: **English Speaking Practice** | **Interview Mock Session**
- Browser-based voice recording (Web Speech API / MediaRecorder)
- Sends audio → transcribes (Web Speech API or Whisper)
- AI evaluates answer for Interview mode
- AI plays conversational role for English mode
- Scores: fluency, accuracy, confidence (mock AI scoring)
- Session history & improvement tracking

#### Voice Session Flow
```
User selects: [Interview Practice] or [English Practice]
          │
          ▼
If Interview: Select topic / question set from Question Bank
If English: Select scenario (casual, formal, debate)
          │
          ▼
VoiceSessionUI loads
├── AI asks first question (text → displayed on screen)
│   (Optional: Text-to-Speech via browser SpeechSynthesis API)
          │
          ▼
User clicks "🎤 Start Talking"
Browser MediaRecorder captures audio
Web Speech API transcribes in real-time (shown as subtitle)
          │
          ▼
User clicks "Done"
Transcript sent to POST /api/voice/evaluate
{
  sessionId,
  mode: "INTERVIEW" | "ENGLISH",
  questionAsked: "...",
  userTranscript: "...",
  audioBase64: "..." (optional, for future Whisper integration)
}
          │
          ▼
Spring Boot routes to Gemini Flash or GPT-4o Mini:
  Interview mode:
    - Score technical accuracy (1-10)
    - Score communication clarity (1-10)
    - Give model answer
    - Suggest improvements

  English mode:
    - Score fluency (1-10)
    - Highlight grammar mistakes
    - Natural conversation reply (continue dialogue)
          │
          ▼
Response streams back → displayed below transcript
Next question auto-loads → session continues
          │
          ▼
Session ends → Summary Report:
  Overall score, strengths, areas to improve, session transcript
```

#### Voice Session Data Model
```
VoiceSession
├── id (UUID)
├── clerkUserId
├── mode (INTERVIEW | ENGLISH)
├── topic (nullable)
├── startedAt / endedAt
├── durationSeconds
├── overallScore (float)
└── Exchanges[]
    ├── id
    ├── sessionId
    ├── questionText
    ├── userTranscript
    ├── aiEvaluation (JSONB)
    │   ├── scores {}
    │   ├── feedback
    │   └── modelAnswer
    └── createdAt
```

---

## 9. Feature 5 — Profile, Settings & GitHub-style Streaks

### Capabilities
- User profile: avatar (S3), name, bio, timezone
- Streak tracking: daily activity = at least 1 note created/edited OR 1 task completed OR 1 AI job OR 1 voice session
- GitHub-style heatmap (52 weeks × 7 days grid)
- Stats: total notes, tasks completed, AI refactors, voice sessions, current streak, longest streak
- Settings: default AI provider, theme (light/dark/system), notification preferences, language
- Account: linked social logins (via Clerk), danger zone (delete account, export data)

### Streak & Activity Tracking

#### ActivityLog Table
```
ActivityLog
├── id (UUID)
├── clerkUserId
├── activityType (NOTE_CREATED | NOTE_EDITED | TASK_COMPLETED |
│                AI_REFACTOR | VOICE_SESSION | NOTE_AI_QUESTION)
├── entityId (UUID - the note/task/session id)
├── activityDate (DATE - timezone-adjusted)
└── createdAt (TIMESTAMP)
```

#### Streak Calculation (Spring Boot)
```java
// StreakService.java
// 1. GROUP BY activityDate for clerkUserId
// 2. Sort dates descending
// 3. Walk backwards counting consecutive days
// 4. currentStreak = count until a day gap is found
// 5. longestStreak = max consecutive run in history
// 6. Cache result in Redis or compute on-demand
```

#### Heatmap Data API
```
GET /api/profile/activity-heatmap?year=2025
Returns: [
  { date: "2025-01-01", count: 3, level: 1 },
  { date: "2025-01-02", count: 12, level: 4 },
  ...
]
// level 0-4 maps to heatmap color intensity (like GitHub)
```

#### Heatmap Frontend Component
```tsx
// ActivityHeatmap.tsx
// 52 columns (weeks) × 7 rows (days)
// Color scale: 5 levels using Tailwind classes
// Tooltip on hover: "Jan 2 — 12 activities"
// Responsive: scrollable on mobile
```

### Settings Data Model
```
UserSettings
├── clerkUserId (PK)
├── defaultAIProvider (GEMINI_FLASH | GPT4O_MINI)
├── theme (LIGHT | DARK | SYSTEM)
├── timezone (IANA timezone string)
├── emailNotifications (boolean)
├── pushNotifications (boolean)
├── language (en | hi | ...)
└── updatedAt
```

---

## 10. Database Schema (PostgreSQL)

```sql
-- Users (synced from Clerk via webhook)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clerk_user_id VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    avatar_url TEXT,
    bio TEXT,
    timezone VARCHAR(100) DEFAULT 'UTC',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Folders (optional note organization)
CREATE TABLE folders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    color VARCHAR(7),
    parent_folder_id UUID REFERENCES folders(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Notes
CREATE TABLE notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    folder_id UUID REFERENCES folders(id) ON DELETE SET NULL,
    title VARCHAR(500) NOT NULL DEFAULT 'Untitled Note',
    content JSONB,                          -- Rich text editor JSON
    content_text TEXT,                      -- Plain text for FTS
    content_vector TSVECTOR,               -- Full-text search vector
    tags TEXT[],
    is_pinned BOOLEAN DEFAULT FALSE,
    is_archived BOOLEAN DEFAULT FALSE,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_notes_user_id ON notes(user_id);
CREATE INDEX idx_notes_content_vector ON notes USING GIN(content_vector);
CREATE INDEX idx_notes_tags ON notes USING GIN(tags);

-- Note Attachments
CREATE TABLE note_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    note_id UUID REFERENCES notes(id) ON DELETE CASCADE,
    file_name VARCHAR(500) NOT NULL,
    file_type VARCHAR(100) NOT NULL,
    s3_key TEXT NOT NULL,
    cdn_url TEXT NOT NULL,
    size_bytes BIGINT,
    upload_status VARCHAR(50) DEFAULT 'PENDING', -- PENDING | CONFIRMED
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- AI Refactor Jobs
CREATE TABLE ai_refactor_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    note_id UUID REFERENCES notes(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,           -- GEMINI_FLASH | GPT4O_MINI
    status VARCHAR(50) DEFAULT 'PENDING',    -- PENDING|PROCESSING|COMPLETED|FAILED
    options JSONB,                           -- { refactor, keyPoints, questions, suggestions }
    result JSONB,                            -- Full AI response
    input_tokens INT,
    output_tokens INT,
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

-- Tasks
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    priority VARCHAR(20) DEFAULT 'MEDIUM',
    status VARCHAR(30) DEFAULT 'TODO',
    due_date TIMESTAMPTZ,
    is_recurring BOOLEAN DEFAULT FALSE,
    recurrence_rule VARCHAR(500),            -- iCal RRULE
    linked_note_id UUID REFERENCES notes(id) ON DELETE SET NULL,
    tags TEXT[],
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Subtasks
CREATE TABLE subtasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID REFERENCES tasks(id) ON DELETE CASCADE,
    title VARCHAR(500) NOT NULL,
    is_completed BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0
);

-- Routines
CREATE TABLE routines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    start_time TIME,
    days_of_week TEXT[],
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Question Bank
CREATE TABLE question_bank (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    source_note_id UUID REFERENCES notes(id) ON DELETE SET NULL,
    source_job_id UUID REFERENCES ai_refactor_jobs(id) ON DELETE SET NULL,
    question_text TEXT NOT NULL,
    difficulty VARCHAR(20),                  -- easy | medium | hard
    topic_tags TEXT[],
    is_bookmarked BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Voice Sessions
CREATE TABLE voice_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    mode VARCHAR(30) NOT NULL,               -- INTERVIEW | ENGLISH
    topic VARCHAR(255),
    overall_score DECIMAL(4,2),
    duration_seconds INT,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    ended_at TIMESTAMPTZ
);

-- Voice Session Exchanges
CREATE TABLE voice_exchanges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES voice_sessions(id) ON DELETE CASCADE,
    question_text TEXT,
    user_transcript TEXT,
    ai_evaluation JSONB,
    exchange_order INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Activity Log (for streaks & heatmap)
CREATE TABLE activity_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    activity_type VARCHAR(50) NOT NULL,
    entity_id UUID,
    activity_date DATE NOT NULL,             -- Timezone-adjusted date
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_activity_user_date ON activity_log(user_id, activity_date);

-- User Settings
CREATE TABLE user_settings (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    default_ai_provider VARCHAR(50) DEFAULT 'GEMINI_FLASH',
    theme VARCHAR(20) DEFAULT 'SYSTEM',
    timezone VARCHAR(100) DEFAULT 'UTC',
    email_notifications BOOLEAN DEFAULT TRUE,
    push_notifications BOOLEAN DEFAULT FALSE,
    language VARCHAR(10) DEFAULT 'en',
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 11. AWS S3 Architecture

### Bucket Configuration
```
Bucket: noteflow-assets-{env}        (env = dev | staging | prod)
Region: ap-south-1 (or your preferred region)

CORS Configuration:
  AllowedOrigins: ["https://noteflow.app", "http://localhost:3000"]
  AllowedMethods: ["GET", "PUT", "POST", "DELETE"]
  AllowedHeaders: ["*"]

Bucket Policy: Private (no public access)
  - All reads via CloudFront signed URLs
  - All writes via pre-signed PUT URLs (15-min TTL)

Lifecycle Rules:
  - Delete PENDING (unconfirmed) uploads > 24 hours
  - Move to S3-IA after 90 days (cost optimization)
```

### CloudFront Distribution
```
Origin: noteflow-assets-{env}.s3.amazonaws.com
Behaviors:
  /users/* → Private (signed URLs for user content)

Spring Boot generates CloudFront signed URLs
TTL: 1 hour for download links
```

### Pre-Signed URL Flow (Spring Boot)
```java
// S3Service.java
public PresignResult generateUploadUrl(String userId, String noteId,
                                        String filename, String contentType) {
    String key = "users/" + userId + "/notes/" + noteId
               + "/attachments/" + UUID.randomUUID() + "/" + filename;

    PresignedPutObjectRequest presigned = s3Presigner.presignPutObject(req ->
        req.putObjectRequest(put -> put
            .bucket(bucketName)
            .key(key)
            .contentType(contentType)
        ).signatureDuration(Duration.ofMinutes(15))
    );

    return new PresignResult(presigned.url().toString(), key);
}
```

---

## 12. API Design (Spring Boot REST)

### Base URL: `/api/v1`

#### Auth Headers
All endpoints require: `Authorization: Bearer {clerkJWT}`

### Notes Endpoints
```
GET    /notes                     → list notes (paginated, filterable)
POST   /notes                     → create note
GET    /notes/{id}                → get note with attachments
PUT    /notes/{id}                → update note
DELETE /notes/{id}                → soft delete
GET    /notes/search?q={query}    → full-text search
POST   /notes/{id}/pin            → toggle pin
POST   /notes/attachments/presign → get S3 pre-signed URL
POST   /notes/attachments/confirm → confirm upload complete
```

### AI Endpoints
```
POST   /ai/refactor               → start AI refactor job
GET    /ai/refactor/{jobId}       → get job status & result
GET    /ai/refactor/stream/{id}   → SSE stream for live output
POST   /ai/refactor/{jobId}/save  → save result as note
```

### Tasks & Calendar
```
GET    /tasks                     → list tasks (date range filter)
POST   /tasks                     → create task
PUT    /tasks/{id}                → update task
DELETE /tasks/{id}                → delete task
PATCH  /tasks/{id}/status         → update status
POST   /tasks/{id}/reschedule     → reschedule
GET    /routines                  → list routines
POST   /routines                  → create routine
```

### Interview & Voice
```
POST   /interview/search          → search & generate questions from notes
GET    /interview/questions       → list bookmarked questions
POST   /interview/questions/{id}/bookmark → toggle bookmark
POST   /voice/sessions            → start voice session
POST   /voice/evaluate            → evaluate one exchange
GET    /voice/sessions            → list past sessions
GET    /voice/sessions/{id}       → get session detail + exchanges
```

### Profile & Settings
```
GET    /profile                   → get profile
PUT    /profile                   → update profile (name, bio, timezone)
POST   /profile/avatar/presign    → get avatar upload URL
GET    /profile/stats             → streaks, totals
GET    /profile/activity-heatmap  → 52-week activity grid
GET    /settings                  → get settings
PUT    /settings                  → update settings
```

### Webhooks
```
POST   /webhooks/clerk            → Clerk user lifecycle events
```

---

## 13. Frontend Architecture (Next.js)

### App Router Structure
```
app/
├── (auth)/
│   ├── sign-in/
│   └── sign-up/
├── (dashboard)/
│   ├── layout.tsx              ← sidebar + header
│   ├── page.tsx                ← dashboard home
│   ├── notes/
│   │   ├── page.tsx            ← notes list
│   │   └── [id]/
│   │       └── page.tsx        ← note editor
│   ├── ai-enhance/
│   │   └── page.tsx            ← AI refactor studio
│   ├── tasks/
│   │   └── page.tsx            ← calendar + todo
│   ├── interview/
│   │   ├── page.tsx            ← question search
│   │   └── practice/
│   │       └── page.tsx        ← voice session
│   └── profile/
│       └── page.tsx            ← profile + settings
├── api/
│   └── webhooks/               ← Next.js webhook routes (optional)
├── layout.tsx                  ← ClerkProvider, ThemeProvider
└── middleware.ts               ← Clerk auth middleware
```

### Key UI Components
```
components/
├── editor/
│   ├── NoteEditor.tsx          ← TipTap rich text editor
│   ├── CodeBlock.tsx           ← code with syntax highlight
│   ├── FileAttachment.tsx      ← attachment chip/preview
│   └── ImageUploader.tsx       ← drag & drop image insert
├── ai/
│   ├── AIRefactorPanel.tsx     ← provider select + options
│   ├── StreamingResponse.tsx   ← live SSE output renderer
│   └── QuestionCard.tsx        ← interview question card
├── calendar/
│   ├── CalendarGrid.tsx
│   ├── TaskCard.tsx
│   └── RoutineBuilder.tsx
├── voice/
│   ├── VoiceRecorder.tsx       ← mic button + transcript
│   ├── SessionFeedback.tsx     ← scores + AI feedback
│   └── SessionSummary.tsx
├── profile/
│   ├── ActivityHeatmap.tsx     ← GitHub-style grid
│   ├── StatsCards.tsx
│   └── StreakBadge.tsx
└── shared/
    ├── Sidebar.tsx
    ├── Header.tsx
    └── ThemeToggle.tsx
```

---

## 14. Data Flow Diagrams

### Note Creation with Attachment
```
User types in NoteEditor
→ Auto-save debounce (2s) → PUT /api/notes/{id}
→ User drops image into editor
→ Frontend: calls /api/notes/attachments/presign
→ Backend: returns S3 presigned URL
→ Frontend: PUT directly to S3
→ Frontend: calls /api/notes/attachments/confirm
→ Backend: stores CDN URL in DB
→ Frontend: replaces placeholder with CDN image
```

### AI Refactor with Streaming
```
User clicks "Enhance Note"
→ POST /api/ai/refactor (returns jobId)
→ EventSource /api/ai/refactor/stream/{jobId}
→ Backend streams tokens via SSE
→ Frontend renders tokens progressively
→ Stream ends: full result stored in DB
→ User clicks Save → POST /api/ai/refactor/{jobId}/save
```

### Streak Update
```
User completes any activity (creates note, completes task, etc.)
→ Activity event fires in Spring Boot Service layer
→ ActivityLogService.record(userId, type, entityId, date)
→ INSERT into activity_log
→ Streak cache invalidated (Redis optional) or computed fresh
→ Profile page fetches updated streak
```

---

## 15. Security Considerations

| Concern | Mitigation |
|---------|-----------|
| Authentication | Clerk JWT on every request; JWKS verification in Spring Security |
| Authorization | Every DB query scoped by `user_id = clerkUserId` — no cross-user access |
| File upload | Pre-signed S3 URLs (15 min TTL), file type validation, size limits (50MB) |
| AI API keys | Stored as environment variables / AWS Secrets Manager — never exposed to client |
| SQL Injection | Spring Data JPA parameterized queries |
| XSS | TipTap/Lexical sanitizes HTML; Next.js CSP headers |
| Clerk webhooks | Svix signature validation before processing |
| Rate limiting | Spring Boot rate limiter on /api/ai/* (protect AI API costs) |
| CORS | Spring Boot CORS config: only allow frontend domain |
| Sensitive data | No AI API keys in frontend; all AI calls proxied through Spring Boot |

---

## 16. Deployment Architecture

```
┌──────────────────────────────────────────┐
│              PRODUCTION                   │
│                                          │
│  Vercel (Next.js)                        │
│  ├── Edge middleware (Clerk auth)         │
│  ├── Static assets on CDN                │
│  └── API routes (webhooks only)          │
│                │                         │
│                │ HTTPS                   │
│                ▼                         │
│  AWS EC2 / ECS (Spring Boot)             │
│  ├── Docker container                    │
│  ├── Auto-scaling group                  │
│  └── Behind AWS ALB (load balancer)      │
│                │                         │
│       ┌────────┼──────────┐              │
│       ▼        ▼          ▼              │
│  AWS RDS   AWS S3    Clerk API           │
│  (Postgres)(Files)  (Auth)               │
│                                          │
│  CloudFront CDN → S3 assets              │
│  AWS Secrets Manager → API keys          │
└──────────────────────────────────────────┘
```

### Environment Variables
```
# Spring Boot
CLERK_SECRET_KEY=sk_live_...
CLERK_JWKS_URL=https://...clerk.accounts.dev/.well-known/jwks.json
DATABASE_URL=jdbc:postgresql://...
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET_NAME=noteflow-assets-prod
CLOUDFRONT_DOMAIN=https://cdn.noteflow.app
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=AIza...

# Next.js
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_...
CLERK_SECRET_KEY=sk_live_...
NEXT_PUBLIC_API_BASE_URL=https://api.noteflow.app
```

---

## 17. Phase 1 Development Roadmap

### Sprint 1 (Weeks 1-2): Foundation
- [ ] Clerk auth integration (Next.js + Spring Boot)
- [ ] PostgreSQL schema setup + Spring Data JPA entities
- [ ] User sync via Clerk webhooks
- [ ] Basic note CRUD (no rich editor yet)
- [ ] S3 bucket setup + pre-signed URL service

### Sprint 2 (Weeks 3-4): Notes Feature
- [ ] TipTap rich text editor integration
- [ ] File/image upload with S3 pre-signed URLs
- [ ] Note search (PostgreSQL full-text)
- [ ] Folders, tags, pin, archive

### Sprint 3 (Weeks 5-6): AI Feature
- [ ] Gemini Flash + GPT-4o Mini service abstraction
- [ ] AI refactor endpoint with SSE streaming
- [ ] Side-by-side view in frontend
- [ ] Question extraction + Question Bank

### Sprint 4 (Weeks 7-8): Tasks & Calendar
- [ ] Task CRUD with priorities, subtasks
- [ ] Calendar view (month/week/day)
- [ ] Recurring tasks (RRULE)
- [ ] Routine builder

### Sprint 5 (Weeks 9-10): Interview & Voice
- [ ] Question search from notes
- [ ] Voice session UI (Web Speech API)
- [ ] AI evaluation endpoint
- [ ] Session history & scores

### Sprint 6 (Weeks 11-12): Profile, Streaks & Polish
- [ ] Activity log + streak calculation
- [ ] GitHub-style heatmap component
- [ ] Profile & settings pages
- [ ] End-to-end testing + deployment

---

*Document version: 1.0 | NoteFlow AI Phase 1 | June 2025*
