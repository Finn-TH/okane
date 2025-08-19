# Okane – Development Plan (MVP → v1)

Goal
Build a frictionless, private-by-default iOS app that turns receipt photos into structured expenses with minimal manual input. MVP focuses on: capture → OCR → parse → categorize → edit → persist → basic analytics.

Principles
- MVP first, local-first (offline-capable), Apple-first frameworks.
- SwiftUI-first, modern APIs, strict concurrency.
- Feature-first architecture (MVVM + unidirectional data flow).
- Security and privacy by default (least privilege, no network for MVP).
- Small steps, tested and shippable at each phase.

Targets and decisions
- Xcode: 16
- iOS minimum: 17
- Language: Swift 6, Swift Concurrency
- UI: SwiftUI
- Capture: Prefer VisionKit (`VNDocumentCameraViewController`) for receipts; allow `PhotosPicker` import; use `AVFoundation` only when custom camera is necessary.
- OCR: Apple Vision (`VNRecognizeTextRequest`)
- Categorization (MVP): on-device rule-based heuristics; future: Apple Intelligence models when available on device.
- Persistence: Core Data (repository pattern)
- Charts: Apple `Charts`
- No third-party dependencies in MVP

Repository layout
- `OkaneApp/` (App entry, `OkaneApp.swift`)
- `Features/ReceiptCapture/`
- `Features/Expenses/` (list, detail, edit)
- `Features/Dashboard/`
- `Core/Models/`
- `Core/Persistence/` (Core Data stack, repositories, migrations)
- `Core/Services/` (`OCRService`, `ReceiptParser`, `CategorizationService`, `AnalyticsService`, `CurrencyFormatter`, `Permissions`)
- `Core/Security/` (privacy manifest, file protection helpers, snapshot redaction)
- `Core/ML/` (provider protocols; MVP `RuleBasedCategorizer`; future Apple Intelligence provider)
- `DesignSystem/` (colors, typography, reusable UI)
- `Resources/` (Assets, Localizations, sample receipts)
- `Tests/`, `UITests/`
- `.github/` (copilot-instructions, issue templates)

Quality gates (every phase)
- Build passes with no warnings.
- Unit tests for new logic; UI tests for critical flows.
- Accessibility labels for primary UI.
- No PII in logs; structured `Logger` usage with privacy annotations.
- App responsive (no main-thread blocking).
- Receipt/expense views wrapped with SwiftUI `.privacySensitive()`.

Roadmap & phases

Phase 0 — Project scaffolding and guardrails
Objectives
- Create the Xcode project, structure the repo, add privacy/security baselines, and verify toolchain + tests.

To-do
- Create iOS app project (SwiftUI lifecycle), set min iOS 17.
- Add targets: `Okane` (app), `OkaneTests`, `OkaneUITests`.
- Implement `CoreDataStack` with lightweight migrations and file protection (`NSPersistentStoreFileProtectionKey` using `NSFileProtectionCompleteUntilFirstUserAuthentication` or `NSFileProtectionCompleteUnlessOpen`).
- Add `Logger` wrappers (OSLog) with privacy annotations.
- Add `PrivacyInfo.xcprivacy` (declare data types; camera usage; no tracking).
- Add `Info.plist` usage descriptions: `NSCameraUsageDescription`, `NSPhotoLibraryAddUsageDescription` (if saving), `NSPhotoLibraryUsageDescription` (if picking).
- Create proposed folder structure and placeholder files.
- Add sample receipt images under `Tests/Fixtures/Receipts`.
- Add `.github/copilot-instructions.md`.
- Add a simple unit test (sanity) and UI test skeleton.

Acceptance criteria
- Project builds and runs on simulator/device.
- Unit/UI tests run green.
- Privacy manifest present; Info.plist descriptions present.
- Repo structure matches plan.

Phase 1 — Receipt capture (VisionKit-first)
Objectives
- Let users capture a receipt scan or import from library; store image reference locally.

To-do
- Prefer `VNDocumentCameraViewController` (VisionKit) for document/receipt scanning with auto-cropping and enhancement. Provide SwiftUI wrapper.
- Optional: `DataScannerViewController` for live text guidance overlay before capture (devices/OS permitting).
- Provide `PhotosPicker` import path.
- Save images to app container in a hashed folder (`FileManager`), return `URL`.
- Add `ReceiptCaptureViewModel` with async capture pipeline.
- Handle permissions just-in-time with clear copy.
- UI test: open capture → scan/import → save draft.

Acceptance criteria
- Capture or pick a receipt and persist image locally.
- Smooth UX; returns to a “Review” screen with captured image.
- Errors shown non-blocking (toast/sheet), with retry.

Deliverables
- `Features/ReceiptCapture/*`, `Core/Services/ImageStore`.

Phase 2 — OCR and parsing (draft creation)
Objectives
- Extract text with Vision and parse amount, merchant, and date into a draft expense.

To-do
- Implement `OCRService` using `VNRecognizeTextRequest` with `.accurate`, language correction enabled, off-main execution.
- Create `ReceiptParser` to:
  - Extract amounts (currency symbols, separators, negatives/returns).
  - Extract dates (various formats, locale-aware).
  - Guess merchant (header lines, high-confidence tokens; ignore “Total/Tax”).
  - Score candidates by confidence and spatial heuristics (e.g., final “Total” near bottom).
- Introduce `ExpenseDraft` model to carry parsed values + raw text blocks.
- Present editable draft view: show image, parsed fields prefilled; allow corrections.
- Unit tests: OCR mocks (inject text blocks); parsing regex + heuristics with fixtures.

Acceptance criteria
- OCR + parser produce a reasonable draft for common printed receipts.
- User can edit amount/date/merchant before saving.

Deliverables
- `Core/Services/OCRService`, `Core/Services/ReceiptParser`, `Features/Expenses/Draft`.

Phase 3 — Categorization (on-device, pluggable)
Objectives
- Assign a category (groceries, utilities, dining, etc.) locally, offline.

To-do
- Define `CategorizationProvider` protocol and `CategorizationService`.
- Implement `RuleBasedCategorizer`:
  - Merchant lexicon (CSV/JSON mapping).
  - Keyword rules from OCR text lines (e.g., “kWh”, “grocery”, “fuel”, “pizza”).
  - Amount/line-item cues (tax/subtotal language).
- Add `Category` enum + display metadata (icon/color).
- Present category suggestion in draft; allow override.
- Unit tests for categorization rules; fixtures cover edge cases.
- Document future providers:
  - Apple Intelligence on-device LLM via “Foundation Models” APIs when available and supported on device/OS; guard with availability, allow offline use; only escalate via Private Cloud Compute with explicit user consent and UI [developer.apple.com](https://developer.apple.com/machine-learning/whats-new/), [apple.com](https://www.apple.com/apple-intelligence/), [developer.apple.com](https://developer.apple.com/apple-intelligence/), [developer.apple.com](https://developer.apple.com/videos/play/wwdc2025/360/), [apple.com](https://www.apple.com/newsroom/2025/06/apple-intelligence-gets-even-more-powerful-with-new-capabilities-across-apple-devices/).

Acceptance criteria
- MVP categorization suggests correct category for common receipts.
- Manual override persists.

Deliverables
- `Core/Services/CategorizationService`, `Core/ML/RuleBasedCategorizer`, `Core/Models/Category`.

Phase 4 — Persistence, editing, and browsing
Objectives
- Persist finalized expenses, browse and edit existing ones.

To-do
- Define `Expense` Core Data entity:
  - `id: UUID`, `amount: Decimal` (store as `NSDecimalNumber`), `currency: String`, `date: Date`,
    `merchant: String?`, `categoryRaw: String?`, `notes: String?`, `receiptImageRef: URL?`,
    `createdAt`, `updatedAt`.
- Implement repository pattern (`ExpenseRepository`) with async CRUD.
- Implement list view (group by month), detail view, edit view.
- Migrate `ExpenseDraft` → `Expense` save pipeline with validation.
- Unit tests for repository and mapping; UI tests for create → edit → save.

Acceptance criteria
- Users can save, view, edit, and delete expenses.
- Data persists across launches; decimals stored accurately.

Deliverables
- `Core/Persistence/*`, `Features/Expenses/*`.

Phase 5 — Dashboard & analytics
Objectives
- Provide basic insights: weekly/monthly totals, by-category breakdowns.

To-do
- Add aggregation service (`AnalyticsService`) that computes rollups in memory.
- Use Apple `Charts` to render:
  - Monthly spend over time.
  - Category distribution (current month, last 30/90 days).
- Respect locale for currency/date formatting (`NumberFormatter`, `DateFormatter`).

Acceptance criteria
- Dashboard shows correct totals from fixtures and real data.
- Charts perform smoothly with sample datasets.

Deliverables
- `Features/Dashboard/*`, `Core/Services/AnalyticsService`.

Phase 6 — UX polish and accessibility
Objectives
- Refine the end-to-end flow and design system.

To-do
- Design tokens in `DesignSystem` (colors, typography, spacing).
- Haptics for key actions; progress indicators during OCR.
- Empty states, error states, pull-to-refresh (if applicable).
- Accessibility: Dynamic Type, VoiceOver labels, contrast checks.
- Use SwiftUI `.privacySensitive()` on receipt and expense detail views to redact app switcher snapshots.
- Adopt system “Writing Tools” in standard text fields as appropriate (notes, descriptions) to leverage on-device Apple Intelligence UX when available [developer.apple.com](https://developer.apple.com/apple-intelligence/).
- App icons and basic branding.

Acceptance criteria
- Smooth “Snap → Review → Save” with minimal friction.
- Accessibility pass for primary flows.

Deliverables
- `DesignSystem/*`, refined feature UIs.

Phase 7 — Security & privacy hardening
Objectives
- Ensure data protection and compliance baselines.

To-do
- Core Data store with file protection via `NSPersistentStoreFileProtectionKey` (choose `CompleteUntilFirstUserAuthentication` unless business case requires otherwise).
- Keychain only if secrets are introduced (not needed for MVP).
- Strict logging policy: `Logger` with privacy annotations; no raw OCR text or image paths in logs.
- Redact images/content from background snapshots (`.privacySensitive()`); observe `UIScreen.capturedDidChangeNotification` to adjust UI if needed.
- Verify `PrivacyInfo.xcprivacy` and App Store privacy questionnaire alignment.

Acceptance criteria
- Data at rest protected; no sensitive data leaked in logs/UI snapshots.
- App Store privacy disclosures accurate.

Deliverables
- `Core/Security/*`, updated manifests and docs.

Future phases (non-MVP)
- Cloud sync: CloudKit (Apple-first) → define zones; attachments for images.
- Improved OCR: receipt-specific models; evaluate Vision/Document improvements before third-party OCR.
- ML categorization: Create ML text classifier; Apple Intelligence on-device models via Foundation Models APIs when supported on target OS/devices.
- Budget planning: envelopes, alerts, goals.
- Multi-device, iPad, widgets, App Intents integrations (shortcuts).
- Monetization (later): plus features; no ads.

Testing strategy
- Unit: parsing, categorization, repositories, analytics.
- UI: capture → draft → save; edit; dashboard displays.
- Fixtures: `Tests/Fixtures/Receipts` with expected parsed structures.
- Performance: measure OCR latency and scrolling performance.
- Accessibility audit before release.

Definition of done (MVP)
- Capture → OCR → categorize → edit → save works reliably on iPhone (iOS 17+).
- Dashboard shows correct aggregates.
- No crashes; no blocking on main thread; acceptable OCR latency (<2–3s typical).
- Privacy and permissions compliant; offline-first works fully.

Notes on backend/APIs
- MVP: no backend. Design repositories/services to be swappable for CloudKit later.
- If/when adding cloud:
  - Prefer CloudKit with mirror of `Expense` schema and attachments for images.
  - Keep categorization on-device; syncing only structured data and image blobs.
  - Conflict resolution: last-write-wins plus merge UI when needed.

Milestones (suggested)
- M0 (Week 1): Phase 0 done.
- M1 (Week 2): Phase 1 + basic OCR call returning text blocks.
- M2 (Week 3): Phase 2 parsing & draft editing.
- M3 (Week 4): Phase 3 categorization MVP.
- M4 (Week 5): Phase 4 persistence & browsing.
- M5 (Week 6): Phase 5 dashboard + Phase 6 polish + Phase 7 hardening.

References
- Apple Intelligence overview and Writing Tools adoption [developer.apple.com](https://developer.apple.com/apple-intelligence/)
- ML frameworks and programmatic access to system models (WWDC25) [developer.apple.com](https://developer.apple.com/videos/play/wwdc2025/360/)
- Foundation Models framework and what’s new in ML [developer.apple.com](https://developer.apple.com/machine-learning/whats-new/)
- Apple Intelligence capabilities, on-device LLM, Private Cloud Compute [apple.com](https://www.apple.com/newsroom/2025/06/apple-intelligence-gets-even-more-powerful-with-new-capabilities-across-apple-devices/), [apple.com](https://www.apple.com/apple-intelligence/)
