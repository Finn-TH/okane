Okane — PRD + Tech Design (MVP → v1)

Owner: Sean
Platform: iOS (Swift)
Last updated: 2025-09-01

1) Purpose & Problem

People want a frictionless, private way to turn receipt photos → structured expenses without tedious typing or cloud dependence.

2) Goals (MVP)
 • End-to-end flow: capture → OCR → parse → categorize → edit → save → basic analytics.
 • Local-first, privacy-first by default; avoid network in MVP.
 • Ship in small, shippable increments with tests and guardrails.

3) In-Scope / Out-of-Scope (MVP guardrails)

In-scope (MVP):
 • Receipt capture (VisionKit) + library import.
 • On-device OCR (Vision), rule-based parsing & categorization.
 • Local persistence (Core Data), simple dashboard (Charts).
 • Accessibility and privacy baselines.

Out-of-scope (MVP):
 • Cloud sync, accounts, monetization.
 • ML categorization (LLM/Apple Intelligence) beyond rules.
 • Third-party OCR/services by default.

Note on privacy/local-first: Local is the baseline. If a blocking gap emerges, a narrowly scoped, opt-in server-side OCR may be prototyped behind a feature flag and explicit user consent. Any remote path must: (a) send only what’s needed, (b) redact PII where possible, (c) document data retention & security. Default build remains offline.

4) Success Metrics (MVP)
 • OCR accuracy ≥ 80% on a curated sample set of printed receipts.
 • Scan → editable draft latency < 3s (typical device, typical receipt).

These two are the only MVP “go/no-go” metrics for now; expand post-MVP.

5) Primary User Stories
 • Scan & draft: As a user, I can scan or import a receipt and immediately see an editable draft with amount/date/merchant prefilled.
 • Correct & save: I can quickly correct fields and save to my local ledger.
 • See basics: I can view monthly totals and a simple by-category breakdown.

6) Principles & Constraints
 • MVP-first, smallest surface area that delivers value.
 • Apple-first frameworks, modern APIs, strict concurrency.
 • MVVM + unidirectional data flow, feature-first structure.
 • No main-thread blocking; .privacySensitive() on sensitive views.
 • No new files created by AI tools unless explicitly requested (project file integrity).

7) Technical Decisions (MVP)
 • Xcode: 16 iOS: 17+ Language: Swift 6 + Swift Concurrency
 • UI: SwiftUI
 • Capture: VisionKit VNDocumentCameraViewController; PhotosPicker fallback
 • OCR: Vision VNRecognizeTextRequest(.accurate)
 • Categorization: rule-based heuristics (keywords, merchant lexicon)
 • Persistence: Core Data (repository pattern)
 • Charts: Apple Charts
 • Dependencies: none (MVP)

8) High-Level Architecture

Feature-first with shared Core services.

/App                      # App entry
/Features
  /ReceiptCapture         # Capture & import, review image
  /Expenses               # Draft, list, detail, edit
  /Dashboard              # Charts & summaries
/Core
  /Models                 # Expense, ExpenseDraft, Category
  /Persistence            # CoreDataStack, ExpenseRepository
  /Services               # OCRService, ReceiptParser, CategorizationService,
                          # AnalyticsService, CurrencyFormatter, Permissions
  /Security               # Privacy manifest, file protection, snapshot redaction
/Configs                  # Info.plist, PrivacyInfo.xcprivacy
/Resources                # Assets, Localizations, sample receipts
/Tests
  /Fixtures               # Fixtures & test suites

Key components
 • OCRService (Vision) → text blocks (off-main).
 • ReceiptParser → amount/date/merchant from text + simple spatial heuristics.
 • CategorizationService → rule-based suggestion (overrideable).
 • ExpenseRepository (Core Data) → async CRUD, precise Decimal handling.
 • AnalyticsService → in-memory rollups for charts.

Data model (MVP)
 • ExpenseDraft: { textBlocks, imageURL, amount?, date?, merchant?, category? }
 • Expense (Core Data):
id: UUID, amount: Decimal (NSDecimalNumber), currency: String,
date: Date, merchant: String?, categoryRaw: String?, notes: String?,
receiptImageRef: URL?, createdAt, updatedAt

9) Quality Gates (apply to every phase)
 • Builds clean (no warnings). No main-thread violations.
 • Unit tests for new logic; UI tests for critical flows.
 • Accessibility labels for primary UI.
 • Structured logging via OSLog with privacy annotations; no PII in logs.
 • Receipt/expense views use .privacySensitive().

10) Roadmap & Phases

Phase 0 — Scaffolding & Guardrails

Objectives: Project template, repo structure, privacy & logging baselines, test skeletons.
To-do:
 • App target + test targets; min iOS 17.
 • CoreDataStack with file protection (NSPersistentStoreFileProtectionKey, choose CompleteUntilFirstUserAuthentication or CompleteUnlessOpen).
 • Logger wrappers (OSLog) with privacy annotations.
 • PrivacyInfo.xcprivacy; Info.plist usage strings (NSCameraUsageDescription, NSPhotoLibraryUsageDescription, etc.).
 • Sample receipt fixtures; placeholder folders; .github/copilot-instructions.md.
Acceptance: Builds & runs; unit/UI tests green; manifests present; structure matches plan.
Deliverables: App skeleton; Core/Persistence/CoreDataStack; logging & privacy files.

Phase 1 — Receipt Capture

Objectives: Scan or import; persist image URL; return to review.
To-do: SwiftUI wrapper for VisionKit; PhotosPicker path; ImageStore (hashed path); ReceiptCaptureViewModel.
Acceptance: Smooth scan/import → local image saved → review screen; non-blocking errors.
Deliverables: Features/ReceiptCapture/*, Core/Services/ImageStore.

Phase 2 — OCR & Parsing (Draft)

Objectives: Recognize text; parse amount/date/merchant; editable draft.
To-do: OCRService (Vision, .accurate, off-main); ReceiptParser (amount, date, merchant; simple spatial heuristics); ExpenseDraft; draft editor.
Tests: OCR mocks (text blocks), parser fixtures.
Acceptance: Reasonable drafts on common printed receipts; editable before save.
Deliverables: Core/Services/OCRService, Core/Services/ReceiptParser, Features/Expenses/Draft.

Phase 3 — Categorization (Rule-based)

Objectives: Suggest category offline; allow override.
To-do: CategorizationProvider + CategorizationService; RuleBasedCategorizer (merchant lexicon + keywords).
Tests: Rule fixtures & edge cases.
Acceptance: Sensible suggestions for common receipts; overrides persist.
Deliverables: Core/Services/CategorizationService, Core/ML/RuleBasedCategorizer, Core/Models/Category.

Phase 4 — Persistence, Editing, Browsing

Objectives: Save finalized expenses; list/detail/edit.
To-do: Expense entity; ExpenseRepository async CRUD; mapping from Draft; list/detail/edit UIs.
Tests: Repository + mapping; UI flow tests.
Acceptance: Create → edit → save; data persists; Decimal safe.
Deliverables: Core/Persistence/*, Features/Expenses/*.

Phase 5 — Dashboard & Analytics

Objectives: Basic insights (totals, by-category).
To-do: AnalyticsService rollups; Apple Charts (monthly spend; category distribution).
Acceptance: Correct aggregates from fixtures and real data; smooth charts.
Deliverables: Features/Dashboard/*, Core/Services/AnalyticsService.

Phase 6 — UX Polish & Accessibility

Objectives: Design tokens; haptics; loading states; accessibility; privacy UX touches.
Deliverables: DesignSystem/*, refined UIs.
Acceptance: “Snap → Review → Save” feels smooth; primary flows accessible.

Phase 7 — Security & Privacy Hardening

Objectives: Verify data-at-rest protection; logging discipline; snapshot redaction; privacy disclosures alignment.
Deliverables: Core/Security/*, updated manifests.

11) Testing Strategy
 • Unit: parsers, categorization, repositories, analytics (fixtures in Tests/Fixtures/Receipts).
 • UI: capture → draft → save; edit; dashboard displays.
 • Performance: OCR latency; scrolling performance.
 • Accessibility: labels, Dynamic Type, contrast.
 • Definition of Done (MVP):
 • End-to-end flow is reliable on iOS 17+.
 • Dashboard aggregates correct.
 • No crashes; OCR latency <3s typical.
 • Privacy/permissions compliant; fully offline by default.

12) Performance Targets (MVP)
 • Latency: capture→draft < 3s (typical receipt, recent iPhone).
 • Memory: no sustained spikes > app memory budget during OCR.
 • Scrolling: 60fps target on lists/charts for typical data sizes.

13) Milestones (suggested solo cadence)
 • M0 (Week 1): Phase 0 done.
 • M1 (Week 2): Phase 1 + OCR stub returns text blocks.
 • M2 (Week 3): Phase 2 parsing + draft editor.
 • M3 (Week 4): Phase 3 categorization MVP.
 • M4 (Week 5): Phase 4 persistence & browsing.
 • M5 (Week 6): Phase 5 dashboard + Phase 6 polish + Phase 7 hardening.

14) AI Collaboration Notes
 • This PRD is the human + LLM context.
 • Operational guardrails for agents (modes, verbosity, tool budgets, safe/unsafe ops) live in docs/AGENTS.md.
 • Default: Speed Mode (minimal reasoning, concise status, verbose/clear code).
 • Never auto-create files or edit .pbxproj without explicit instruction.

15) Appendix — Future Phases (post-MVP)
 • Cloud sync: CloudKit zones; attachments for images; conflict resolution.
 • Improved OCR: evaluate Vision/Document updates before 3rd-party.
 • ML categorization: Create ML or on-device Apple Intelligence when available; offline-first with explicit consent if PCC used.
 • Budgeting: envelopes, alerts, goals.
 • Multi-device: iPad, widgets, App Intents.
 • Monetization: later; no ads.

16) References
 • Apple Intelligence & Writing Tools: developer.apple.com
 • Foundation Models & WWDC25 ML sessions: developer.apple.com
 • Apple newsroom updates on Apple Intelligence: apple.com
