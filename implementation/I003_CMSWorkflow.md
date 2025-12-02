# I003: CMS Workflow Implementation

## Overview

Implementation notes for the content management workflow, covering song ingestion, audio upload, LRC parsing, and admin operations.

## Related Decisions

- D002: Database Schema
- D006: Audio Storage (Cloudflare R2)
- D011: Lyrics Format (LRC → JSON)

## Song Ingestion Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     SONG INGESTION FLOW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. ADMIN SELECTS FILES                                          │
│     ├── MP3 audio file (required)                                │
│     ├── LRC lyrics file (required)                               │
│     └── Cover image (optional)                                   │
│                              │                                   │
│                              ▼                                   │
│  2. CLIENT-SIDE VALIDATION                                       │
│     ├── File type check (audio/mpeg, text/plain)                │
│     ├── File size check (MP3 ≤ 50MB, LRC ≤ 1MB)                 │
│     └── Basic LRC format validation                              │
│                              │                                   │
│                              ▼                                   │
│  3. REQUEST PRESIGNED URLS                                       │
│     POST /api/trpc/admin.song.getUploadUrls                     │
│     Response: { audioUrl, lrcUrl, coverUrl }                    │
│                              │                                   │
│                              ▼                                   │
│  4. UPLOAD TO R2                                                 │
│     ├── PUT audioUrl → MP3 file                                 │
│     ├── PUT lrcUrl → LRC file                                   │
│     └── PUT coverUrl → Image file                               │
│                              │                                   │
│                              ▼                                   │
│  5. CREATE SONG RECORD                                           │
│     POST /api/trpc/admin.song.create                            │
│     ├── Parse LRC → JSON                                        │
│     ├── Extract audio duration                                   │
│     ├── Generate slug                                            │
│     └── Save to database                                         │
│                              │                                   │
│                              ▼                                   │
│  6. SONG CREATED (DRAFT)                                         │
│     Admin can preview, edit, then publish                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## File Upload Implementation

### Presigned URL Generation

```typescript
// packages/trpc/routers/admin/song.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'

const r2Client = new S3Client({
  region: 'auto',
  endpoint: process.env.CLOUDFLARE_R2_ENDPOINT,
  credentials: {
    accessKeyId: process.env.CLOUDFLARE_R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.CLOUDFLARE_R2_SECRET_ACCESS_KEY!,
  },
})

export const adminSongRouter = router({
  getUploadUrls: adminProcedure
    .input(z.object({
      songId: z.string().uuid(),
      hasAudio: z.boolean(),
      hasLrc: z.boolean(),
      hasCover: z.boolean(),
    }))
    .mutation(async ({ input }) => {
      const { songId, hasAudio, hasLrc, hasCover } = input
      const urls: Record<string, string> = {}

      if (hasAudio) {
        urls.audioUrl = await getSignedUrl(
          r2Client,
          new PutObjectCommand({
            Bucket: process.env.CLOUDFLARE_R2_BUCKET,
            Key: `audio/${songId}/audio.mp3`,
            ContentType: 'audio/mpeg',
          }),
          { expiresIn: 3600 }
        )
      }

      if (hasLrc) {
        urls.lrcUrl = await getSignedUrl(
          r2Client,
          new PutObjectCommand({
            Bucket: process.env.CLOUDFLARE_R2_BUCKET,
            Key: `lyrics/${songId}/lyrics.lrc`,
            ContentType: 'text/plain',
          }),
          { expiresIn: 3600 }
        )
      }

      if (hasCover) {
        urls.coverUrl = await getSignedUrl(
          r2Client,
          new PutObjectCommand({
            Bucket: process.env.CLOUDFLARE_R2_BUCKET,
            Key: `covers/${songId}/cover.jpg`,
            ContentType: 'image/jpeg',
          }),
          { expiresIn: 3600 }
        )
      }

      return urls
    }),
})
```

### Client-Side Upload Component

```typescript
// apps/admin/components/AudioUploader.tsx
'use client'

import { useState, useCallback } from 'react'
import { useDropzone } from 'react-dropzone'
import { trpc } from '@/lib/admin-trpc'

interface AudioUploaderProps {
  songId: string
  onUploadComplete: (url: string) => void
}

export function AudioUploader({ songId, onUploadComplete }: AudioUploaderProps) {
  const [progress, setProgress] = useState(0)
  const [uploading, setUploading] = useState(false)

  const getUploadUrls = trpc.admin.song.getUploadUrls.useMutation()

  const onDrop = useCallback(async (acceptedFiles: File[]) => {
    const file = acceptedFiles[0]
    if (!file) return

    setUploading(true)
    setProgress(0)

    try {
      // Get presigned URL
      const { audioUrl } = await getUploadUrls.mutateAsync({
        songId,
        hasAudio: true,
        hasLrc: false,
        hasCover: false,
      })

      // Upload to R2
      const xhr = new XMLHttpRequest()
      xhr.upload.addEventListener('progress', (e) => {
        if (e.lengthComputable) {
          setProgress(Math.round((e.loaded / e.total) * 100))
        }
      })

      await new Promise((resolve, reject) => {
        xhr.onload = () => resolve(xhr.response)
        xhr.onerror = () => reject(new Error('Upload failed'))
        xhr.open('PUT', audioUrl)
        xhr.setRequestHeader('Content-Type', 'audio/mpeg')
        xhr.send(file)
      })

      // Return the public URL
      const publicUrl = `${process.env.NEXT_PUBLIC_R2_PUBLIC_URL}/audio/${songId}/audio.mp3`
      onUploadComplete(publicUrl)
    } catch (error) {
      console.error('Upload error:', error)
    } finally {
      setUploading(false)
    }
  }, [songId, getUploadUrls, onUploadComplete])

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: { 'audio/mpeg': ['.mp3'] },
    maxSize: 50 * 1024 * 1024, // 50MB
    multiple: false,
  })

  return (
    <div
      {...getRootProps()}
      className={`border-2 border-dashed rounded-lg p-8 text-center cursor-pointer
        ${isDragActive ? 'border-primary bg-primary/10' : 'border-gray-300'}
        ${uploading ? 'pointer-events-none opacity-50' : ''}`}
    >
      <input {...getInputProps()} />
      {uploading ? (
        <div>
          <p>Uploading... {progress}%</p>
          <div className="w-full bg-gray-200 rounded-full h-2 mt-2">
            <div
              className="bg-primary h-2 rounded-full transition-all"
              style={{ width: `${progress}%` }}
            />
          </div>
        </div>
      ) : isDragActive ? (
        <p>Drop the MP3 file here...</p>
      ) : (
        <p>Drag & drop an MP3 file, or click to select</p>
      )}
    </div>
  )
}
```

## LRC Parser Implementation

```typescript
// packages/utils/lrc-parser.ts

export interface LyricsLine {
  t: number  // Time in seconds
  l: string  // Lyric text
}

export interface ParseResult {
  lyrics: LyricsLine[]
  metadata: Record<string, string>
  errors: string[]
}

export function parseLrc(lrc: string): ParseResult {
  const lines = lrc.split(/\r?\n/)
  const lyrics: LyricsLine[] = []
  const metadata: Record<string, string> = {}
  const errors: string[] = []

  const timeRegex = /\[(\d+):(\d+)(?:\.(\d+))?\]/g
  const metaRegex = /^\[([a-z]+):(.+)\]$/i

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i].trim()
    if (!line) continue

    // Check for metadata (e.g., [ti:Title], [ar:Artist])
    const metaMatch = line.match(metaRegex)
    if (metaMatch && !line.match(timeRegex)) {
      metadata[metaMatch[1].toLowerCase()] = metaMatch[2].trim()
      continue
    }

    // Parse timestamps
    const timestamps: number[] = []
    let match: RegExpExecArray | null

    while ((match = timeRegex.exec(line)) !== null) {
      const minutes = parseInt(match[1], 10)
      const seconds = parseInt(match[2], 10)
      const ms = match[3]
        ? parseInt(match[3].padEnd(3, '0'), 10)
        : 0

      const time = minutes * 60 + seconds + ms / 1000
      timestamps.push(time)
    }

    // Get lyric text (everything after timestamps)
    const text = line.replace(timeRegex, '').trim()

    // Create lyric entries for each timestamp
    for (const t of timestamps) {
      if (t < 0) {
        errors.push(`Line ${i + 1}: Invalid negative timestamp`)
        continue
      }
      lyrics.push({ t, l: text })
    }
  }

  // Sort by time
  lyrics.sort((a, b) => a.t - b.t)

  // Validate for gaps or overlaps
  for (let i = 1; i < lyrics.length; i++) {
    if (lyrics[i].t === lyrics[i - 1].t && lyrics[i].l !== lyrics[i - 1].l) {
      errors.push(`Duplicate timestamp at ${lyrics[i].t}s`)
    }
  }

  return { lyrics, metadata, errors }
}

// Validate LRC file before upload
export function validateLrc(lrc: string): { valid: boolean; errors: string[] } {
  const result = parseLrc(lrc)

  const errors: string[] = [...result.errors]

  if (result.lyrics.length === 0) {
    errors.push('No lyrics found in LRC file')
  }

  if (result.lyrics.length < 5) {
    errors.push('Too few lyrics lines (minimum 5)')
  }

  return {
    valid: errors.length === 0,
    errors,
  }
}
```

## Song Creation Flow

```typescript
// packages/trpc/routers/admin/song.ts

const CreateSongInput = z.object({
  title: z.string().min(1).max(200),
  artistId: z.string().uuid(),
  categoryId: z.string().uuid(),
  audioUrl: z.string().url(),
  lrcContent: z.string().min(1),
  coverUrl: z.string().url().optional(),
  language: z.string().optional(),
})

export const adminSongRouter = router({
  create: adminProcedure
    .input(CreateSongInput)
    .mutation(async ({ input, ctx }) => {
      const { title, artistId, categoryId, audioUrl, lrcContent, coverUrl, language } = input

      // Parse LRC
      const { lyrics, errors } = parseLrc(lrcContent)
      if (errors.length > 0) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: `LRC parse errors: ${errors.join(', ')}`,
        })
      }

      // Extract duration (from last lyric timestamp + buffer)
      const durationSec = lyrics.length > 0
        ? Math.ceil(lyrics[lyrics.length - 1].t + 30)
        : 0

      // Generate slug
      const slug = generateSlug(title)

      // Check slug uniqueness
      const existing = await ctx.prisma.song.findUnique({ where: { slug } })
      if (existing) {
        throw new TRPCError({
          code: 'CONFLICT',
          message: 'A song with this title already exists',
        })
      }

      // Create song
      const song = await ctx.prisma.song.create({
        data: {
          title,
          slug,
          artistId,
          categoryId,
          audioUrl,
          lyricsJson: lyrics,
          coverUrl,
          durationSec,
          language,
          isPublished: false, // Draft by default
        },
        include: { artist: true, category: true },
      })

      // Log admin action
      await ctx.prisma.adminAuditLog.create({
        data: {
          adminId: ctx.user.id,
          action: 'song.create',
          targetType: 'song',
          targetId: song.id,
          details: { title },
        },
      })

      return song
    }),

  publish: adminProcedure
    .input(z.object({ id: z.string().uuid(), publish: z.boolean() }))
    .mutation(async ({ input, ctx }) => {
      const song = await ctx.prisma.song.update({
        where: { id: input.id },
        data: {
          isPublished: input.publish,
          publishedAt: input.publish ? new Date() : null,
        },
      })

      await ctx.prisma.adminAuditLog.create({
        data: {
          adminId: ctx.user.id,
          action: input.publish ? 'song.publish' : 'song.unpublish',
          targetType: 'song',
          targetId: song.id,
        },
      })

      return song
    }),
})
```

## Admin Song Form Component

```typescript
// apps/admin/components/forms/SongForm.tsx
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { AudioUploader } from '../AudioUploader'
import { LrcUploader } from '../LrcUploader'
import { trpc } from '@/lib/admin-trpc'

const formSchema = z.object({
  title: z.string().min(1, 'Title is required'),
  artistId: z.string().uuid('Select an artist'),
  categoryId: z.string().uuid('Select a category'),
  audioUrl: z.string().url('Upload audio file'),
  lrcContent: z.string().min(1, 'Upload LRC file'),
  language: z.string().optional(),
})

type FormValues = z.infer<typeof formSchema>

export function SongForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
  })

  const createSong = trpc.admin.song.create.useMutation()
  const { data: artists } = trpc.admin.artist.list.useQuery()
  const { data: categories } = trpc.admin.category.list.useQuery()

  const onSubmit = async (data: FormValues) => {
    try {
      await createSong.mutateAsync(data)
      // Redirect to song list or show success
    } catch (error) {
      // Handle error
    }
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
      {/* Title */}
      <div>
        <label>Title</label>
        <input {...form.register('title')} />
        {form.formState.errors.title && (
          <p className="text-red-500">{form.formState.errors.title.message}</p>
        )}
      </div>

      {/* Artist select */}
      <div>
        <label>Artist</label>
        <select {...form.register('artistId')}>
          <option value="">Select artist...</option>
          {artists?.map((a) => (
            <option key={a.id} value={a.id}>{a.name}</option>
          ))}
        </select>
      </div>

      {/* Category select */}
      <div>
        <label>Category</label>
        <select {...form.register('categoryId')}>
          <option value="">Select category...</option>
          {categories?.map((c) => (
            <option key={c.id} value={c.id}>{c.name}</option>
          ))}
        </select>
      </div>

      {/* Audio upload */}
      <div>
        <label>Audio File (MP3)</label>
        <AudioUploader
          songId={form.watch('title') ? generateTempId() : ''}
          onUploadComplete={(url) => form.setValue('audioUrl', url)}
        />
      </div>

      {/* LRC upload */}
      <div>
        <label>Lyrics File (LRC)</label>
        <LrcUploader
          onParsed={(content) => form.setValue('lrcContent', content)}
        />
      </div>

      <button type="submit" disabled={createSong.isPending}>
        {createSong.isPending ? 'Creating...' : 'Create Song'}
      </button>
    </form>
  )
}
```

## Checklist

- [ ] Implement R2 presigned URL generation
- [ ] Create file upload components
- [ ] Implement LRC parser with validation
- [ ] Create admin song CRUD routes
- [ ] Implement song form with validation
- [ ] Add audit logging
- [ ] Create lyrics editor component
- [ ] Test upload flow end-to-end
