# System Architecture — 5D Planner: Community Collaboration Hub

## Goals
- Allow users to publish and discover designs (images, 2D/3D scene references).
- Provide structured feedback and mentorship flows.
- Support real-time co-design (collaboration) and asynchronous collaboration.
- Scale for growing user engagement and media storage.

---

## High-level architecture (summary)
- **Frontend:** React (web) + React Three Fiber for 3D previews. Mobile clients can be React Native or Flutter.
- **Backend / API:** Node.js + Express (REST + WebSocket endpoints).
- **Real-time layer:** Socket.IO (WebSockets) for co-design sessions; WebRTC for peer-to-peer live editing if low latency is required.
- **Database:** PostgreSQL for relational data (users, posts, comments, challenges); Redis for caching, session storage and pub/sub.
- **Storage:** AWS S3 for design files, renders, and thumbnails.
- **Auth & Security:** OAuth2 / JWT for API authentication; role-based access (user, mentor, moderator).
- **Search & Discovery:** ElasticSearch (or Postgres full-text + indexes) for feed, tags, and search.
- **Background jobs:** Worker queue (Bull / Redis) for heavy tasks — renders, image thumbnails, challenge scoring.
- **Monitoring:** Prometheus + Grafana (or third-party like Datadog).

---

## Data model (core tables)
- **users**: id, name, email, role, avatar, bio, created_at
- **designs**: id, user_id, title, description, scene_reference (link to S3 / scene json), preview_image, visibility (public/private), tags, created_at
- **comments**: id, design_id, user_id, parent_comment_id, content, rating, created_at
- **collaborations**: id, design_id, user_id, role, permissions, last_active
- **challenges**: id, title, description, rules, start_at, end_at, submissions_count
- **submissions**: id, challenge_id, design_id, votes, created_at
- **mentorship_sessions**: id, mentor_id, user_id, scheduled_at, notes_link

---

## Component interactions (sequence)
1. User posts a design → frontend uploads scene file/preview to S3 → frontend POST /designs with S3 links → backend stores metadata in PostgreSQL.
2. Another user comments → frontend POST /designs/:id/comments → backend stores comment → backend emits WebSocket event to design owner/subscribers.
3. Co-design session start → one user creates session → backend creates session record and token → other users join via Socket.IO or WebRTC to sync edits (patch messages).
4. Challenge flow → backend creates challenge → users submit designs → worker triggers thumbnail generation and scoring → results published in feed.

---

## Real-time collaboration design
- Use Socket.IO rooms per `collaboration_session:{session_id}`.
- Messages: `scene_patch` (delta updates), `chat_message`, `presence_update`, `save_snapshot`.
- To keep consistency: majority of state stored in server snapshots + periodic autosave to S3 and DB. For conflict resolution use Operational Transforms (OT) for simple vector transforms or CRDTs for complex document merging.
- For initial MVP: allow co-editing of placements / comments only (not full parametric edits). This reduces conflict complexity but enables the social experience.

---

## API endpoints (examples)
- `POST /api/v1/designs` — create design (metadata + S3 URLs)
- `GET /api/v1/designs/:id` — get design + comments
- `POST /api/v1/designs/:id/comments` — create comment / feedback
- `POST /api/v1/collab/sessions` — create co-design session
- `WS /socket` — real-time edits / presence

---

## Feasibility & trade-offs
- **Why feasible:** The stack (React + Node + Postgres + S3 + Socket.IO) is common, widely supported, and possible for a small team. Third-party services (Firebase, Supabase, or Pusher) can accelerate prototyping.
- **Performance:** Heavy media and 3D assets push cost to S3 + CDN. Use lazy loading of 3D assets and cached thumbnails to keep UX fast.
- **Complexity:** Full OT/CRDT is hard — start with limited collaborative primitives (comments, presence, synchronized camera) and evolve to full scene syncing.
- **Security & Moderation:** Include rate limits, spam filters, and moderation tools for reports and content takedowns.

---

## Scalability & future enhancements
- Move real-time sessions to a dedicated cluster (k8s) and use Redis for cross-instance pub/sub.
- Add analytics events for measuring retention uplift from community interactions.
- Add a recommendation engine for feed personalization (collaborative filtering using user activity).

---

## Diagram (Mermaid)
```mermaid
flowchart LR
  A[React Frontend] -->|REST API| B(Node/Express API)
  A -->|Socket.IO| C(Real-time Layer)
  B --> D[PostgreSQL]
  B --> E[AWS S3]
  B --> F[Redis]
  B --> G[Worker Queue]
  G --> E
  C --> F
  B --> H[ElasticSearch]
  H --> A
