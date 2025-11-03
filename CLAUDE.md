# ObsidianSync-BEP - Project Guidelines

## Project Type

**Personal Productivity Tool** (iOS App Development)
- Goal: Self-hosted BEP sync for Obsidian Mobile as alternative to Obsidian Sync
- User: Single user (Guillaume) - dogfooding tool for personal workflow
- Status: Pre-Development (architecture complete, ready for implementation)
- License: MPL 2.0 (fork of Sushitrain)

See [planning/framing.md](planning/framing.md) for complete product vision and roadmap.

## Project Overview

**Architecture**:
```
Desktop: Obsidian ←→ Syncthing ←→ [BEP Protocol] ←→ iOS: Sushitrain Fork
                          ↓
                    Push Server (Rust) ←→ APNs ←→ iOS Background Wake
                                                        ↓
                                              File Provider Extension
                                                        ↓
                                                 Obsidian Mobile
```

**Components**:
1. **Sushitrain fork** (Swift + Go): BEP client for iOS
2. **File Provider extension** (Swift): Expose vault to iOS Files app
3. **Push server** (Rust): Monitor changes, send APNs notifications
4. **Obsidian plugin** (TypeScript): Optional sync status/controls

**Key Constraint**: 15,000 files in vault (half markdown, half images/PDFs)
- Incremental sync: 5-50 files typical (fits 30-second iOS limit)
- Initial sync: Metadata only, files downloaded on-demand

## Development Workflow

### Team Delegation Protocol

**ALWAYS delegate to specialized agents** for ALL tasks (even simple ones)

**Core Team**:
- **solution-architect**: iOS architecture, APNs design, File Provider patterns
- **developer**: Swift/Go/Rust/TypeScript implementation
- **integration-specialist**: BEP protocol, iOS APIs, File Provider integration
- **git-workflow-manager**: ALL commits (via git-workflow skill)

**Standard Flow**: Architecture → Implementation → Commit

### Model Selection

Use appropriate model based on task complexity:
- **Haiku**: Well-defined Swift UI, straightforward Rust server code
- **Sonnet**: iOS File Provider architecture, BEP protocol integration, APNs design

### Git Practices

**Fork Management**:
- Sushitrain as git submodule (track upstream cleanly)
- Feature branches for each phase
- Commits via git-workflow-manager agent only

**Upstream Tracking**:
```bash
# Sushitrain upstream
cd sushitrain-fork
git remote add upstream https://github.com/pixelspark/sushitrain.git
git fetch upstream
```

**Commit Format**:
- `feat:` for new features
- `fix:` for bug fixes
- `refactor:` for code improvements
- `docs:` for documentation

## Development Principles

### iOS-Specific Standards

**Swift Conventions**:
- Follow Apple Swift style guide
- Use `async/await` for concurrency (not completion handlers)
- Protocol-oriented design where appropriate
- Descriptive names (readability > brevity)

**File Provider Best Practices**:
- Lazy enumeration for large vaults
- On-demand download with intelligent caching
- LRU cache management
- Error handling for network failures

**APNs Silent Push**:
- Content-available: 1 (silent push)
- Max 1 push per minute (1-minute batching)
- Fallback to BGProcessingTask for overnight sync

### Go (SushitrainCore) Conventions

- Keep Go code minimal (Syncthing wrapper only)
- Don't modify Syncthing core unless absolutely necessary
- Document any gomobile bridge changes

### Rust (Push Server) Conventions

- Follow vault-server patterns (same developer)
- Use `tokio` for async runtime
- Error handling with `anyhow`
- Configuration via environment variables

### Testing Strategy

**TDD Unit-First**:
- Write unit tests for pure functions before integration tests
- Test pyramid: Unit → Integration → BDD

**iOS-Specific Testing**:
- XCTest for Swift unit tests
- UITest for File Provider integration
- TestFlight for real-device validation

**Critical Tests**:
- APNs push reception (silent push wakes app)
- 30-second timeout handling (state persistence)
- File Provider on-demand download
- 15,000-file vault metadata sync

## Git Commit Protocol

**ALL commits MUST use git-workflow-manager agent** (via git-workflow skill)

**Exception**: None - even single-file markdown changes go through git specialist

**Why**: Prevents partial commits, ensures atomic history, validates build

## Project Structure

```
~/dev/tools/obsidian-sync-bep/
├── CLAUDE.md                       # This file
├── planning/
│   ├── framing.md                  # Product vision
│   ├── decision-journal.md         # Architecture Decision Records
│   └── implementation-plan.md      # Phased execution plan
├── sushitrain-fork/                # Git submodule
│   ├── Sushitrain/                 # iOS app (Swift)
│   ├── FileProviderExtension/      # NEW: Files integration
│   └── SushitrainCore/             # BEP engine (Go)
├── push-server/                    # Rust APNs server
│   ├── src/
│   │   ├── main.rs
│   │   ├── apns_client.rs
│   │   ├── change_monitor.rs
│   │   └── device_registry.rs
│   └── Cargo.toml
├── obsidian-plugin/                # TypeScript plugin
│   ├── src/
│   │   ├── main.ts
│   │   └── sync-status.ts
│   └── package.json
└── docs/                           # User documentation
    ├── setup-guide.md
    └── troubleshooting.md
```

## API Scope

**In Scope (Phases 1-3)**:
- BEP sync via Syncthing wrapper
- APNs silent push notifications
- File Provider extension (on-demand download)
- Obsidian plugin (sync status/controls)

**Deferred to Phase 4+**:
- Custom BEP server (only if Syncthing insufficient)
- Advanced conflict resolution
- Selective sync

**Explicitly NOT Building**:
- Android support (different architecture)
- Web-based sync interface
- Multi-user/multi-vault management

## Key Technical Constraints

### iOS Background Execution

**30-second limit** for silent push background tasks
- Typical sync (5-50 files): < 5 seconds ✅
- Large batch (1000+ files): May timeout ⚠️
- Mitigation: State persistence, resume on next wake

**Battery Budget**:
- Target: < 5% per day
- 1-minute push intervals (responsive sync)
- WiFi + cellular supported

### APNs Throttling

**Hard limit**: ~5 pushes/device/day
**Strategy**: 1-minute batching window
- Multiple vault changes → single push per minute
- Prevents APNs exhaustion
- Provides responsive sync feel

### Vault Scale

**15,000 files total**:
- Half markdown, half images/PDFs
- Largest file: ~50 MB
- **Critical**: File Provider on-demand download essential at this scale

## Development Phases

### Phase 1: Core Sync
- Fork Sushitrain, rebrand
- APNs infrastructure (Rust server)
- iOS background sync handler
- Dogfooding: 2 weeks

### Phase 2: File Provider Integration
- Extension target setup
- On-demand download architecture
- LRU cache management
- Dogfooding: 1 week

### Phase 3: Obsidian Plugin
- Plugin scaffold
- Sync status display
- Manual sync trigger
- Dogfooding: 2 weeks

**Decision Point**: Evaluate Phase 4 (custom BEP server) after 3 months dogfooding

## Success Criteria

**Personal Dogfooding**:
- [ ] Daily use for 2+ weeks per phase
- [ ] Zero data loss/corruption
- [ ] Sync reliability ≥ Obsidian Sync experience
- [ ] Battery usage < 5% per day
- [ ] Smooth cross-device note-taking (1-minute sync latency)

**Long-Term** (6+ months):
- [ ] Continuous use without reverting to Obsidian Sync
- [ ] No architectural blockers for future enhancements

## Project Learnings

### 2025-11-03 - Architecture & Planning

**Methodological: Agent Over-Delegation Anti-Pattern**
- Used agent for simple text edits based on user feedback
- Delegation protocol is for architecture/design, not find-replace
- **Learning**: Do simple content edits directly, delegate complexity

**Methodological: Project Folder Discipline**
- Initially created planning docs in vault-server folder
- Polluted unrelated project with ObsidianSync planning
- **Learning**: Create dedicated project folder from day 1

**Technical: 15K File Vault Scale**
- User clarified: 15K total files, not 15K changes per sync
- Initial sync = metadata only (one-time challenge)
- Incremental sync = 5-50 files (typical case)
- **Impact**: File Provider on-demand download is CRITICAL, not optional

## References

- **Framing**: [planning/framing.md](planning/framing.md)
- **Sushitrain**: https://github.com/pixelspark/sushitrain
- **BEP Spec**: https://docs.syncthing.net/specs/bep-v1.html
- **File Provider**: https://developer.apple.com/documentation/fileprovider
- **Obsidian Plugin API**: https://docs.obsidian.md/Plugins/Getting+started/Build+a+plugin
