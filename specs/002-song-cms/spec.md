# Feature Specification: Song Content Management System

**Branch**: `002-song-cms`
**Created**: 2024-12-02
**Status**: Ready for Planning

---

## Summary

Enable administrators to upload, manage, and publish songs with their associated audio files, lyrics (LRC format), and metadata. This is the foundation for populating the karaoke catalog.

---

## User Scenarios & Testing

### US1: Upload New Song (P1 - MVP Critical)

**Context**: Admins need to add new songs to the catalog with all required assets.

**Story**: As an admin, I want to upload a song with its audio file, lyrics, and metadata so that users can access it for karaoke.

**Acceptance Scenarios**:

```gherkin
Scenario: Upload song with all required files
  Given I am logged in as an admin
  And I am on the "Add Song" page
  When I fill in the song title "បទសំណព្វចិត្ត"
  And I select an artist from the dropdown
  And I select a category from the dropdown
  And I upload an MP3 file (size < 50MB)
  And I upload an LRC lyrics file
  And I click "Create Song"
  Then the song should be created in draft status
  And I should see a success message
  And I should be redirected to the song detail page

Scenario: Validate LRC file format
  Given I am uploading a new song
  When I upload an LRC file
  Then the system should parse and validate the LRC content
  And display a preview of lyrics with timestamps
  And show any parsing errors or warnings

Scenario: Upload optional cover image
  Given I am creating a new song
  When I upload a cover image (JPEG/PNG, < 5MB)
  Then the image should be stored
  And a thumbnail should be generated
  And the cover should display on the song card

Scenario: Reject invalid files
  Given I am uploading song assets
  When I upload an invalid file type (e.g., .exe, .doc)
  Then the upload should be rejected
  And an error message should explain allowed file types
```

**Edge Cases**:
- LRC file with invalid timestamps
- LRC file with encoding issues (UTF-8 vs other)
- Very large MP3 file (>50MB)
- Duplicate song title for same artist
- Network interruption during upload

---

### US2: Preview Song Before Publishing (P1 - MVP Critical)

**Context**: Admins need to verify all song data is correct before making it public.

**Story**: As an admin, I want to preview a song in draft status so that I can verify everything is correct before publishing.

**Acceptance Scenarios**:

```gherkin
Scenario: Preview lyrics synchronization
  Given a song is in draft status
  When I click "Preview"
  Then I should see the lyrics panel
  And I should be able to play the audio
  And lyrics should highlight in sync with audio

Scenario: Edit song details before publishing
  Given I am previewing a draft song
  When I notice the artist is incorrect
  And I click "Edit"
  Then I should be able to modify any field
  And save changes without publishing

Scenario: Replace audio file
  Given a draft song has an audio file
  When I upload a new MP3 file
  Then the old file should be replaced
  And the song duration should update automatically
```

**Edge Cases**:
- Audio file corrupt or unplayable
- Lyrics timestamps don't match audio duration
- Preview on mobile admin interface

---

### US3: Publish/Unpublish Song (P1 - MVP Critical)

**Context**: Admins control which songs are visible to users.

**Story**: As an admin, I want to publish or unpublish songs so that I can control what users see.

**Acceptance Scenarios**:

```gherkin
Scenario: Publish a draft song
  Given a song is in draft status
  And all required fields are complete (title, artist, audio, lyrics)
  When I click "Publish"
  Then the song should become visible to users
  And the published timestamp should be recorded
  And the song should appear in browse/search

Scenario: Unpublish a live song
  Given a song is published
  When I click "Unpublish"
  Then the song should be hidden from users
  And existing direct links should show "Song unavailable"
  And the song should remain in admin list as draft

Scenario: Cannot publish incomplete song
  Given a song is missing lyrics
  When I try to publish
  Then I should see an error "Cannot publish: Missing lyrics"
  And the song should remain in draft status
```

**Edge Cases**:
- Publishing while user is mid-karaoke session
- Bulk publish multiple songs
- Publish scheduled for future date (future feature)

---

### US4: Manage Artists (P2 - Supporting)

**Context**: Songs require artist associations; admins need to manage the artist catalog.

**Story**: As an admin, I want to create and manage artists so that songs can be properly attributed.

**Acceptance Scenarios**:

```gherkin
Scenario: Create new artist
  Given I am on the Artists page
  When I click "Add Artist"
  And I fill in the name "ស៊ីន ស៊ីសាមុត"
  And I optionally add a bio and image
  And I click "Save"
  Then the artist should be created
  And be available in song artist dropdown

Scenario: Edit existing artist
  Given an artist exists in the system
  When I edit their name or details
  Then all associated songs should reflect the change
  And the artist's page should update

Scenario: View artist's songs
  Given an artist has multiple songs
  When I view the artist detail page
  Then I should see a list of all their songs
  And the song count should be accurate
```

**Edge Cases**:
- Artist name with special characters
- Duplicate artist names
- Deleting artist with existing songs (should prevent or reassign)

---

### US5: Manage Categories (P2 - Supporting)

**Context**: Categories help users browse and discover songs.

**Story**: As an admin, I want to create and manage categories so that songs can be organized.

**Acceptance Scenarios**:

```gherkin
Scenario: Create new category
  Given I am on the Categories page
  When I click "Add Category"
  And I enter name "ចំរៀងអូន" (Classic)
  And I click "Save"
  Then the category should be created
  And be available in song category dropdown

Scenario: Reorder categories
  Given multiple categories exist
  When I drag to reorder them
  Then the display order should update
  And the new order should reflect in the user-facing browse page
```

**Edge Cases**:
- Category with no songs
- Deleting category with existing songs

---

### US6: Search and Filter Songs (P2 - Supporting)

**Context**: As the catalog grows, admins need efficient ways to find songs.

**Story**: As an admin, I want to search and filter the song list so that I can quickly find specific songs.

**Acceptance Scenarios**:

```gherkin
Scenario: Search by title
  Given multiple songs exist
  When I type "love" in the search box
  Then I should see songs with "love" in the title
  And results should update as I type (debounced)

Scenario: Filter by status
  Given songs exist in draft and published states
  When I filter by "Draft"
  Then only draft songs should be displayed

Scenario: Filter by category
  Given songs exist in multiple categories
  When I filter by "Pop"
  Then only songs in Pop category should be shown

Scenario: Combine filters
  Given I have filtered by "Draft"
  When I also filter by category "Rock"
  Then only draft Rock songs should appear
```

---

## Requirements

### Functional Requirements

| ID | Requirement | User Story |
|----|-------------|------------|
| FR1 | Upload MP3 audio files (≤50MB) | US1 |
| FR2 | Upload and parse LRC lyrics files | US1 |
| FR3 | Upload cover images (JPEG/PNG, ≤5MB) | US1 |
| FR4 | Generate presigned URLs for R2 uploads | US1 |
| FR5 | Validate LRC format and display preview | US1 |
| FR6 | Create song records in draft status | US1 |
| FR7 | Preview song with lyrics sync | US2 |
| FR8 | Edit song metadata and replace files | US2 |
| FR9 | Publish/unpublish songs | US3 |
| FR10 | CRUD operations for artists | US4 |
| FR11 | CRUD operations for categories | US5 |
| FR12 | Search songs by title/artist | US6 |
| FR13 | Filter songs by status/category | US6 |
| FR14 | Audit log for admin actions | All |

### Non-Functional Requirements

| ID | Requirement | Metric |
|----|-------------|--------|
| NFR1 | File upload progress indication | Show % complete |
| NFR2 | Upload timeout | 5 minutes max per file |
| NFR3 | Admin page load time | ≤ 2 seconds |
| NFR4 | Search response time | ≤ 500ms |
| NFR5 | Support 10,000+ songs | Pagination required |

### Key Entities

| Entity | Description |
|--------|-------------|
| Song | Core entity with title, slug, audio URL, lyrics JSON |
| Artist | Name, bio, image, song count |
| Category | Name, display order, song count |
| AdminAuditLog | Admin ID, action, target, timestamp |

---

## Success Criteria

| Metric | Target | Measurement |
|--------|--------|-------------|
| Song upload success rate | ≥ 99% | Error logs |
| Average upload time (30MB file) | ≤ 60 seconds | Performance monitoring |
| LRC parse success rate | ≥ 95% | Error logs |
| Admin task completion time | ≤ 5 minutes | User feedback |

---

## Requirement Completeness Checklist

- [x] All user stories have acceptance scenarios
- [x] Each user story is independently testable
- [x] Success criteria are measurable
- [x] Edge cases are documented
- [x] Non-functional requirements have specific metrics
- [x] All `[NEEDS CLARIFICATION]` items resolved

---

## Open Questions

None at this time.
