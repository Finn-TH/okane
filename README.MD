Okane 📸💸

Okane is an iOS app that turns receipt photos into structured expenses with minimal manual input, giving users a simple way to track and analyze their spending.
✨ Features (MVP)

    📷 Capture receipts via camera (VisionKit) or import from Photos
    🔎 On‑device OCR with Apple Vision
    🧾 Parse amount, date, and merchant
    🗂 Rule‑based expense categorization (with user override)
    💾 Local‑only storage with Core Data
    📊 Simple dashboard with expense summaries

🛠 Tech Stack

    Language: Swift 6
    UI: SwiftUI
    iOS: 17+
    Frameworks: Vision, VisionKit, PhotosUI, CoreData, Charts

📂 Project Structure

Okane/
├─ App/                 # App entry & lifecycle
├─ Features/            # Feature-first (ReceiptCapture, Expenses, Dashboard)
├─ Core/                # Models, utilities, extensions
├─ Services/            # OCRService, StorageService, etc.
├─ Data/                # Core Data stack & repositories
├─ Resources/           # Assets.xcassets, Info.plist, localizations
├─ Tests/ & UITests/    # Unit & UI test targets
├─ docs/                # Documentation (plan.md, ADRs, etc.)
└─ .github/             # Workflows, issue templates, copilot-instructions.md

🤝 Contributing

    Fork & branch
    Add feature/fix with tests
    Open a Pull Request

👉 Follow the agreed folder structure & coding standards. Keep external dependencies to an absolute minimum.
🔒 Privacy & Security

    ✅ All data stays on-device
    🛡 Sensitive views redacted with .privacySensitive()
    🔔 Permissions requested just‑in‑time
    📑 Privacy manifest (PrivacyInfo.xcprivacy) kept up to date
