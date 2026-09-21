# Android 14 Navigation & Low-Storage Optimization Blueprint

This document outlines the architectural design implications, user navigation flows, Android 14 platform optimizations, and low-storage strategy for **Code On The Go** (on unrooted Android 14 devices).

---

## 1. Application Vision & High-Level Architecture

Code On The Go (CoGo) is a full-featured, on-device mobile IDE capable of building and running Android applications offline.

### Core Architectural Layers
- **UI Layer:** Jetpack Compose (for new screens like Plugin Manager / External File Installer) + Material Android Views & Fragments (for Editor / Main Dashboard).
- **Architecture Pattern:** Unidirectional Data Flow (UDF) with Koin Dependency Injection, `ViewModel`, `StateFlow`, and `SharedFlow` for single-shot events.
- **Build & Execution Engine:** Out-of-process Gradle build via `tooling-api` and embedded Termux terminal toolchain.
- **Persistence & Storage:** Room database for structured recent projects data, DataStore/SharedPreferences for user settings, and raw file system for workspace directories (`/sdcard/AndroidIDEProjects/`).

---

## 2. Android 14 Navigation & UX Optimization Flow

### Key Navigation Pathway
```
                      ┌───────────────────────┐
                      │    SplashActivity     │
                      └───────────┬───────────┘
                                  │
                   ┌──────────────┴──────────────┐
                   ▼                             ▼
         ┌───────────────────┐         ┌───────────────────┐
         │ OnboardingActivity│         │   MainActivity    │ (Project Dashboard)
         └─────────┬─────────┘         └─────────┬─────────┘
                   │                             │
                   └──────────────┬──────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
┌──────────────────┐    ┌──────────────────┐    ┌────────────────────┐
│ EditorActivityKt │    │ TerminalActivity │    │PluginManagerAct... │
│ (Sora Editor/LSP)│    │(Termux Shell UI) │    │  (Compose Screen)  │
└──────────────────┘    └──────────────────┘    └────────────────────┘
```

### Android 14 Navigation Improvements
1. **Predictive Back Gestures:**
   - Manifest configured with `android:enableOnBackInvokedCallback="true"`.
   - Activities and Compose screens handle back invocation gracefully without state corruption or losing un-saved editor buffers.
2. **Edge-to-Edge & System Insets:**
   - Protecting system bars (top status bar & bottom gesture navigation bar) using `WindowInsetsExtensions.kt`.
   - Responsive layouts across orientation shifts, font scale changes (up to 2.0x for accessibility), and split-screen multitasking.
3. **Intent Handling & SAF (Storage Access Framework):**
   - Direct handling of `.cgp` and `.cgt` bundles through `ExternalFileInstallActivity`.
   - File access configured for Android 14 Scoped Storage via `IDEDocumentsProvider` and `IDEFileProvider`.

---

## 3. Low-Storage Optimization Strategy

For unrooted devices operating in storage-constrained environments, the following optimizations are specified:

### A. Asset & Bootstrap Distribution Management
- **Lazy Asset Unpacking:** Unpack Termux runtime, JDK, and Gradle distributions on-demand during first launch or feature initialization rather than extracting everything upfront.
- **Compressed Asset Bundling:** Maintain compressed `assets/release/v8/` packaging and leverage native lib compression configurations (`extractNativeLibs="true"` with AGP deflate optimization).

### B. Cache Pruning & Garbage Collection
- **Build & LSP Cache Pruning:** Automatic cleanup policy for Gradle project caches (`.gradle/caches`), build build directories (`build/`), and language server index caches (`.lsp-index`).
- **Temporary Installation File Reclamation:** Strict lifecycle management in `InstallTempFiles.kt` ensuring extracted project templates and plugins clear temp files post-installation.

### C. Storage Threshold Adaptation
- **Adaptive Storage Floor:** Modify rigid storage constraints in `StorageUtils.kt` (currently requiring 4GB–6GB free space) to allow lightweight editor usage and project editing on devices with limited remaining storage (e.g., 1.5GB–2GB floor with warning indicators).

---

## 4. Execution & Implementation Roadmap

1. **Phase 1: Navigation & Android 14 Insets Refinement**
   - Verify predictive back handling across `EditorActivityKt` and `MainActivity`.
   - Ensure back-stack integrity during file import and plugin installations.
2. **Phase 2: Storage Optimization Implementation**
   - Update `StorageUtils.kt` minimum thresholds.
   - Implement temporary file eviction routines for LSP index logs and build artifacts.
3. **Phase 3: Performance Verification**
   - Validate 2.0x font scaling across all modified screens.
   - Test on ARM64 Android 14 target device.
