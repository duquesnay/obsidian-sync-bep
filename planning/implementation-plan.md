# Implementation Plan: ObsidianSync-BEP

**Purpose**: Phased rollout plan with incremental value delivery

**Timeline**: 14 weeks total (7 + 4 + 3) to complete MVP through Phase 3

**Methodology**: Each phase is independently shippable and delivers user value

---

## Pre-Phase: Project Setup (Week 0)

### Goals
- [ ] Create project repository
- [ ] Setup development environment
- [ ] Apple Developer Account configured
- [ ] Initial planning documents committed

### Tasks

**Day 1: Repository Initialization**
```bash
# Create project
mkdir ~/dev/tools/obsidian-sync-bep
cd ~/dev/tools/obsidian-sync-bep

# Initialize git
git init
git remote add origin https://github.com/your-username/obsidian-sync-bep.git

# Add Sushitrain as submodule
git submodule add https://github.com/pixelspark/sushitrain.git sushitrain-fork
git submodule update --init --recursive

# Create structure
mkdir -p planning docs push-server obsidian-plugin

# Symlink CLAUDE.md
ln -s ~/.claude-memories/claude_mds/dev/tools/obsidian-sync-bep/CLAUDE.md CLAUDE.md

# Planning documents already in place (framing.md, decision-journal.md, implementation-plan.md)

# Initial commit
git add .
git commit -m "chore: initial project structure with planning documents"
git push -u origin main
```

**Day 2: Apple Developer Setup**
- [ ] Purchase Apple Developer Account ($99/year) if not existing
- [ ] Create App ID: `com.yourname.obsidiansync`
- [ ] Enable capabilities:
  - [x] App Groups (`group.com.yourname.obsidiansync`)
  - [x] Push Notifications
  - [x] File Provider Extension
- [ ] Generate APNs authentication key (.p8 file)
- [ ] Download and store securely (1Password, Keychain)

**Day 3: Development Environment Setup**
- [ ] Install Rust (for push-server): `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
- [ ] Install Node.js (for Obsidian plugin): `brew install node`
- [ ] Install Xcode Command Line Tools: `xcode-select --install`
- [ ] Install Syncthing: `brew install syncthing`
- [ ] Test Sushitrain build:
  ```bash
  cd sushitrain-fork/SushitrainCore
  make
  # Should build successfully
  ```

**Day 4-5: Technical Validation**

Test assumptions from `ios-sync-integration-analysis.md`:

```bash
# Test 1: Syncthing + Sushitrain connectivity
# 1. Start Syncthing on macOS
syncthing -home ~/.config/syncthing

# 2. Build and run Sushitrain on iOS simulator
cd sushitrain-fork
open Sushitrain.xcodeproj
# Build and run on iOS simulator

# 3. Configure connection, add test folder
# 4. Sync 100 test files, measure time
```

**Validation Checklist**:
- [ ] Sushitrain connects to macOS Syncthing successfully
- [ ] 100-file sync completes in < 10 seconds (WiFi)
- [ ] File content is identical (diff test)
- [ ] Measure network bandwidth usage
- [ ] Monitor iOS app memory/CPU during sync

**Deliverables**:
- ✅ Project repository initialized
- ✅ Planning documents committed
- ✅ Apple Developer configured
- ✅ Development environment working
- ✅ Syncthing + Sushitrain validated

---

## Phase 1: MVP - Core Sync (Weeks 1-7)

**Goal**: Automatic Obsidian vault sync from Desktop to iOS using Syncthing + APNs push

**Success Criteria**:
- [ ] Vault changes on Desktop sync to iOS within 5 minutes (typical)
- [ ] Sync success rate > 85% on WiFi
- [ ] No data loss or corruption
- [ ] Battery usage < 5% per day

---

### Week 1: Fork Sushitrain & Project Structure

**Tasks**:

1. **Fork Sushitrain** (Day 1)
   ```bash
   cd sushitrain-fork
   git remote rename origin upstream
   git remote add origin https://github.com/your-username/sushitrain-fork.git

   # Create development branch
   git checkout -b feature/obsidiansync-integration
   ```

2. **Rebrand to ObsidianSync-BEP** (Day 2-3)
   - Update `Info.plist`:
     - Bundle ID: `com.yourname.obsidiansync`
     - Display name: "ObsidianSync-BEP"
   - Update icons (or keep Sushitrain icons for now)
   - Update entitlements:
     ```xml
     <key>com.apple.security.application-groups</key>
     <array>
         <string>group.com.yourname.obsidiansync</string>
     </array>
     ```

3. **Setup push-server Rust Project** (Day 4-5)
   ```bash
   cd push-server
   cargo init --name obsidiansync-push

   # Add dependencies to Cargo.toml
   [dependencies]
   tokio = { version = "1", features = ["full"] }
   a2 = "0.9"  # APNs client
   notify = "6.1"  # File system watching
   rusqlite = "0.30"  # Device token storage
   serde = { version = "1", features = ["derive"] }
   serde_json = "1"
   ```

**Deliverables**:
- ✅ Sushitrain forked and rebranded
- ✅ push-server Rust project initialized
- ✅ App Group entitlements configured
- ✅ iOS app builds successfully

---

### Week 2-3: APNs Infrastructure

**Week 2: APNs Client Implementation**

1. **APNs Client Module** (`push-server/src/apns_client.rs`)
   ```rust
   use a2::{Client, Endpoint, NotificationBuilder};

   pub struct APNsClient {
       client: Client,
       bundle_id: String,
   }

   impl APNsClient {
       pub async fn new(key_path: &str, team_id: &str, key_id: &str) -> Result<Self> {
           let client = Client::token(
               File::open(key_path)?,
               key_id,
               team_id,
               Endpoint::Production,
           )?;

           Ok(Self {
               client,
               bundle_id: "com.yourname.obsidiansync".to_string(),
           })
       }

       pub async fn send_silent_push(&self, device_token: &str, sync_hint: SyncHint) -> Result<()> {
           let payload = json!({
               "aps": {
                   "content-available": 1,
               },
               "sync-hint": sync_hint,
           });

           let notification = NotificationBuilder::new()
               .set_content_available()
               .set_priority(5)
               .build(&device_token, &payload)?;

           self.client.send(notification).await?;
           Ok(())
       }
   }

   #[derive(Serialize)]
   pub struct SyncHint {
       pub file_count: usize,
       pub estimated_kb: usize,
       pub batch_id: String,
   }
   ```

2. **Device Token Storage** (`push-server/src/device_registry.rs`)
   ```rust
   use rusqlite::{Connection, params};

   pub struct DeviceRegistry {
       db: Connection,
   }

   impl DeviceRegistry {
       pub fn new(db_path: &str) -> Result<Self> {
           let db = Connection::open(db_path)?;

           db.execute(
               "CREATE TABLE IF NOT EXISTS devices (
                   token TEXT PRIMARY KEY,
                   registered_at INTEGER,
                   last_push_at INTEGER
               )",
               [],
           )?;

           Ok(Self { db })
       }

       pub fn register_device(&mut self, token: &str) -> Result<()> {
           let now = SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs();

           self.db.execute(
               "INSERT OR REPLACE INTO devices (token, registered_at, last_push_at) VALUES (?, ?, ?)",
               params![token, now, 0],
           )?;

           Ok(())
       }

       pub fn get_all_tokens(&self) -> Result<Vec<String>> {
           let mut stmt = self.db.prepare("SELECT token FROM devices")?;
           let tokens = stmt.query_map([], |row| row.get(0))?
               .collect::<Result<Vec<String>, _>>()?;

           Ok(tokens)
       }
   }
   ```

3. **HTTP Endpoint for Device Registration** (`push-server/src/main.rs`)
   ```rust
   use axum::{Router, routing::post, Json};

   #[derive(Deserialize)]
   struct RegisterRequest {
       device_token: String,
   }

   async fn register_device(
       Json(payload): Json<RegisterRequest>,
   ) -> &'static str {
       let mut registry = DeviceRegistry::new("devices.db").unwrap();
       registry.register_device(&payload.device_token).unwrap();
       "OK"
   }

   #[tokio::main]
   async fn main() {
       let app = Router::new()
           .route("/register", post(register_device));

       axum::Server::bind(&"0.0.0.0:3000".parse().unwrap())
           .serve(app.into_make_service())
           .await
           .unwrap();
   }
   ```

**Week 3: iOS APNs Registration**

1. **iOS App: Request Push Permissions** (Sushitrain fork)
   ```swift
   // AppDelegate.swift or App.swift
   func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
       UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound, .badge]) { granted, error in
           if granted {
               DispatchQueue.main.async {
                   application.registerForRemoteNotifications()
               }
           }
       }
       return true
   }

   func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
       let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
       print("Device Token: \(token)")

       // Send to push-server
       registerDeviceToken(token)
   }

   private func registerDeviceToken(_ token: String) {
       let url = URL(string: "http://your-mac-ip:3000/register")!
       var request = URLRequest(url: url)
       request.httpMethod = "POST"
       request.setValue("application/json", forHTTPHeaderField: "Content-Type")

       let payload = ["device_token": token]
       request.httpBody = try? JSONSerialization.data(withJSONObject: payload)

       URLSession.shared.dataTask(with: request).resume()
   }
   ```

2. **Test APNs Push**
   ```bash
   # Start push-server
   cd push-server
   cargo run

   # Launch iOS app (simulator or device)
   # App registers device token

   # Send test push
   curl -X POST http://localhost:3000/test-push
   ```

**Deliverables**:
- ✅ APNs client implemented and tested
- ✅ Device token storage working
- ✅ iOS app registers for push notifications
- ✅ Test silent push successfully wakes iOS app

---

### Week 4: File System Monitoring & Push Trigger

1. **File System Watcher** (`push-server/src/change_monitor.rs`)
   ```rust
   use notify::{Watcher, RecursiveMode, Result, Event};

   pub struct ChangeMonitor {
       vault_path: PathBuf,
       throttler: PushThrottler,
       apns_client: APNsClient,
   }

   impl ChangeMonitor {
       pub async fn watch(&mut self) -> Result<()> {
           let (tx, rx) = channel();
           let mut watcher = notify::recommended_watcher(tx)?;

           watcher.watch(&self.vault_path, RecursiveMode::Recursive)?;

           let mut pending_changes = Vec::new();

           for event in rx {
               match event {
                   Ok(Event { kind: EventKind::Modify(_), paths, .. }) => {
                       pending_changes.extend(paths);

                       // Debounce: wait 2 seconds for more changes
                       tokio::time::sleep(Duration::from_secs(2)).await;

                       // Check throttle
                       if self.throttler.should_push(&pending_changes) {
                           self.send_push_notification(&pending_changes).await?;
                           pending_changes.clear();
                       }
                   }
                   _ => {}
               }
           }

           Ok(())
       }

       async fn send_push_notification(&self, changes: &[PathBuf]) -> Result<()> {
           let sync_hint = SyncHint {
               file_count: changes.len(),
               estimated_kb: self.estimate_size(changes),
               batch_id: Uuid::new_v4().to_string(),
           };

           let tokens = self.device_registry.get_all_tokens()?;

           for token in tokens {
               self.apns_client.send_silent_push(&token, sync_hint).await?;
           }

           Ok(())
       }
   }
   ```

2. **Push Throttler** (from ADR-005)
   ```rust
   pub struct PushThrottler {
       last_push: Instant,
       min_interval: Duration,  // 30 minutes
   }

   impl PushThrottler {
       pub fn should_push(&mut self, changes: &[PathBuf]) -> bool {
           let elapsed = self.last_push.elapsed();

           if elapsed < self.min_interval {
               return false;  // Batch for next window
           }

           self.last_push = Instant::now();
           true
       }
   }
   ```

3. **Syncthing Integration**
   ```bash
   # Configure Syncthing to share vault folder
   syncthing -home ~/.config/syncthing

   # Add folder via web UI (http://localhost:8384)
   # - Folder path: /Users/you/Documents/ObsidianVault
   # - Share with iOS device (Sushitrain)

   # Start push-server monitoring the same folder
   cargo run -- --vault-path /Users/you/Documents/ObsidianVault
   ```

**Deliverables**:
- ✅ File system watcher detects vault changes
- ✅ Push throttler enforces 30-minute minimum
- ✅ APNs push sent when changes detected
- ✅ Syncthing configured and sharing vault

---

### Week 5-6: iOS App Modifications

**Week 5: Silent Push Handler**

1. **Background Sync Trigger** (iOS app)
   ```swift
   // AppDelegate.swift
   func application(_ application: UIApplication, didReceiveRemoteNotification userInfo: [AnyHashable : Any], fetchCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {

       guard let syncHint = userInfo["sync-hint"] as? [String: Any],
             let fileCount = syncHint["file-count"] as? Int else {
           completionHandler(.noData)
           return
       }

       print("Silent push received: \(fileCount) files to sync")

       // Start background sync
       Task {
           do {
               try await backgroundSyncManager.performSync(fileCount: fileCount)
               completionHandler(.newData)
           } catch {
               print("Sync failed: \(error)")
               completionHandler(.failed)
           }
       }
   }
   ```

2. **Background Sync Manager**
   ```swift
   class BackgroundSyncManager {
       let networkDetector = NetworkQualityDetector()
       let syncEngine: SushitrainSyncEngine

       func performSync(fileCount: Int) async throws {
           let startTime = Date()
           let timeout: TimeInterval = 28  // Leave 2-second margin

           // Adaptive batch sizing
           let maxBatch = networkDetector.estimateBatchSize()
           let filesToSync = min(fileCount, maxBatch)

           print("Syncing \(filesToSync) files (max batch: \(maxBatch))")

           // Start BEP sync
           try await withTimeout(timeout) {
               try await syncEngine.sync(limit: filesToSync)
           }

           let elapsed = Date().timeIntervalSince(startTime)
           print("Sync completed in \(elapsed)s")

           // Save checkpoint for next sync
           if fileCount > filesToSync {
               saveCheckpoint(synced: filesToSync, remaining: fileCount - filesToSync)
           }
       }

       private func saveCheckpoint(synced: Int, remaining: Int) {
           UserDefaults.standard.set(synced, forKey: "last_sync_count")
           UserDefaults.standard.set(remaining, forKey: "pending_sync_count")
       }
   }
   ```

**Week 6: Network Quality Detection & State Persistence**

1. **Network Quality Detector** (from ADR-006)
   ```swift
   class NetworkQualityDetector {
       func estimateBatchSize() -> Int {
           let networkType = detectNetworkType()
           let latency = measureLatency()

           switch (networkType, latency) {
           case (.wifi, .low):  return 2000
           case (.wifi, .high): return 1000
           case (.lte, .low):   return 1000
           case (.lte, .high):  return 500
           case (.cellular, _): return 500
           default:             return 500
           }
       }

       private func detectNetworkType() -> NetworkType {
           let reachability = NetworkReachability.forInternetConnection()
           if reachability.isReachableViaWiFi() {
               return .wifi
           } else if reachability.isReachableViaWWAN() {
               return .lte
           } else {
               return .cellular
           }
       }

       private func measureLatency() -> Latency {
           // Ping Syncthing server, measure RTT
           // Implementation using URLSession or SimplePing
           // < 100ms = .low, >= 100ms = .high
       }
   }
   ```

2. **State Persistence**
   ```swift
   // Save state before 30-second timeout
   func saveState() {
       let state = SyncState(
           lastSequenceNumber: syncEngine.currentSequence,
           pendingFiles: syncEngine.pendingFiles,
           timestamp: Date()
       )

       // Save to UserDefaults or SQLite
       let data = try! JSONEncoder().encode(state)
       UserDefaults.standard.set(data, forKey: "sync_state")
   }

   // Resume on next wake
   func resumeSync() {
       guard let data = UserDefaults.standard.data(forKey: "sync_state"),
             let state = try? JSONDecoder().decode(SyncState.self, from: data) else {
           return
       }

       syncEngine.resumeFrom(sequence: state.lastSequenceNumber)
   }
   ```

**Deliverables**:
- ✅ iOS app handles silent push notifications
- ✅ Background sync completes within 30-second window (typical case)
- ✅ Network quality detection working
- ✅ State persistence for timeout recovery

---

### Week 7: Integration Testing & Optimization

**Integration Test Plan**:

1. **Functional Tests**
   - [ ] Create/edit/delete file on Desktop → syncs to iOS within 5 min
   - [ ] 100-file batch sync completes successfully
   - [ ] 500-file batch sync completes successfully
   - [ ] 1000-file batch sync completes successfully
   - [ ] 2000-file batch sync (WiFi only)

2. **Network Reliability Tests**
   - [ ] WiFi (low latency): Measure success rate for 2000 files
   - [ ] WiFi (high latency via Network Link Conditioner): 1000 files
   - [ ] LTE (good signal): 1000 files
   - [ ] LTE (poor signal): 500 files
   - [ ] Airplane mode toggle during sync: Resume works

3. **Battery Usage Tests**
   - [ ] Monitor battery drain over 24 hours
   - [ ] Target: < 5% per day
   - [ ] Adjust throttling if needed

4. **Edge Cases**
   - [ ] APNs push while app in foreground
   - [ ] APNs push while syncing already in progress
   - [ ] Multiple rapid pushes (throttling verification)
   - [ ] Large file sync (> 10 MB)
   - [ ] Conflict detection (simultaneous edit on Desktop and iOS)

**Performance Optimization**:
- [ ] Profile sync time for different file counts
- [ ] Optimize batch size based on empirical data
- [ ] Adjust throttling interval if battery usage high

**Bug Fixes**:
- [ ] Fix any crashes or data corruption issues
- [ ] Handle edge cases gracefully
- [ ] Add error logging for debugging

**Deliverables**:
- ✅ All integration tests passing
- ✅ Sync success rate > 85% on WiFi (validated)
- ✅ Battery usage acceptable
- ✅ No known critical bugs
- ✅ Phase 1 ready for dogfooding

**Dogfooding Checkpoint**: Use daily for 2 weeks, document issues, fix critical bugs

---

## Phase 2: File Provider Extension (Weeks 8-11)

**Goal**: Expose synced vault to iOS Files app for native Obsidian Mobile integration

**Success Criteria**:
- [ ] Obsidian Mobile opens vault from Files app
- [ ] File changes reflected in Files app within 5 seconds
- [ ] Performance acceptable for 2000+ file vaults

---

### Week 8: Extension Target Setup

1. **Create File Provider Extension Target** (XCode)
   - Open `sushitrain-fork/Sushitrain.xcodeproj`
   - Add new target: File > New > Target > File Provider Extension
   - Name: `ObsidianSyncFileProvider`
   - Bundle ID: `com.yourname.obsidiansync.fileprovider`

2. **Configure Entitlements**
   ```xml
   <!-- ObsidianSyncFileProvider.entitlements -->
   <key>com.apple.security.application-groups</key>
   <array>
       <string>group.com.yourname.obsidiansync</string>
   </array>
   ```

3. **Shared Container Setup**
   ```swift
   // Shared/Constants.swift (shared between app and extension)
   let sharedContainerURL = FileManager.default.containerURL(
       forSecurityApplicationGroupIdentifier: "group.com.yourname.obsidiansync"
   )!

   let vaultPath = sharedContainerURL.appendingPathComponent("vault")
   let metadataDBPath = sharedContainerURL.appendingPathComponent("metadata.db")
   ```

4. **File Provider Extension Scaffold**
   ```swift
   // FileProviderExtension/FileProviderExtension.swift
   import FileProvider

   class FileProviderExtension: NSFileProviderExtension {

       override init() {
           super.init()
           // Initialize vault bridge
           self.vaultBridge = SushitrainVaultBridge(sharedContainerURL: sharedContainerURL)
       }

       override func item(for identifier: NSFileProviderItemIdentifier) throws -> NSFileProviderItem {
           return try vaultBridge.getItem(identifier)
       }

       override func enumerator(for containerItemIdentifier: NSFileProviderItemIdentifier) throws -> NSFileProviderEnumerator {
           return VaultEnumerator(containerID: containerItemIdentifier, storage: vaultBridge)
       }
   }
   ```

**Deliverables**:
- ✅ File Provider extension target created
- ✅ App Group configured and working
- ✅ Shared container accessible from both app and extension
- ✅ Extension compiles and installs

---

### Week 9: Storage Bridge Implementation

1. **SushitrainVaultBridge** (Shared framework)
   ```swift
   // Shared/SushitrainVaultBridge.swift
   class SushitrainVaultBridge {
       private let vaultPath: URL
       private let database: MetadataDatabase

       init(sharedContainerURL: URL) {
           self.vaultPath = sharedContainerURL.appendingPathComponent("vault")
           self.database = MetadataDatabase(at: sharedContainerURL)

           // Create vault directory if needed
           try? FileManager.default.createDirectory(at: vaultPath, withIntermediateDirectories: true)
       }

       func getItem(_ identifier: NSFileProviderItemIdentifier) throws -> VaultItem {
           return try database.fetchItem(identifier)
       }

       func enumerateItems(in directory: NSFileProviderItemIdentifier) -> [VaultItem] {
           return database.listDirectory(directory)
       }

       func fetchContent(_ identifier: NSFileProviderItemIdentifier) -> URL {
           let item = try! database.fetchItem(identifier)
           return vaultPath.appendingPathComponent(item.relativePath)
       }
   }
   ```

2. **Metadata Database** (SQLite)
   ```swift
   class MetadataDatabase {
       private let db: Connection

       init(at containerURL: URL) {
           let dbPath = containerURL.appendingPathComponent("metadata.db").path
           self.db = try! Connection(dbPath)

           try! db.run("""
               CREATE TABLE IF NOT EXISTS file_metadata (
                   identifier TEXT PRIMARY KEY,
                   filename TEXT NOT NULL,
                   parent_identifier TEXT,
                   relative_path TEXT NOT NULL,
                   size INTEGER,
                   modified_date INTEGER,
                   content_type TEXT
               )
           """)
       }

       func insertItem(_ item: VaultItem) throws {
           let insert = Table("file_metadata").insert(
               Expression<String>("identifier") <- item.identifier,
               Expression<String>("filename") <- item.filename,
               // ... other fields
           )
           try db.run(insert)
       }

       func fetchItem(_ identifier: NSFileProviderItemIdentifier) throws -> VaultItem {
           let query = Table("file_metadata").filter(Expression<String>("identifier") == identifier.rawValue)
           guard let row = try db.pluck(query) else {
               throw NSFileProviderError(.noSuchItem)
           }
           return VaultItem(from: row)
       }
   }
   ```

3. **VaultItem** (NSFileProviderItem conformance)
   ```swift
   class VaultItem: NSObject, NSFileProviderItem {
       let identifier: NSFileProviderItemIdentifier
       let parentItemIdentifier: NSFileProviderItemIdentifier
       let filename: String
       let contentType: UTType
       let fileSize: Int64?
       let lastModifiedDate: Date?

       var itemIdentifier: NSFileProviderItemIdentifier { identifier }
       var parentItemIdentifier: NSFileProviderItemIdentifier { parentItemIdentifier }
       var filename: String { filename }
       var contentType: UTType { contentType }
       var documentSize: NSNumber? { fileSize.map { NSNumber(value: $0) } }
       var contentModificationDate: Date? { lastModifiedDate }

       // ... other NSFileProviderItem properties
   }
   ```

**Deliverables**:
- ✅ Vault bridge implemented
- ✅ SQLite metadata storage working
- ✅ File enumeration returns correct items
- ✅ Content fetching returns file URLs

---

### Week 10: Real-Time Updates

1. **Sync Completion Hook** (Main app)
   ```swift
   // SyncManager.swift (in main Sushitrain app)
   func syncCompleted(updatedFiles: [String]) {
       // Update metadata database
       for file in updatedFiles {
           let item = VaultItem(from: file)
           try? metadataDB.insertOrUpdate(item)
       }

       // Signal File Provider
       NSFileProviderManager.default.signalEnumerator(for: .workingSet) { error in
           if let error = error {
               print("Failed to signal File Provider: \(error)")
           } else {
               print("File Provider notified of changes")
           }
       }
   }
   ```

2. **Working Set Enumerator** (Extension)
   ```swift
   class VaultEnumerator: NSObject, NSFileProviderEnumerator {
       private let storage: SushitrainVaultBridge
       private let containerID: NSFileProviderItemIdentifier

       func enumerateItems(for observer: NSFileProviderEnumerationObserver, startingAt page: NSFileProviderPage) {
           let items = storage.enumerateItems(in: containerID)

           observer.didEnumerate(items)
           observer.finishEnumerating(upTo: nil)
       }

       func enumerateChanges(for observer: NSFileProviderChangeObserver, from syncAnchor: NSFileProviderSyncAnchor) {
           // Incremental changes since last enumeration
           let (addedItems, modifiedItems, deletedIDs) = storage.changesSince(syncAnchor)

           observer.didUpdate(modifiedItems)
           observer.didDeleteItems(withIdentifiers: deletedIDs)

           let newAnchor = storage.currentSyncAnchor()
           observer.finishEnumeratingChanges(upTo: newAnchor, moreComing: false)
       }
   }
   ```

3. **Testing Real-Time Updates**
   ```bash
   # Test flow:
   # 1. Sync files via Sushitrain
   # 2. Open Files app
   # 3. Navigate to ObsidianSync folder
   # 4. Edit file on Desktop
   # 5. Wait for sync (< 5 min)
   # 6. Files app should show updated file
   ```

**Deliverables**:
- ✅ Sync completion triggers File Provider notification
- ✅ Files app reflects changes within 5 seconds
- ✅ Incremental enumeration working
- ✅ No full re-scan needed on updates

---

### Week 11: Obsidian Mobile Integration

1. **Test Obsidian Mobile Vault Opening**
   - [ ] Install Obsidian Mobile on iOS device/simulator
   - [ ] Launch Obsidian, tap "Open folder as vault"
   - [ ] Navigate to Files app → ObsidianSync
   - [ ] Select vault folder
   - [ ] Verify Obsidian opens vault successfully

2. **Document Picker Integration**
   ```swift
   // Verify extension provides correct UTType for markdown files
   class VaultItem: NSObject, NSFileProviderItem {
       var contentType: UTType {
           if filename.hasSuffix(".md") {
               return .markdown
           } else {
               return .data
           }
       }
   }
   ```

3. **Performance Testing**
   - [ ] Open 2000-file vault in Obsidian Mobile
   - [ ] Measure file list load time (target < 2 seconds)
   - [ ] Test file opening (target < 500ms)
   - [ ] Test search performance

4. **Lazy Loading Optimization**
   ```swift
   // Only enumerate visible items initially
   func enumerateItems(for observer: NSFileProviderEnumerationObserver, startingAt page: NSFileProviderPage) {
       let pageSize = 100
       let items = storage.enumerateItems(in: containerID, limit: pageSize, offset: page)

       observer.didEnumerate(items)

       if items.count == pageSize {
           let nextPage = NSFileProviderPage(page.rawValue + pageSize)
           observer.finishEnumerating(upTo: nextPage)
       } else {
           observer.finishEnumerating(upTo: nil)
       }
   }
   ```

**Deliverables**:
- ✅ Obsidian Mobile opens vault from Files app
- ✅ File browsing performance acceptable (< 2s for 2000 files)
- ✅ File editing works (changes saved to vault)
- ✅ Obsidian detects external file changes
- ✅ Phase 2 ready for dogfooding

**Dogfooding Checkpoint**: Use as primary iOS workflow for 1 week

---

## Phase 3: Obsidian Plugin (Weeks 12-14)

**Goal**: Deep Obsidian Mobile integration with sync status and manual controls

**Success Criteria**:
- [ ] Sync status visible within Obsidian
- [ ] Manual sync trigger works
- [ ] Conflicts displayed and resolvable

---

### Week 12: Plugin Scaffold

1. **Initialize Plugin Project**
   ```bash
   cd obsidian-plugin
   npm init -y
   npm install --save-dev @types/node typescript obsidian

   # Create plugin structure
   mkdir src
   touch src/main.ts
   touch manifest.json
   touch tsconfig.json
   ```

2. **Plugin Manifest** (`manifest.json`)
   ```json
   {
     "id": "obsidiansync-plugin",
     "name": "ObsidianSync Status",
     "version": "0.1.0",
     "minAppVersion": "0.15.0",
     "description": "Displays sync status for ObsidianSync-BEP",
     "author": "Your Name",
     "authorUrl": "https://github.com/your-username/obsidian-sync-bep",
     "isDesktopOnly": false
   }
   ```

3. **Main Plugin Class** (`src/main.ts`)
   ```typescript
   import { Plugin, Notice } from 'obsidian';
   import { ObsidianSyncSettings, DEFAULT_SETTINGS } from './settings';

   export default class ObsidianSyncPlugin extends Plugin {
       settings: ObsidianSyncSettings;
       statusBarItem: HTMLElement;

       async onload() {
           console.log('Loading ObsidianSync plugin');

           await this.loadSettings();

           // Add ribbon icon
           this.addRibbonIcon('sync', 'ObsidianSync Status', () => {
               this.openSyncStatusView();
           });

           // Add status bar item
           this.statusBarItem = this.addStatusBarItem();
           this.updateSyncStatus();

           // Register commands
           this.addCommand({
               id: 'trigger-manual-sync',
               name: 'Trigger Manual Sync',
               callback: () => this.triggerManualSync()
           });
       }

       async loadSettings() {
           this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());
       }

       async updateSyncStatus() {
           // Read .obsidian/sync-status.json
           const statusFile = '.obsidian/sync-status.json';
           try {
               const content = await this.app.vault.adapter.read(statusFile);
               const status = JSON.parse(content);

               this.statusBarItem.setText(
                   `Sync: ${status.state} (${status.filesInSync}/${status.totalFiles})`
               );
           } catch (e) {
               this.statusBarItem.setText('Sync: Unknown');
           }
       }
   }
   ```

4. **Build Configuration** (`tsconfig.json`)
   ```json
   {
     "compilerOptions": {
       "target": "ES2018",
       "module": "commonjs",
       "lib": ["es2018"],
       "outDir": "./",
       "strict": true,
       "moduleResolution": "node",
       "esModuleInterop": true,
       "skipLibCheck": true
     },
     "include": ["src/**/*.ts"]
   }
   ```

**Deliverables**:
- ✅ Plugin project initialized
- ✅ Plugin loads in Obsidian Mobile
- ✅ Ribbon icon and status bar visible
- ✅ Basic structure working

---

### Week 13: Sync Status Display

1. **Sync Status Metadata Writer** (iOS app)
   ```swift
   // SyncManager.swift
   func updateSyncStatusFile() {
       let status = SyncStatus(
           state: currentSyncState.rawValue,
           filesInSync: syncedFileCount,
           totalFiles: totalFileCount,
           lastSyncTime: lastSyncDate.ISO8601Format(),
           pendingChanges: pendingChangeCount,
           conflicts: conflictFiles.map { ConflictInfo(file: $0.path, ...) }
       )

       let data = try! JSONEncoder().encode(status)
       let statusPath = vaultPath.appendingPathComponent(".obsidian/sync-status.json")
       try? data.write(to: statusPath)
   }
   ```

2. **Sync Status View** (`src/sync-status.ts`)
   ```typescript
   import { ItemView, WorkspaceLeaf } from 'obsidian';

   export const VIEW_TYPE_SYNC_STATUS = 'obsidiansync-status-view';

   export class SyncStatusView extends ItemView {
       constructor(leaf: WorkspaceLeaf) {
           super(leaf);
       }

       getViewType(): string {
           return VIEW_TYPE_SYNC_STATUS;
       }

       getDisplayText(): string {
           return 'Sync Status';
       }

       async onOpen() {
           const container = this.containerEl.children[1];
           container.empty();

           // Read sync status
           const status = await this.readSyncStatus();

           container.createEl('h4', { text: 'ObsidianSync Status' });

           const statusTable = container.createEl('table');
           statusTable.createEl('tr').innerHTML = `
               <td>State:</td>
               <td class="sync-state-${status.state}">${status.state}</td>
           `;
           statusTable.createEl('tr').innerHTML = `
               <td>Files in sync:</td>
               <td>${status.filesInSync} / ${status.totalFiles}</td>
           `;
           statusTable.createEl('tr').innerHTML = `
               <td>Last sync:</td>
               <td>${new Date(status.lastSyncTime).toLocaleString()}</td>
           `;

           // Show conflicts if any
           if (status.conflicts && status.conflicts.length > 0) {
               container.createEl('h5', { text: 'Conflicts' });
               const conflictList = container.createEl('ul');
               status.conflicts.forEach(conflict => {
                   conflictList.createEl('li', { text: conflict.file });
               });
           }

           // Add manual sync button
           const syncBtn = container.createEl('button', { text: 'Trigger Sync' });
           syncBtn.onclick = () => this.plugin.triggerManualSync();
       }

       async readSyncStatus(): Promise<SyncStatus> {
           try {
               const content = await this.app.vault.adapter.read('.obsidian/sync-status.json');
               return JSON.parse(content);
           } catch {
               return {
                   state: 'unknown',
                   filesInSync: 0,
                   totalFiles: 0,
                   lastSyncTime: new Date().toISOString(),
                   pendingChanges: 0,
                   conflicts: []
               };
           }
       }
   }
   ```

3. **Auto-Refresh Status**
   ```typescript
   // main.ts
   onload() {
       // ... existing code ...

       // Refresh status every 10 seconds
       this.registerInterval(
           window.setInterval(() => this.updateSyncStatus(), 10000)
       );
   }
   ```

**Deliverables**:
- ✅ Sync status displayed in plugin UI
- ✅ Status updates in real-time
- ✅ Conflicts shown when present
- ✅ Status bar reflects current state

---

### Week 14: Manual Sync Trigger & Settings

1. **URL Scheme Integration** (iOS app)
   ```swift
   // SceneDelegate.swift
   func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
       guard let url = URLContexts.first?.url else { return }

       if url.scheme == "obsidiansync" {
           handleURLScheme(url)
       }
   }

   func handleURLScheme(_ url: URL) {
       if url.host == "sync" {
           // Trigger manual sync
           Task {
               try await syncManager.performManualSync()
           }
       }
   }
   ```

2. **Plugin Manual Sync Command** (`src/main.ts`)
   ```typescript
   async triggerManualSync() {
       // Open Sushitrain app via URL scheme
       const url = 'obsidiansync://sync?folder=obsidian-vault';

       // On iOS, this will switch to Sushitrain app
       window.open(url, '_blank');

       new Notice('Sync triggered - switching to ObsidianSync app...');

       // Poll for sync completion
       setTimeout(() => this.checkSyncCompletion(), 5000);
   }

   async checkSyncCompletion() {
       // Read sync-status.json, check if state changed
       const status = await this.readSyncStatus();

       if (status.state === 'idle') {
           new Notice('Sync completed!');
           this.updateSyncStatus();
       } else {
           // Still syncing, check again in 5 seconds
           setTimeout(() => this.checkSyncCompletion(), 5000);
       }
   }
   ```

3. **Plugin Settings** (`src/settings.ts`)
   ```typescript
   import { App, PluginSettingTab, Setting } from 'obsidian';

   export interface ObsidianSyncSettings {
       syncMode: 'immediate' | 'balanced' | 'conservative';
       wifiOnly: boolean;
       showNotifications: boolean;
   }

   export const DEFAULT_SETTINGS: ObsidianSyncSettings = {
       syncMode: 'balanced',
       wifiOnly: true,
       showNotifications: true
   };

   export class ObsidianSyncSettingTab extends PluginSettingTab {
       plugin: ObsidianSyncPlugin;

       constructor(app: App, plugin: ObsidianSyncPlugin) {
           super(app, plugin);
           this.plugin = plugin;
       }

       display(): void {
           let { containerEl } = this;
           containerEl.empty();

           new Setting(containerEl)
               .setName('Sync Mode')
               .setDesc('Immediate (5-min), Balanced (30-min), Conservative (1-hour)')
               .addDropdown(dropdown => dropdown
                   .addOption('immediate', 'Immediate')
                   .addOption('balanced', 'Balanced')
                   .addOption('conservative', 'Conservative')
                   .setValue(this.plugin.settings.syncMode)
                   .onChange(async (value) => {
                       this.plugin.settings.syncMode = value;
                       await this.plugin.saveSettings();
                   }));

           new Setting(containerEl)
               .setName('WiFi Only')
               .setDesc('Only sync on WiFi (disable for cellular sync)')
               .addToggle(toggle => toggle
                   .setValue(this.plugin.settings.wifiOnly)
                   .onChange(async (value) => {
                       this.plugin.settings.wifiOnly = value;
                       await this.plugin.saveSettings();
                   }));
       }
   }
   ```

4. **Build & Install**
   ```bash
   cd obsidian-plugin
   npm run build

   # Copy to Obsidian vault
   cp main.js manifest.json /path/to/vault/.obsidian/plugins/obsidiansync-plugin/
   ```

**Deliverables**:
- ✅ Manual sync trigger works via URL scheme
- ✅ Plugin settings tab functional
- ✅ User can configure sync mode and preferences
- ✅ Settings persisted correctly
- ✅ Phase 3 complete

**Dogfooding Checkpoint**: Daily use for 2 weeks

---

## Phase 4+: Future Enhancements (Post-MVP)

**Evaluate after 3 months Phase 3 dogfooding**

### Potential Work

1. **Custom BEP Server** (11+ weeks)
   - Only if Syncthing wrapper proves insufficient
   - Implement BEP v1 protocol in Rust
   - Integrate with vault-server architecture

2. **Advanced Conflict Resolution** (2-3 weeks)
   - Three-way merge UI
   - Automatic resolution strategies
   - Conflict prevention (optimistic locking)

3. **Performance Optimizations** (1-2 weeks)
   - Progressive sync prioritization
   - Better batch size algorithms
   - Cache optimization

4. **Community Features** (3-4 weeks)
   - Plugin distribution via Obsidian community
   - Documentation and setup guides
   - Issue tracker and support

5. **Multi-Device Orchestration** (4-5 weeks)
   - Support for multiple iOS devices
   - Conflict resolution across devices
   - Centralized sync status dashboard

---

## Success Metrics & KPIs

### Phase 1 Metrics

**Reliability**:
- [ ] Sync success rate > 85% on WiFi
- [ ] < 5 data loss/corruption incidents per month

**Performance**:
- [ ] Desktop → iOS sync latency < 5 minutes (p95)
- [ ] Battery usage < 5% per day

**Usability**:
- [ ] Guillaume uses daily for 2+ weeks without reverting
- [ ] < 1 manual intervention needed per week

### Phase 2 Metrics

**Integration**:
- [ ] Obsidian Mobile vault opening success rate > 95%
- [ ] File changes reflected in Files app < 5 seconds (p95)

**Performance**:
- [ ] 2000-file vault enumeration < 2 seconds
- [ ] File opening latency < 500ms

### Phase 3 Metrics

**Usability**:
- [ ] Sync status checked within Obsidian (not switching apps)
- [ ] Manual sync trigger used successfully > 90% of the time

### Long-Term Metrics (6+ months)

**Adoption**:
- [ ] Continuous use for 6+ months
- [ ] Community interest (GitHub stars, forks)
- [ ] No major architectural blockers

**Cost Savings**:
- [ ] Total cost (dev time + infra) < $120/year (Obsidian Sync cost)

---

## Risk Mitigation Checklist

### Before Starting Phase 1
- [ ] Apple Developer Account setup complete
- [ ] APNs authentication key generated and tested
- [ ] Syncthing + Sushitrain connectivity validated
- [ ] Network reliability matrix validated empirically

### Before Phase 2
- [ ] Phase 1 dogfooding successful (2 weeks minimum)
- [ ] Sync reliability > 85% confirmed
- [ ] No data loss incidents
- [ ] Battery usage acceptable

### Before Phase 3
- [ ] File Provider working in Files app
- [ ] Obsidian Mobile opens vault successfully
- [ ] Performance acceptable for 2000+ files

### Before Phase 4
- [ ] 3 months Phase 3 dogfooding complete
- [ ] Empirical data shows need for custom BEP server OR
- [ ] Community demand justifies additional features

---

## Development Workflow

### Daily Workflow
1. **Check todo list** (use `TodoWrite` tool)
2. **Implement feature** (delegate to developer agent with appropriate model)
3. **Write tests first** (TDD unit-first approach)
4. **Run tests** before committing
5. **Commit via git-workflow-manager** (all commits through specialist)

### Weekly Workflow
1. **Review progress** against implementation plan
2. **Update planning documents** (backlog, decision journal)
3. **Dogfooding checkpoint** (test in real workflow)
4. **Adjust priorities** based on findings

### Phase Transitions
1. **Complete all phase deliverables**
2. **Dogfooding checkpoint** (1-2 weeks)
3. **Retrospective** (what went well, what to improve)
4. **Update planning docs** with learnings
5. **Begin next phase**

---

## Appendix: Quick Reference

### Useful Commands

**Git Submodule Management**:
```bash
# Update Sushitrain upstream
cd sushitrain-fork
git fetch upstream
git checkout main
git merge upstream/main
cd ..
git add sushitrain-fork
git commit -m "chore: update Sushitrain"
```

**APNs Testing**:
```bash
# Send test push
cd push-server
cargo run -- --test-push <device-token>
```

**iOS Build**:
```bash
cd sushitrain-fork
xcodebuild -project Sushitrain.xcodeproj -scheme Synctrain -configuration Debug
```

**Plugin Build**:
```bash
cd obsidian-plugin
npm run build
cp main.js manifest.json /path/to/vault/.obsidian/plugins/obsidiansync-plugin/
```

### File Locations

**Planning Documents**:
- `planning/framing.md` - Product vision and roadmap
- `planning/decision-journal.md` - Architecture Decision Records
- `planning/implementation-plan.md` - This file

**Project Root**:
- `~/dev/tools/obsidian-sync-bep/`

**Key Components**:
- `sushitrain-fork/` - iOS BEP client
- `push-server/` - APNs notification server
- `obsidian-plugin/` - Obsidian Mobile plugin

### Contact & Support

**Developer**: Guillaume (personal use)
**Issues**: GitHub Issues (after community release)
**Documentation**: `docs/` folder in project root
