# Decision Journal: ObsidianSync-BEP

**Purpose**: Track architectural decisions, trade-offs, and rationale for ObsidianSync-BEP project

**Format**: Lightweight Architecture Decision Records (ADRs)

---

## ADR-001: Syncthing Wrapper vs Custom BEP Server

**Date**: 2025-11-03
**Status**: DECIDED - Syncthing Wrapper (Option 1)
**Context**: Need to choose BEP sync implementation approach

### Decision

Start with **Syncthing wrapper + APNs push** (Option 1), defer custom BEP server (Option 2) until empirical data shows clear need.

### Options Considered

**Option 1: Syncthing Wrapper**
```
macOS Syncthing ←─BEP─→ iOS Sushitrain Fork
      ↑
File System Monitor → APNs Push Server → iOS App Wake
```

**Pros**:
- ✅ Proven stack (Sushitrain already works with Syncthing)
- ✅ Lower risk (battle-tested BEP implementation)
- ✅ Faster development (7 weeks vs 11+ weeks)
- ✅ Full Syncthing features (discovery, relay, conflict resolution)
- ✅ Lower maintenance (Syncthing team handles protocol updates)

**Cons**:
- ⚠️ Requires running Syncthing on macOS (always-on)
- ⚠️ Less control over sync behavior
- ⚠️ Still subject to iOS 30-second background limit

**Option 2: Custom BEP Server**
```
iOS Sushitrain Fork ←─BEP─→ vault-server (Custom Rust BEP)
                                    ↓
                              APNs Push Direct
```

**Pros**:
- ✅ Full control over protocol implementation
- ✅ Potential Obsidian-specific optimizations
- ✅ Integrated with vault-server architecture

**Cons**:
- ❌ High complexity (BEP v1 is complex protocol)
- ❌ Unknown iOS client compatibility (first custom BEP server for Sushitrain)
- ❌ 9-11 week development estimate
- ❌ Ongoing maintenance burden (protocol updates, iOS changes)
- ❌ No discovery/relay infrastructure (iOS clients may expect this)

### Decision Rationale

**Primary Reason**: **Risk management**

The BEP protocol analysis (see `ios-sync-integration-analysis.md`) revealed:
1. **2000-file metadata sync is feasible** (~200 KB compressed, fits in single message)
2. **iOS 30-second background limit is the PRIMARY constraint**, not BEP protocol
3. **No evidence of custom BEP servers in production** with iOS clients
4. **Reliability depends on network quality**, not protocol choice

**Key Insight**: Both options face the same iOS background execution challenge. Custom BEP server adds development complexity without solving the core constraint.

### Success Metrics for Reevaluation

**Evaluate Option 2 if**:
- Syncthing wrapper reliability < 70% after 3 months dogfooding
- Specific Obsidian optimization blocked by Syncthing architecture
- Syncthing maintenance becomes problematic (crashes, config drift)

**Stay with Option 1 if**:
- Reliability > 85% on WiFi (target met)
- User experience acceptable
- No major Syncthing-specific issues

**Decision Point**: After 3 months Phase 1 dogfooding

### Implementation Notes

**Option 1 Requirements**:
- [ ] Install Syncthing on macOS (Homebrew: `brew install syncthing`)
- [ ] Configure Syncthing to run as launch daemon (always-on)
- [ ] Setup vault folder as Syncthing shared folder
- [ ] Configure iOS Sushitrain to connect to macOS Syncthing
- [ ] Build `push-server` to monitor Syncthing events and trigger APNs

---

## ADR-002: Git Repository Structure (Monorepo vs Separate Repos)

**Date**: 2025-11-03
**Status**: DECIDED - Monorepo with Submodule
**Context**: Need to organize Sushitrain fork, plugin, and push-server

### Decision

Use **monorepo** structure with **git submodule** for Sushitrain fork.

### Options Considered

**Option 1: Monorepo (CHOSEN)**
```
obsidian-sync-bep/
├── sushitrain-fork/        # Git submodule
├── obsidian-plugin/        # TypeScript plugin
├── push-server/            # Rust APNs server
└── planning/               # Shared planning docs
```

**Pros**:
- ✅ Single clone for developers
- ✅ Coordinated releases (tag all components together)
- ✅ Shared planning documents
- ✅ Cross-component refactoring easier

**Cons**:
- ⚠️ Larger repository size
- ⚠️ Need to learn git submodule workflow

**Option 2: Separate Repositories**
```
obsidiansync-ios/           # Sushitrain fork
obsidiansync-plugin/        # Obsidian plugin
obsidiansync-server/        # Push notification server
obsidiansync-docs/          # Planning and documentation
```

**Pros**:
- ✅ Clean separation of concerns
- ✅ Independent versioning
- ✅ Easier to open-source specific components

**Cons**:
- ❌ Coordination overhead (4 repos to manage)
- ❌ Cross-component changes require multiple PRs
- ❌ Harder to track project-wide progress

### Decision Rationale

**Primary Reason**: **Single user (personal use) favors simplicity**

For personal dogfooding, benefits of monorepo (single clone, coordinated releases) outweigh separation benefits. If project goes community, can split later.

**Git Submodule for Sushitrain**:
- Cleanly track upstream changes
- Easy to update Sushitrain fork independently
- Clear separation between fork and custom components

### Implementation Commands

```bash
# Setup monorepo
mkdir ~/dev/tools/obsidian-sync-bep
cd ~/dev/tools/obsidian-sync-bep
git init

# Add Sushitrain as submodule
git submodule add https://github.com/pixelspark/sushitrain.git sushitrain-fork
git submodule update --init --recursive

# Track upstream Sushitrain
cd sushitrain-fork
git remote rename origin upstream
git remote add origin https://github.com/your-username/sushitrain-fork.git

# Update upstream
git fetch upstream
git checkout main
git pull upstream main
cd ..
git add sushitrain-fork
git commit -m "chore: update Sushitrain to latest upstream"
```

### Reevaluation Criteria

**Split into separate repos if**:
- Community contributors join (need independent component releases)
- Plugin distributed via Obsidian community (needs standalone repo)
- Maintenance overhead of monorepo becomes problematic

---

## ADR-003: File Provider Storage Strategy

**Date**: 2025-11-03
**Status**: DECIDED - App Group Shared Container
**Context**: File Provider extension needs access to synced vault files

### Decision

Use **App Group shared container** for vault storage, with **on-demand download** for large vaults.

### Architecture

```
┌─────────────────────────────────────────────┐
│          App Group Container                │
│  group.com.yourname.obsidiansync            │
├─────────────────────────────────────────────┤
│                                             │
│  vault/                  # Synced files    │
│  ├── notes/                                 │
│  ├── attachments/                           │
│  └── ...                                    │
│                                             │
│  metadata.db             # SQLite DB       │
│  ├── file_metadata       # NSFileProviderItem data │
│  ├── sync_state          # BEP index sequence │
│  └── thumbnail_cache     # Image previews  │
│                                             │
└─────────────────────────────────────────────┘
         ↑                      ↑
         │                      │
    Sushitrain App    File Provider Extension
```

### Options Considered

**Option 1: App Group Shared Container (CHOSEN)**

**Pros**:
- ✅ Standard iOS approach for app/extension data sharing
- ✅ Both app and extension can read/write vault files
- ✅ SQLite database for file metadata (fast enumeration)
- ✅ Supports on-demand download (fetch content when Files app requests)

**Cons**:
- ⚠️ Requires App Group entitlement (easy to setup)
- ⚠️ Storage limit depends on iOS device capacity

**Option 2: iCloud Drive Storage**

**Pros**:
- ✅ Automatic cloud backup
- ✅ Cross-device access

**Cons**:
- ❌ Introduces dependency on iCloud
- ❌ Conflicts with Syncthing sync (dual sync mechanisms)
- ❌ User may not have iCloud enabled

**Option 3: Direct File Access (No Shared Container)**

**Pros**:
- ✅ Simplest architecture

**Cons**:
- ❌ Extension cannot access app's Documents folder (iOS sandbox)
- ❌ Would require copying files between app and extension (wasteful)

### Decision Rationale

**Primary Reason**: File Provider extensions REQUIRE shared storage to access files synced by main app.

iOS sandbox prevents File Provider extension from accessing main app's Documents folder. App Group shared container is Apple's recommended approach for this use case.

### Implementation Details

**Entitlements Configuration**:
```xml
<!-- Sushitrain.entitlements -->
<key>com.apple.security.application-groups</key>
<array>
    <string>group.com.yourname.obsidiansync</string>
</array>

<!-- FileProviderExtension.entitlements -->
<key>com.apple.security.application-groups</key>
<array>
    <string>group.com.yourname.obsidiansync</string>
</array>
```

**Swift Code**:
```swift
// Get shared container URL
let containerURL = FileManager.default.containerURL(
    forSecurityApplicationGroupIdentifier: "group.com.yourname.obsidiansync"
)!

// Vault storage path
let vaultPath = containerURL.appendingPathComponent("vault")

// Metadata database
let dbPath = containerURL.appendingPathComponent("metadata.db")
```

**On-Demand Download Strategy**:
1. Store all file metadata in SQLite (fast enumeration)
2. Download file content only when Files app requests it
3. Implement LRU cache eviction for frequently accessed files
4. Background fetch for recently synced files (proactive caching)

### Performance Considerations

**Large Vault Optimization** (2000+ files):
- Enumerate metadata only (cheap - from SQLite)
- Download content on first access
- Cache recent files (keep 100 most accessed)
- Evict old files when storage > 80% capacity

**Thumbnail Strategy**:
- Leverage Sushitrain's existing thumbnail support
- Generate markdown previews on-demand
- Share thumbnail cache between app and extension

---

## ADR-004: Obsidian Plugin Distribution Strategy

**Date**: 2025-11-03
**Status**: DECIDED - Personal Use First, Community Later
**Context**: Determine how to distribute Obsidian Mobile plugin

### Decision

**Phase 1-3**: Personal use only (direct install from source)
**Phase 4+**: Consider Obsidian community plugin submission after dogfooding

### Options Considered

**Option 1: Personal Use Only (CHOSEN for now)**

**Installation**:
```bash
# Manual installation
cd /path/to/obsidian/vault/.obsidian/plugins/
git clone https://github.com/your-username/obsidiansync-plugin.git
cd obsidiansync-plugin
npm install
npm run build
```

**Pros**:
- ✅ No approval process
- ✅ Fast iteration during development
- ✅ Can test breaking changes freely
- ✅ Full control over release schedule

**Cons**:
- ⚠️ Manual installation required
- ⚠️ No automatic updates

**Option 2: Obsidian Community Plugin**

**Process**:
1. Submit to https://github.com/obsidianmd/obsidian-releases
2. Wait for review and approval
3. Listed in Obsidian's Community Plugins browser

**Pros**:
- ✅ Easy discovery for other users
- ✅ Automatic updates via Obsidian
- ✅ Community feedback and contributions

**Cons**:
- ❌ Approval process delays releases
- ❌ Must follow Obsidian plugin guidelines strictly
- ❌ Breaking changes require careful versioning

### Decision Rationale

**Primary Reason**: **Dogfooding phase needs flexibility**

During personal use phase, need to iterate quickly without approval delays. Plugin is non-critical enhancement (core sync works without it).

**Timeline for Community Release**:
- After 3 months successful Phase 3 dogfooding
- Once plugin is stable and well-tested
- If there's community interest (e.g., feedback from users of similar setups)

### Reevaluation Criteria

**Submit to Obsidian community if**:
- Guillaume uses plugin successfully for 3+ months
- No major architectural issues identified
- Community expresses interest (e.g., GitHub stars, requests)
- Plugin provides clear value beyond personal use case

---

## ADR-005: APNs Push Notification Throttling Strategy

**Date**: 2025-11-03
**Status**: DECIDED - 30-Minute Batching Window
**Context**: APNs has ~5 push/day hard limit, need to balance responsiveness vs throttling

### Decision

**Default**: 30-minute minimum between silent push notifications
**User Configurable**: Allow "immediate sync" mode (5-minute window) with battery warning

### Options Considered

**Option 1: Aggressive (5-minute window)**
- Max 12 pushes/hour → likely to hit daily limit quickly
- Better responsiveness
- Higher battery usage

**Option 2: Conservative (30-minute window) (CHOSEN)**
- Max 48 pushes/day → well below APNs limit
- Acceptable latency for most workflows
- Lower battery impact

**Option 3: Very Conservative (1-hour window)**
- Max 24 pushes/day
- Too slow for responsive sync experience

### Decision Rationale

**Primary Reason**: **Balance responsiveness with APNs limits**

Analysis from `ios-sync-integration-analysis.md`:
- Official recommendation: 2-3 notifications/hour
- Hard limit: ~5/device/day
- Over-sending = APNs drops notifications

**30-minute window**:
- Keeps well below limits (48 theoretical max vs 5 hard limit is safe margin)
- Typical workflow: Edit notes → 30 min later → iOS synced
- User can trigger manual sync if urgent

### Implementation

```rust
// push-server/src/change_monitor.rs

struct PushThrottler {
    last_push: Instant,
    min_interval: Duration,  // Default: 30 minutes
}

impl PushThrottler {
    fn should_push(&mut self, changes: &[FileEvent]) -> bool {
        let elapsed = self.last_push.elapsed();

        if elapsed < self.min_interval {
            // Batch for next window
            return false;
        }

        // Allow push
        self.last_push = Instant::now();
        true
    }
}
```

**User Preference** (Obsidian plugin settings):
```json
{
  "sync_mode": "balanced",  // "immediate" | "balanced" | "conservative"
  "sync_interval_minutes": 30,
  "wifi_only": true,
  "battery_saver": false
}
```

### Modes

**Immediate Mode** (5-minute window):
- For users who need low-latency sync
- Warning: "Higher battery usage, may hit daily push limit"

**Balanced Mode** (30-minute window, DEFAULT):
- Good trade-off for most users
- Recommendation for personal use

**Conservative Mode** (1-hour window):
- For battery-conscious users
- Use BGProcessingTask for overnight sync

### BGProcessingTask Fallback

If silent push fails 3 times:
```swift
// Schedule overnight sync
BGTaskScheduler.shared.submit(
    BGProcessingTaskRequest(identifier: "nl.yourname.obsidiansync.full-sync")
)
```

---

## ADR-006: Network Quality Adaptive Batching

**Date**: 2025-11-03
**Status**: DECIDED - Dynamic Batch Sizing Based on Network Type
**Context**: iOS 30-second timeout requires different batch sizes for different networks

### Decision

Implement **adaptive batch sizing** based on real-time network quality detection.

### Batch Size Strategy

From reliability matrix (see `ios-sync-integration-analysis.md`):

| Network | Latency | Recommended Batch Size | Expected Success Rate |
|---------|---------|------------------------|----------------------|
| WiFi    | Low     | 2000 files            | 90%                  |
| WiFi    | High    | 1000 files            | 90%                  |
| LTE     | Low     | 1000 files            | 90%                  |
| LTE     | High    | 500 files             | 85%                  |
| 3G      | Any     | 500 files             | 70%                  |

### Implementation

```swift
// iOS App: Network Quality Detection

class NetworkQualityDetector {
    func estimateBatchSize() -> Int {
        let networkType = detectNetworkType()  // WiFi, LTE, 3G
        let latency = measureLatency()         // Ping server

        switch (networkType, latency) {
        case (.wifi, .low):  return 2000
        case (.wifi, .high): return 1000
        case (.lte, .low):   return 1000
        case (.lte, .high):  return 500
        case (.cellular, _): return 500
        default:             return 500  // Conservative fallback
        }
    }

    private func measureLatency() -> Latency {
        // Ping Syncthing server, measure RTT
        // < 100ms = low, >= 100ms = high
    }
}
```

**Sync Flow**:
```swift
func handleSilentPush(payload: APNsPayload) {
    let fileCount = payload.syncHint.fileCount
    let maxBatch = NetworkQualityDetector().estimateBatchSize()

    if fileCount <= maxBatch {
        // Sync all files
        syncManager.syncAll()
    } else {
        // Progressive sync: prioritize recent changes
        let priorityFiles = syncManager.getPriorityFiles(limit: maxBatch)
        syncManager.syncFiles(priorityFiles)

        // Defer remaining to next wake
        syncManager.saveCheckpoint(afterFile: maxBatch)
    }
}
```

### Progressive Sync Priority

**Tier 1** (sync first):
- Files modified in last 24 hours
- Files currently open in Obsidian
- Files in "daily notes" folder

**Tier 2** (sync if time permits):
- Files modified in last week
- Frequently accessed files

**Tier 3** (defer to next sync):
- Old files (> 1 month unchanged)
- Archive folders

### Decision Rationale

**Primary Reason**: **30-second timeout is non-negotiable**, must adapt to network conditions

Static batch size (always 2000) fails on poor networks (40% success on LTE high latency). Adaptive approach maintains > 85% success rate across network types.

**Trade-off**: Complexity vs reliability. Worth the implementation cost.

---

## ADR-007: Testing Strategy (TDD Approach)

**Date**: 2025-11-03
**Status**: DECIDED - TDD Unit-First with Integration Tests
**Context**: Align with vault-server testing methodology

### Decision

Follow **TDD unit-first** approach (same as vault-server):
1. Write unit tests before implementation
2. Integration tests for iOS APIs and BEP protocol
3. BDD tests for user workflows

### Test Pyramid

```
        ┌─────────────┐
        │  BDD Tests  │  ← User workflows (slow, few)
        │   (Few)     │
        └─────────────┘
       ┌───────────────┐
       │ Integration   │  ← iOS APIs, BEP (medium speed, moderate)
       │     Tests     │
       └───────────────┘
      ┌─────────────────┐
      │   Unit Tests    │  ← Pure functions (fast, many)
      │     (Many)      │
      └─────────────────┘
```

### Rationale

From vault-server learnings (see `vault-server/planning/decision-journal.md` - ADR on test pyramid):

**Unit tests provide design pressure**:
- Force extraction of pure functions
- Prevent coupling to HTTP/iOS frameworks
- Fast feedback loop

**Integration tests validate boundaries**:
- iOS File Provider API behavior
- BEP protocol compliance
- APNs push delivery

**BDD tests enforce user perspective**:
- "I want to sync vault" (not "BEP IndexUpdate returns 200")
- Catch usability issues

### Test Examples

**Unit Test** (Rust - push-server):
```rust
#[test]
fn throttler_should_batch_frequent_changes() {
    let mut throttler = PushThrottler::new(Duration::from_secs(1800)); // 30 min

    assert!(throttler.should_push(&changes1)); // First push allowed
    assert!(!throttler.should_push(&changes2)); // Batched (< 30 min)

    std::thread::sleep(Duration::from_secs(1801));
    assert!(throttler.should_push(&changes3)); // Next window
}
```

**Integration Test** (Swift - File Provider):
```swift
func testFileProviderEnumerateVault() async throws {
    // Setup: Sync 100 files via BEP
    let files = await syncTestVault(fileCount: 100)

    // Exercise: Enumerate via File Provider API
    let enumerator = try fileProvider.enumerator(for: .rootContainer)
    let items = try await enumerator.enumerateItems()

    // Verify: All files enumerated
    XCTAssertEqual(items.count, 100)
    XCTAssertEqual(Set(items.map(\.filename)), Set(files.map(\.name)))
}
```

**BDD Test** (Gherkin - user workflow):
```gherkin
Feature: Automatic iOS Vault Sync

  Scenario: Desktop edit syncs to iOS
    Given I have Obsidian vault with 100 notes
    And iOS app is configured and connected
    When I edit "daily-note.md" on Desktop
    And I wait 5 minutes
    Then iOS app receives sync notification
    And "daily-note.md" is updated on iOS
    And Obsidian Mobile shows latest content
```

### Test Coverage Targets

**Phase 1** (Core Sync):
- Unit tests: 80%+ coverage (Rust push-server)
- Integration tests: All critical paths (APNs, BEP handshake, file sync)
- BDD tests: 3-5 key workflows

**Phase 2** (File Provider):
- Integration tests: All File Provider API methods
- Performance tests: 2000-file enumeration < 1 second

**Phase 3** (Obsidian Plugin):
- Integration tests: Plugin API integration
- Manual testing: UI workflows (no automated UI tests for Obsidian plugins)

---

## Future ADRs (To Be Written)

**ADR-008**: Conflict Resolution Strategy (Phase 2)
- How to detect conflicts?
- UI for conflict resolution?
- Automatic merge strategies?

**ADR-009**: Performance Monitoring (Phase 3)
- Metrics to track (sync latency, battery usage)
- Monitoring infrastructure
- Alerting thresholds

**ADR-010**: Community Release Strategy (Phase 4+)
- Open source license considerations
- Contribution guidelines
- Support model

---

## Decision Changelog

| ADR | Date | Decision | Status |
|-----|------|----------|--------|
| ADR-001 | 2025-11-03 | Syncthing Wrapper vs Custom BEP | DECIDED - Syncthing |
| ADR-002 | 2025-11-03 | Monorepo vs Separate Repos | DECIDED - Monorepo |
| ADR-003 | 2025-11-03 | File Provider Storage | DECIDED - App Group |
| ADR-004 | 2025-11-03 | Plugin Distribution | DECIDED - Personal Use |
| ADR-005 | 2025-11-03 | APNs Throttling | DECIDED - 30min window |
| ADR-006 | 2025-11-03 | Adaptive Batching | DECIDED - Network-based |
| ADR-007 | 2025-11-03 | Testing Strategy | DECIDED - TDD Unit-First |
