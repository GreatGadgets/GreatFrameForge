# Tiered Application Architecture & Licensing Design

## 1. Overview
This document outlines the architecture for transforming GreatFrameForge (GFF) into a tiered application with license management via **LemonSqueezy**. The system is designed to be modular, allowing the licensing and registration logic to be extracted as a shared module for other "GreatGadget" applications.

## 2. Tier Definitions

| Feature | Free Tier | Base Tier ($20 AUD) | Tier 3 (DLC) |
| :--- | :--- | :--- | :--- |
| **Transform Controls (Pan, Zoom, Rotate, Mirror)**| ✅ Full Access | ✅ Full Access | ✅ Full Access |
| **Design Tools (Sliders & Color Pickers)** | ✅ Full Access | ✅ Full Access | ✅ Full Access |
| **Base Image** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Presets** | 1 Named Preset | Unlimited | Unlimited |
| **Default Preset** | Read-only | Overwritable | Overwritable |
| **Inset Layers** | Max 1 | Unlimited | Unlimited |
| **Gradient Layers** | Max 1 | Unlimited | Unlimited |
| **Text Layers** | Max 1 | Unlimited | Unlimited |
| **Safe Zones** | ❌ None | ✅ YT Zones | ✅ YT + Social Media Pack |
| **Import/Export** | ❌ None | ✅ Yes | ✅ Yes |
| **Social Presets** | ❌ None | ❌ None | ✅ IG, X, YT, TikTok, FB |
| **Text Gradients** | ❌ None | ✅ Yes | ✅ Yes |
| **Text Font Selectors** | ❌ None | ✅ Yes | ✅ Yes | 

## 3. Core Architectural Components

### 3.1 `LicenseManager` (Shared Module)
- **Location**: `assets/js/core/LicenseManager.js`
- **Responsibility**: 
    - Secure storage and retrieval of license keys.
    - Communication with LemonSqueezy API for activation and validation.
    - Handling "Heartbeat" checks (standardized at once every 7 days).
    - Managing device activation counts.
- **Persistence Strategy**:
    - Primary: `localStorage` for session-to-session persistence.
    - Secondary: `IndexedDB` as a redundant backup to survive some cache clearing scenarios.
    - *Note*: Browser-level "Clear Site Data" will still wipe these. We will implement a "Export License Backup" feature to allow users to save a small `.gfk` (GreatKey) file locally.

### 3.2 `FeatureGate` (Utility)
- **Location**: `assets/js/utils/FeatureGate.js`
- **Responsibility**:
    - Centralized logic to check if a specific action is permitted.
    - Provides simple API: `FeatureGate.canAddTextLayer()`, `FeatureGate.hasImportExport()`, etc.
    - Decouples `gtg.js` logic from the specific tier implementation details.

### 3.3 `RegistrationUI` (Component)
- **Location**: `assets/js/components/RegistrationUI.js`
- **Responsibility**:
    - Rendering the "Cog" icon in the header.
    - Managing the hover/modal window for key entry.
    - Displaying affiliate links (e.g., 1Password).
    - Warning users when changing an unsaved key.

## 4. Implementation Strategy

### 4.1 Modular Integration
Instead of bloating `gtg.js`, the licensing system will be initialized in `gtg.js` via:
```javascript
import { LicenseManager } from './core/LicenseManager.js';
import { FeatureGate } from './utils/FeatureGate.js';

// On startup
await LicenseManager.init();
FeatureGate.setTier(LicenseManager.getTier());
```

### 4.2 LemonSqueezy Validation Flow
1. **Activation**: When a user enters a key, we call `/v1/licenses/activate`.
2. **Validation (Startup)**:
    - Check if a cached validation exists (less than 7 days old).
    - If expired/missing, call `/v1/licenses/validate`.
3. **Grace Period**: If the server is unreachable, allow a 3nd-day grace period before falling back to the Free tier.

### 4.3 UI Gating Logic
- **Buttons**: If a feature is locked, buttons will be rendered with a `lock` icon and an `opacity-50` state. Clicking them will trigger a "Tier Upgrade" notification.
- **Limits**: The `addLayer` functions will check `FeatureGate` and prevent execution if limits are reached.
- **Presets**: The "Save" logic will be modified to check for the "named preset" limit in the Free tier.

## 5. Shared Module Design (GreatGadget Standard)
The `LicenseManager` and `RegistrationUI` will be designed as standalone classes that only require a `config` object:
```javascript
const GFF_LICENSE_CONFIG = {
    productId: 'GFF_BASE',
    dlcIds: ['GFF_SOCIAL_PACK'],
    affiliateLinks: {
        onePassword: 'https://1password.com/...'
    }
};
```
This ensures that the next application (e.g., GreatAudioForge) can simply import these modules and provide its own configuration.

## 6. Future Tier / DLC Support
The architecture uses a **Bitmask** or **Array-based** permission system. Each license key returned by LemonSqueezy includes "Variant IDs" or "License Name". We map these to internal capability sets. Adding a new DLC simply involves adding a new ID to the configuration and a corresponding entry in `FeatureGate`.

## 7. Registration Window Design (UX)
- **Trigger**: A subtle silver cog in the top-right header.
- **Contents**:
    - Current Tier Status (Badge).
    - Key Input Field (Masked by default).
    - "Save & Validate" Button.
    - Last Validated timestamp.
    - "Manage Devices" link (pointing to LemonSqueezy Customer Portal).
    - "Protect your keys" Affiliate Section (1Password).
- **Security**: Warn users: *"Changing this key will reset your tier until validated. Continue?"* before allowing edits to an existing valid key.
