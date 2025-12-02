# Tasks: Song CMS Implementation

**Feature**: `002-song-cms`
**Generated**: 2024-12-02
**Source**: `plan.md`

---

## Task Format

`[ID] [P?] [Story] Description`

---

## Phase 1: Setup & Database

- [ ] `[1.1]` Add Song, Artist, Category, AdminAuditLog models to Prisma schema
- [ ] `[1.2]` Run database migration
- [ ] `[1.3]` Seed test data (artists, categories)
- [ ] `[1.4]` Configure R2 environment variables

**Checkpoint**: Database ready with schema and seed data

---

## Phase 2: Utils Package (Foundational)

- [ ] `[2.1] [P]` Write tests for LRC parser (`lrc-parser.test.ts`)
- [ ] `[2.2]` Implement LRC parser (`lrc-parser.ts`)
- [ ] `[2.3] [P]` Implement slug generator (`slug-generator.ts`)
- [ ] `[2.4]` Export from utils package index

**Checkpoint**: LRC parser works with real LRC files

---

## Phase 3: Admin API - Upload (US1)

- [ ] `[3.1]` Implement R2 client configuration
- [ ] `[3.2]` Implement `admin.upload.getUrls` procedure (presigned URLs)
- [ ] `[3.3]` Write contract tests for upload procedure

**Checkpoint**: Can generate valid presigned URLs for R2

---

## Phase 4: Admin API - Songs (US1, US2, US3)

- [ ] `[4.1] [P]` Implement `admin.song.create` procedure
- [ ] `[4.2] [P]` Implement `admin.song.update` procedure
- [ ] `[4.3] [P]` Implement `admin.song.delete` procedure
- [ ] `[4.4] [P]` Implement `admin.song.get` procedure
- [ ] `[4.5] [P]` Implement `admin.song.list` procedure (with filters)
- [ ] `[4.6]` Implement `admin.song.publish` procedure
- [ ] `[4.7]` Write contract tests for song CRUD

**Checkpoint**: Song API complete with tests passing

---

## Phase 5: Admin API - Artists & Categories (US4, US5)

- [ ] `[5.1] [P]` Implement `admin.artist.create` procedure
- [ ] `[5.2] [P]` Implement `admin.artist.update` procedure
- [ ] `[5.3] [P]` Implement `admin.artist.list` procedure
- [ ] `[5.4] [P]` Implement `admin.category.create` procedure
- [ ] `[5.5] [P]` Implement `admin.category.update` procedure
- [ ] `[5.6] [P]` Implement `admin.category.list` procedure
- [ ] `[5.7]` Implement `admin.category.reorder` procedure
- [ ] `[5.8]` Write contract tests for artist/category

**Checkpoint**: All admin APIs complete

---

## Phase 6: Admin UI - Layout & Auth

- [ ] `[6.1]` Create admin layout with sidebar (`layout.tsx`)
- [ ] `[6.2]` Implement admin auth guard (role check)
- [ ] `[6.3]` Create AdminSidebar component
- [ ] `[6.4]` Create AdminHeader component
- [ ] `[6.5]` Set up admin tRPC client

**Checkpoint**: Admin shell working with auth protection

---

## Phase 7: Admin UI - Uploaders (US1)

- [ ] `[7.1] [P]` Implement AudioUploader component (drag-drop, progress)
- [ ] `[7.2] [P]` Implement LrcUploader component (parse & preview)
- [ ] `[7.3] [P]` Implement ImageUploader component
- [ ] `[7.4]` Test uploaders with R2 integration

**Checkpoint**: File upload working end-to-end

---

## Phase 8: Admin UI - Forms (US1, US4, US5)

- [ ] `[8.1]` Implement SongForm component
- [ ] `[8.2]` Integrate uploaders into SongForm
- [ ] `[8.3] [P]` Implement ArtistForm component
- [ ] `[8.4] [P]` Implement CategoryForm component

**Checkpoint**: All forms functional

---

## Phase 9: Admin UI - Pages (US1, US2, US3, US6)

### Songs

- [ ] `[9.1]` Implement songs list page with DataTable
- [ ] `[9.2]` Add search and filter controls
- [ ] `[9.3]` Implement create song page
- [ ] `[9.4]` Implement song detail page with preview
- [ ] `[9.5]` Implement edit song page
- [ ] `[9.6]` Add publish/unpublish actions

### Artists

- [ ] `[9.7]` Implement artists list page
- [ ] `[9.8]` Implement create/edit artist modal

### Categories

- [ ] `[9.9]` Implement categories list page
- [ ] `[9.10]` Implement drag-to-reorder functionality

**Checkpoint**: All admin pages functional

---

## Phase 10: Polish

- [ ] `[10.1]` Add loading states to all forms
- [ ] `[10.2]` Add error handling with toast notifications
- [ ] `[10.3]` Implement audit logging for all admin actions
- [ ] `[10.4]` Add confirmation dialogs for destructive actions
- [ ] `[10.5]` Optimize list page with pagination
- [ ] `[10.6]` Test responsive layout on tablet

**Checkpoint**: Production-ready admin panel

---

## Parallel Task Groups

**Group A** (Utils - no dependencies):
- `[2.1]`, `[2.3]`

**Group B** (Song API - after schema):
- `[4.1]`, `[4.2]`, `[4.3]`, `[4.4]`, `[4.5]`

**Group C** (Artist/Category API - after schema):
- `[5.1]`, `[5.2]`, `[5.3]`, `[5.4]`, `[5.5]`, `[5.6]`

**Group D** (Uploaders - after auth):
- `[7.1]`, `[7.2]`, `[7.3]`

**Group E** (Forms - after uploaders):
- `[8.3]`, `[8.4]`

---

## Estimated Effort

| Phase | Tasks | Estimate |
|-------|-------|----------|
| Setup | 4 | 0.5 days |
| Utils | 4 | 0.5 days |
| Upload API | 3 | 0.5 days |
| Song API | 7 | 1 day |
| Artist/Category API | 8 | 1 day |
| Admin Layout | 5 | 0.5 days |
| Uploaders | 4 | 1 day |
| Forms | 4 | 1 day |
| Pages | 10 | 2 days |
| Polish | 6 | 1 day |

**Total**: ~9 days
