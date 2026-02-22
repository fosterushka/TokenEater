# Drop Cookie System — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Supprimer l'auth cookies, passer OAuth Keychain only, sandboxer l'app, proxy widget via AppIntent.

**Architecture:** App et widget lisent le Keychain directement (zero partage fichier). App stocke proxy dans UserDefaults. Widget recoit proxy via AppIntentConfiguration. Chaque cible cache independamment dans son propre container sandbox.

**Tech Stack:** Swift 5.9, macOS 14+, Security framework, WidgetKit, AppIntents

---

### Task 1: Bump deployment target macOS 14

`AppIntentConfiguration` (pour le proxy widget) requiert macOS 14+. macOS 14 Sonoma est sorti en sept. 2023.

**Files:**
- Modify: `project.yml`

**Step 1: Mettre a jour project.yml**

Dans `project.yml`, remplacer toutes les occurrences de `"13.0"` par `"14.0"` :

```yaml
options:
  deploymentTarget:
    macOS: "14.0"

settings:
  base:
    MACOSX_DEPLOYMENT_TARGET: "14.0"
```

**Step 2: Commit**

```bash
git add project.yml
git commit -m "chore: bump deployment target macOS 14 (requis pour AppIntentConfiguration)"
```

---

### Task 2: Sandboxer l'app hote

**Files:**
- Modify: `ClaudeUsageApp/ClaudeUsageApp.entitlements`

**Step 1: Ajouter sandbox + network.client**

Remplacer le contenu de `ClaudeUsageApp/ClaudeUsageApp.entitlements` :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
</dict>
</plist>
```

**Step 2: Commit**

```bash
git add ClaudeUsageApp/ClaudeUsageApp.entitlements
git commit -m "feat: sandbox app hote (app-sandbox + network.client)"
```

---

### Task 3: Rewrite UsageModels.swift — supprimer SharedConfig/SharedStorage

Supprimer `SharedConfig`, `SharedStorage`, `AppConstants.widgetBundleID`. Ajouter `ProxyConfig` et un cache local.

**Files:**
- Modify: `Shared/UsageModels.swift`

**Step 1: Remplacer entierement le fichier**

Garder : `UsageResponse`, `UsageBucket`, `CachedUsage`.
Supprimer : `AppConstants`, `SharedConfig`, `SharedStorage`.
Ajouter : `ProxyConfig`, `LocalCache`.

```swift
import Foundation

// MARK: - API Response

struct UsageResponse: Codable {
    let fiveHour: UsageBucket?
    let sevenDay: UsageBucket?
    let sevenDaySonnet: UsageBucket?
    let sevenDayOauthApps: UsageBucket?
    let sevenDayOpus: UsageBucket?
    let sevenDayCowork: UsageBucket?

    enum CodingKeys: String, CodingKey {
        case fiveHour = "five_hour"
        case sevenDay = "seven_day"
        case sevenDaySonnet = "seven_day_sonnet"
        case sevenDayOauthApps = "seven_day_oauth_apps"
        case sevenDayOpus = "seven_day_opus"
        case sevenDayCowork = "seven_day_cowork"
    }

    init(fiveHour: UsageBucket? = nil, sevenDay: UsageBucket? = nil, sevenDaySonnet: UsageBucket? = nil,
         sevenDayOauthApps: UsageBucket? = nil, sevenDayOpus: UsageBucket? = nil, sevenDayCowork: UsageBucket? = nil) {
        self.fiveHour = fiveHour
        self.sevenDay = sevenDay
        self.sevenDaySonnet = sevenDaySonnet
        self.sevenDayOauthApps = sevenDayOauthApps
        self.sevenDayOpus = sevenDayOpus
        self.sevenDayCowork = sevenDayCowork
    }

    // Decode tolerantly: unknown keys are ignored, broken buckets become nil
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        fiveHour = try? container.decode(UsageBucket.self, forKey: .fiveHour)
        sevenDay = try? container.decode(UsageBucket.self, forKey: .sevenDay)
        sevenDaySonnet = try? container.decode(UsageBucket.self, forKey: .sevenDaySonnet)
        sevenDayOauthApps = try? container.decode(UsageBucket.self, forKey: .sevenDayOauthApps)
        sevenDayOpus = try? container.decode(UsageBucket.self, forKey: .sevenDayOpus)
        sevenDayCowork = try? container.decode(UsageBucket.self, forKey: .sevenDayCowork)
    }
}

struct UsageBucket: Codable {
    let utilization: Double
    let resetsAt: String?

    enum CodingKeys: String, CodingKey {
        case utilization
        case resetsAt = "resets_at"
    }

    var resetsAtDate: Date? {
        guard let resetsAt else { return nil }
        let withFractional = ISO8601DateFormatter()
        withFractional.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
        if let date = withFractional.date(from: resetsAt) {
            return date
        }
        let withoutFractional = ISO8601DateFormatter()
        withoutFractional.formatOptions = [.withInternetDateTime]
        return withoutFractional.date(from: resetsAt)
    }
}

// MARK: - Cached Usage (for offline support)

struct CachedUsage: Codable {
    let usage: UsageResponse
    let fetchDate: Date
}

// MARK: - Proxy Config (injectable — app uses UserDefaults, widget uses AppIntent)

struct ProxyConfig {
    var enabled: Bool
    var host: String
    var port: Int

    init(enabled: Bool = false, host: String = "127.0.0.1", port: Int = 1080) {
        self.enabled = enabled
        self.host = host
        self.port = port
    }
}

// MARK: - Local Cache (each target writes to its own sandbox Application Support)

enum LocalCache {
    private static let cacheFileName = "claude-usage-cache.json"

    private static var cacheURL: URL {
        let appSupport = FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask).first!
        try? FileManager.default.createDirectory(at: appSupport, withIntermediateDirectories: true)
        return appSupport.appendingPathComponent(cacheFileName)
    }

    static func write(_ cache: CachedUsage) {
        try? JSONEncoder().encode(cache).write(to: cacheURL)
    }

    static func read() -> CachedUsage? {
        guard let data = try? Data(contentsOf: cacheURL) else { return nil }
        return try? JSONDecoder().decode(CachedUsage.self, from: data)
    }
}
```

**Step 2: Commit**

```bash
git add Shared/UsageModels.swift
git commit -m "refactor: suppression SharedConfig/SharedStorage, ajout ProxyConfig + LocalCache"
```

---

### Task 4: Rewrite ClaudeAPIClient.swift — OAuth only

**Files:**
- Modify: `Shared/ClaudeAPIClient.swift`

**Step 1: Remplacer entierement le fichier**

```swift
import Foundation

final class ClaudeAPIClient {
    static let shared = ClaudeAPIClient()

    private let oauthURL = URL(string: "https://api.anthropic.com/api/oauth/usage")!

    /// Set by host app (from UserDefaults) or widget (from AppIntent)
    var proxyConfig: ProxyConfig?

    private var session: URLSession {
        guard let proxy = proxyConfig, proxy.enabled else { return .shared }
        let c = URLSessionConfiguration.default
        c.connectionProxyDictionary = [
            kCFNetworkProxiesSOCKSEnable as String: true,
            kCFNetworkProxiesSOCKSProxy as String: proxy.host,
            kCFNetworkProxiesSOCKSPort as String: proxy.port,
        ]
        return URLSession(configuration: c)
    }

    // MARK: - Auth

    var isConfigured: Bool {
        KeychainOAuthReader.readClaudeCodeToken() != nil
    }

    // MARK: - Fetch Usage

    func fetchUsage() async throws -> UsageResponse {
        guard let oauth = KeychainOAuthReader.readClaudeCodeToken() else {
            throw ClaudeAPIError.noToken
        }

        var request = URLRequest(url: oauthURL)
        request.httpMethod = "GET"
        request.setValue("Bearer \(oauth.accessToken)", forHTTPHeaderField: "Authorization")
        request.setValue("oauth-2025-04-20", forHTTPHeaderField: "anthropic-beta")

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw ClaudeAPIError.invalidResponse
        }

        switch httpResponse.statusCode {
        case 200:
            let usage = try JSONDecoder().decode(UsageResponse.self, from: data)
            LocalCache.write(CachedUsage(usage: usage, fetchDate: Date()))
            return usage
        case 401, 403:
            throw ClaudeAPIError.tokenExpired
        default:
            throw ClaudeAPIError.httpError(httpResponse.statusCode)
        }
    }

    // MARK: - Test Connection

    func testConnection() async -> ConnectionTestResult {
        guard let oauth = KeychainOAuthReader.readClaudeCodeToken() else {
            return ConnectionTestResult(success: false, message: String(localized: "error.notoken"))
        }

        var request = URLRequest(url: oauthURL)
        request.httpMethod = "GET"
        request.setValue("Bearer \(oauth.accessToken)", forHTTPHeaderField: "Authorization")
        request.setValue("oauth-2025-04-20", forHTTPHeaderField: "anthropic-beta")

        do {
            let (data, response) = try await session.data(for: request)
            guard let httpResponse = response as? HTTPURLResponse else {
                return ConnectionTestResult(success: false, message: String(localized: "error.invalidresponse.short"))
            }

            if httpResponse.statusCode == 200 {
                guard let usage = try? JSONDecoder().decode(UsageResponse.self, from: data) else {
                    return ConnectionTestResult(success: false, message: String(localized: "error.unsupportedplan"))
                }
                let sessionPct = usage.fiveHour?.utilization ?? 0
                return ConnectionTestResult(success: true, message: String(format: String(localized: "test.success"), Int(sessionPct)))
            } else if httpResponse.statusCode == 401 || httpResponse.statusCode == 403 {
                return ConnectionTestResult(success: false, message: String(format: String(localized: "test.expired"), httpResponse.statusCode))
            } else {
                return ConnectionTestResult(success: false, message: String(format: String(localized: "test.http"), httpResponse.statusCode))
            }
        } catch {
            return ConnectionTestResult(success: false, message: String(format: String(localized: "error.network"), error.localizedDescription))
        }
    }

    // MARK: - Cache

    func loadCachedUsage() -> CachedUsage? {
        LocalCache.read()
    }
}

// MARK: - Error

enum ClaudeAPIError: LocalizedError {
    case noToken
    case invalidResponse
    case tokenExpired
    case unsupportedPlan
    case httpError(Int)

    var errorDescription: String? {
        switch self {
        case .noToken:
            return String(localized: "error.notoken")
        case .invalidResponse:
            return String(localized: "error.invalidresponse")
        case .tokenExpired:
            return String(localized: "error.tokenexpired")
        case .unsupportedPlan:
            return String(localized: "error.unsupportedplan")
        case .httpError(let code):
            return String(format: String(localized: "error.http"), code)
        }
    }
}

// MARK: - Test Result

struct ConnectionTestResult {
    let success: Bool
    let message: String
}
```

**Step 2: Commit**

```bash
git add Shared/ClaudeAPIClient.swift
git commit -m "refactor: ClaudeAPIClient OAuth only, proxy injectable, cache local"
```

---

### Task 5: Mettre a jour ClaudeUsageApp.swift

**Files:**
- Modify: `ClaudeUsageApp/ClaudeUsageApp.swift`

**Step 1: Retirer isHostApp et ajouter sync proxy**

```swift
import SwiftUI

@main
struct ClaudeUsageApp: App {
    @StateObject private var menuBarVM = MenuBarViewModel()
    @AppStorage("showMenuBar") private var showMenuBar = true

    init() {
        syncProxyConfig()
    }

    var body: some Scene {
        WindowGroup(id: "settings") {
            SettingsView(onConfigSaved: { [weak menuBarVM] in
                menuBarVM?.reloadConfig()
                syncProxyConfig()
            })
        }
        .windowResizability(.contentSize)

        MenuBarExtra(isInserted: $showMenuBar) {
            MenuBarPopoverView(viewModel: menuBarVM)
        } label: {
            Image(nsImage: menuBarVM.menuBarImage)
        }
        .menuBarExtraStyle(.window)
    }

    private func syncProxyConfig() {
        ClaudeAPIClient.shared.proxyConfig = ProxyConfig(
            enabled: UserDefaults.standard.bool(forKey: "proxyEnabled"),
            host: UserDefaults.standard.string(forKey: "proxyHost") ?? "127.0.0.1",
            port: {
                let port = UserDefaults.standard.integer(forKey: "proxyPort")
                return port > 0 ? port : 1080
            }()
        )
    }
}
```

Note : `syncProxyConfig()` est une free function dans l'App struct (pas un static). C'est appele au init et a chaque sauvegarde des settings.

**Step 2: Commit**

```bash
git add ClaudeUsageApp/ClaudeUsageApp.swift
git commit -m "refactor: retrait isHostApp, sync proxy via UserDefaults"
```

---

### Task 6: Simplifier MenuBarView.swift

**Files:**
- Modify: `ClaudeUsageApp/MenuBarView.swift`

**Step 1: Remplacer resolveAuthMethod() par isConfigured**

Ligne 66, remplacer :
```swift
hasConfig = ClaudeAPIClient.shared.resolveAuthMethod() != nil
```
par :
```swift
hasConfig = ClaudeAPIClient.shared.isConfigured
```

Ligne 134, remplacer :
```swift
guard ClaudeAPIClient.shared.resolveAuthMethod() != nil else {
```
par :
```swift
guard ClaudeAPIClient.shared.isConfigured else {
```

Ligne 157, remplacer :
```swift
hasConfig = ClaudeAPIClient.shared.resolveAuthMethod() != nil
```
par :
```swift
hasConfig = ClaudeAPIClient.shared.isConfigured
```

**Step 2: Commit**

```bash
git add ClaudeUsageApp/MenuBarView.swift
git commit -m "refactor: MenuBarViewModel utilise isConfigured"
```

---

### Task 7: Creer ProxyIntent pour le widget

**Files:**
- Create: `ClaudeUsageWidget/ProxyIntent.swift`

**Step 1: Creer le fichier**

```swift
import AppIntents
import WidgetKit

struct ProxyIntent: WidgetConfigurationIntent {
    static var title: LocalizedStringResource = "TokenEater Configuration"
    static var description: IntentDescription = "Configure proxy settings for the widget"

    @Parameter(title: "Enable SOCKS5 Proxy", default: false)
    var proxyEnabled: Bool

    @Parameter(title: "Proxy Host", default: "127.0.0.1")
    var proxyHost: String

    @Parameter(title: "Proxy Port", default: 1080)
    var proxyPort: Int
}
```

**Step 2: Commit**

```bash
git add ClaudeUsageWidget/ProxyIntent.swift
git commit -m "feat: ajout ProxyIntent pour config proxy widget"
```

---

### Task 8: Migrer Provider vers AppIntentTimelineProvider

**Files:**
- Modify: `ClaudeUsageWidget/Provider.swift`

**Step 1: Remplacer entierement le fichier**

```swift
import WidgetKit
import Foundation

struct Provider: AppIntentTimelineProvider {
    private let apiClient = ClaudeAPIClient.shared

    func placeholder(in context: Context) -> UsageEntry {
        .placeholder
    }

    func snapshot(for configuration: ProxyIntent, in context: Context) async -> UsageEntry {
        if context.isPreview {
            return .placeholder
        }
        applyProxy(configuration)
        return await fetchEntry()
    }

    func timeline(for configuration: ProxyIntent, in context: Context) async -> Timeline<UsageEntry> {
        applyProxy(configuration)
        let entry = await fetchEntry()
        let nextUpdate = Calendar.current.date(byAdding: .minute, value: 15, to: Date()) ?? Date().addingTimeInterval(900)
        return Timeline(entries: [entry], policy: .after(nextUpdate))
    }

    private func applyProxy(_ configuration: ProxyIntent) {
        apiClient.proxyConfig = ProxyConfig(
            enabled: configuration.proxyEnabled,
            host: configuration.proxyHost,
            port: configuration.proxyPort
        )
    }

    private func fetchEntry() async -> UsageEntry {
        guard apiClient.isConfigured else {
            return .unconfigured
        }

        do {
            let usage = try await apiClient.fetchUsage()
            return UsageEntry(date: Date(), usage: usage)
        } catch {
            if let cached = apiClient.loadCachedUsage() {
                return UsageEntry(
                    date: Date(),
                    usage: cached.usage,
                    error: nil,
                    isStale: true
                )
            }
            return UsageEntry(date: Date(), usage: nil, error: error.localizedDescription)
        }
    }
}
```

**Step 2: Commit**

```bash
git add ClaudeUsageWidget/Provider.swift
git commit -m "refactor: Provider migre vers AppIntentTimelineProvider"
```

---

### Task 9: Migrer widgets vers AppIntentConfiguration

**Files:**
- Modify: `ClaudeUsageWidget/ClaudeUsageWidget.swift`

**Step 1: Remplacer StaticConfiguration par AppIntentConfiguration**

```swift
import WidgetKit
import SwiftUI

struct ClaudeUsageWidget: Widget {
    let kind: String = "ClaudeUsageWidget"

    var body: some WidgetConfiguration {
        AppIntentConfiguration(kind: kind, intent: ProxyIntent.self, provider: Provider()) { entry in
            UsageWidgetView(entry: entry)
        }
        .configurationDisplayName("TokenEater")
        .description("Affiche votre consommation Claude en temps réel")
        .supportedFamilies([.systemMedium, .systemLarge])
    }
}

struct PacingWidget: Widget {
    let kind: String = "PacingWidget"

    var body: some WidgetConfiguration {
        AppIntentConfiguration(kind: kind, intent: ProxyIntent.self, provider: Provider()) { entry in
            PacingWidgetView(entry: entry)
        }
        .configurationDisplayName("TokenEater Pacing")
        .description("Votre rythme de consommation Claude")
        .supportedFamilies([.systemSmall])
    }
}

@main
struct ClaudeUsageWidgetBundle: WidgetBundle {
    var body: some Widget {
        ClaudeUsageWidget()
        PacingWidget()
    }
}
```

**Step 2: Commit**

```bash
git add ClaudeUsageWidget/ClaudeUsageWidget.swift
git commit -m "feat: widgets migres vers AppIntentConfiguration (proxy editable)"
```

---

### Task 10: Simplifier SettingsView.swift — OAuth only

C'est la task la plus lourde. On supprime toute l'UI cookies.

**Files:**
- Modify: `ClaudeUsageApp/SettingsView.swift`

**Step 1: Remplacer entierement le fichier**

```swift
import SwiftUI
import WidgetKit

extension Notification.Name {
    static let displaySettingsDidChange = Notification.Name("displaySettingsDidChange")
}

struct SettingsView: View {
    var onConfigSaved: (() -> Void)?

    @State private var testResult: ConnectionTestResult?
    @State private var isTesting = false
    @State private var showGuide = false
    @State private var isImporting = false
    @State private var importMessage: String?
    @State private var importSuccess = false
    @State private var authMethodLabel = ""

    @AppStorage("showMenuBar") private var showMenuBar = true
    @AppStorage("pacingDisplayMode") private var pacingDisplayMode = "dotDelta"

    @State private var pinnedFiveHour = true
    @State private var pinnedSevenDay = true
    @State private var pinnedSonnet = false
    @State private var pinnedPacing = false

    @AppStorage("proxyEnabled") private var proxyEnabled = false
    @AppStorage("proxyHost") private var proxyHost = "127.0.0.1"
    @AppStorage("proxyPort") private var proxyPort = 1080

    // Colors for guide sheet
    private let sheetBg = Color(hex: "#141416")
    private let sheetCard = Color.white.opacity(0.04)
    private let accent = Color(hex: "#FF9F0A")

    var body: some View {
        VStack(spacing: 0) {
            // App header
            HStack(spacing: 12) {
                Image(nsImage: NSImage(named: "AppIcon") ?? NSApp.applicationIconImage)
                    .resizable()
                    .frame(width: 40, height: 40)
                    .clipShape(RoundedRectangle(cornerRadius: 8))
                VStack(alignment: .leading, spacing: 1) {
                    Text("TokenEater")
                        .font(.system(size: 16, weight: .bold, design: .rounded))
                    Text("settings.subtitle")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
                Spacer()
                Text("v2.0.0")
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }
            .padding(.horizontal, 20)
            .padding(.top, 16)
            .padding(.bottom, 8)

            TabView {
                connectionTab
                    .tabItem {
                        Label("settings.tab.connection", systemImage: "bolt.horizontal.fill")
                    }
                displayTab
                    .tabItem {
                        Label("settings.tab.display", systemImage: "menubar.rectangle")
                    }
                proxyTab
                    .tabItem {
                        Label("settings.tab.proxy", systemImage: "network")
                    }
            }
        }
        .frame(width: 500, height: 400)
        .onAppear { loadConfig() }
        .sheet(isPresented: $showGuide) { guideSheet }
    }

    // MARK: - Connection Tab

    private var connectionTab: some View {
        Form {
            Section {
                HStack {
                    if isImporting {
                        ProgressView()
                            .controlSize(.small)
                    }
                    VStack(alignment: .leading, spacing: 2) {
                        Text("connect.button")
                            .fontWeight(.medium)
                        Text("connect.subtitle")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    Spacer()
                    if !authMethodLabel.isEmpty {
                        Text(authMethodLabel)
                            .font(.caption2)
                            .fontWeight(.medium)
                            .foregroundStyle(.green)
                            .padding(.horizontal, 8)
                            .padding(.vertical, 3)
                            .background(.green.opacity(0.12))
                            .clipShape(Capsule())
                    }
                    Button("connect.button") {
                        connectAutoDetect()
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(isImporting)
                }

                if let message = importMessage {
                    Label(message, systemImage: importSuccess ? "checkmark.circle.fill" : "info.circle.fill")
                        .font(.caption)
                        .foregroundStyle(importSuccess ? .green : .orange)
                }
            }

            Section {
                HStack(spacing: 12) {
                    Button {
                        testConnection()
                    } label: {
                        if isTesting {
                            ProgressView()
                                .controlSize(.small)
                        } else {
                            Label("settings.test", systemImage: "bolt.fill")
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(isTesting)

                    Button {
                        WidgetCenter.shared.reloadAllTimelines()
                    } label: {
                        Label("settings.refresh", systemImage: "arrow.clockwise")
                    }

                    Spacer()

                    Button {
                        showGuide = true
                    } label: {
                        Label("settings.guide", systemImage: "questionmark.circle")
                    }
                }

                if let result = testResult {
                    Label(
                        result.message,
                        systemImage: result.success ? "checkmark.circle.fill" : "xmark.circle.fill"
                    )
                    .foregroundStyle(result.success ? .green : .red)
                }
            }
        }
        .formStyle(.grouped)
    }

    // MARK: - Display Tab

    private var displayTab: some View {
        Form {
            Section("settings.menubar.title") {
                Toggle("settings.menubar.toggle", isOn: $showMenuBar)
            }

            Section {
                Toggle("metric.session", isOn: $pinnedFiveHour)
                Toggle("metric.weekly", isOn: $pinnedSevenDay)
                Toggle("metric.sonnet", isOn: $pinnedSonnet)
                Toggle("pacing.label", isOn: $pinnedPacing)
            } header: {
                Text("settings.metrics.pinned")
            } footer: {
                Text("settings.metrics.pinned.footer")
                    .fixedSize(horizontal: false, vertical: true)
            }
            Section("settings.pacing.display") {
                Picker("Mode", selection: $pacingDisplayMode) {
                    Text("settings.pacing.dot").tag("dot")
                    Text("settings.pacing.dotdelta").tag("dotDelta")
                }
                .pickerStyle(.radioGroup)
            }
        }
        .formStyle(.grouped)
        .onAppear { loadPinnedMetrics() }
        .onChange(of: pinnedFiveHour) { _ in savePinnedMetrics() }
        .onChange(of: pinnedSevenDay) { _ in savePinnedMetrics() }
        .onChange(of: pinnedSonnet) { _ in savePinnedMetrics() }
        .onChange(of: pinnedPacing) { _ in savePinnedMetrics() }
        .onChange(of: pacingDisplayMode) { _ in
            NotificationCenter.default.post(name: .displaySettingsDidChange, object: nil)
        }
    }

    // MARK: - Proxy Tab

    private var proxyTab: some View {
        Form {
            Section {
                Toggle("settings.proxy.toggle", isOn: $proxyEnabled)
            } footer: {
                Text("settings.proxy.footer")
            }

            Section("settings.proxy.config") {
                LabeledContent("settings.proxy.host") {
                    TextField("127.0.0.1", text: $proxyHost)
                        .textFieldStyle(.roundedBorder)
                        .font(.system(.body, design: .monospaced))
                }
                LabeledContent("settings.proxy.port") {
                    TextField("1080", value: $proxyPort, format: .number)
                        .textFieldStyle(.roundedBorder)
                        .font(.system(.body, design: .monospaced))
                        .frame(width: 80)
                }
            }
            .disabled(!proxyEnabled)

            Section {
                Text("settings.proxy.hint")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
        .formStyle(.grouped)
        .onChange(of: proxyEnabled) { _ in onConfigSaved?() }
        .onChange(of: proxyHost) { _ in onConfigSaved?() }
        .onChange(of: proxyPort) { _ in onConfigSaved?() }
    }

    // MARK: - Guide Sheet

    private var guideSheet: some View {
        ZStack {
            sheetBg.ignoresSafeArea()

            ScrollView {
                VStack(spacing: 0) {
                    HStack {
                        Text("guide.title")
                            .font(.system(size: 16, weight: .bold, design: .rounded))
                            .foregroundStyle(.white)
                        Spacer()
                        Button {
                            showGuide = false
                        } label: {
                            Image(systemName: "xmark.circle.fill")
                                .font(.system(size: 20))
                                .foregroundStyle(.white.opacity(0.3))
                        }
                        .buttonStyle(.plain)
                    }
                    .padding(.bottom, 20)

                    // Method 1: Claude Code (only method now)
                    guideSection(
                        icon: "terminal.fill",
                        color: Color(hex: "#22C55E"),
                        title: String(localized: "guide.oauth.title"),
                        badge: String(localized: "guide.oauth.badge"),
                        steps: [
                            String(localized: "guide.oauth.step1"),
                            String(localized: "guide.oauth.step2"),
                            String(localized: "guide.oauth.step3"),
                        ]
                    )

                    // Add widget
                    HStack(spacing: 12) {
                        Image(systemName: "square.grid.2x2")
                            .font(.system(size: 14))
                            .foregroundStyle(accent)
                            .frame(width: 32, height: 32)
                            .background(accent.opacity(0.12))
                            .clipShape(RoundedRectangle(cornerRadius: 8))

                        VStack(alignment: .leading, spacing: 2) {
                            Text("guide.widget")
                                .font(.system(size: 13, weight: .semibold))
                                .foregroundStyle(.white.opacity(0.9))
                            Text("guide.widget.detail")
                                .font(.system(size: 11))
                                .foregroundStyle(.white.opacity(0.45))
                        }
                        Spacer()
                    }
                    .padding(14)
                    .background(
                        RoundedRectangle(cornerRadius: 12)
                            .fill(accent.opacity(0.06))
                            .overlay(
                                RoundedRectangle(cornerRadius: 12)
                                    .stroke(accent.opacity(0.15), lineWidth: 1)
                            )
                    )
                }
                .padding(24)
            }
        }
        .frame(width: 460, height: 360)
    }

    private func guideSection(icon: String, color: Color, title: String, badge: String? = nil, steps: [String]) -> some View {
        VStack(alignment: .leading, spacing: 10) {
            HStack(spacing: 8) {
                Image(systemName: icon)
                    .font(.system(size: 13, weight: .semibold))
                    .foregroundStyle(color)
                Text(title)
                    .font(.system(size: 14, weight: .bold, design: .rounded))
                    .foregroundStyle(.white.opacity(0.9))
                if let badge = badge {
                    Text(badge)
                        .font(.system(size: 9, weight: .bold))
                        .foregroundStyle(color)
                        .padding(.horizontal, 6)
                        .padding(.vertical, 2)
                        .background(color.opacity(0.15))
                        .clipShape(Capsule())
                }
            }

            VStack(spacing: 6) {
                ForEach(Array(steps.enumerated()), id: \.offset) { index, step in
                    HStack(alignment: .top, spacing: 10) {
                        Text("\(index + 1)")
                            .font(.system(size: 10, weight: .bold, design: .rounded))
                            .foregroundStyle(color.opacity(0.8))
                            .frame(width: 18, height: 18)
                            .background(color.opacity(0.1))
                            .clipShape(Circle())
                        Text(.init(step))
                            .font(.system(size: 12))
                            .foregroundStyle(.white.opacity(0.7))
                            .fixedSize(horizontal: false, vertical: true)
                        Spacer()
                    }
                }
            }
            .padding(12)
            .background(
                RoundedRectangle(cornerRadius: 10)
                    .fill(sheetCard)
                    .overlay(
                        RoundedRectangle(cornerRadius: 10)
                            .stroke(Color.white.opacity(0.05), lineWidth: 1)
                    )
            )
        }
        .padding(.bottom, 16)
    }

    // MARK: - Config

    private func loadConfig() {
        loadPinnedMetrics()
        if KeychainOAuthReader.readClaudeCodeToken() != nil {
            authMethodLabel = String(localized: "connect.method.oauth")
        }
    }

    private func loadPinnedMetrics() {
        if let saved = UserDefaults.standard.stringArray(forKey: "pinnedMetrics") {
            let set = Set(saved)
            pinnedFiveHour = set.contains(MetricID.fiveHour.rawValue)
            pinnedSevenDay = set.contains(MetricID.sevenDay.rawValue)
            pinnedSonnet = set.contains(MetricID.sonnet.rawValue)
            pinnedPacing = set.contains(MetricID.pacing.rawValue)
        }
    }

    private func savePinnedMetrics() {
        var metrics: [String] = []
        if pinnedFiveHour { metrics.append(MetricID.fiveHour.rawValue) }
        if pinnedSevenDay { metrics.append(MetricID.sevenDay.rawValue) }
        if pinnedSonnet { metrics.append(MetricID.sonnet.rawValue) }
        if pinnedPacing { metrics.append(MetricID.pacing.rawValue) }
        if metrics.isEmpty { metrics.append(MetricID.fiveHour.rawValue); pinnedFiveHour = true }
        UserDefaults.standard.set(metrics, forKey: "pinnedMetrics")
        NotificationCenter.default.post(name: .displaySettingsDidChange, object: nil)
    }

    // MARK: - Actions

    private func testConnection() {
        isTesting = true
        testResult = nil

        Task {
            let result = await ClaudeAPIClient.shared.testConnection()
            await MainActor.run {
                testResult = result
                isTesting = false
                if result.success {
                    WidgetCenter.shared.reloadAllTimelines()
                }
            }
        }
    }

    private func connectAutoDetect() {
        isImporting = true
        importMessage = nil

        guard KeychainOAuthReader.readClaudeCodeToken() != nil else {
            isImporting = false
            importMessage = String(localized: "connect.noclaudecode")
            importSuccess = false
            return
        }

        Task {
            let result = await ClaudeAPIClient.shared.testConnection()
            await MainActor.run {
                isImporting = false
                if result.success {
                    authMethodLabel = String(localized: "connect.method.oauth")
                    importMessage = String(localized: "connect.oauth.success")
                    importSuccess = true
                    onConfigSaved?()
                } else {
                    importMessage = result.message
                    importSuccess = false
                }
            }
        }
    }
}
```

**Step 2: Commit**

```bash
git add ClaudeUsageApp/SettingsView.swift
git commit -m "refactor: SettingsView OAuth only, proxy via @AppStorage, suppression UI cookies"
```

---

### Task 11: Supprimer BrowserCookieReader.swift

**Files:**
- Delete: `ClaudeUsageApp/BrowserCookieReader.swift`

**Step 1: Supprimer le fichier**

```bash
git rm ClaudeUsageApp/BrowserCookieReader.swift
```

**Step 2: Verifier qu'aucune reference ne subsiste**

Chercher `BrowserCookieReader`, `DetectedBrowser`, `importCookies`, `detectBrowsers` dans tout le codebase. Il ne devrait plus y en avoir apres la Task 10.

**Step 3: Commit**

```bash
git commit -m "chore: suppression BrowserCookieReader (343 lignes crypto/sqlite)"
```

---

### Task 12: Mettre a jour UsageEntry.swift

**Files:**
- Modify: `ClaudeUsageWidget/UsageEntry.swift`

**Step 1: Mettre a jour le message d'erreur unconfigured**

Ligne 35, remplacer :
```swift
static var unconfigured: UsageEntry {
    UsageEntry(date: Date(), usage: nil, error: String(localized: "error.nosessionkey"))
}
```
par :
```swift
static var unconfigured: UsageEntry {
    UsageEntry(date: Date(), usage: nil, error: String(localized: "error.notoken"))
}
```

**Step 2: Commit**

```bash
git add ClaudeUsageWidget/UsageEntry.swift
git commit -m "fix: message erreur widget utilise error.notoken"
```

---

### Task 13: Nettoyer les localization strings

**Files:**
- Modify: `Shared/en.lproj/Localizable.strings`
- Modify: `Shared/fr.lproj/Localizable.strings`

**Step 1: EN — Supprimer les cles mortes et modifier**

Supprimer ces lignes dans `Shared/en.lproj/Localizable.strings` :

```
"settings.sessionkey" = ...
"settings.orgid" = ...
"settings.orgid.hint" = ...
"settings.footer" = ...
"settings.manual" = ...
"connect.method.cookies" = ...
"import.button" = ...
"import.subtitle" = ...
"import.success" = ...
"import.loading" = ...
"import.nobroser" = ...
"import.picker.title" = ...
"import.profiles" = ...
"import.expires" = ...
"guide.browser.title" = ...
"guide.browser.step1" = ...
"guide.browser.step2" = ...
"guide.browser.step3" = ...
"guide.manual.title" = ...
"guide.manual.step1" = ...
"guide.manual.step2" = ...
"guide.manual.step3" = ...
"guide.manual.step4" = ...
"guide.cookie.warning" = ...
"error.nosessionkey" = ...
"error.noorgid" = ...
"error.sessionexpired" = ...
"error.keychain.denied" = ...
"error.keychain.notfound" = ...
"error.db.copy" = ...
"error.db.open" = ...
"error.nocookies" = ...
"error.decryption" = ...
"error.missing.both" = ...
"error.missing.session" = ...
"error.missing.org" = ...
"error.missing.cookie" = ...
```

Modifier :
```
"connect.subtitle" = "Auto-detect Claude Code OAuth token";
```

Ajouter apres la section `/* Connect */` :
```
"connect.noclaudecode" = "Claude Code not detected. Install it and sign in first.";
```

Ajouter dans la section `/* Errors - API */` :
```
"error.notoken" = "No OAuth token found. Install Claude Code and click Connect.";
"error.tokenexpired" = "OAuth token expired — relaunch Claude Code to refresh it";
```

**Step 2: FR — Memes modifications**

Supprimer les memes cles dans `Shared/fr.lproj/Localizable.strings`.

Modifier :
```
"connect.subtitle" = "Détection auto du token OAuth Claude Code";
```

Ajouter :
```
"connect.noclaudecode" = "Claude Code non détecté. Installez-le et connectez-vous d'abord.";
"error.notoken" = "Aucun token OAuth trouvé. Installez Claude Code et cliquez sur Connexion.";
"error.tokenexpired" = "Token OAuth expiré — relancez Claude Code pour le rafraîchir";
```

**Step 3: Commit**

```bash
git add Shared/en.lproj/Localizable.strings Shared/fr.lproj/Localizable.strings
git commit -m "chore: nettoyage localization, suppression cles cookies"
```

---

### Task 14: Mettre a jour CLAUDE.md et README.md

**Files:**
- Modify: `CLAUDE.md`
- Modify: `README.md`

**Step 1: CLAUDE.md**

- Section API : retirer le bloc "Cookies navigateur (fallback)"
- Section Features : retirer "Auth cookies navigateur (Chrome, Arc, Brave, Edge, Chromium)", ajouter "App sandboxee"
- Section Architecture : changer "OAuth/cookie auth" → "OAuth auth"
- Section TODO : retirer "Simplification auth" et "Securite (feedback communaute)" (c'est fait)
- Mettre a jour la version dans les commandes build si necessaire

**Step 2: README.md**

- Section Authentication : retirer methodes 2 et 3 (browser cookies, manual), garder OAuth uniquement
- Section Configure : retirer "Auto-import from browser" et "Manual setup"
- Section "Supported Browsers" : supprimer entierement
- Section "How it works" : retirer le bloc cookies, garder uniquement OAuth
- Section Architecture : retirer la mention "cookie auth", ajouter "sandboxed"
- Ajouter mention "Requires macOS 14+" dans les badges ou prerequisites

**Step 3: Commit**

```bash
git add CLAUDE.md README.md
git commit -m "docs: mise a jour docs — OAuth only, app sandboxee, macOS 14+"
```

---

### Task 15: Build et test

**Step 1: Regenerer le projet Xcode**

```bash
xcodegen generate
plutil -insert NSExtension -json '{"NSExtensionPointIdentifier":"com.apple.widgetkit-extension"}' ClaudeUsageWidget/Info.plist 2>/dev/null || true
```

**Step 2: Build**

```bash
xcodebuild -project ClaudeUsageWidget.xcodeproj -scheme ClaudeUsageApp -configuration Debug -derivedDataPath build build 2>&1 | tail -5
```

Expected : `** BUILD SUCCEEDED **`

Si erreurs de compilation, les corriger et re-builder.

**Step 3: Tests manuels**

1. `open build/Build/Products/Debug/TokenEater.app`
2. Verifier que "Claude Code (auto)" s'affiche au clic "Connect"
3. Cliquer "Test" → doit afficher "Connected — Session: XX%"
4. Verifier les settings proxy (toggle, host, port persistent)
5. **Critique** : verifier que l'app sandboxee peut lire le Keychain. Si prompt macOS → cliquer "Always Allow". Si ca echoue silencieusement → retirer `com.apple.security.app-sandbox` des entitlements et re-builder.
6. Verifier que le widget se refresh et affiche les donnees

**Step 4: Commit final**

```bash
git add -A
git commit -m "feat: v2.0.0 — OAuth only, app sandboxee, suppression cookies"
```

---

## Resume des fichiers

| Action | Fichier | Raison |
|--------|---------|--------|
| **MODIFY** | `project.yml` | macOS 14+ |
| **MODIFY** | `ClaudeUsageApp/ClaudeUsageApp.entitlements` | Sandbox |
| **REWRITE** | `Shared/UsageModels.swift` | Suppression SharedConfig/Storage |
| **REWRITE** | `Shared/ClaudeAPIClient.swift` | OAuth only |
| **MODIFY** | `ClaudeUsageApp/ClaudeUsageApp.swift` | Retrait isHostApp |
| **MODIFY** | `ClaudeUsageApp/MenuBarView.swift` | isConfigured |
| **CREATE** | `ClaudeUsageWidget/ProxyIntent.swift` | AppIntent proxy |
| **REWRITE** | `ClaudeUsageWidget/Provider.swift` | AppIntentTimelineProvider |
| **REWRITE** | `ClaudeUsageWidget/ClaudeUsageWidget.swift` | AppIntentConfiguration |
| **REWRITE** | `ClaudeUsageApp/SettingsView.swift` | OAuth only UI |
| **DELETE** | `ClaudeUsageApp/BrowserCookieReader.swift` | 343 lignes mortes |
| **MODIFY** | `ClaudeUsageWidget/UsageEntry.swift` | Message erreur |
| **MODIFY** | `Shared/en.lproj/Localizable.strings` | Nettoyage |
| **MODIFY** | `Shared/fr.lproj/Localizable.strings` | Nettoyage |
| **MODIFY** | `CLAUDE.md` | Docs |
| **MODIFY** | `README.md` | Docs |
