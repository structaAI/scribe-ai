# ScribeAI Architecture Plan

## 1. Product Overview
- **Goal**: Capture live meeting audio (mic or shared tab), stream it to Google Gemini for low-latency transcription, store full sessions in Postgres, and deliver structured summaries plus exports.
- **Primary Users**: Knowledge workers who need automated meeting notes and searchable transcripts.
- **Experience Flow**: Authenticate → choose audio source → record/monitor live transcript → pause/resume as needed → stop to trigger post-processing summary → review/download session history.

## 2. Functional Requirements
1. Authentication via Better Auth (email/password + OAuth-ready) with session persistence in Next.js App Router.
2. Audio capture options:
   - `getUserMedia` microphone.
   - `getDisplayMedia` with system audio for Meet/Zoom tab sharing.
3. Chunked streaming (<=30s PCM/Opus blobs) delivered to the backend through WebSockets (Socket.io) for transcription and reliability acknowledgements.
4. Gemini-based transcription (streaming endpoint) with incremental updates broadcast to clients.
5. Post-processing: summary, action items, decisions using a tailored Gemini prompt after session stop.
6. Session history UI with filtering, search, and export (txt/json).
7. Resilience: auto-reconnect, buffering, and backpressure handling for >1 hr sessions.

## 3. High-Level Architecture
```mermaid
flowchart LR
    subgraph Client (Next.js App Router)
        UI[React + XState Recorder UI] --> WS_CLIENT[(Socket.io Client)]
        UI --> MediaRecorder[MediaRecorder Worker]
    end

    subgraph Edge/Backend
        WS_CLIENT -->|Opus Chunks| WS_SERVER[(Socket.io Gateway)]
        WS_SERVER --> ChunkQueue[(Redis Stream / In-memory Queue)]
        ChunkQueue --> Transcriber[Gemini Stream Worker]
        Transcriber -->|Partial drafts| WS_SERVER
        Transcriber --> Postgres[(Prisma ORM)]
        Postgres --> SummaryWorker[Post-processing Worker]
        SummaryWorker --> WS_SERVER
        WS_SERVER --> UI
    end
```

## 4. Key Components
- **Next.js App Router (frontend)**: Dashboard (session list) and recorder page. Uses Server Actions + Route Handlers for RESTful operations and `socket.io-client` for streams. Tailwind + shadcn for UI.
- **Recorder State Machine**: XState chart controlling states (`idle` → `permission` → `recording` → `paused` → `processing` → `completed`). Handles reconnection/resume logic.
- **Media Worker**: Dedicated Web Worker that uses `MediaRecorder` to chunk audio and push onto a queue with backpressure control before emitting via WebSocket (ack-based).
- **Node Socket Gateway**: Separate server (co-located with Next.js or standalone) managing namespaces per session, auth tokens, and binary chunk ingestion.
- **Processing Workers**: Use BullMQ / simple queue to decouple ingestion from Gemini calls. Ensures long-session stability and retries on Gemini errors.
- **Prisma/Postgres**: Stores users, sessions, transcript segments, and summaries (schema draft in section 5).

## 5. Data Modeling (Prisma Draft)
```prisma
model User {
  id            String   @id @default(cuid())
  email         String   @unique
  authProvider  String
  sessions      Session[]
  createdAt     DateTime @default(now())
}

model Session {
  id            String   @id @default(cuid())
  userId        String
  title         String
  status        SessionStatus @default(RECORDING)
  startedAt     DateTime @default(now())
  endedAt       DateTime?
  transcriptUrl String?
  summaryId     String?
  source        SessionSource
  durationMs    Int?
  user          User     @relation(fields: [userId], references: [id])
  segments      TranscriptSegment[]
}

model TranscriptSegment {
  id          String   @id @default(cuid())
  sessionId   String
  sequence    Int
  speakerTag  String?
  text        String
  startedAtMs Int
  endedAtMs   Int
  session     Session @relation(fields: [sessionId], references: [id])
}

model Summary {
  id         String   @id @default(cuid())
  sessionId  String   @unique
  overview   String
  keyPoints  Json
  actionItems Json
  decisions  Json
  session    Session  @relation(fields: [sessionId], references: [id])
}

enum SessionStatus {
  RECORDING
  PAUSED
  PROCESSING
  COMPLETED
  FAILED
}

enum SessionSource {
  MICROPHONE
  TAB
}
```

## 6. Streaming & Resilience Strategy
1. **Chunking**: 30s max chunks @ 16kHz mono PCM (or Opus). Each chunk includes metadata (`sequence`, `timestamp`, `sessionId`) and is compressed before sending.
2. **Acknowledgements**: Gateway emits `chunk:accepted` to advance client queue; otherwise the client retries the chunk (with exponential backoff).
3. **Reconnect**: If the device sleeps or network drops, UI enters `reconnecting`, caches pending chunks, and re-authenticates before resuming.
4. **Long Sessions**: Periodic checkpoints stored in Postgres every N chunks for quick recovery and to bound memory.
5. **Multi-speaker diarization**: Gemini prompt includes speaker tags seeded from VAD (Voice Activity Detection) hints. Optional WebRTC insertable streams for per-speaker metadata.

## 7. Prompt Engineering for Gemini
- **Transcription prompt**: Supply context on meeting type, expected jargon, and diarization hints. Example system prompt snippet:
  > "You are ScribeAI. Produce partial transcripts in JSON {speaker, text, confidence}. Maintain running numbering and do not repeat earlier lines."
- **Summary prompt**: On stop, aggregate transcript text and send:
  > "Summarize this meeting with sections: Overview, Key Decisions, Action Items (with owners), Risks. Keep bullets concise."
- Use `safetySettings` to minimize hallucinations and `responseMimeType: application/json` for structured output.

## 8. Security & Privacy
- All socket connections require Better Auth JWT (short-lived) signed by Next.js route handler.
- Audio chunks stored only temporarily (S3-compatible bucket with lifecycle rules).
- Encrypt secrets with `.env.local` + Doppler/1Password for deployment.

## 9. Implementation Roadmap
1. **Infra setup**: Docker Compose (Postgres + Redis), Prisma init, Better Auth scaffolding.
2. **Core UI shell**: Dashboard, session cards, start modal.
3. **Recorder pipeline**: Media permissions, chunk queue, socket wiring, live transcript rendering.
4. **Backend workers**: Socket gateway, chunk persistence, Gemini stream integration.
5. **Post-processing**: Summary worker, exports, notifications.
6. **Polish & QA**: Offline scenarios, end-to-end tests, video walkthrough.

## 10. Testing Strategy
- Unit tests for utility libs (`lib/audio`, `lib/gemini`).
- Integration tests for route handlers using `supertest`.
- Cypress component tests for recorder UI and session list.
- Load testing with k6 for 1-hr session simulation (ensure <500ms latency per chunk).

This document should guide the step-by-step implementation of ScribeAI while keeping scalability and reliability front-of-mind.

