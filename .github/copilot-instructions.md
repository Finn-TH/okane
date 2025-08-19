# Okane — Copilot Repository Instructions

Project overview
- iOS app to turn receipt photos into structured expenses with minimal manual input. MVP flow: capture → OCR (Apple Vision) → parse → categorize (on-device rules) → edit → save (Core Data) → dashboard.

Targets and stack
- iOS 17+, Swift 6, SwiftUI-first, Xcode 16.
- Frameworks: `Vision`, `VisionKit`, `AVFoundation`, `PhotosUI`, `CoreData`, `Charts`, `OSLog`.
- Dependencies: prefer SPM; no third-party OCR/LLMs in MVP.

Repository & folder structure (Xcode 16 buildable folders)
- On-disk folders must mirror Xcode (no virtual groups). Place new files under real paths.
- Top-level:
  - `App/` — app entry, routing, `OkaneApp.swift`
  - `Features/` — `ReceiptCapture/`, `Expenses/`, `Dashboard/`, `Settings/` (each: `Views/`, `ViewModels/`, `Services/`, `Resources/`)
  - `Core/` — shared models, design system, utilities, extensions
  - `Services/` — cross-cutting services (`OCRService`, `LLMService`, `StorageService`)
  - `Data/` — persistence (`CoreData/` model, migrations, repositories)
  - `Resources/` — global assets (`Assets.xcassets`), `Info.plist`, localizations
  - `Tests/`, `UITests/` — mirror source layout; shared fixtures under `Tests/Fixtures/`
  - `Packages/` — optional local Swift packages for large features
  - `ThirdParty/` — vendored frameworks only if SPM not possible
  - `Configs/` — `.xcconfig`, `Secrets.example.plist` (real secrets ignored)
  - `docs/` — `plan.md`, `architecture.md`, ADRs
  - `.github/` — workflows, templates, `copilot-instructions.md`

Architecture & coding standards
- MVVM + unidirectional data flow. Dependency injection via initializers; protocols for services.
- Concurrency: use `async/await`; mark UI-bound view models `@MainActor`. Keep heavy work off the main thread.
- Naming: types `PascalCase`; members `lowerCamelCase`. Views end with `View`, view models with `ViewModel`, services with `Service`. Extensions named `Type+Context.swift`.
- Currency: use `Decimal`/`NSDecimalNumber`; never use `Double` for money.

Feature rules
- Receipt capture: prefer `VisionKit` (`VNDocumentCameraViewController` for scans; `DataScannerViewController` for live guidance where available). Allow `PhotosPicker` import. Fallback to `AVFoundation` only if necessary. Save images in app container and persist `URL` references (not blobs in Core Data).
- OCR: use `VNRecognizeTextRequest` with `.accurate` for still images (use `.fast` only for live previews), language correction on, run off-main. Expose a simple `OCRService` abstraction.
- Parsing: keep parsing deterministic and testable (`ReceiptParser`). Show an editable draft before save; never auto-commit OCR output.
- Categorization: define `CategorizationService` protocol; default `RuleBasedCategorizer`. On-device only in MVP; allow user override and persist it. Keep network model providers behind a feature flag for later.

Data & persistence
- Local-only in MVP. Core Data with lightweight migrations. Repositories expose async CRUD and avoid main-thread blocking.

Security, privacy, and logging
- Local-first; no transmitting user data. Store secrets in Keychain; apply file protection to Core Data store (`NSPersistentStoreFileProtectionKey`).
- Include privacy manifest (`PrivacyInfo.xcprivacy`) and accurate `Info.plist` purpose strings (camera/photos).
- Mark sensitive views `.privacySensitive()`. Do not log PII, raw OCR text, or image paths. Use `Logger` with privacy annotations.

Testing & quality
- Unit tests for `ReceiptParser`, `CategorizationService`, repositories, and analytics aggregation. UI tests for capture → draft → save → edit.
- Tests mirror folder structure; fixtures in `Tests/Fixtures/Receipts`.
- Compile often; prefer small, focused PRs.

When generating code (Copilot)
- Place files in the correct on-disk folder. Add matching tests under `Tests/...`.
- Use Swift concurrency patterns; avoid new dependencies. If adding any, leave a TODO and request approval.
- Prefer Apple system frameworks and on-device processing. Keep code small, composable, and testable.

Non-goals (MVP)
- No sign-in, cloud sync, external OCR/LLMs, ads, or analytics SDKs. No off-device data storage.

See docs
- For details (entity fields, parsing heuristics, UI patterns, migration strategy): see `docs/plan.md`
