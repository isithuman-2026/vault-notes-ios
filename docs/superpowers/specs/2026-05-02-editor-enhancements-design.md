# Editor Enhancements Design

## Scope

Five independent features added on top of the base vault-notes app:

1. UITextView editor (prerequisite for all other features)
2. Floating Markdown keyboard bar
3. Note navigator (headings + wikilinks)
4. Navigation history (back stack)
5. Code block rendering (preview)
6. SMB offline cache and sync
7. App lock (FaceID / device passcode)

Apple Intelligence integration is out of scope for this iteration (wishlist).

---

## 1. UITextView Editor

### Problem

SwiftUI `TextEditor` does not expose `selectedRange` or programmatic scroll control. Both are required for wrap-selection and heading navigation.

### Design

Replace `TextEditor` in `MarkdownEditorView.swift` with a `UIViewRepresentable` wrapping `UITextView`. The swap is internal to that file; `NoteView` and all callers are unchanged.

The `Coordinator` conforms to `UITextViewDelegate` and handles two-way text binding (`textViewDidChange`). Direct manipulation (insert, wrap, scroll) is exposed through an `EditorController` class:

```swift
final class EditorController: ObservableObject {
    weak var textView: UITextView?
    func insertMarkdown(syntax: MarkdownSyntax) { ... }
    func scrollToRange(_ range: NSRange) { ... }
    func currentScrollOffset() -> CGFloat { ... }
}
```

`NoteView` owns `@StateObject var editorController = EditorController()` and passes it to both `MarkdownEditorView` (which sets `editorController.textView` on make) and `MarkdownBarView` / `NoteNavigatorView` (which call its methods).

Theming (background, foreground, font) is applied directly to `UITextView` properties, matching existing `AppTheme` values.

**Files changed:**
- `VaultNotes/Views/Editor/MarkdownEditorView.swift` — full replacement
- `VaultNotes/Views/Editor/EditorController.swift` — new `ObservableObject`

---

## 2. Floating Markdown Bar

### Design

A SwiftUI `HStack` rendered in `NoteView` via `.safeAreaInset(edge: .bottom)`, visible only when the keyboard is shown. Keyboard visibility is tracked by subscribing to `UIResponder.keyboardWillShowNotification` and `UIResponder.keyboardWillHideNotification`.

The bar has three zones:

| Zone | Content |
|------|---------|
| Left | Dismiss keyboard button (`keyboard.chevron.compact.down`) |
| Centre | Horizontal `ScrollView` of Markdown insert buttons |
| Right | Navigator button (`list.bullet.indent`) |

**Character strip order:**
`#` `##` `###` `**` `_` `~~` `` ` `` ` ``` ` `>` `-` `- [ ]` `[[` `[` `]` `(` `)` `|` `---`

**Insert behaviour:**

- Selected text present: wrap with opening and closing syntax, keep selection
- No selection: insert pair with cursor positioned inside (e.g. `**|**`)
- Fenced code block (` ``` `): inserts `\`\`\`\n\n\`\`\`` with cursor on the first line after the opening fence so user types the language, then moves to body on Return

All insertions go through a single `insertMarkdown(syntax:)` method on the `Coordinator` that reads and writes `UITextView.selectedRange` directly.

**Files:**
- `VaultNotes/Views/Editor/MarkdownBarView.swift` — new floating bar component
- `VaultNotes/Views/Editor/MarkdownSyntax.swift` — enum of all supported syntax tokens with their open/close strings and cursor offset

---

## 3. Note Navigator

### MarkdownParser service

Parses raw note string into a flat array of `ParsedHeading` structs:

```swift
struct ParsedHeading {
    let level: Int          // 1-6
    let text: String        // heading text without # prefix
    let lineStart: Int      // character offset in the full string
    let wikilinks: [String] // [[link]] targets found before next heading
}
```

Parsing is done in a single linear pass: collect heading lines (`^#{1,6} `), collect `[[...]]` matches in the content between consecutive headings. O(n) on note length.

### Navigator sheet

Triggered by the navigator button in the Markdown bar. Presented as a SwiftUI sheet.

Content: `List` with each `ParsedHeading` as a row. Heading indentation is `(level - 1) * 16` pt leading padding. Wikilinks appear as sub-rows beneath their heading with a `link` SF Symbol prefix.

**Tap heading:** dismisses sheet, calls `scrollToRange` on the `UITextView` coordinator to jump to `lineStart`, moves cursor to start of that line.

**Tap wikilink:** dismisses sheet, searches the vault's current folder (and recursively the vault root) for a file named `<linkTarget>.md`. If found, pushes current note onto the navigation history stack and opens the linked note. If not found, shows a brief toast "Note not found".

**Empty state:** "No headings found" when the note contains no heading lines.

**Files:**
- `VaultNotes/Services/MarkdownParser.swift` — new service
- `VaultNotes/Views/Editor/NoteNavigatorView.swift` — new sheet

---

## 4. Navigation History (Back Stack)

### Design

`VaultBrowserView` owns two new state properties:

```swift
@State private var navigationHistory: [(url: URL, scrollOffset: CGFloat)] = []
```

When a wikilink opens a note (from the navigator): current `selectedFile` and its scroll offset are pushed onto the stack, `selectedFile` is set to the linked note's URL.

When the user manually taps a file in `FileListView`: stack is cleared.

`NoteView` receives an optional `onBack: (() -> Void)?` closure. When non-nil, a back button appears in the toolbar showing the previous note's filename (truncated to 20 chars, `chevron.left` prefix). Tapping calls `onBack`, which pops the stack in `VaultBrowserView` and restores `selectedFile` and scroll position.

Scroll position is stored as a `CGFloat` offset and restored by calling `scrollView.setContentOffset` on the `UITextView`'s enclosing scroll view.

`NoteView` also receives `vaultID: UUID?` as a parameter (passed from `VaultBrowserView` via `AppSettings.activeVaultID`) so `SyncService` knows which vault's manifest to consult.

**Files changed:**
- `VaultNotes/Views/Vault/VaultBrowserView.swift`
- `VaultNotes/Views/Editor/NoteView.swift`

---

## 5. Code Block Rendering (Preview)

### Design

`MarkdownPreviewView` registers a custom block style via swift-markdown-ui's `.markdownBlockStyle(\.codeBlock)`. The custom `CodeBlockView` receives the raw code string and language identifier.

Layout:
- Header row: language label (small caps, secondary foreground) left, copy button (`doc.on.doc`) right
- Body: `ScrollView(.horizontal)` containing a `HStack` of a line-number column and the code column, both monospaced

Line numbers: fixed-width, right-aligned, secondary foreground. Count derived from splitting on `\n`.

Copy button: taps write to `UIPasteboard.general.string`. Button icon switches to `checkmark` for 1.5 seconds then reverts (driven by a `@State var copied: Bool`).

Background color uses `theme.editorBackground` darkened slightly (opacity overlay).

**Files:**
- `VaultNotes/Views/Editor/CodeBlockView.swift` — new component
- `VaultNotes/Views/Editor/MarkdownPreviewView.swift` — register block style

---

## 6. SMB Offline Cache and Sync

### Applicability

Cloud provider vaults (iCloud Drive, Dropbox, OneDrive, Google Drive) have offline sync managed by the OS FileProvider — no app work required. This feature applies only to vaults the user marks as "Cache for offline use" when adding them. The flag is stored on `VaultBookmark` as `isSMBCached: Bool`.

### Cache layout

```
Documents/
  VaultCache/
    <vault-id>/
      manifest.json
      <mirrored vault directory tree of .md files>
```

`manifest.json` schema:
```json
{
  "relative/path/note.md": {
    "sha256": "abc123...",
    "cachedAt": "2026-05-02T10:00:00Z",
    "serverModifiedAt": "2026-05-02T09:55:00Z",
    "localModifiedAt": "2026-05-02T10:05:00Z"
  }
}
```

`localModifiedAt` is updated every time the user saves a note. It is `null` if the user has not edited the note since the last cache pull.

### Sync lifecycle

**Initial vault add (SMB cached):**
Background `Task` downloads all `.md` files recursively from the SMB URL, writes to cache directory, builds manifest. `VaultBrowserView` shows a progress banner ("Downloading vault… 42/138 notes") during this phase. Vault is accessible (read from cache) immediately after each file downloads.

**Note open (lazy check):**
When `NoteView` appears, `SyncService.checkNote(at:vaultID:)` runs async: reads server file, computes SHA256, compares to manifest. Three outcomes:
- Hash matches: serve cache, no action
- Server newer only (user hasn't edited locally since last sync): silently update cache
- Both sides changed (local cache modified after `cachedAt`, server hash differs): trigger conflict resolution

**Background poll:**
`Timer.scheduledTimer(withTimeInterval: 300, ...)` fires every 5 minutes while app is active. Iterates manifest, checks recently-accessed notes (opened in last 24h) against server.

**Offline detection:**
`NWPathMonitor` watches network status. When offline, reads from cache silently. When connectivity returns, resumes polling.

### Conflict resolution

A sheet presented over `NoteView` with two options:

1. **"Use newer"** — compares `serverModifiedAt` to `cachedAt`, discards older version automatically, no further user action
2. **"Compare versions"** — opens the diff overlay

**Diff overlay** — full-screen modal sheet:
- Unified diff format (single column)
- Monospace font, line numbers
- Removed lines: red background, `-` prefix
- Added lines: green background, `+` prefix
- Unchanged context lines: neutral, no prefix (3 lines of context above/below each change)
- Visual language matches Claude Code's diff viewer
- Pinned footer: "Keep cached version" | "Keep server version"
- Choosing either dismisses the overlay, writes the chosen version to cache and (if keeping cached) uploads to server

Diff computation: Myers diff algorithm on line arrays. Implement as `DiffEngine.diff(from: [String], to: [String]) -> [DiffLine]`. No third-party dependency.

**Files:**
- `VaultNotes/Models/VaultBookmark.swift` — add `isSMBCached` flag
- `VaultNotes/Services/SyncService.swift` — new: manifest read/write, hash check, download, poll
- `VaultNotes/Services/DiffEngine.swift` — new: Myers diff algorithm
- `VaultNotes/Views/Sync/SyncConflictView.swift` — new: conflict resolution sheet
- `VaultNotes/Views/Sync/DiffOverlayView.swift` — new: unified diff viewer
- `VaultNotes/Views/Vault/VaultBrowserView.swift` — progress banner, NWPathMonitor

---

## 7. App Lock

### Design

`LocalAuthentication` framework. Policy: `.deviceOwnerAuthentication` — covers FaceID, TouchID, and device passcode fallback with no custom PIN required.

**Lock triggers:**
- App launch (cold start)
- App returns from background after the configured timeout

**Timeout options:** Immediately, 1 minute, 5 minutes (default), Never.

**Implementation:**
`AppLockService` (singleton `ObservableObject`) owns `isLocked: Bool` and `backgroundedAt: Date?`. `RootView` overlays a full-screen blur (`ultraThinMaterial`) + lock icon when `isLocked == true`. No app content is visible through the blur. FaceID prompt is presented automatically on top.

`ScenePhase` changes drive the logic:
- `.background`: record `backgroundedAt`
- `.active`: if `backgroundedAt` is set and elapsed time exceeds timeout, set `isLocked = true`, clear `backgroundedAt`, call `LAContext.evaluatePolicy`

Settings: toggle "Require authentication" (stored in `AppSettings`), timeout picker.

**Files:**
- `VaultNotes/Services/AppLockService.swift` — new
- `VaultNotes/Views/RootView.swift` — overlay and scene phase handling
- `VaultNotes/Models/AppSettings.swift` — add `appLockEnabled`, `appLockTimeout`
- `VaultNotes/Views/Settings/SettingsView.swift` — lock settings rows

---

## Out of Scope

- Apple Intelligence / Writing Tools integration (wishlist)
- Full backlink indexing (notes that link *to* current note — requires vault-wide index)
- Real-time collaborative editing
- Syntax highlighting in editor (only in preview)
- Custom PIN (device passcode handles fallback)
