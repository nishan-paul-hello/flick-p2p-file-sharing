# Software Requirements Specification (SRS)

## Flick - Peer-to-Peer File Sharing Application

**Version:** 0.1.0  
**Date:** January 1, 2026  
**Status:** Production Ready

---

## 1. INTRODUCTION

### 1.1 Purpose

This Software Requirements Specification (SRS) defines the complete functional and non-functional requirements for **Flick**, a peer-to-peer file sharing Progressive Web Application. This document serves as the authoritative blueprint for development, testing, deployment, and maintenance.

### 1.2 Scope

**Flick** is a client-side, zero-backend Progressive Web Application enabling direct peer-to-peer file transfer between devices using WebRTC technology. The system supports unlimited file sizes, bidirectional transfers, and operates across desktop, mobile, and tablet platforms.

**Key Capabilities:**

- Room-based P2P connections via 6-character codes
- Real-time bidirectional file transfers with progress tracking
- Dual-mode storage strategy (OPFS for large files, RAM for compatibility)
- Advanced NAT traversal using STUN/TURN servers
- Complete privacy with zero server-side data persistence
- Installable PWA with offline support

### 1.3 Definitions, Acronyms, and Abbreviations

| Term       | Definition                                                                          |
| ---------- | ----------------------------------------------------------------------------------- |
| **P2P**    | Peer-to-Peer: Direct communication between two devices without intermediary servers |
| **WebRTC** | Web Real-Time Communication: Browser API for P2P data/media transfer                |
| **PeerJS** | JavaScript library abstracting WebRTC complexity                                    |
| **STUN**   | Session Traversal Utilities for NAT: Protocol for NAT discovery                     |
| **TURN**   | Traversal Using Relays around NAT: Relay server for restrictive NATs                |
| **ICE**    | Interactive Connectivity Establishment: Framework for NAT traversal                 |
| **OPFS**   | Origin Private File System: Browser-based private file storage API                  |
| **PWA**    | Progressive Web App: Installable web application with offline capabilities          |
| **NAT**    | Network Address Translation: Router mechanism that can block P2P                    |
| **SSR**    | Server-Side Rendering: Next.js rendering strategy                                   |
| **CSR**    | Client-Side Rendering: Browser-based rendering                                      |

### 1.4 Document Conventions

- **MUST**: Mandatory requirement
- **SHOULD**: Recommended but not mandatory
- **MAY**: Optional feature
- Code blocks use TypeScript/JavaScript syntax
- File paths use UNIX-style forward slashes

---

## 2. OVERALL DESCRIPTION

### 2.1 Product Perspective

Flick is a standalone web application built on Next.js 15+ with React 19. It operates entirely client-side with no backend infrastructure, using public STUN/TURN servers for NAT traversal.

**System Context:**

```
┌─────────────┐                    ┌─────────────┐
│   Device A  │◄──── WebRTC ──────►│   Device B  │
│  (Browser)  │     Direct P2P     │  (Browser)  │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       └────────► STUN/TURN ◄────────────┘
                  (NAT Traversal)
```

### 2.2 Product Functions

#### 2.2.1 Core Functions

1. **Room Management**
    - Generate unique 6-character alphanumeric room codes
    - Host room creation with automatic peer initialization
    - Guest room joining with connection establishment
    - Room disconnection and cleanup

2. **File Transfer**
    - Multi-file selection via drag-and-drop or file picker
    - Chunked transfer (64KB chunks) with backpressure control
    - Real-time progress tracking (per-file and per-chunk)
    - Bidirectional simultaneous transfers
    - Support for unlimited file sizes using OPFS
    - Automatic retry logic for failed chunks

3. **Connection Management**
    - Automatic peer discovery and connection
    - ICE candidate gathering and exchange
    - Connection quality monitoring (excellent/good/poor/disconnected)
    - Automatic reconnection on temporary disconnects
    - TURN relay fallback for restrictive NATs

4. **Storage Management**
    - Automatic storage mode detection (Power/Compatibility)
    - OPFS-based disk streaming for large files (Power Mode)
    - RAM-based storage for compatibility (Compatibility Mode)
    - IndexedDB persistence for session restoration
    - Storage quota monitoring

5. **User Interface**
    - Real-time connection status indicators
    - File transfer history with sent/received tabs
    - Activity log panel with filtering
    - Download individual files or batch ZIP
    - Responsive design for all screen sizes

### 2.3 User Classes and Characteristics

| User Class      | Characteristics                                   | Technical Proficiency                 |
| --------------- | ------------------------------------------------- | ------------------------------------- |
| **Casual User** | Personal file sharing between own devices         | Low - No technical knowledge required |
| **Power User**  | Frequent transfers, large files, multiple devices | Medium - Understands basic networking |
| **Developer**   | Testing, debugging, extending functionality       | High - Full technical understanding   |

### 2.4 Operating Environment

**Client Requirements:**

- Modern web browser with WebRTC support:
    - Chrome/Edge 89+ (recommended for OPFS)
    - Firefox 90+
    - Safari 15.4+ (limited OPFS support)
- JavaScript enabled
- Minimum 100MB available RAM
- Network connectivity (WiFi, cellular, or ethernet)

**Network Requirements:**

- Outbound UDP/TCP ports for WebRTC (typically 49152-65535)
- Access to STUN servers (ports 3478, 19302)
- Access to TURN servers (ports 80, 443, 3478) for restrictive NATs

**Platform Support:**

- Desktop: Windows, macOS, Linux
- Mobile: iOS 15.4+, Android 10+
- Tablet: iPad OS 15.4+, Android tablets

### 2.5 Design and Implementation Constraints

**Technical Constraints:**

1. **No Backend**: All logic must execute client-side
2. **Browser APIs**: Limited to Web Platform APIs (WebRTC, OPFS, IndexedDB)
3. **Storage Limits**: Browser quota restrictions (typically 60% of available disk)
4. **Memory Limits**: Mobile browsers may have 1-2GB RAM limits
5. **TURN Quota**: Free TURN servers have bandwidth limits (500MB-20GB/month)

**Architectural Constraints:**

1. **Framework**: Next.js 15+ with App Router (not Pages Router)
2. **Language**: TypeScript with strict mode enabled
3. **State Management**: Zustand with IndexedDB persistence
4. **UI Library**: React 19 with shadcn/ui components
5. **Styling**: Tailwind CSS with custom design system

---

## 3. SYSTEM FEATURES

### 3.1 Room-Based Connection System

**Priority:** Critical  
**Description:** Users create or join rooms using 6-character codes to establish P2P connections.

#### 3.1.1 Functional Requirements

**FR-3.1.1:** Room Code Generation  
The system MUST generate unique 6-character alphanumeric room codes (uppercase A-Z, 0-9).

**FR-3.1.2:** Host Room Creation  
When a user creates a room, the system MUST:

- Initialize a PeerJS instance with the room code as peer ID
- Configure ICE servers (STUN/TURN) for NAT traversal
- Display the room code prominently with copy-to-clipboard functionality
- Listen for incoming peer connections

**FR-3.1.3:** Guest Room Joining  
When a user joins a room, the system MUST:

- Validate the room code format (6 alphanumeric characters)
- Initialize a PeerJS instance with a random peer ID
- Attempt connection to the host peer using the room code
- Implement 30-second connection timeout with error handling
- Handle "room already exists" scenario gracefully

**FR-3.1.4:** Connection State Management  
The system MUST track and display connection states:

- `disconnected`: No active connection
- `connecting`: Connection attempt in progress
- `connected`: Active P2P connection established
- `reconnecting`: Attempting to restore lost connection

**FR-3.1.5:** Room Disconnection  
The system MUST allow users to leave rooms, which:

- Closes all active data channels
- Destroys the PeerJS instance
- Clears room code from state
- Optionally preserves transfer history

#### 3.1.2 Non-Functional Requirements

**NFR-3.1.1:** Connection Establishment Time  
The system SHOULD establish P2P connections within 5 seconds under normal network conditions.

**NFR-3.1.2:** Room Code Uniqueness  
Room codes MUST have a collision probability < 0.001% (36^6 = 2.1 billion combinations).

### 3.2 File Transfer System

**Priority:** Critical  
**Description:** Bidirectional file transfer with chunking, progress tracking, and error recovery.

#### 3.2.1 Functional Requirements

**FR-3.2.1:** File Selection  
The system MUST support file selection via:

- Drag-and-drop onto designated drop zone
- Click-to-browse file picker
- Multiple file selection (unlimited count)

**FR-3.2.2:** Chunked Transfer Protocol  
The system MUST implement a three-phase transfer protocol:

1. **Metadata Phase:**

    ```typescript
    {
      type: 'metadata',
      transferId: string,
      metadata: { name, size, type, timestamp },
      totalChunks: number
    }
    ```

2. **Chunk Phase:**

    ```typescript
    {
      type: 'chunk',
      transferId: string,
      chunkIndex: number,
      data: ArrayBuffer
    }
    ```

3. **Completion Phase:**
    ```typescript
    {
      type: 'complete',
      transferId: string
    }
    ```

**FR-3.2.3:** Chunk Size and Backpressure

- Chunk size MUST be 64KB (65,536 bytes)
- System MUST monitor `bufferedAmount` on data channel
- System MUST pause sending when `bufferedAmount` > 1MB
- System MUST resume sending when buffer drains

**FR-3.2.4:** Progress Tracking  
The system MUST display:

- Per-file progress percentage (0-100%)
- Transfer speed (MB/s)
- Estimated time remaining
- Current chunk / total chunks

**FR-3.2.5:** Simultaneous Transfers  
The system MUST support:

- Multiple outgoing transfers concurrently
- Multiple incoming transfers concurrently
- Bidirectional transfers (both peers sending/receiving)

**FR-3.2.6:** Transfer States  
Each transfer MUST have one of these states:

- `pending`: Queued for transfer
- `transferring`: Active transfer in progress
- `completed`: Successfully transferred
- `failed`: Transfer failed with error

#### 3.2.2 Non-Functional Requirements

**NFR-3.2.1:** Transfer Speed  
The system SHOULD achieve transfer speeds of:

- LAN: 50-100 MB/s
- Direct WiFi: 10-50 MB/s
- TURN relay: 1-10 MB/s

**NFR-3.2.2:** File Size Support

- Power Mode (OPFS): Unlimited file size
- Compatibility Mode (RAM): Limited by available memory (typically 500MB-2GB)

**NFR-3.2.3:** Progress Update Frequency  
Progress updates MUST be throttled to maximum 1% increments to prevent UI lag.

### 3.3 Storage Management System

**Priority:** High  
**Description:** Dual-mode storage strategy adapting to browser capabilities.

#### 3.3.1 Functional Requirements

**FR-3.3.1:** Storage Mode Detection  
On application load, the system MUST:

- Test for OPFS API availability (`navigator.storage.getDirectory`)
- Test for `FileSystemWritableFileStream` support
- Perform actual OPFS write test (create/write/delete test file)
- Set storage mode to `power` if all tests pass, otherwise `compatibility`

**FR-3.3.2:** Power Mode (OPFS)  
When in Power Mode, the system MUST:

- Create a dedicated directory: `flick-transfers/`
- Stream incoming chunks directly to OPFS using `FileSystemWritableFileStream`
- Write chunks at correct offsets using `seek()` and `write()`
- Serialize writes per transfer to prevent race conditions
- Close writable streams on transfer completion
- Clean up OPFS files on user request

**FR-3.3.3:** Compatibility Mode (RAM)  
When in Compatibility Mode, the system MUST:

- Store chunks in memory as `ArrayBuffer[]`
- Maintain chunk order by index
- Assemble complete file as `Blob` on download
- Clear chunks from memory after download

**FR-3.3.4:** Storage Indicator  
The system MUST display current storage mode to users with:

- Icon indicator (Zap for Power, ZapOff for Compatibility)
- Mode label ("Power Mode" or "Compatibility Mode")
- Tooltip explaining implications

**FR-3.3.5:** Storage Quota Monitoring  
The system SHOULD:

- Query `navigator.storage.estimate()` for quota information
- Warn users when approaching quota limits (>80% usage)
- Provide option to clear transfer history to free space

#### 3.3.2 Non-Functional Requirements

**NFR-3.3.1:** OPFS Performance  
OPFS writes SHOULD complete within 10ms per 64KB chunk on modern hardware.

**NFR-3.3.2:** Memory Efficiency  
Compatibility Mode SHOULD limit memory usage to < 500MB for mobile devices.

### 3.4 State Persistence System

**Priority:** Medium  
**Description:** Session restoration using IndexedDB for seamless user experience.

#### 3.4.1 Functional Requirements

**FR-3.4.1:** Persisted State  
The system MUST persist to IndexedDB:

- Room code (if host)
- Peer ID (if host)
- Host/guest role
- Transfer history (received and sent files)
- Activity logs (last 100 entries)
- Storage capabilities

**FR-3.4.2:** State Exclusions  
The system MUST NOT persist:

- Active WebRTC connections
- File chunks (too large for IndexedDB)
- In-progress transfers (marked as `failed` on restore)
- Temporary UI state

**FR-3.4.3:** Session Restoration  
On application load, the system MUST:

- Rehydrate state from IndexedDB
- Set `hasHydrated` flag when complete
- Show loading screen until hydration completes
- Attempt to restore OPFS file handles for completed transfers

**FR-3.4.4:** State Cleanup  
The system MUST provide options to:

- Clear all transfer history
- Clear activity logs
- Reset to initial state

### 3.5 NAT Traversal and Connection Quality

**Priority:** Critical  
**Description:** Robust connection establishment across diverse network topologies.

#### 3.5.1 Functional Requirements

**FR-3.5.1:** ICE Server Configuration  
The system MUST configure PeerJS with:

- Multiple STUN servers for redundancy (Google, Twilio)
- TURN servers with credentials (Xirsys)
- ICE candidate pool size: 15
- ICE transport policy: `all` (UDP, TCP, TLS)
- Bundle policy: `max-bundle`
- RTCP mux policy: `require`

**FR-3.5.2:** Dynamic TURN Credentials  
The system SHOULD:

- Fetch fresh TURN credentials from Xirsys API on peer initialization
- Use user-provided UI settings for API credentials
- Log TURN server status for debugging

**FR-3.5.3:** ICE State Monitoring  
The system MUST monitor ICE connection states:

- `new`: Initial state
- `checking`: Gathering and testing candidates
- `connected`: Connection established
- `completed`: All candidates tested
- `failed`: Connection failed
- `disconnected`: Temporary disconnect
- `closed`: Connection terminated

**FR-3.5.4:** Connection Quality Indicators  
The system MUST map ICE states to user-facing quality levels:

- `excellent`: ICE state `connected` or `completed`
- `good`: Stable connection with minor latency
- `poor`: ICE state `disconnected` or high packet loss
- `disconnected`: ICE state `failed` or `closed`

**FR-3.5.5:** Candidate Logging  
In development mode, the system MUST log:

- ICE candidate types (host, srflx, relay)
- TURN relay usage detection
- Connection establishment path (direct vs. relayed)

#### 3.5.2 Non-Functional Requirements

**NFR-3.5.1:** NAT Traversal Success Rate  
The system SHOULD achieve >95% connection success rate across:

- Symmetric NAT (mobile networks)
- Port-restricted NAT (corporate firewalls)
- Full-cone NAT (home routers)

**NFR-3.5.2:** Reconnection Time  
The system SHOULD restore connections within 10 seconds after temporary disconnects.

### 3.6 User Interface and Experience

**Priority:** High  
**Description:** Modern, responsive, accessible interface with premium aesthetics.

#### 3.6.1 Functional Requirements

**FR-3.6.1:** Responsive Layout  
The system MUST provide optimized layouts for:

- Mobile: 320px - 767px (single column)
- Tablet: 768px - 1023px (adaptive grid)
- Desktop: 1024px+ (multi-column grid)

**FR-3.6.2:** Component Structure  
The UI MUST include these components:

- **Header**: App branding, theme toggle, log panel toggle
- **ConnectionPanel**: Room creation/joining, status, storage mode
- **FileDropZone**: Drag-and-drop and file picker
- **FileList**: Tabbed view (Received/Sent) with file cards
- **LogPanel**: Sliding panel with activity logs
- **Footer**: Attribution and links

**FR-3.6.3:** File Card Display  
Each file card MUST show:

- File name (truncated with ellipsis if long)
- File size (human-readable: KB, MB, GB)
- File type icon
- Transfer status badge
- Progress bar (if transferring)
- Download button (if completed)
- Remove button

**FR-3.6.4:** Activity Log  
The log panel MUST:

- Display last 100 log entries
- Show timestamp, type (info/success/warning/error), message, description
- Auto-scroll to newest entry
- Indicate unread logs with badge
- Mark logs as read when panel opened

**FR-3.6.5:** Animations and Transitions  
The system MUST implement:

- Page transitions (fade in/out)
- Component mount animations (slide, scale)
- Progress bar animations (smooth width transitions)
- Icon animations (pulse for active transfers)
- Loading skeletons for async operations

**FR-3.6.6:** Accessibility  
The system MUST provide:

- ARIA labels for all interactive elements
- Semantic HTML5 elements
- Keyboard navigation support
- Focus indicators
- Screen reader announcements for state changes

#### 3.6.2 Non-Functional Requirements

**NFR-3.6.1:** Performance

- First Contentful Paint (FCP): < 1.5s
- Time to Interactive (TTI): < 3s
- Cumulative Layout Shift (CLS): < 0.1

**NFR-3.6.2:** Design System  
The UI MUST use:

- Color palette: Zinc/Slate dark mode with Sky/Cyan accents
- Typography: Inter font family
- Spacing: Fluid spacing scale (clamp-based)
- Glassmorphism: backdrop-blur with semi-transparent backgrounds

### 3.7 Progressive Web App Features

**Priority:** Medium  
**Description:** PWA capabilities for installation and offline support.

#### 3.7.1 Functional Requirements

**FR-3.7.1:** Web App Manifest  
The system MUST provide `manifest.json` with:

- App name: "Flick - P2P File Sharing"
- Short name: "Flick"
- Theme color: `#020617`
- Background color: `#020617`
- Display mode: `standalone`
- Icons: 192x192, 512x512, SVG
- Start URL: `/`

**FR-3.7.2:** Service Worker  
The system SHOULD implement service worker for:

- Offline fallback page
- Static asset caching (CSS, JS, fonts)
- Runtime caching for API requests

**FR-3.7.3:** Install Prompt  
The system MAY:

- Detect `beforeinstallprompt` event
- Show custom install button
- Defer prompt until user interaction

**FR-3.7.4:** iOS Support  
The system MUST include meta tags for iOS:

- `apple-mobile-web-app-capable`
- `apple-mobile-web-app-status-bar-style`
- `apple-touch-icon`

---

## 4. EXTERNAL INTERFACE REQUIREMENTS

### 4.1 User Interfaces

**UI-1: Home Page**

- Layout: Responsive grid (1 column mobile, 2 columns desktop)
- Components: ConnectionPanel, FileDropZone, FileList tabs
- State: Loading, Active, Error

**UI-2: Loading Screen**

- Full-screen overlay with animated logo
- Minimum display duration: 1 second
- Fade-out transition

**UI-3: Error Page**

- Custom 404 page with navigation back to home
- Error boundary for runtime errors

**UI-4: Log Panel**

- Sliding panel from left edge
- Width: 320px (mobile), 384px (desktop)
- Overlay on mobile, sidebar on desktop

### 4.2 Hardware Interfaces

**HW-1: Network Interface**

- System MUST utilize device network interfaces for WebRTC
- Support WiFi, Ethernet, and cellular data connections

**HW-2: Storage Interface**

- System MUST access browser storage APIs (IndexedDB, OPFS)
- Respect browser storage quotas

### 4.3 Software Interfaces

**SW-1: PeerJS Library**

- Version: 1.5.4+
- Purpose: WebRTC abstraction
- Interface: JavaScript API

**SW-2: Next.js Framework**

- Version: 15.1.0+
- Purpose: React framework with SSR/SSG
- Interface: App Router API

**SW-3: Zustand State Management**

- Version: 5.0.9+
- Purpose: Global state management
- Interface: React hooks

**SW-4: IndexedDB (idb-keyval)**

- Version: 6.2.2+
- Purpose: State persistence
- Interface: Promise-based key-value API

**SW-5: STUN/TURN Servers**

- Google STUN: `stun.l.google.com:19302`
- Xirsys TURN: Dynamic credentials via API

### 4.4 Communications Interfaces

**COM-1: WebRTC Data Channels**

- Protocol: SCTP over DTLS
- Serialization: Binary (ArrayBuffer)
- Reliability: Reliable ordered delivery
- Max message size: 64KB

**COM-2: PeerJS Signaling**

- Protocol: WebSocket to PeerJS cloud server
- Purpose: Peer discovery and SDP exchange
- Endpoint: `0.peerjs.com` (default)

**COM-3: TURN API (Xirsys)**

- Protocol: HTTPS REST API
- Method: PUT to `https://global.xirsys.net/_turn/{channel}`
- Authentication: Basic Auth (Base64 encoded)
- Response: JSON with ICE servers

---

## 5. NON-FUNCTIONAL REQUIREMENTS

### 5.1 Performance Requirements

**PERF-1:** The system MUST handle file transfers up to 100GB in Power Mode.

**PERF-2:** The system MUST maintain UI responsiveness (<100ms input latency) during active transfers.

**PERF-3:** The system MUST achieve transfer speeds of at least 10 MB/s on LAN connections.

**PERF-4:** The system MUST support at least 10 simultaneous file transfers per peer.

**PERF-5:** IndexedDB operations MUST complete within 500ms for state persistence.

### 5.2 Safety Requirements

**SAFE-1:** The system MUST prevent browser tab crashes by limiting RAM usage in Compatibility Mode.

**SAFE-2:** The system MUST gracefully handle out-of-memory errors with user notifications.

**SAFE-3:** The system MUST close all OPFS writable streams on connection loss to prevent data corruption.

### 5.3 Security Requirements

**SEC-1:** The system MUST NOT store user data on any server.

**SEC-2:** The system MUST use HTTPS in production to enable WebRTC.

**SEC-3:** The system MUST sanitize room codes to prevent XSS (alphanumeric only).

**SEC-4:** The system MUST implement Content Security Policy headers.

**SEC-5:** The system MUST NOT expose TURN credentials in public client-side code (use environment variables or private local storage).

**SEC-6:** The system SHOULD use DTLS encryption for WebRTC data channels (automatic in WebRTC).

### 5.4 Software Quality Attributes

**QUAL-1: Reliability**

- System uptime: 99.9% (excluding user network issues)
- Connection success rate: >95%
- Transfer completion rate: >98%

**QUAL-2: Maintainability**

- Code coverage: >80% for critical paths
- TypeScript strict mode: Enabled
- ESLint compliance: Zero errors
- Documentation: JSDoc for all public APIs

**QUAL-3: Usability**

- First-time user success rate: >90% (create/join room)
- Average time to first transfer: <60 seconds
- User error rate: <5%

**QUAL-4: Scalability**

- Support for files up to 100GB
- Support for 100+ files in transfer history
- Support for 100 log entries

**QUAL-5: Portability**

- Browser support: Chrome, Firefox, Safari, Edge
- OS support: Windows, macOS, Linux, iOS, Android
- No platform-specific code

### 5.5 Business Rules

**BR-1:** The system MUST NOT require user authentication or accounts.

**BR-2:** The system MUST NOT store files on any server.

**BR-3:** The system MUST NOT implement analytics or tracking.

**BR-4:** The system MUST be free to use with no subscription model.

**BR-5:** Room codes MUST expire when the host disconnects.

---

## 6. SYSTEM ARCHITECTURE

### 6.1 Architectural Style

**Pattern:** Client-Side Single Page Application (SPA) with Server-Side Rendering (SSR) for initial load

**Key Principles:**

- Zero backend dependency (pure frontend + P2P)
- Component-based architecture (React)
- Unidirectional data flow (Zustand)
- Separation of concerns (presentation, business logic, data)

### 6.2 Component Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Next.js App Router                  │
├─────────────────────────────────────────────────────────┤
│  app/                                                   │
│  ├── layout.tsx          (Root layout, providers)      │
│  ├── page.tsx            (Home page, main UI)          │
│  ├── loading.tsx         (Loading screen)              │
│  ├── error.tsx           (Error boundary)              │
│  └── not-found.tsx       (404 page)                    │
├─────────────────────────────────────────────────────────┤
│  components/                                            │
│  ├── ConnectionPanel     (Room management)             │
│  ├── FileDropZone        (File selection)              │
│  ├── FileList            (Transfer history)            │
│  ├── LogPanel            (Activity logs)               │
│  ├── Header              (App header)                  │
│  ├── Footer              (App footer)                  │
│  └── ui/                 (shadcn/ui primitives)        │
├─────────────────────────────────────────────────────────┤
│  lib/                                                   │
│  ├── store/              (Zustand state management)    │
│  │   ├── slices/                                       │
│  │   │   ├── peer-slice.ts      (P2P connection)      │
│  │   │   ├── transfer-slice.ts  (File transfers)      │
│  │   │   ├── log-slice.ts       (Activity logs)       │
│  │   │   ├── ui-slice.ts        (UI state)            │
│  │   │   └── storage-slice.ts   (Storage mode)        │
│  │   ├── index.ts        (Store composition)          │
│  │   └── types.ts        (Store type definitions)     │
│  ├── hooks/              (Custom React hooks)          │
│  ├── opfs-manager.ts     (OPFS operations)            │
│  ├── storage-mode.ts     (Storage detection)          │
│  ├── ice-servers.ts      (TURN credential fetching)   │
│  ├── constants.ts        (App constants)              │
│  ├── types.ts            (Type definitions)           │
│  └── utils.ts            (Utility functions)          │
└─────────────────────────────────────────────────────────┘
```

### 6.3 Data Flow Architecture

```
┌──────────────┐
│  User Action │
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│  React Component │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  Zustand Action  │ ◄─── (State update)
└──────┬───────────┘
       │
       ├──────────────────────┐
       │                      │
       ▼                      ▼
┌──────────────┐      ┌──────────────┐
│  PeerJS API  │      │  OPFS/RAM    │
└──────┬───────┘      └──────┬───────┘
       │                     │
       ▼                     ▼
┌──────────────┐      ┌──────────────┐
│   WebRTC     │      │  IndexedDB   │
└──────────────┘      └──────────────┘
```

### 6.4 State Management Architecture

**Store Slices:**

1. **PeerSlice** (`peer-slice.ts`)
    - Manages PeerJS instance lifecycle
    - Handles connection establishment
    - Monitors ICE states
    - Processes incoming messages

2. **TransferSlice** (`transfer-slice.ts`)
    - Manages file transfer queue
    - Implements chunking logic
    - Tracks transfer progress
    - Handles downloads

3. **LogSlice** (`log-slice.ts`)
    - Maintains activity log
    - Implements log rotation (max 100 entries)
    - Tracks unread status

4. **UISlice** (`ui-slice.ts`)
    - Manages UI state (log panel open/closed)
    - Tracks hydration status

5. **StorageSlice** (`storage-slice.ts`)
    - Detects storage capabilities
    - Manages storage mode

**State Persistence:**

- Middleware: Zustand `persist` middleware
- Storage: IndexedDB via `idb-keyval`
- Partialize: Selective state persistence (excludes active connections, chunks)

### 6.5 File Transfer Protocol

**Message Types:**

1. **Metadata Message**

    ```typescript
    {
      type: 'metadata',
      transferId: string,        // Unique transfer ID
      metadata: {
        name: string,            // File name
        size: number,            // File size in bytes
        type: string,            // MIME type
        timestamp: number        // Unix timestamp
      },
      totalChunks: number        // Total number of chunks
    }
    ```

2. **Chunk Message**

    ```typescript
    {
      type: 'chunk',
      transferId: string,        // Transfer ID
      chunkIndex: number,        // Chunk index (0-based)
      data: ArrayBuffer          // 64KB chunk data
    }
    ```

3. **Complete Message**
    ```typescript
    {
      type: 'complete',
      transferId: string         // Transfer ID
    }
    ```

**Transfer Flow:**

```
Sender                          Receiver
  │                                │
  ├─── metadata ─────────────────►│ (Create transfer record)
  │                                │
  ├─── chunk[0] ─────────────────►│ (Write to OPFS/RAM)
  ├─── chunk[1] ─────────────────►│
  ├─── chunk[2] ─────────────────►│
  │         ...                    │
  ├─── chunk[N] ─────────────────►│
  │                                │
  ├─── complete ─────────────────►│ (Mark as completed)
  │                                │
```

---

## 7. DATA REQUIREMENTS

### 7.1 Data Models

#### 7.1.1 FileMetadata

```typescript
interface FileMetadata {
    name: string; // File name with extension
    size: number; // File size in bytes
    type: string; // MIME type (e.g., "image/png")
    timestamp: number; // Unix timestamp (ms)
}
```

#### 7.1.2 FileTransfer

```typescript
interface FileTransfer {
    id: string; // Unique transfer ID
    metadata: FileMetadata; // File metadata
    progress: number; // Progress percentage (0-100)
    status: 'pending' | 'transferring' | 'completed' | 'failed';
    totalChunks: number; // Total number of chunks
    storageMode: 'power' | 'compatibility';

    // Compatibility mode (RAM)
    chunks?: ArrayBuffer[]; // Array of chunks

    // Power mode (OPFS)
    opfsPath?: string; // Path in OPFS

    // UI state
    downloaded?: boolean; // User has downloaded
}
```

#### 7.1.3 LogEntry

```typescript
interface LogEntry {
    id: string; // Unique log ID
    timestamp: number; // Unix timestamp (ms)
    type: 'info' | 'success' | 'warning' | 'error';
    message: string; // Primary message
    description?: string; // Optional details
}
```

#### 7.1.4 StorageCapabilities

```typescript
interface StorageCapabilities {
    mode: 'power' | 'compatibility';
    supportsOPFS: boolean;
    browserInfo: string; // Browser name
}
```

### 7.2 Data Storage

**IndexedDB Schema:**

```
Database: flick-peer-storage
├── Key: "flick-peer-storage"
└── Value: {
      roomCode: string | null,
      peerId: string | null,
      isHost: boolean,
      receivedFiles: FileTransfer[],
      outgoingFiles: FileTransfer[],
      storageCapabilities: StorageCapabilities,
      logs: LogEntry[],
      hasUnreadLogs: boolean
    }
```

**OPFS Structure:**

```
OPFS Root
└── flick-transfers/
    ├── {transferId1}.bin
    ├── {transferId2}.bin
    └── {transferId3}.bin
```

### 7.3 Data Constraints

**DC-1:** Room codes MUST be exactly 6 characters (uppercase A-Z, 0-9).

**DC-2:** Transfer IDs MUST be unique (format: `{timestamp}-{random}`).

**DC-3:** Log entries MUST NOT exceed 100 items (FIFO rotation).

**DC-4:** File names MUST be preserved exactly as uploaded.

**DC-5:** Chunk indices MUST be sequential starting from 0.

---

## 8. DEPLOYMENT REQUIREMENTS

### 8.1 Build Configuration

**Build Command:**

```bash
npm run build
```

**Output:**

- `.next/standalone/` - Standalone server bundle
- `.next/static/` - Static assets (CSS, JS, images)
- `public/` - Public assets (manifest, icons)

**Environment Variables:**

```bash
# Optional: Custom PeerJS server
NEXT_PUBLIC_PEER_HOST=
NEXT_PUBLIC_PEER_PORT=
NEXT_PUBLIC_PEER_PATH=

# Required: Xirsys TURN credentials
NEXT_PUBLIC_XIRSYS_IDENT=your_username
NEXT_PUBLIC_XIRSYS_SECRET=your_secret
NEXT_PUBLIC_XIRSYS_CHANNEL=your_channel
```

### 8.2 Deployment Platforms

**Recommended: Vercel**

- Automatic deployments from Git
- Edge network CDN
- Zero configuration
- Free tier available

**Alternative: Docker**

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
EXPOSE ${PORT}
CMD ["npm", "start", "--", "--port", "${PORT}"]
```

### 8.3 Production Checklist

- [ ] Environment variables configured
- [ ] HTTPS enabled (required for WebRTC)
- [ ] Security headers configured (CSP, HSTS, etc.)
- [ ] Error tracking enabled (optional: Sentry)
- [ ] Analytics disabled (privacy requirement)
- [ ] Service worker registered
- [ ] PWA manifest validated
- [ ] Icons generated (192x192, 512x512)
- [ ] Lighthouse score >90 (Performance, Accessibility, Best Practices, SEO)

---

## 9. TESTING REQUIREMENTS

### 9.1 Unit Testing

**UT-1:** All utility functions MUST have unit tests with >90% coverage.

**UT-2:** State management slices MUST have unit tests for all actions.

**UT-3:** OPFS manager MUST have tests for all CRUD operations.

**Framework:** Jest + React Testing Library

### 9.2 Integration Testing

**IT-1:** File transfer flow MUST be tested end-to-end (send → receive → download).

**IT-2:** Connection establishment MUST be tested across different network conditions.

**IT-3:** Storage mode detection MUST be tested with mocked browser APIs.

### 9.3 End-to-End Testing

**E2E-1:** User can create room and receive room code.

**E2E-2:** User can join room with valid code.

**E2E-3:** User can send file and peer receives it.

**E2E-4:** User can download received file.

**E2E-5:** User can disconnect and reconnect.

**Framework:** Playwright

### 9.4 Performance Testing

**PT-1:** Transfer speed MUST be measured for files of varying sizes (1MB, 100MB, 1GB).

**PT-2:** Memory usage MUST be profiled during large file transfers.

**PT-3:** UI responsiveness MUST be measured during active transfers (FPS, input latency).

### 9.5 Compatibility Testing

**CT-1:** Application MUST be tested on:

- Chrome 89+ (Windows, macOS, Linux, Android)
- Firefox 90+ (Windows, macOS, Linux)
- Safari 15.4+ (macOS, iOS)
- Edge 89+ (Windows)

**CT-2:** OPFS support MUST be verified on each browser.

**CT-3:** WebRTC compatibility MUST be verified on each browser.

### 9.6 Security Testing

**ST-1:** XSS vulnerability testing for room code input.

**ST-2:** CSP header validation.

**ST-3:** HTTPS enforcement verification.

**ST-4:** Data leakage testing (ensure no server-side storage).

---

## 10. MAINTENANCE AND SUPPORT

### 10.1 Monitoring

**MON-1:** Client-side error logging (console errors).

**MON-2:** Connection failure rate tracking.

**MON-3:** Transfer completion rate tracking.

**MON-4:** Browser compatibility issues tracking.

### 10.2 Updates and Versioning

**Versioning Scheme:** Semantic Versioning (MAJOR.MINOR.PATCH)

**Update Policy:**

- **Major:** Breaking changes (e.g., protocol changes)
- **Minor:** New features (backward compatible)
- **Patch:** Bug fixes and performance improvements

### 10.3 Backup and Recovery

**BACK-1:** Users SHOULD be advised to download files immediately (no server-side backup).

**BACK-2:** Transfer history is backed up to IndexedDB (survives browser restarts).

**BACK-3:** OPFS files persist until explicitly deleted by user.

### 10.4 Documentation

**DOC-1:** README.md with quick start guide.

**DOC-2:** API documentation for all public functions (JSDoc).

**DOC-3:** Architecture documentation (this SRS).

**DOC-4:** Deployment guide for self-hosting.

---

## 11. APPENDICES

### 11.1 Glossary

| Term              | Definition                                        |
| ----------------- | ------------------------------------------------- |
| **Chunk**         | 64KB segment of a file for transfer               |
| **Host**          | Peer that creates a room                          |
| **Guest**         | Peer that joins an existing room                  |
| **Transfer ID**   | Unique identifier for a file transfer             |
| **Room Code**     | 6-character code for room identification          |
| **ICE Candidate** | Network address candidate for P2P connection      |
| **OPFS**          | Browser-based private file system                 |
| **Backpressure**  | Flow control mechanism to prevent buffer overflow |

### 11.2 Technology Stack Summary

| Category    | Technology    | Version   | Purpose                  |
| ----------- | ------------- | --------- | ------------------------ |
| Framework   | Next.js       | 15.1.0+   | React framework with SSR |
| UI Library  | React         | 19.0.0+   | Component-based UI       |
| Language    | TypeScript    | 5.0+      | Type-safe JavaScript     |
| Styling     | Tailwind CSS  | 3.4.1+    | Utility-first CSS        |
| P2P         | PeerJS        | 1.5.4+    | WebRTC abstraction       |
| State       | Zustand       | 5.0.9+    | State management         |
| Persistence | idb-keyval    | 6.2.2+    | IndexedDB wrapper        |
| Animation   | Framer Motion | 11.11.17+ | Animation library        |
| Components  | shadcn/ui     | Latest    | Radix UI primitives      |
| Icons       | Lucide React  | 0.469.0+  | Icon library             |
| Compression | JSZip         | 3.10.1+   | ZIP file creation        |

### 11.3 Browser Compatibility Matrix

| Feature        | Chrome  | Firefox    | Safari      | Edge    |
| -------------- | ------- | ---------- | ----------- | ------- |
| WebRTC         | ✅ 89+  | ✅ 90+     | ✅ 15.4+    | ✅ 89+  |
| OPFS           | ✅ 102+ | ⚠️ 111+    | ❌ No       | ✅ 102+ |
| IndexedDB      | ✅ All  | ✅ All     | ✅ All      | ✅ All  |
| Service Worker | ✅ All  | ✅ All     | ✅ 11.1+    | ✅ All  |
| PWA Install    | ✅ All  | ⚠️ Limited | ⚠️ iOS only | ✅ All  |

**Legend:**

- ✅ Full support
- ⚠️ Partial support
- ❌ No support

### 11.4 Performance Benchmarks

**Target Metrics (Desktop, LAN):**

- Connection establishment: < 5 seconds
- Transfer speed: 50-100 MB/s
- UI responsiveness: < 100ms input latency
- Memory usage: < 200MB (excluding file data)

**Target Metrics (Mobile, WiFi):**

- Connection establishment: < 10 seconds
- Transfer speed: 10-50 MB/s
- UI responsiveness: < 150ms input latency
- Memory usage: < 150MB (excluding file data)

### 11.5 Known Limitations

**LIM-1:** Safari on iOS does not support OPFS (falls back to Compatibility Mode).

**LIM-2:** Free TURN servers have bandwidth quotas (500MB-20GB/month).

**LIM-3:** Compatibility Mode limited by browser memory (typically 500MB-2GB).

**LIM-4:** Connection may fail on highly restrictive corporate networks blocking WebRTC.

**LIM-5:** Transfer speed limited by slowest peer's network connection.

### 11.6 Future Enhancements (Out of Scope)

**FE-1:** Multi-peer rooms (more than 2 participants)

**FE-2:** Encrypted file transfers (end-to-end encryption)

**FE-3:** Resume interrupted transfers

**FE-4:** File preview before download

**FE-5:** QR code for room joining

**FE-6:** Voice/video calling

**FE-7:** Text chat functionality

**FE-8:** File compression before transfer

---

## 12. APPROVAL

This Software Requirements Specification has been reviewed and approved by:

**Document Version:** 1.0  
**Last Updated:** January 1, 2026  
**Next Review Date:** July 1, 2026

---

**End of Document**
