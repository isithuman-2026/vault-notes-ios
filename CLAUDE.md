# CLAUDE.md

## Writing Style

- Use plain, technical language. Clear enough to understand without being simplified.
- No em dashes. Use a comma, colon, semicolon, or rewrite the sentence.
- No emoji unless explicitly asked.
- No filler phrases. Keep descriptions succinct.

## Privacy and Security Directive

Never include in git commits:
- Filesystem paths containing usernames or home directories
- Usernames, email addresses, real names, or any personal identifiers
- SSH keys, API tokens, credentials, or secrets of any kind
- Hostnames, internal IPs, or network details
- Any other PII

Make changes path-agnostic rather than hardcoding local paths. When in doubt, leave it out.

## What This Is

vault-notes is a minimal iOS Markdown notes app that opens vaults stored on SMB shares (accessed via VPN or the iOS Files app) or cloud storage providers (iCloud Drive, Dropbox, OneDrive, Google Drive). Built with Swift 6 and SwiftUI targeting iOS 17+. Follows Apple Human Interface Guidelines throughout.

## Commands

```bash
# Open in Xcode
open VaultNotes.xcodeproj

# Build (command line)
xcodebuild -scheme VaultNotes -destination 'platform=iOS Simulator,name=iPhone 16' build

# Test (unit)
xcodebuild -scheme VaultNotes -destination 'platform=iOS Simulator,name=iPhone 16' test

# Test (single file)
xcodebuild -scheme VaultNotes -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:VaultNotesTests/FileSystemServiceTests test
```

## Architecture Summary

- `VaultNotes/Models/` — SwiftData models: VaultBookmark, AppSettings
- `VaultNotes/Services/FileSystemService.swift` — file read/write/list via security-scoped URLs
- `VaultNotes/Services/VaultManager.swift` — resolve bookmarks, open/close vault access
- `VaultNotes/Services/TemplateService.swift` — detect templates folder, list and apply templates
- `VaultNotes/Services/ThemeService.swift` — built-in color schemes, active theme management
- `VaultNotes/Views/Vault/` — vault picker, vault switcher, folder tree, file list
- `VaultNotes/Views/Editor/` — Markdown editor and preview toggle
- `VaultNotes/Views/Settings/` — color scheme picker, app settings
- `VaultNotes/App.swift` — entry point, SwiftData container setup
- `VaultNotesTests/` — XCTest unit tests mirroring service layer

## Tech Stack

- Swift 6, SwiftUI, iOS 17+
- SwiftData (vault bookmark persistence, settings)
- swift-markdown-ui (Markdown rendering in preview mode)
- UIDocumentPickerViewController (vault directory selection, covers SMB + all cloud providers)
- Security-scoped bookmarks (persistent cross-session vault access)

## Documentation Maintenance

Keep these files current whenever code changes:

| Change | Update required |
|--------|----------------|
| Function added, renamed, moved, or removed | `CODE_INDEX.md` — update or add the entry with correct `file:line` |
| New source file added | `CODE_INDEX.md` — add a section; `CLAUDE.md` Architecture Summary — add the file |
| Architecture or pipeline changes | `CLAUDE.md` Architecture Summary |
| New dependency | `CLAUDE.md` Tech Stack section |

Do not wait until the end of a task — update documentation as part of the same change.
