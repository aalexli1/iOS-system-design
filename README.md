# iOS System Design Interview Guide

This document outlines a reusable, high-leverage framework for tackling iOS system design interviews. It is intentionally structured, comprehensive, and optimized for a 60-minute interview covering architecture, state management, scalability, offline support, performance, and evolvability.

It's focused on UIKit but SwiftUI should modify the presentation layer only.

This is opinionated toward the MVVM-C architecture since I've found it to be the most scalable for large projects.

## Mindset & Framing

In an iOS system design interview, the real evaluation is not about memorizing patterns but demonstrating:
- Ownership of large-scale UIKit app architecture
- Ability to define the problem like a tech lead
- Comfort with state management, real-time data, offline-first constraints
- Understanding of performance, battery, and reliability considerations
- Clarity in trade-offs, incremental evolution, and testing strategies

Your goal is to steer the interview using this structure. In a perfect world where the interviewer asks no questions and you drive the entire time, this is what the script would look like:
1.	Clarify product & constraints
2.	Identify core flows & entities
3.	Present layered MVVM-C architecture
4.	Discuss state management
5.	Define API + real-time strategy
6.	Add offline-first model
7.	Address performance & battery
8.	Cover reliability & lifecycle handling
9.	End with evolution, modularity & testing strategy

However, they'll jump in at any time and probably dig deeper into one of these areas. Be ready to pivot.

## Clarifying Product & Constraints

Start by zooming out before you architect:

Business & User Context
- Who is the primary user?
- What problem are we solving?

Platform Context
- iOS only
- UIKit
- iOS version assumptions (e.g., iOS 16+)

Functional Scope
- Real-time interactions?
- Feed vs chat vs transactional UI?
- Creation vs consumption?

Non-Functional Requirements
- Latency expectations
- Concurrency / scale (e.g., high DAU, rapid event streams)
- Offline-first?
- Battery-sensitivity?

Interview Framing

State explicitly:

“I’ll focus on architecture, state management, API integration, offline behavior, performance, and how the system evolves over time.”

This positions you as the driver.

## Core User Flows & Domain Modeling

Identify 2-3 major flows:

Examples:
- Login → Home
- Feed browsing (UITableView/UICollectionView)
- Item details page
- Real-time chat/messages
- Compose/submit workflow

Define core entities:
- User
- FeedItem / Post
- Message
- Room / Channel

Relationships:
- User ↔ Rooms
- Room → Messages
- Post → Comments

Call out that these flows and entities drive module boundaries.

## High-Level iOS Architecture (MVVM-C)

Use a layered approach.

### Layers

Presentation Layer
- UIKit view controllers
- UIKit custom views
- Coordinators for navigation (MVVM-C)
- One coordinator per feature

View Model Layer
- Holds feature state
- Exposes reactive-ish outputs (Combine, RxSwift, or closures)
- Converts domain models → view models
- Processes user intents → triggers use cases

Domain / Use Case Layer
- Encapsulates business logic
- E.g., LoadFeedUseCase, SendMessageUseCase, RefreshProfileUseCase

Data Layer
- Repositories
- Interfaces hiding remote + local data sources
- Remote: URLSession / WebSocket
- Local: Core Data / SQLite / Realm

## State Management (MVVM)

State management is a major focus.

### Pattern Choice

Use MVVM with unidirectional data flow.

State Structure Example

```swift
struct FeedViewState {
    var items: [FeedItemViewModel]
    var isLoading: Bool
    var errorMessage: String?
    var isOffline: Bool
}
```

View Model Responsibilities
- Owns ViewState
- Emits state updates (CurrentValueSubject, PassthroughSubject, or custom observer)
- Handles user intents
- Delegates real work to use cases

View Controller Responsibilities
- Subscribes to VM outputs
- Applies changes to UIKit views
- Forwards user interactions as intents

### State Boundaries
- Local UI state → view controller (input text, focus)
- Feature state → view model
- Global state → session/service layer (auth, flags)

### Real-time State Handling
- View model subscribes to observeMessages() or observeFeedUpdates()
- Merge initial REST load + WebSocket events
- Use ID-based deduplication

⸻

## API Design & Data Contracts

### Repository Protocols

Repositories abstract the data sources.

Example:
```swift
protocol MessageRepository {
    func observeMessages(roomID: String) -> AnyPublisher<[Message], Error>
    func sendMessage(roomID: String, text: String) -> AnyPublisher<Void, Error>
}
```

### Remote Data Source
- URLSession/URLSessionWebSocketTask
- REST for initial fetches
- WebSocket/SSE for push updates

### Real-Time Data Strategy

- Maintain a single WebSocket service
- Reconnect on foregrounding with backoff
- Order messages via server timestamp / sequence number
- Use idempotent operations

### Data Model Structure
- DTOs → Domain Models → View Models
- Support API versioning & feature flags

## Offline-First Architecture

When offline constraints are introduced, be ready to talk about the following.

### Local Database as Primary Data Source

Recommended options:
- SwiftData (maybe more for SwiftUI though)
- Core Data (first-party but tied into Apple constraints)
- SQLite (most freedom)
- Realm

Repositories:
- All reads come from local DB first
- Remote refresh overlays new data

### Outbox / Sync Queue Pattern

Local queue of pending operations:
- SendMessageOp
- CreatePostOp

SyncManager:
- Uses NWPathMonitor (maybe there's a newer one now) for connectivity
- Flushes queue on connectivity restoration
- Retries with exponential backoff

### Optimistic UI
- UI immediately reflects user changes
- Local DB marks records as pending
- Pending UI states styled with partial opacity or some icon

### Conflict Resolution
- Simple: last-write-wins
- Complex: server authoritative or field-level reconciliation

### Background Sync
- Use BGProcessingTask for periodic sync
- Use background URLSession for uploads
- Respect system battery constraints

## Performance & Battery Considerations

Performance is one of the most important sections if it comes up.

### Main Thread Discipline
- Keep UI updates on main
- Move JSON decoding, image processing, DB querying off-main
- Use DispatchQueue.global(), OperationQueue, or structured concurrency

### Efficient Lists (UITableView / UICollectionView)
- Use diffable data sources
- Pre-calc heights when possible
- Use Compositional Layout for complex grids
- Use cell prefetching APIs

### Image Optimization
- NSCache or custom LRU cache
- Decode/resize off-main
- Lazy loading and preheating

### Network & Battery
- Prefer WebSockets or APNs push (there are limits to push) over polling
- Batch network requests
- Respect Low Power Mode
- Avoid excessive timers or constant background execution

### Instrumentation
- Instruments: Know the main ones and when to use them
  - Time Profiler
  - Allocations
  - Leaks
- os_signpost for tracing critical paths
- Logging for network, sync, UI

## Reliability & Failure Handling

### Error Handling
- Error banners / toast views
- Retry actions exposed at the view model level
- Detect stale data and recover gracefully

### Lifecycle Resilience
- Handle foreground/background transitions explicitly
- Pause heavy tasks on background transition
- Resume real-time streams on foreground
- Persist critical state before suspension

### Degraded Modes
- Use cached local data when offline
- Skeleton views for partial data
- Avoid blank screens or blocking errors

## Evolution, Modularity & Testing

I feel like this is more advanced and deals with packaging multiple features.

### Modularization

Organize into Swift packages or frameworks:
- FeedFeature
- ChatFeature
- ProfileFeature
- Networking
- Persistence
- DesignSystem

### Feature Flags & Remote Config
- Toggle UI or algorithmic experiments
- Decouple release frequency from feature rollout

### Testing Strategy

#### Unit Tests
- ViewModels: test state transitions with mocked repositories
- Use cases: test business rules
- Repositories: test with in-memory DB or mock network

#### UI Tests
- Use XCUITest for full flows (login, feed load, offline message sending)

#### Snapshot Tests
- Validate key screens & cell states

#### Integration Tests
- Smoke tests on real devices and ideally real backend.

## Unified Architecture Diagram (One Picture)

Use this as your whiteboard anchor. Start here, then zoom into the specific flow (Feed, Chat, Offline Edit, Upload).

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                                   UI                                          │
│  ViewControllers (screens)                                                    │
│  - user input, scroll/keyboard math, diffable apply                           │
│  - forwards intents + UIKit-derived facts                                     │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │ intents (tap, refresh, paginate, send)
                                ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                                ViewModels                                     │
│  - holds ViewState + draft state (compose)                                    │
│  - subscribes to domain streams (observeFeed/observeMessages/observeProfile)  │
│  - calls UseCases for commands (refresh, loadNext, send, save, retry)         │
└───────────────┬────────────────────────────────────────────────┬──────────────┘
                │ observe (queries)                              │ commands
                ▼                                                ▼
┌────────────────────────────────────────┐                 ┌───────────────────────────────────────┐
│         Domain Repos                   │                 │            UseCases                   │
│  observeX() -> Publisher               │                 │  - intent + policy: when/whether/order│
│  - map DB -> Domain                    │                 │  - rate limit, cancel, coalesce       │
│  - merge server + local                │                 │  - orchestrate repo calls             │
│  - for reads: GET via Network          │                 └──────────────┬────────────────────────┘
│  - for writes: persist + enqueue Outbox│                (calls repos)   │
└───────────────┬────────────────────────┘                                ▼
                │ DB read/write (transactions)            ┌──────────────────────────────┐
                ▼                                         │      Outbox Repository       │
┌───────────────────────────────────────────────────────────────────────────────┐
│                                Local DB                                       │
│  Tables/Entities:                                                             │
│   - feed_items (id, sortKey, serverUpdatedAt, syncStatus, localOverlays...)   │
│   - feed_metadata (feedID, nextCursor, hasMore, lastRefreshedAt)              │
│   - messages (tempID, serverID, deliveryState, serverTimestamp, lastError...) │
│   - outbox_ops (opID, type, entityID, payload, state, attemptCount, nextAt...)│
│   - profile/settings drafts + sync fields                                     │
│  DB Observations drive UI updates (single source of truth)                    │
└───────────────┬───────────────────────────────────────────────┬───────────────┘
                │ observe domain tables                         │ observe outbox
                ▼                                               ▼
┌───────────────────────────────┐                 ┌─────────────────────────────────┐
│       Repo observeX() streams │                 │    SyncManager (headless)       │
│ (Publishers/observers)        │                 │  - observes ready outbox ops    │
└───────────────────────────────┘                 │  - connectivity/lifecycle gates │
                                                  │  - drains ops w/ backoff        │
                                                  │  - applies results via repos    │
                                                  └──────────────┬──────────────────┘
                                                                 │ network I/O
                                                                 ▼
                                                  ┌────────────────────────────────┐
                                                  │          NetworkClient         │
                                                  │  REST/GraphQL (GET + mutation) │
                                                  │  WebSocket (events)            │
                                                  │  Background URLSession (upload)│
                                                  └──────────────┬─────────────────┘
                                                                 ▼
                                                  ┌──────────────────────────────┐
                                                  │             Server           │
                                                  │  Feed/Search APIs (cursor)   │
                                                  │  Chat APIs + WS events       │
                                                  │  Profile/Content mutations   │
                                                  └──────────────────────────────┘
Legend:
- Queries: VM subscribes to repo.observeX() → DB drives updates.
- Commands: VM calls UseCase → Repo persists + enqueues outbox (if durable).
- Sync: SyncManager drains outbox → Network → Repo applies ack → DB emits.
```

## Diagrams & Detailed Sequences

Use this section as your “walkthrough script.” For each scenario:
- Start with the diagram (boxes + arrows)
- Name the API surface (protocols)
- Walk the sequence (happy path)
- Call out 2–3 edge cases + where they live

### Infinite Feed (Pagination + Caching + Reconciliation)

Key APIs (what to mention)
```swift
// VM depends on these
protocol FeedRepository {
    func observeFeed(feedID: String) -> AnyPublisher<[FeedItem], Never>
    func readMetadata(feedID: String) async -> FeedMetadata?

    func refresh(feedID: String, limit: Int) async throws
    func loadNextPage(feedID: String, cursor: String, limit: Int) async throws
}

protocol RefreshFeedUseCase {
    func refresh(feedID: String) async
}

protocol LoadNextFeedPageUseCase {
    func loadNext(feedID: String) async
}

struct FeedMetadata {
    var nextCursor: String?
    var hasMore: Bool
    var lastRefreshedAt: Date
}
```

Architecture Diagram
```
┌──────────────────────────────────┐
│        UIKit Layer               │
| FeedViewController               │
│ - scroll events / pull-to-refresh│
│ - apply snapshot (diffable)      │
└──────────────┬───────────────────┘
               │ intents
               ▼
┌──────────────────────────────┐
│ FeedViewModel                │
│ - subscribe to observeFeed() │
│ - state: isLoading/isPaging  │
│ - guards: canLoadMore        │
└──────────────┬───────────────┘
               │ commands
               ▼
┌──────────────────────────────┐
│ UseCases                     │
│ RefreshFeed / LoadNextPage   │
│ - policy: rate limit, cancel │
└──────────────┬───────────────┘
               │ data access
               ▼
┌────────────────────────────┐         ┌────────────────────────────┐
|        FeedRepository      │         │ Local DB (Source of Truth) │
│ - REST fetch + DB upsert   │───────► │ - feed_items               │
│ - merge server+local fields│         │ - feed_metadata(cursor)    │ 
└────────────────────────────┘         └────────────────────────────┘
               │ remote
               ▼
     ┌──────────────────┐
     │ REST/GraphQL API │
     │ cursor-based     │
     └──────────────────┘
```

Sequence Details
1.	Bind (screen start)

- VC creates VM (or coordinator injects), calls vm.start(feedID)
- VM subscribes once:
- repo.observeFeed(feedID) → maps domain → cell view models → updates state.items

2.	Initial refresh / pull-to-refresh

- VC → VM: refresh()
- VM sets isLoading = true then calls RefreshFeedUseCase.refresh(feedID)
- UseCase calls repo.refresh(feedID, limit)
- Repo fetches page1, upserts items + updates feed_metadata.nextCursor
- DB triggers the observation → VM updates list → VC applies snapshot

3.	Pagination

- VC detects near-bottom (prefetch/scroll), calls vm.loadNextPage()
- VM guards: isPaging == false && canLoadMore == true
- VM calls LoadNextFeedPageUseCase.loadNext(feedID)
- UseCase reads cursor via repo.readMetadata() (or caches in memory), then calls repo.loadNextPage(feedID, cursor, limit)
- Repo upserts, updates cursor → DB triggers observeFeed

Reconciliation notes (mention briefly)
- Prefer cursor-based pagination; DB primary key dedupes repeated items.
- Upsert server-owned fields while preserving local overlays (likes, pending states).

Edge Cases
- Double pagination: VM guards (isPaging, canLoadMore).
- Cursor invalid / 410 Gone: UseCase triggers full refresh; repo resets metadata.
- Items re-ordered between pages: DB sortKey controls ordering; UI diff handles moves.
