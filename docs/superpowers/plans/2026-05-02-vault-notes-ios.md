# Sidian iOS Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a minimal iOS Markdown notes app that opens Obsidian-compatible vaults from SMB shares or cloud storage providers, with a folder tree browser, Markdown editor/preview, template support, and a selectable color scheme.

**Architecture:** UIDocumentPickerViewController selects a vault directory; a security-scoped bookmark is persisted in SwiftData so the vault reopens across sessions. All file I/O goes through a thin FileSystemService. The UI is pure SwiftUI (iOS 17+) following Apple HIG, with swift-markdown-ui handling Markdown preview rendering.

**Tech Stack:** Swift 6, SwiftUI, SwiftData, swift-markdown-ui, UIDocumentPickerViewController, security-scoped bookmarks, XCTest

---

## File Structure

```
Sidian/
├── App.swift                         # Entry point, SwiftData container
├── Models/
│   ├── VaultBookmark.swift           # SwiftData model: persisted vault
│   └── AppSettings.swift             # SwiftData model: active vault, theme, last path
├── Services/
│   ├── FileSystemService.swift       # list/read/write files via scoped URLs
│   ├── VaultManager.swift            # resolve bookmarks, start/stop security scope
│   ├── TemplateService.swift         # detect templates/, list, apply
│   └── ThemeService.swift            # built-in schemes, active theme
├── Views/
│   ├── RootView.swift                # top-level nav: vault selected vs onboarding
│   ├── Vault/
│   │   ├── OnboardingView.swift      # first-launch, pick vault
│   │   ├── VaultSwitcherView.swift   # list of saved vaults, add/remove
│   │   ├── FolderTreeView.swift      # recursive folder tree sidebar
│   │   └── FileListView.swift        # file list for selected folder
│   ├── Editor/
│   │   ├── NoteView.swift            # container: editor + preview toggle
│   │   ├── MarkdownEditorView.swift  # plain TextEditor wrapper
│   │   └── MarkdownPreviewView.swift # swift-markdown-ui renderer
│   └── Settings/
│       ├── SettingsView.swift        # settings root
│       └── ThemePickerView.swift     # color scheme grid
├── Theme/
│   └── AppTheme.swift                # Theme struct + all built-in schemes
└── Extensions/
    └── Color+Hex.swift               # Color init from hex string

SidianTests/
├── FileSystemServiceTests.swift
├── VaultManagerTests.swift
├── TemplateServiceTests.swift
└── ThemeServiceTests.swift
```

---

## Task 1: Xcode Project and Dependencies

**Files:**
- Create: `Sidian.xcodeproj` (via Xcode)
- Create: `Sidian/App.swift`
- Modify: `Package.swift` / SPM via Xcode (add swift-markdown-ui)

- [ ] **Step 1: Create Xcode project**

In Xcode: File > New > Project > App. Name: `Sidian`, Team: personal, Bundle ID: `com.vaultnotes.app`, Interface: SwiftUI, Language: Swift, Storage: SwiftData. Minimum deployment target: iOS 17.0.

- [ ] **Step 2: Add swift-markdown-ui dependency**

In Xcode: File > Add Package Dependencies. URL: `https://github.com/gonzalezreal/swift-markdown-ui`. Version: Up to Next Major from `2.4.0`. Add `MarkdownUI` to Sidian target.

- [ ] **Step 3: Configure entitlements**

In `Sidian.entitlements`, ensure these keys are present:
```xml
<key>com.apple.security.files.user-selected.read-write</key>
<true/>
<key>com.apple.security.network.client</key>
<true/>
```

- [ ] **Step 4: Replace App.swift**

```swift
import SwiftUI
import SwiftData

@main
struct SidianApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
        }
        .modelContainer(for: [VaultBookmark.self, AppSettings.self])
    }
}
```

- [ ] **Step 5: Build to verify clean project**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' build`
Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 6: Commit**

```bash
git init
git add .
git commit -m "feat: initial Xcode project with SwiftData and swift-markdown-ui"
```

---

## Task 2: Data Models

**Files:**
- Create: `Sidian/Models/VaultBookmark.swift`
- Create: `Sidian/Models/AppSettings.swift`
- Create: `SidianTests/VaultBookmarkTests.swift`

- [ ] **Step 1: Write failing test**

```swift
// SidianTests/VaultBookmarkTests.swift
import XCTest
import SwiftData
@testable import Sidian

final class VaultBookmarkTests: XCTestCase {
    func test_vaultBookmark_init_setsFields() {
        let data = Data([0x01, 0x02])
        let bookmark = VaultBookmark(name: "MyVault", bookmarkData: data)

        XCTAssertEqual(bookmark.name, "MyVault")
        XCTAssertEqual(bookmark.bookmarkData, data)
        XCTAssertNotNil(bookmark.id)
    }

    func test_appSettings_defaultTheme_isDefault() {
        let settings = AppSettings()
        XCTAssertEqual(settings.activeThemeID, "default")
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/VaultBookmarkTests test`
Expected: FAIL — `VaultBookmark` not defined.

- [ ] **Step 3: Implement VaultBookmark**

```swift
// Sidian/Models/VaultBookmark.swift
import Foundation
import SwiftData

@Model
final class VaultBookmark {
    var id: UUID
    var name: String
    var bookmarkData: Data
    var lastOpened: Date

    init(name: String, bookmarkData: Data) {
        self.id = UUID()
        self.name = name
        self.bookmarkData = bookmarkData
        self.lastOpened = Date()
    }
}
```

- [ ] **Step 4: Implement AppSettings**

```swift
// Sidian/Models/AppSettings.swift
import Foundation
import SwiftData

@Model
final class AppSettings {
    var activeVaultID: UUID?
    var activeThemeID: String
    var lastFolderPath: String?

    init() {
        self.activeThemeID = "default"
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/VaultBookmarkTests test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add Sidian/Models/ SidianTests/VaultBookmarkTests.swift
git commit -m "feat: SwiftData models for VaultBookmark and AppSettings"
```

---

## Task 3: FileSystemService

**Files:**
- Create: `Sidian/Services/FileSystemService.swift`
- Create: `SidianTests/FileSystemServiceTests.swift`

- [ ] **Step 1: Write failing tests**

```swift
// SidianTests/FileSystemServiceTests.swift
import XCTest
@testable import Sidian

final class FileSystemServiceTests: XCTestCase {
    var tempDir: URL!
    var service: FileSystemService!

    override func setUp() {
        super.setUp()
        tempDir = FileManager.default.temporaryDirectory
            .appendingPathComponent(UUID().uuidString)
        try! FileManager.default.createDirectory(at: tempDir, withIntermediateDirectories: true)
        service = FileSystemService()
    }

    override func tearDown() {
        try? FileManager.default.removeItem(at: tempDir)
        super.tearDown()
    }

    func test_listContents_returnsFiles() throws {
        let fileURL = tempDir.appendingPathComponent("note.md")
        try "# Hello".write(to: fileURL, atomically: true, encoding: .utf8)

        let contents = try service.listContents(of: tempDir)

        XCTAssertEqual(contents.count, 1)
        XCTAssertEqual(contents[0].name, "note.md")
        XCTAssertFalse(contents[0].isDirectory)
    }

    func test_listContents_marksDirectories() throws {
        let subDir = tempDir.appendingPathComponent("subfolder")
        try FileManager.default.createDirectory(at: subDir, withIntermediateDirectories: true)

        let contents = try service.listContents(of: tempDir)

        XCTAssertEqual(contents.count, 1)
        XCTAssertTrue(contents[0].isDirectory)
    }

    func test_readFile_returnsContent() throws {
        let fileURL = tempDir.appendingPathComponent("note.md")
        try "# Title".write(to: fileURL, atomically: true, encoding: .utf8)

        let content = try service.readFile(at: fileURL)

        XCTAssertEqual(content, "# Title")
    }

    func test_writeFile_persistsContent() throws {
        let fileURL = tempDir.appendingPathComponent("new.md")

        try service.writeFile(content: "## Section", to: fileURL)

        let stored = try String(contentsOf: fileURL, encoding: .utf8)
        XCTAssertEqual(stored, "## Section")
    }

    func test_createFile_returnsURLAndExists() throws {
        let url = try service.createFile(named: "test.md", in: tempDir, content: "body")

        XCTAssertTrue(FileManager.default.fileExists(atPath: url.path))
        let content = try String(contentsOf: url, encoding: .utf8)
        XCTAssertEqual(content, "body")
    }

    func test_listContents_sortsAlpha_directoriesFirst() throws {
        let subDir = tempDir.appendingPathComponent("alpha")
        try FileManager.default.createDirectory(at: subDir, withIntermediateDirectories: true)
        try "x".write(to: tempDir.appendingPathComponent("zebra.md"), atomically: true, encoding: .utf8)
        try "x".write(to: tempDir.appendingPathComponent("apple.md"), atomically: true, encoding: .utf8)

        let contents = try service.listContents(of: tempDir)

        XCTAssertTrue(contents[0].isDirectory)
        XCTAssertEqual(contents[1].name, "apple.md")
        XCTAssertEqual(contents[2].name, "zebra.md")
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/FileSystemServiceTests test`
Expected: FAIL — `FileSystemService` not defined.

- [ ] **Step 3: Implement FileSystemService**

```swift
// Sidian/Services/FileSystemService.swift
import Foundation

struct VaultFile: Identifiable, Hashable {
    let id: UUID = UUID()
    let name: String
    let url: URL
    let isDirectory: Bool
    let modifiedDate: Date?
}

final class FileSystemService {
    private let fm = FileManager.default

    func listContents(of url: URL) throws -> [VaultFile] {
        let keys: [URLResourceKey] = [.isDirectoryKey, .contentModificationDateKey, .nameKey]
        let contents = try fm.contentsOfDirectory(
            at: url,
            includingPropertiesForKeys: keys,
            options: [.skipsHiddenFiles]
        )
        return contents
            .compactMap { fileURL -> VaultFile? in
                let rv = try? fileURL.resourceValues(forKeys: Set(keys))
                let isDir = rv?.isDirectory ?? false
                return VaultFile(
                    name: fileURL.lastPathComponent,
                    url: fileURL,
                    isDirectory: isDir,
                    modifiedDate: rv?.contentModificationDate
                )
            }
            .sorted {
                if $0.isDirectory != $1.isDirectory { return $0.isDirectory }
                return $0.name.localizedCaseInsensitiveCompare($1.name) == .orderedAscending
            }
    }

    func readFile(at url: URL) throws -> String {
        try String(contentsOf: url, encoding: .utf8)
    }

    func writeFile(content: String, to url: URL) throws {
        try content.write(to: url, atomically: true, encoding: .utf8)
    }

    func createFile(named name: String, in directory: URL, content: String) throws -> URL {
        let url = directory.appendingPathComponent(name)
        try content.write(to: url, atomically: true, encoding: .utf8)
        return url
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/FileSystemServiceTests test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add Sidian/Services/FileSystemService.swift SidianTests/FileSystemServiceTests.swift
git commit -m "feat: FileSystemService with list, read, write, create"
```

---

## Task 4: VaultManager

**Files:**
- Create: `Sidian/Services/VaultManager.swift`
- Create: `SidianTests/VaultManagerTests.swift`

- [ ] **Step 1: Write failing tests**

```swift
// SidianTests/VaultManagerTests.swift
import XCTest
@testable import Sidian

final class VaultManagerTests: XCTestCase {
    var tempDir: URL!

    override func setUp() {
        super.setUp()
        tempDir = FileManager.default.temporaryDirectory
            .appendingPathComponent(UUID().uuidString)
        try! FileManager.default.createDirectory(at: tempDir, withIntermediateDirectories: true)
    }

    override func tearDown() {
        try? FileManager.default.removeItem(at: tempDir)
        super.tearDown()
    }

    func test_bookmarkData_roundtrip() throws {
        let data = try VaultManager.bookmarkData(for: tempDir)
        XCTAssertFalse(data.isEmpty)

        let resolved = try VaultManager.resolveBookmark(data)
        XCTAssertEqual(resolved.path, tempDir.path)
    }

    func test_vaultName_fromURL() {
        let name = VaultManager.vaultName(from: tempDir)
        XCTAssertEqual(name, tempDir.lastPathComponent)
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/VaultManagerTests test`
Expected: FAIL — `VaultManager` not defined.

- [ ] **Step 3: Implement VaultManager**

```swift
// Sidian/Services/VaultManager.swift
import Foundation

enum VaultManagerError: Error {
    case bookmarkStale
    case bookmarkResolutionFailed(Error)
}

final class VaultManager {
    static func bookmarkData(for url: URL) throws -> Data {
        try url.bookmarkData(
            options: .minimalBookmark,
            includingResourceValuesForKeys: nil,
            relativeTo: nil
        )
    }

    static func resolveBookmark(_ data: Data) throws -> URL {
        var isStale = false
        do {
            let url = try URL(
                resolvingBookmarkData: data,
                options: [],
                relativeTo: nil,
                bookmarkDataIsStale: &isStale
            )
            if isStale { throw VaultManagerError.bookmarkStale }
            return url
        } catch let e as VaultManagerError {
            throw e
        } catch {
            throw VaultManagerError.bookmarkResolutionFailed(error)
        }
    }

    static func vaultName(from url: URL) -> String {
        url.lastPathComponent
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/VaultManagerTests test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add Sidian/Services/VaultManager.swift SidianTests/VaultManagerTests.swift
git commit -m "feat: VaultManager for bookmark creation and resolution"
```

---

## Task 5: Theme System

**Files:**
- Create: `Sidian/Theme/AppTheme.swift`
- Create: `Sidian/Extensions/Color+Hex.swift`
- Create: `SidianTests/ThemeServiceTests.swift`

- [ ] **Step 1: Write failing tests**

```swift
// SidianTests/ThemeServiceTests.swift
import XCTest
import SwiftUI
@testable import Sidian

final class ThemeServiceTests: XCTestCase {
    func test_allThemes_notEmpty() {
        XCTAssertFalse(AppTheme.all.isEmpty)
    }

    func test_defaultTheme_exists() {
        let theme = AppTheme.all.first(where: { $0.id == "default" })
        XCTAssertNotNil(theme)
    }

    func test_theme_forID_returnsCorrectTheme() {
        let theme = AppTheme.theme(for: "dracula")
        XCTAssertEqual(theme?.name, "Dracula")
    }

    func test_theme_forUnknownID_returnsNil() {
        XCTAssertNil(AppTheme.theme(for: "does-not-exist"))
    }

    func test_colorHex_initFromSixChar() {
        let color = Color(hex: "#FF5733")
        XCTAssertNotNil(color)
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/ThemeServiceTests test`
Expected: FAIL — `AppTheme` not defined.

- [ ] **Step 3: Implement Color+Hex**

```swift
// Sidian/Extensions/Color+Hex.swift
import SwiftUI

extension Color {
    init?(hex: String) {
        var str = hex.trimmingCharacters(in: .whitespacesAndNewlines)
        if str.hasPrefix("#") { str = String(str.dropFirst()) }
        guard str.count == 6, let value = UInt64(str, radix: 16) else { return nil }
        self.init(
            red: Double((value >> 16) & 0xFF) / 255,
            green: Double((value >> 8) & 0xFF) / 255,
            blue: Double(value & 0xFF) / 255
        )
    }
}
```

- [ ] **Step 4: Implement AppTheme**

```swift
// Sidian/Theme/AppTheme.swift
import SwiftUI

struct AppTheme: Identifiable {
    let id: String
    let name: String
    let background: Color
    let foreground: Color
    let secondaryForeground: Color
    let accent: Color
    let editorBackground: Color
    let sidebarBackground: Color

    static let all: [AppTheme] = [
        .default,
        .dracula,
        .nord,
        .solarizedLight,
        .solarizedDark,
        .tokyoNight,
        .catppuccinMocha,
        .gruvboxDark,
    ]

    static func theme(for id: String) -> AppTheme? {
        all.first(where: { $0.id == id })
    }
}

extension AppTheme {
    static let `default` = AppTheme(
        id: "default",
        name: "Default",
        background: Color(UIColor.systemBackground),
        foreground: Color(UIColor.label),
        secondaryForeground: Color(UIColor.secondaryLabel),
        accent: Color.accentColor,
        editorBackground: Color(UIColor.systemBackground),
        sidebarBackground: Color(UIColor.secondarySystemBackground)
    )

    static let dracula = AppTheme(
        id: "dracula",
        name: "Dracula",
        background: Color(hex: "#282a36")!,
        foreground: Color(hex: "#f8f8f2")!,
        secondaryForeground: Color(hex: "#6272a4")!,
        accent: Color(hex: "#bd93f9")!,
        editorBackground: Color(hex: "#282a36")!,
        sidebarBackground: Color(hex: "#21222c")!
    )

    static let nord = AppTheme(
        id: "nord",
        name: "Nord",
        background: Color(hex: "#2e3440")!,
        foreground: Color(hex: "#eceff4")!,
        secondaryForeground: Color(hex: "#4c566a")!,
        accent: Color(hex: "#88c0d0")!,
        editorBackground: Color(hex: "#2e3440")!,
        sidebarBackground: Color(hex: "#3b4252")!
    )

    static let solarizedLight = AppTheme(
        id: "solarized-light",
        name: "Solarized Light",
        background: Color(hex: "#fdf6e3")!,
        foreground: Color(hex: "#657b83")!,
        secondaryForeground: Color(hex: "#93a1a1")!,
        accent: Color(hex: "#268bd2")!,
        editorBackground: Color(hex: "#fdf6e3")!,
        sidebarBackground: Color(hex: "#eee8d5")!
    )

    static let solarizedDark = AppTheme(
        id: "solarized-dark",
        name: "Solarized Dark",
        background: Color(hex: "#002b36")!,
        foreground: Color(hex: "#839496")!,
        secondaryForeground: Color(hex: "#586e75")!,
        accent: Color(hex: "#268bd2")!,
        editorBackground: Color(hex: "#002b36")!,
        sidebarBackground: Color(hex: "#073642")!
    )

    static let tokyoNight = AppTheme(
        id: "tokyo-night",
        name: "Tokyo Night",
        background: Color(hex: "#1a1b26")!,
        foreground: Color(hex: "#a9b1d6")!,
        secondaryForeground: Color(hex: "#565f89")!,
        accent: Color(hex: "#7aa2f7")!,
        editorBackground: Color(hex: "#1a1b26")!,
        sidebarBackground: Color(hex: "#16161e")!
    )

    static let catppuccinMocha = AppTheme(
        id: "catppuccin-mocha",
        name: "Catppuccin Mocha",
        background: Color(hex: "#1e1e2e")!,
        foreground: Color(hex: "#cdd6f4")!,
        secondaryForeground: Color(hex: "#6c7086")!,
        accent: Color(hex: "#cba6f7")!,
        editorBackground: Color(hex: "#1e1e2e")!,
        sidebarBackground: Color(hex: "#181825")!
    )

    static let gruvboxDark = AppTheme(
        id: "gruvbox-dark",
        name: "Gruvbox Dark",
        background: Color(hex: "#282828")!,
        foreground: Color(hex: "#ebdbb2")!,
        secondaryForeground: Color(hex: "#928374")!,
        accent: Color(hex: "#fabd2f")!,
        editorBackground: Color(hex: "#282828")!,
        sidebarBackground: Color(hex: "#1d2021")!
    )
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/ThemeServiceTests test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add Sidian/Theme/ Sidian/Extensions/ SidianTests/ThemeServiceTests.swift
git commit -m "feat: AppTheme with 8 built-in color schemes"
```

---

## Task 6: TemplateService

**Files:**
- Create: `Sidian/Services/TemplateService.swift`
- Create: `SidianTests/TemplateServiceTests.swift`

- [ ] **Step 1: Write failing tests**

```swift
// SidianTests/TemplateServiceTests.swift
import XCTest
@testable import Sidian

final class TemplateServiceTests: XCTestCase {
    var tempDir: URL!
    var templatesDir: URL!
    var service: TemplateService!

    override func setUp() {
        super.setUp()
        tempDir = FileManager.default.temporaryDirectory
            .appendingPathComponent(UUID().uuidString)
        templatesDir = tempDir.appendingPathComponent("Templates")
        try! FileManager.default.createDirectory(at: templatesDir, withIntermediateDirectories: true)
        service = TemplateService(fileService: FileSystemService())
    }

    override func tearDown() {
        try? FileManager.default.removeItem(at: tempDir)
        super.tearDown()
    }

    func test_templatesFolder_detected_whenExists() {
        let result = service.templatesFolder(in: tempDir)
        XCTAssertNotNil(result)
    }

    func test_templatesFolder_returnsNil_whenAbsent() {
        let emptyDir = FileManager.default.temporaryDirectory
            .appendingPathComponent(UUID().uuidString)
        try! FileManager.default.createDirectory(at: emptyDir, withIntermediateDirectories: true)
        defer { try? FileManager.default.removeItem(at: emptyDir) }

        XCTAssertNil(service.templatesFolder(in: emptyDir))
    }

    func test_listTemplates_returnsMarkdownFiles() throws {
        try "# Template".write(to: templatesDir.appendingPathComponent("Daily Note.md"), atomically: true, encoding: .utf8)
        try "# Other".write(to: templatesDir.appendingPathComponent("Meeting.md"), atomically: true, encoding: .utf8)

        let templates = try service.listTemplates(in: templatesDir)

        XCTAssertEqual(templates.count, 2)
        XCTAssertTrue(templates.allSatisfy { $0.name.hasSuffix(".md") })
    }

    func test_applyTemplate_returnsContent() throws {
        let templateURL = templatesDir.appendingPathComponent("Daily.md")
        try "# Daily Note\n\n## Tasks\n".write(to: templateURL, atomically: true, encoding: .utf8)

        let content = try service.applyTemplate(at: templateURL)

        XCTAssertEqual(content, "# Daily Note\n\n## Tasks\n")
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/TemplateServiceTests test`
Expected: FAIL — `TemplateService` not defined.

- [ ] **Step 3: Implement TemplateService**

```swift
// Sidian/Services/TemplateService.swift
import Foundation

final class TemplateService {
    private let fileService: FileSystemService
    private let templatesFolderNames = ["Templates", "templates", "_templates"]

    init(fileService: FileSystemService) {
        self.fileService = fileService
    }

    func templatesFolder(in vaultRoot: URL) -> URL? {
        for name in templatesFolderNames {
            let candidate = vaultRoot.appendingPathComponent(name)
            var isDir: ObjCBool = false
            if FileManager.default.fileExists(atPath: candidate.path, isDirectory: &isDir), isDir.boolValue {
                return candidate
            }
        }
        return nil
    }

    func listTemplates(in folder: URL) throws -> [VaultFile] {
        let contents = try fileService.listContents(of: folder)
        return contents.filter { !$0.isDirectory && $0.name.hasSuffix(".md") }
    }

    func applyTemplate(at url: URL) throws -> String {
        try fileService.readFile(at: url)
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `xcodebuild -scheme Sidian -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:SidianTests/TemplateServiceTests test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add Sidian/Services/TemplateService.swift SidianTests/TemplateServiceTests.swift
git commit -m "feat: TemplateService detects templates folder and lists templates"
```

---

## Task 7: RootView and Onboarding

**Files:**
- Create: `Sidian/Views/RootView.swift`
- Create: `Sidian/Views/Vault/OnboardingView.swift`

These views are wired to SwiftData — no unit test, verify visually in simulator.

- [ ] **Step 1: Implement OnboardingView**

```swift
// Sidian/Views/Vault/OnboardingView.swift
import SwiftUI
import SwiftData

struct OnboardingView: View {
    @Environment(\.modelContext) private var modelContext
    @State private var showingPicker = false
    var onVaultAdded: () -> Void

    var body: some View {
        VStack(spacing: 24) {
            Spacer()
            Image(systemName: "doc.text.magnifyingglass")
                .font(.system(size: 64))
                .foregroundStyle(.secondary)
            Text("No Vault Selected")
                .font(.title2)
                .fontWeight(.semibold)
            Text("Choose a folder to use as your vault. Works with iCloud Drive, SMB shares (added in Files), Dropbox, OneDrive, and Google Drive.")
                .font(.body)
                .multilineTextAlignment(.center)
                .foregroundStyle(.secondary)
                .padding(.horizontal, 32)
            Button("Open Vault Folder") {
                showingPicker = true
            }
            .buttonStyle(.borderedProminent)
            Spacer()
        }
        .sheet(isPresented: $showingPicker) {
            DocumentPickerView { url in
                addVault(url: url)
            }
        }
    }

    private func addVault(url: URL) {
        guard url.startAccessingSecurityScopedResource() else { return }
        defer { url.stopAccessingSecurityScopedResource() }
        guard let data = try? VaultManager.bookmarkData(for: url) else { return }
        let bookmark = VaultBookmark(name: VaultManager.vaultName(from: url), bookmarkData: data)
        modelContext.insert(bookmark)
        let settings = AppSettings()
        settings.activeVaultID = bookmark.id
        modelContext.insert(settings)
        onVaultAdded()
    }
}
```

- [ ] **Step 2: Implement DocumentPickerView (UIKit wrapper)**

```swift
// Sidian/Views/Vault/DocumentPickerView.swift
import SwiftUI
import UniformTypeIdentifiers

struct DocumentPickerView: UIViewControllerRepresentable {
    var onPick: (URL) -> Void

    func makeCoordinator() -> Coordinator { Coordinator(onPick: onPick) }

    func makeUIViewController(context: Context) -> UIDocumentPickerViewController {
        let picker = UIDocumentPickerViewController(forOpeningContentTypes: [.folder])
        picker.allowsMultipleSelection = false
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIDocumentPickerViewController, context: Context) {}

    final class Coordinator: NSObject, UIDocumentPickerDelegate {
        let onPick: (URL) -> Void
        init(onPick: @escaping (URL) -> Void) { self.onPick = onPick }

        func documentPicker(_ controller: UIDocumentPickerViewController, didPickDocumentsAt urls: [URL]) {
            guard let url = urls.first else { return }
            onPick(url)
        }
    }
}
```

- [ ] **Step 3: Implement RootView**

```swift
// Sidian/Views/RootView.swift
import SwiftUI
import SwiftData

struct RootView: View {
    @Query private var bookmarks: [VaultBookmark]
    @Query private var settings: [AppSettings]
    @State private var activeVaultURL: URL?
    @State private var refreshToken = UUID()

    private var activeSettings: AppSettings? { settings.first }

    var body: some View {
        Group {
            if let url = activeVaultURL {
                VaultBrowserView(vaultURL: url)
                    .id(refreshToken)
            } else {
                OnboardingView {
                    resolveActiveVault()
                }
            }
        }
        .onAppear { resolveActiveVault() }
    }

    private func resolveActiveVault() {
        guard let activeID = activeSettings?.activeVaultID,
              let bookmark = bookmarks.first(where: { $0.id == activeID }),
              let url = try? VaultManager.resolveBookmark(bookmark.bookmarkData) else {
            activeVaultURL = nil
            return
        }
        activeVaultURL = url
        refreshToken = UUID()
    }
}
```

- [ ] **Step 4: Build and run in simulator**

Run in Xcode on iPhone 16 simulator. Expected: onboarding screen with "Open Vault Folder" button. Tapping it opens the Files picker.

- [ ] **Step 5: Commit**

```bash
git add Sidian/Views/
git commit -m "feat: onboarding flow with document picker and bookmark persistence"
```

---

## Task 8: Vault Browser — Folder Tree and File List

**Files:**
- Create: `Sidian/Views/Vault/VaultBrowserView.swift`
- Create: `Sidian/Views/Vault/FolderTreeView.swift`
- Create: `Sidian/Views/Vault/FileListView.swift`
- Create: `Sidian/Views/Vault/VaultSwitcherView.swift`

- [ ] **Step 1: Implement VaultBrowserView (NavigationSplitView)**

```swift
// Sidian/Views/Vault/VaultBrowserView.swift
import SwiftUI

struct VaultBrowserView: View {
    let vaultURL: URL
    @State private var selectedFolder: URL?
    @State private var selectedFile: URL?
    @State private var showingSwitcher = false

    var body: some View {
        NavigationSplitView {
            FolderTreeView(rootURL: vaultURL, selectedFolder: $selectedFolder)
                .navigationTitle(vaultURL.lastPathComponent)
                .toolbar {
                    ToolbarItem(placement: .topBarLeading) {
                        Button("Vaults", systemImage: "tray.2") {
                            showingSwitcher = true
                        }
                    }
                    ToolbarItem(placement: .topBarTrailing) {
                        NavigationLink(destination: SettingsView()) {
                            Image(systemName: "gear")
                        }
                    }
                }
        } content: {
            if let folder = selectedFolder ?? vaultURL as URL? {
                FileListView(folderURL: folder, selectedFile: $selectedFile, vaultURL: vaultURL)
            } else {
                ContentUnavailableView("Select a folder", systemImage: "folder")
            }
        } detail: {
            if let file = selectedFile {
                NoteView(fileURL: file)
            } else {
                ContentUnavailableView("Select a note", systemImage: "doc.text")
            }
        }
        .sheet(isPresented: $showingSwitcher) {
            VaultSwitcherView()
        }
    }
}
```

- [ ] **Step 2: Implement FolderTreeView**

```swift
// Sidian/Views/Vault/FolderTreeView.swift
import SwiftUI

struct FolderTreeView: View {
    let rootURL: URL
    @Binding var selectedFolder: URL?
    @State private var folders: [VaultFile] = []
    private let fileService = FileSystemService()

    var body: some View {
        List(selection: $selectedFolder) {
            Section("Vault Root") {
                FolderRowView(
                    file: VaultFile(name: rootURL.lastPathComponent, url: rootURL, isDirectory: true, modifiedDate: nil),
                    selectedFolder: $selectedFolder,
                    fileService: fileService
                )
            }
        }
        .listStyle(.sidebar)
    }
}

struct FolderRowView: View {
    let file: VaultFile
    @Binding var selectedFolder: URL?
    let fileService: FileSystemService
    @State private var children: [VaultFile] = []
    @State private var isExpanded = false
    @State private var isLoaded = false

    var body: some View {
        DisclosureGroup(isExpanded: $isExpanded) {
            ForEach(children.filter(\.isDirectory)) { child in
                FolderRowView(file: child, selectedFolder: $selectedFolder, fileService: fileService)
            }
        } label: {
            Label(file.name, systemImage: "folder")
                .onTapGesture { selectedFolder = file.url }
        }
        .onChange(of: isExpanded) { _, expanded in
            if expanded, !isLoaded { loadChildren() }
        }
    }

    private func loadChildren() {
        isLoaded = true
        children = (try? fileService.listContents(of: file.url).filter(\.isDirectory)) ?? []
    }
}
```

- [ ] **Step 3: Implement FileListView**

```swift
// Sidian/Views/Vault/FileListView.swift
import SwiftUI

struct FileListView: View {
    let folderURL: URL
    @Binding var selectedFile: URL?
    let vaultURL: URL
    @State private var files: [VaultFile] = []
    @State private var showingNewNote = false
    @State private var newNoteName = ""
    @State private var selectedTemplate: VaultFile?
    @State private var templates: [VaultFile] = []
    @State private var showingTemplatePicker = false
    private let fileService = FileSystemService()
    private let templateService = TemplateService(fileService: FileSystemService())

    var body: some View {
        List(files.filter { !$0.isDirectory }, id: \.url, selection: $selectedFile) { file in
            VStack(alignment: .leading, spacing: 2) {
                Text(file.name.replacingOccurrences(of: ".md", with: ""))
                    .font(.body)
                if let date = file.modifiedDate {
                    Text(date.formatted(date: .abbreviated, time: .shortened))
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .tag(file.url)
        }
        .navigationTitle(folderURL.lastPathComponent)
        .toolbar {
            ToolbarItem(placement: .topBarTrailing) {
                Button("New Note", systemImage: "square.and.pencil") {
                    prepareNewNote()
                }
            }
        }
        .onAppear { loadFiles() }
        .onChange(of: folderURL) { _, _ in loadFiles() }
        .alert("New Note", isPresented: $showingNewNote) {
            TextField("Note name", text: $newNoteName)
            if !templates.isEmpty {
                Button("Choose Template") { showingTemplatePicker = true }
            }
            Button("Create") { createNote() }
            Button("Cancel", role: .cancel) {}
        }
        .sheet(isPresented: $showingTemplatePicker) {
            TemplatePickerView(templates: templates, selected: $selectedTemplate)
        }
    }

    private func loadFiles() {
        files = (try? fileService.listContents(of: folderURL)) ?? []
    }

    private func prepareNewNote() {
        newNoteName = ""
        selectedTemplate = nil
        if let templatesFolder = templateService.templatesFolder(in: vaultURL) {
            templates = (try? templateService.listTemplates(in: templatesFolder)) ?? []
        }
        showingNewNote = true
    }

    private func createNote() {
        var content = ""
        if let template = selectedTemplate {
            content = (try? templateService.applyTemplate(at: template.url)) ?? ""
        }
        var name = newNoteName.trimmingCharacters(in: .whitespacesAndNewlines)
        if !name.hasSuffix(".md") { name += ".md" }
        if let url = try? fileService.createFile(named: name, in: folderURL, content: content) {
            loadFiles()
            selectedFile = url
        }
    }
}
```

- [ ] **Step 4: Implement VaultSwitcherView**

```swift
// Sidian/Views/Vault/VaultSwitcherView.swift
import SwiftUI
import SwiftData

struct VaultSwitcherView: View {
    @Environment(\.dismiss) private var dismiss
    @Environment(\.modelContext) private var modelContext
    @Query private var bookmarks: [VaultBookmark]
    @Query private var settings: [AppSettings]
    @State private var showingPicker = false

    private var activeSettings: AppSettings? { settings.first }

    var body: some View {
        NavigationStack {
            List {
                ForEach(bookmarks) { bookmark in
                    HStack {
                        VStack(alignment: .leading) {
                            Text(bookmark.name).font(.body)
                            Text(bookmark.lastOpened.formatted(date: .abbreviated, time: .omitted))
                                .font(.caption).foregroundStyle(.secondary)
                        }
                        Spacer()
                        if activeSettings?.activeVaultID == bookmark.id {
                            Image(systemName: "checkmark").foregroundStyle(.accent)
                        }
                    }
                    .contentShape(Rectangle())
                    .onTapGesture { activate(bookmark) }
                }
                .onDelete { indexSet in
                    for i in indexSet { modelContext.delete(bookmarks[i]) }
                }
            }
            .navigationTitle("Vaults")
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    Button("Add Vault", systemImage: "plus") { showingPicker = true }
                }
                ToolbarItem(placement: .topBarLeading) {
                    Button("Done") { dismiss() }
                }
            }
        }
        .sheet(isPresented: $showingPicker) {
            DocumentPickerView { url in
                addVault(url: url)
            }
        }
    }

    private func activate(_ bookmark: VaultBookmark) {
        bookmark.lastOpened = Date()
        activeSettings?.activeVaultID = bookmark.id
        dismiss()
    }

    private func addVault(url: URL) {
        guard url.startAccessingSecurityScopedResource() else { return }
        defer { url.stopAccessingSecurityScopedResource() }
        guard let data = try? VaultManager.bookmarkData(for: url) else { return }
        let bookmark = VaultBookmark(name: VaultManager.vaultName(from: url), bookmarkData: data)
        modelContext.insert(bookmark)
    }
}
```

- [ ] **Step 5: Build and run in simulator, verify folder tree and file list**

Open a test folder. Expected: sidebar shows folder tree with disclosure groups. Content area shows .md files with name and date. "New Note" button triggers alert.

- [ ] **Step 6: Commit**

```bash
git add Sidian/Views/Vault/
git commit -m "feat: vault browser with folder tree, file list, and vault switcher"
```

---

## Task 9: Template Picker View

**Files:**
- Create: `Sidian/Views/Vault/TemplatePickerView.swift`

- [ ] **Step 1: Implement TemplatePickerView**

```swift
// Sidian/Views/Vault/TemplatePickerView.swift
import SwiftUI

struct TemplatePickerView: View {
    let templates: [VaultFile]
    @Binding var selected: VaultFile?
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            List(templates, id: \.url, selection: $selected) { template in
                HStack {
                    Text(template.name.replacingOccurrences(of: ".md", with: ""))
                    Spacer()
                    if selected?.url == template.url {
                        Image(systemName: "checkmark").foregroundStyle(.accent)
                    }
                }
                .contentShape(Rectangle())
                .onTapGesture {
                    selected = template
                    dismiss()
                }
            }
            .navigationTitle("Choose Template")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button("No Template") {
                        selected = nil
                        dismiss()
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 2: Build and run**

Create a `Templates/` folder inside a test vault with two `.md` files. Tap "New Note". Expected: "Choose Template" button appears. Tapping it shows the template picker list.

- [ ] **Step 3: Commit**

```bash
git add Sidian/Views/Vault/TemplatePickerView.swift
git commit -m "feat: template picker for new note creation"
```

---

## Task 10: Note Editor and Preview

**Files:**
- Create: `Sidian/Views/Editor/NoteView.swift`
- Create: `Sidian/Views/Editor/MarkdownEditorView.swift`
- Create: `Sidian/Views/Editor/MarkdownPreviewView.swift`

- [ ] **Step 1: Implement MarkdownEditorView**

```swift
// Sidian/Views/Editor/MarkdownEditorView.swift
import SwiftUI

struct MarkdownEditorView: View {
    @Binding var text: String
    var theme: AppTheme

    var body: some View {
        TextEditor(text: $text)
            .font(.system(.body, design: .monospaced))
            .foregroundStyle(theme.foreground)
            .scrollContentBackground(.hidden)
            .background(theme.editorBackground)
            .padding(.horizontal, 8)
    }
}
```

- [ ] **Step 2: Implement MarkdownPreviewView**

```swift
// Sidian/Views/Editor/MarkdownPreviewView.swift
import SwiftUI
import MarkdownUI

struct MarkdownPreviewView: View {
    let text: String
    var theme: AppTheme

    var body: some View {
        ScrollView {
            Markdown(text)
                .markdownTheme(.gitHub)
                .padding()
                .frame(maxWidth: .infinity, alignment: .leading)
        }
        .background(theme.editorBackground)
    }
}
```

- [ ] **Step 3: Implement NoteView**

```swift
// Sidian/Views/Editor/NoteView.swift
import SwiftUI
import SwiftData

struct NoteView: View {
    let fileURL: URL
    @State private var content: String = ""
    @State private var isEditing: Bool = false
    @State private var isSaved: Bool = true
    @Query private var settings: [AppSettings]
    private let fileService = FileSystemService()

    private var activeTheme: AppTheme {
        guard let id = settings.first?.activeThemeID else { return .default }
        return AppTheme.theme(for: id) ?? .default
    }

    var body: some View {
        Group {
            if isEditing {
                MarkdownEditorView(text: $content, theme: activeTheme)
                    .onChange(of: content) { _, _ in isSaved = false }
            } else {
                MarkdownPreviewView(text: content, theme: activeTheme)
            }
        }
        .background(activeTheme.editorBackground)
        .navigationTitle(fileURL.deletingPathExtension().lastPathComponent)
        .navigationBarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItem(placement: .topBarTrailing) {
                HStack {
                    if !isSaved {
                        Button("Save", systemImage: "arrow.down.doc") { save() }
                            .tint(.orange)
                    }
                    Button(isEditing ? "Preview" : "Edit",
                           systemImage: isEditing ? "eye" : "pencil") {
                        if isEditing { save() }
                        isEditing.toggle()
                    }
                }
            }
        }
        .onAppear { load() }
        .onChange(of: fileURL) { _, _ in load() }
    }

    private func load() {
        content = (try? fileService.readFile(at: fileURL)) ?? ""
        isSaved = true
        isEditing = false
    }

    private func save() {
        try? fileService.writeFile(content: content, to: fileURL)
        isSaved = true
    }
}
```

- [ ] **Step 4: Build and run in simulator**

Open a vault, select a `.md` file. Expected: preview rendered by swift-markdown-ui. Tap "Edit" to switch to plain text editor. Tap "Save" + "Preview" to return. Unsaved indicator (orange Save button) appears when content changes.

- [ ] **Step 5: Commit**

```bash
git add Sidian/Views/Editor/
git commit -m "feat: Markdown editor and preview with save/edit toggle"
```

---

## Task 11: Settings and Theme Picker

**Files:**
- Create: `Sidian/Views/Settings/SettingsView.swift`
- Create: `Sidian/Views/Settings/ThemePickerView.swift`

- [ ] **Step 1: Implement ThemePickerView**

```swift
// Sidian/Views/Settings/ThemePickerView.swift
import SwiftUI
import SwiftData

struct ThemePickerView: View {
    @Query private var settings: [AppSettings]
    @Environment(\.modelContext) private var modelContext

    private var activeSettings: AppSettings? { settings.first }
    private let columns = [GridItem(.adaptive(minimum: 140))]

    var body: some View {
        ScrollView {
            LazyVGrid(columns: columns, spacing: 16) {
                ForEach(AppTheme.all) { theme in
                    ThemeSwatchView(theme: theme, isActive: activeSettings?.activeThemeID == theme.id)
                        .onTapGesture { activeSettings?.activeThemeID = theme.id }
                }
            }
            .padding()
        }
        .navigationTitle("Color Scheme")
    }
}

struct ThemeSwatchView: View {
    let theme: AppTheme
    let isActive: Bool

    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            ZStack {
                RoundedRectangle(cornerRadius: 8, style: .continuous)
                    .fill(theme.background)
                    .frame(height: 80)
                VStack(alignment: .leading, spacing: 4) {
                    Text("## Heading")
                        .font(.system(size: 11, weight: .bold, design: .monospaced))
                        .foregroundStyle(theme.foreground)
                    Text("Normal text body")
                        .font(.system(size: 10, design: .monospaced))
                        .foregroundStyle(theme.secondaryForeground)
                    Text("[[Link]]")
                        .font(.system(size: 10, design: .monospaced))
                        .foregroundStyle(theme.accent)
                }
                .padding(8)
            }
            Text(theme.name)
                .font(.caption)
                .padding(.top, 4)
        }
        .overlay(
            RoundedRectangle(cornerRadius: 8, style: .continuous)
                .strokeBorder(isActive ? theme.accent : .clear, lineWidth: 2)
        )
    }
}
```

- [ ] **Step 2: Implement SettingsView**

```swift
// Sidian/Views/Settings/SettingsView.swift
import SwiftUI

struct SettingsView: View {
    var body: some View {
        List {
            NavigationLink("Color Scheme") {
                ThemePickerView()
            }
        }
        .navigationTitle("Settings")
    }
}
```

- [ ] **Step 3: Build and run in simulator**

Open Settings from vault browser toolbar. Tap "Color Scheme". Expected: grid of 8 theme swatches with mini text preview. Tap a swatch. Expected: active swatch gains accent border. Return to vault, open a note. Expected: note uses selected theme colors.

- [ ] **Step 4: Commit**

```bash
git add Sidian/Views/Settings/
git commit -m "feat: settings screen and color scheme picker with live swatches"
```

---

## Task 12: Security-Scope Lifecycle and Background Access

iOS revokes security-scoped access when the app goes to background. Wrap vault file access so it restarts scope on foreground.

**Files:**
- Modify: `Sidian/Views/Vault/VaultBrowserView.swift`
- Modify: `Sidian/Views/Editor/NoteView.swift`

- [ ] **Step 1: Add scoped access management to VaultBrowserView**

In `VaultBrowserView`, add `@State private var scopeActive = false` and handle `.onAppear` / scene phase changes:

```swift
// Add to VaultBrowserView body modifiers:
.onAppear { startScope() }
.onDisappear { stopScope() }
.onChange(of: scenePhase) { _, phase in
    if phase == .active { startScope() }
    else if phase == .background { stopScope() }
}
```

Add `@Environment(\.scenePhase) private var scenePhase` at top of struct, and:

```swift
private func startScope() {
    guard !scopeActive else { return }
    scopeActive = vaultURL.startAccessingSecurityScopedResource()
}

private func stopScope() {
    guard scopeActive else { return }
    vaultURL.stopAccessingSecurityScopedResource()
    scopeActive = false
}
```

- [ ] **Step 2: Apply same pattern in NoteView for save**

In `NoteView.save()`, wrap file write in security scope:

```swift
private func save() {
    let accessed = fileURL.startAccessingSecurityScopedResource()
    defer { if accessed { fileURL.stopAccessingSecurityScopedResource() } }
    try? fileService.writeFile(content: content, to: fileURL)
    isSaved = true
}
```

- [ ] **Step 3: Test background/foreground cycle manually**

Run on simulator. Open a vault note, edit content, background the app (Home button), foreground it, tap Save. Expected: file saves without error.

- [ ] **Step 4: Commit**

```bash
git add Sidian/Views/Vault/VaultBrowserView.swift Sidian/Views/Editor/NoteView.swift
git commit -m "fix: restart security-scoped access on app foreground"
```

---

## Task 13: App Icon and Launch Screen

**Files:**
- Modify: `Assets.xcassets/AppIcon.appiconset/`

- [ ] **Step 1: Generate app icon**

Use SF Symbol `doc.text` as basis. In Xcode, generate icon set at 1024x1024 using an icon tool (e.g., https://appicon.co or Sketch/Figma). Icon: dark background matching Dracula theme (`#282a36`), white `doc.text` SF Symbol centered.

- [ ] **Step 2: Set in Asset Catalog**

Drag generated images into `Assets.xcassets/AppIcon.appiconset/` in Xcode. Verify no missing size warnings.

- [ ] **Step 3: Verify launch**

Build and run. Expected: app icon appears correctly in simulator home screen. No launch screen delay (default behavior).

- [ ] **Step 4: Commit**

```bash
git add Assets.xcassets/
git commit -m "feat: app icon"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] SMB share access via Files app: covered by `UIDocumentPickerViewController` (iOS Files app supports SMB shares added by user)
- [x] Cloud sync (OneDrive/Dropbox/GDrive/iCloud): covered by same document picker (providers expose via FileProvider if app installed)
- [x] Multiple vault selection: `VaultSwitcherView` + `VaultBookmark` list
- [x] Folder tree: `FolderTreeView` with recursive `DisclosureGroup`
- [x] Note reading: `MarkdownPreviewView` with swift-markdown-ui
- [x] Note writing: `MarkdownEditorView` with `TextEditor`
- [x] Apple HIG design: SwiftUI `NavigationSplitView`, `List`, system SF Symbols, system fonts
- [x] Templates folder detection: `TemplateService.templatesFolder(in:)` checks `Templates/`, `templates/`, `_templates/`
- [x] Template picker on new note: `TemplatePickerView` sheet
- [x] Color schemes: 8 built-in schemes in `AppTheme.all`
- [x] Color scheme picker: `ThemePickerView` with visual swatches
- [x] GitHub project: created separately via `gh` CLI (see setup notes)

**Gaps identified:** None.

**Placeholder scan:** None found.

**Type consistency:**
- `VaultFile` defined Task 3, used throughout Tasks 7, 8, 9, 10 consistently
- `AppTheme` defined Task 5, used in Tasks 10, 11 consistently
- `TemplateService` init signature matches between Task 6 and Task 8
- `VaultManager.bookmarkData(for:)` and `resolveBookmark(_:)` signatures consistent across Tasks 4, 7, 8
