# Product Framing: ObsidianSync-BEP

**Project Type**: Personal Productivity Tool → Potential Community Release
**Status**: Pre-Development (Architecture & Planning Phase)
**User**: Single user (Guillaume) initially, then community dogfooding
**License**: MPL 2.0 (inherited from Sushitrain fork)

---

## Product Vision

**What**: An alternative vault server for Obsidian Mobile that syncs vaults using self-hosted BEP protocol (Syncthing) infrastructure, with File Provider integration and optional Obsidian Mobile plugin.

**Why**: Own your sync infrastructure without subscription fees. Smooth note-taking flow across devices with responsive, automatic vault synchronization.

**For Whom**:
- Primary: Guillaume (personal use, dogfooding)
- Secondary: Obsidian + iOS users seeking free sync alternative
- Tertiary: Open source community (MPL 2.0 fork of Sushitrain)

**Unique Value**:
- **Self-Hosted**: Own your sync infrastructure, no subscription dependency
- **Free & Open Source**: MPL 2.0 license (fork of Sushitrain)
- **iOS Files App Integration**: Native Files app support via File Provider for on-demand file access
- **Responsive Sync**: 1-minute sync intervals enable smooth cross-device note-taking
- **Obsidian Integration**: Optional plugin for sync status and controls
- **Proven Foundation**: Built on Sushitrain (production-ready BEP client)

---

## Problem Statement

**Current Pain Points**:
1. **Cost**: Obsidian Sync costs $10/month ($120/year)
2. **Lock-in**: Proprietary sync mechanism, no self-hosting option
3. **Manual Workflow**: iOS requires Obsidian Sync or manual iCloud/Dropbox management
4. **Limited Control**: Cannot customize sync behavior or infrastructure

**User Story**:
> "As an Obsidian power user with iOS devices, I want automatic vault sync without paying for Obsidian Sync, so I can control my own infrastructure and avoid subscription fees."

---

## Solution Approach

### High-Level Architecture

```
Desktop:    Obsidian Desktop ←→ Syncthing Desktop ←→ [BEP Protocol]
                                        ↓
                                 File System Monitor
                                        ↓
                                  APNs Push Server

iOS:        Sushitrain Fork (BEP client) ←→ Local Vault Storage
                    ↓                              ↓
            File Provider Extension      ←→  iOS Files App
                                                   ↓
                                          Obsidian Mobile
                                                   ↓
                                    ObsidianSync Plugin (optional)
```

### Core Components

1. **Sushitrain Fork** (iOS BEP client)
   - Handles BEP protocol sync with macOS Syncthing
   - Responds to APNs silent push notifications
   - Manages local vault storage

2. **File Provider Extension** (iOS)
   - Exposes synced vault to iOS Files app
   - Enables Obsidian Mobile to open vault natively
   - Real-time updates when sync completes

3. **Push Notification Server** (macOS/server-side)
   - Monitors file system for vault changes
   - Sends APNs silent push to iOS when changes detected
   - Throttles notifications to respect APNs limits

4. **Obsidian Mobile Plugin** (optional enhancement)
   - Displays sync status within Obsidian
   - Manual sync trigger
   - Conflict resolution UI
   - Settings integration

---

## Definition of Done

### Phase 1: MVP - Core Sync

**Must Have**:
1. **Automatic Sync**
   - Vault changes on Desktop trigger iOS sync within 1 minute
   - Success rate > 85% on WiFi and cellular
   - No data loss or corruption for 15,000-file vault
   - Incremental changes (5-50 files) sync efficiently

2. **iOS Integration**
   - APNs silent push wakes iOS app
   - BEP sync completes within 30-second background limit for typical changes
   - State persistence for resume on timeout
   - Both WiFi and cellular networks supported

3. **Battery Efficiency**
   - Battery drain < 5% per day from background sync
   - Throttled push notifications (max 1 per minute for responsive feel)

**Won't Have (Phase 1)**:
- File Provider integration (Phase 2)
- Obsidian plugin (Phase 3)
- Custom BEP server (Phase 4+, only if needed)

**Success Metrics**:
- Guillaume uses it daily for 2+ weeks without issues
- Sync reliability matches or exceeds Obsidian Sync experience
- No manual intervention needed for typical workflow
- Responsive cross-device experience (smooth note-taking flow)

---

### Phase 2: File Provider Integration

**Must Have**:
1. **Files App Exposure**
   - Synced vault visible in iOS Files app
   - Obsidian Mobile can open vault from Files
   - File changes reflected in Files app within 5 seconds

2. **On-Demand Download Strategy**
   - Lazy loading for large vaults (15,000 files total)
   - Metadata index sync completes in background
   - Individual files downloaded on-demand when accessed
   - Intelligent caching for frequently accessed files
   - LRU cache management for device storage constraints

3. **Performance**
   - Thumbnail generation for markdown previews
   - File browser responsive even with 15,000+ files
   - ~50 MB max file handling without timeout

**Success Metrics**:
- Obsidian Mobile uses Files app as primary vault access method
- No performance degradation vs. Obsidian Sync
- Native iOS experience
- Large vault accessible without storing all files locally

---

### Phase 3: Obsidian Plugin

**Must Have**:
1. **Sync Status Display**
   - Status bar indicator
   - Detailed sync view (files in sync, last sync time)
   - Conflict detection and display

2. **User Controls**
   - Manual sync trigger
   - Settings for sync preferences (cellular, WiFi, etc.)

**Success Metrics**:
- Guillaume rarely needs to leave Obsidian to check sync status
- Conflicts easily identified and resolved
- Clarify: Does plugin provide user-facing controls, or is it informational only?

---

## Scope

### In Scope (v1.0 - Phase 1-3)

**Core Sync**:
- BEP protocol sync (Syncthing wrapper approach)
- APNs silent push notifications
- Background sync with iOS 30-second constraint handling
- File system monitoring and change detection
- Device token management

**iOS Integration**:
- File Provider extension
- iOS Files app integration
- Obsidian Mobile compatibility

**Obsidian Enhancement**:
- Mobile plugin for sync status and controls
- Conflict resolution UI
- Manual sync trigger

### Out of Scope (Defer/Never)

**Deferred to Phase 4+**:
- Custom BEP server implementation (only if Syncthing wrapper insufficient)
- Advanced conflict resolution strategies
- Multi-device orchestration (beyond Syncthing's capabilities)

**Explicitly NOT Building**:
- Android support (different architecture, separate project)
- Web-based sync interface
- Windows/Linux iOS sync (macOS-only for personal use)
- Multi-user/multi-vault sync management
- Automated vault backups (trust Syncthing's versioning)

---

## Technical Constraints

### iOS Background Execution Limits

**Critical Constraint**: iOS terminates background tasks after 30 seconds

**Vault Scale**: 15,000 total files (half markdown, half images/PDFs)
- **Largest file**: ~50 MB max
- **Typical changes**: 5-50 files per sync cycle (incremental)
- **Initial sync challenge**: One-time metadata index sync for 15,000 files
- **Incremental sync**: Small file counts allow 30-second completion window

**Implications**:
- Initial metadata sync requires strategic batching or background task scheduling
- Incremental syncs (typical case) complete within 30-second window
- Reliability depends heavily on network quality:
  - WiFi (low latency): 90% success rate
  - LTE/Cellular (normal latency): 75% success rate
  - Poor network: 40% success rate
- Mitigation: Adaptive batch sizing, state persistence, resume on next wake

**Reference**: See `/Users/guillaume/dev/tools/vault-server/planning/ios-sync-integration-analysis.md` for detailed analysis

### APNs Throttling & Sync Latency

**Official Limit**: ~2-3 notifications per hour recommended
**Hard Limit**: ~5 per device per day

**Decision**: Use 1-minute aggressive sync (1-minute intervals for batching changes)
- Rationale: Smooth note-taking flow from one device to next, sometimes simultaneously
- Aligns with user preference for responsive cross-device experience

**Implications**:
- 1-minute batching window allows responsive sync without APNs rate-limit violation
- Multiple changes batched into single push notification
- Individual file changes visible within 1-2 minutes

**Mitigation**:
- Batch multiple changes into single push per 1-minute window
- BGProcessingTask fallback for overnight sync when app backgrounded
- Intelligent batching prevents APNs exhaustion

---

## Open Questions

### Technical Decisions Needed

**Q1: Syncthing vs Custom BEP Server?**
- **Option 1 (Recommended)**: Syncthing wrapper + APNs push (lower risk)
- **Option 2 (Future consideration)**: Custom BEP server in Rust (higher risk)
- **Decision**: Start with Option 1, evaluate after 3 months dogfooding

**Q2: File Provider Storage Strategy? [DECIDED]**
- **Decision**: On-demand download with intelligent caching
- Rationale: 15,000-file vault cannot fit on typical iOS device
- All files sync (metadata tracked), but not all downloaded locally
- Individual files fetched when accessed via Files app
- LRU cache management for device storage constraints

**Q3: Plugin Distribution?**
- **Personal Use**: Direct install from source
- **Community Release**: Obsidian community plugin submission after validation
- **Decision**: Personal use for Phase 1-3, consider community release after dogfooding

### Clarifications Needed

**Q1: Plugin Scope?**
- Does plugin provide user-facing configuration controls (e.g., sync frequency preferences)?
- Or is it primarily informational (sync status display, manual trigger)?
- This affects Phase 3 scope and complexity.

**Q2: Network Strategy? [DECIDED]**
- **Decision**: Support both WiFi and cellular
- No WiFi-only default (user wants smooth cross-device experience on any network)

---

## Success Criteria (Personal Dogfooding)

### Phase 1 Success (Core Sync)
- [ ] Guillaume uses ObsidianSync-BEP as primary iOS sync for 2+ weeks
- [ ] Zero data loss or corruption incidents
- [ ] Sync reliability subjectively "as good as Obsidian Sync"
- [ ] Battery usage acceptable (< 5% per day)
- [ ] No manual intervention needed for typical daily workflow
- [ ] 1-minute sync latency provides smooth cross-device note-taking experience

### Phase 2 Success (File Provider)
- [ ] Obsidian Mobile vault opened exclusively via Files app
- [ ] File changes in Obsidian reflected in Files app within 5 seconds
- [ ] Performance acceptable for 15,000-file vault
- [ ] On-demand download works seamlessly (files loaded as needed)
- [ ] No UX degradation vs. Obsidian Sync workflow
- [ ] LRU cache prevents device storage exhaustion

### Phase 3 Success (Obsidian Plugin)
- [ ] Sync status checked within Obsidian (not switching apps)
- [ ] Conflicts identified and resolved without leaving Obsidian
- [ ] Manual sync trigger works when needed
- [ ] Plugin provides expected controls/information (clarify scope)

### Long-Term Success (6+ months)
- [ ] Continuously used for 6+ months without reverting to Obsidian Sync
- [ ] Community interest warranting public release
- [ ] No major architectural issues blocking future enhancements
- [ ] Total cost (time + infrastructure) < $120/year (Obsidian Sync cost)

---

## Risks & Mitigation

### High-Priority Risks

**RISK-1: iOS 30-second Background Limit**
- **Impact**: Incomplete syncs, inconsistent vault state
- **Probability**: 15-30% on poor networks
- **Mitigation**: Adaptive batch sizing, state persistence, resume logic, network quality detection

**RISK-2: APNs Throttling**
- **Impact**: Delayed sync (hours), poor UX
- **Probability**: HIGH for frequent vault editors
- **Mitigation**: Batch changes (30-min window), BGProcessingTask fallback, user notification of delayed sync

**RISK-3: Syncthing Dependency**
- **Impact**: Must run Syncthing on macOS (always-on requirement)
- **Probability**: Certain (architectural requirement)
- **Mitigation**: Document setup clearly, provide automated installer, fallback to custom BEP server if Syncthing proves problematic

### Medium-Priority Risks

**RISK-4: File Provider Complexity**
- **Impact**: Bugs, performance issues, iOS compatibility problems
- **Probability**: MEDIUM (standard but complex iOS API)
- **Mitigation**: Study Apple samples, extensive testing, prototype early

**RISK-5: Obsidian Mobile API Limitations**
- **Impact**: Plugin features limited by Obsidian Mobile API
- **Probability**: MEDIUM (mobile API is subset of desktop)
- **Mitigation**: Verify API capabilities early, design within constraints, graceful degradation

**RISK-6: Battery Drain**
- **Impact**: User disables app, negative experience
- **Probability**: LOW-MEDIUM (depends on sync frequency)
- **Mitigation**: Aggressive throttling, WiFi-only preference, monitor battery usage, adjust strategy

### Low-Priority Risks

**RISK-7: Large File Handling**
- **Impact**: Timeout during large file sync
- **Probability**: LOW (typical Obsidian vault = small markdown files)
- **Mitigation**: Detect large files, warn user, defer to BGProcessingTask

**RISK-8: Upstream Sushitrain Changes**
- **Impact**: Breaking changes in Sushitrain requiring fork updates
- **Probability**: LOW (stable project, infrequent updates)
- **Mitigation**: Git submodule strategy, track upstream, test before merging

---

## Development Phases

### Phase 1: Core Sync
- Fork Sushitrain, project setup
- APNs infrastructure (server-side)
- File system monitoring & push trigger logic
- iOS app modifications (push handling, sync optimization)
- Integration testing, reliability validation

### Phase 2: File Provider Integration
- Extension target setup, App Group configuration
- On-demand download architecture
- Storage bridge implementation
- Real-time update mechanism
- Obsidian Mobile integration testing
- Cache management and optimization

### Phase 3: Obsidian Plugin
- Plugin scaffold, basic UI
- Sync status display and monitoring
- Manual sync trigger, configuration controls
- Conflict detection and reporting

### Phase 4+: Optimization (Ongoing)
- Evaluate after 3 months Phase 3 dogfooding
- Data-driven decisions on custom BEP server, advanced features
- Community feedback incorporation

---

## Organization & Methodology

### Project Team (Same as vault-server)

**Core Team** (systematic consultation):
- **solution-architect**: Architecture design, implementation planning
- **developer**: Code implementation (Rust, Swift, TypeScript)
- **integration-specialist**: BEP protocol, iOS API integration
- **performance-optimizer**: Sync optimization, battery efficiency (consult as needed)

**Support Team** (on-demand):
- **code-quality-analyst**: Code reviews, architecture validation
- **git-workflow-manager**: Commit management, PR creation

**Collaboration Pattern**:
1. Solution-architect designs architecture (this document)
2. Integration-specialist validates iOS/BEP integration feasibility
3. Developer implements with appropriate model (Haiku for simple, Sonnet for complex)
4. Git-workflow-manager handles all commits (atomic, clean history)

### Development Workflow

**Delegation Protocol**:
- ALWAYS delegate to specialized agents (even simple tasks)
- Use Haiku for well-defined implementation, Sonnet for architecture/design
- Standard flow: Architecture → Implementation → Commit

**Git Strategy**:
- All commits via git-workflow-manager agent
- Feature branches for each phase
- Clean atomic commits (one concern per commit)

**Testing Strategy**:
- TDD unit-first for pure functions
- Integration tests for iOS APIs, BEP protocol
- BDD tests for user workflows (vault sync, file access)

---

## Project Structure

```
~/dev/tools/obsidian-sync-bep/
├── planning/
│   ├── framing.md                  # This document
│   ├── decision-journal.md         # ADRs
│   ├── implementation-plan.md      # Phased rollout
│   ├── backlog.md                  # Feature backlog
│   └── ios-sync-analysis.md        # Technical deep-dive
├── sushitrain-fork/                # Git submodule
├── obsidian-plugin/                # TypeScript plugin
├── push-server/                    # Rust APNs server
└── docs/                           # User documentation
```

---

## Next Steps

1. **Create Project Repository**
   - Initialize git repository
   - Setup git submodule for Sushitrain
   - Create planning documents
   - Initial commit

2. **Apple Developer Setup**
   - Ensure Apple Developer account available ($99/year)
   - Generate APNs authentication key
   - Register App ID and capabilities

3. **Prototype APNs Push**
   - Minimal Rust server sending test push
   - iOS app receiving silent push
   - Validate 30-second execution window

4. **Technical Validation**
   - Test Sushitrain with macOS Syncthing
   - Measure sync time across vault sizes (focus on 15,000-file scenario)
   - Validate network reliability matrix (WiFi, cellular, poor network)
   - Confirm iOS background behavior and metadata sync approach

5. **Phase 1 Implementation**
   - Follow implementation plan
   - Regular dogfooding checkpoints
   - Adjust based on empirical findings

---

## References

- **Technical Analysis**: `/Users/guillaume/dev/tools/vault-server/planning/ios-sync-integration-analysis.md`
- **Sushitrain GitHub**: https://github.com/pixelspark/sushitrain
- **BEP Protocol Spec**: https://docs.syncthing.net/specs/bep-v1.html
- **iOS File Provider**: https://developer.apple.com/documentation/fileprovider
- **Obsidian Plugin API**: https://docs.obsidian.md/Plugins/Getting+started/Build+a+plugin
