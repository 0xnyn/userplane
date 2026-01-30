# Recorder App - Vercel Composition Patterns Audit

**App:** apps/recorder
**Framework:** Astro 5.15.9 + React 19.2.0
**Date:** January 30, 2026
**Auditor:** Claude Code

---

## Executive Summary

The Recorder app is an Astro-based application with embedded React components for video recording functionality. It has a clean architecture with a single context provider and well-organized custom hooks. The main improvement opportunity is migrating from `useContext` to React 19's `use()` hook.

| Category | Score | Issues Found |
|----------|-------|--------------|
| Boolean Props | 7/10 | 15+ instances across 8 files |
| forwardRef Usage | 10/10 | None found (excellent) |
| useContext Usage | 5/10 | Uses legacy pattern, needs React 19 migration |
| Prop Complexity | 7/10 | 5 components with 5+ props |
| Callback Props | 7/10 | 20+ callbacks, well-structured |
| State Lifting | 8/10 | Good centralized state in AppModeContext |

---

## 1. React 19 Migration Required

### 1.1 useContext to use() Migration

**Priority: MEDIUM** | **Impact: MEDIUM**

**File:** `apps/recorder/src/contexts/AppModeContext.tsx`

**Line 11 - Import:**
```typescript
// CURRENT
import {
  createContext,
  type ReactNode,
  useContext,  // ⚠️ Legacy pattern
  useEffect,
  useMemo,
  useRef,
  useState,
} from 'react';
```

**Line 277 - Usage:**
```typescript
// CURRENT (React 18 pattern)
export function useAppMode(): AppModeContextValue {
  const context = useContext(AppModeContext);
  if (!context) {
    throw new Error('useAppMode must be used within an AppModeProvider');
  }
  return context;
}
```

**RECOMMENDED (React 19 pattern):**
```typescript
import { use } from 'react';

export function useAppMode(): AppModeContextValue {
  const context = use(AppModeContext);
  if (!context) {
    throw new Error('useAppMode must be used within an AppModeProvider');
  }
  return context;
}
```

**Benefits of `use()`:**
- Can be called conditionally
- Works seamlessly with Suspense
- Better TypeScript inference

### 1.2 forwardRef Usage

**Status: ✅ EXCELLENT - No instances found**

The recorder app does not use `forwardRef` anywhere, which is the correct pattern for React 19.

---

## 2. Boolean Props Analysis

**Total Instances: 15+ across 8 files**

### 2.1 Complete Boolean Props Inventory

| File | Line | Prop | Severity |
|------|------|------|----------|
| `VideoPlayer.tsx` | 14 | `autoPlay?: boolean` | 🟢 Low |
| `VideoPlayer.tsx` | 16 | `muted?: boolean` | 🟢 Low |
| `VideoPlayer.tsx` | 18 | `showControls?: boolean` | 🟡 Medium |
| `VideoPlayer.tsx` | 41 | `isPlaying` (state) | 🟢 Internal |
| `VideoPlayer.tsx` | 42 | `hasError` (state) | 🟢 Internal |
| `SettingsDropdown.tsx` | 18 | `isBlurEnabled: boolean` | 🟡 Medium |
| `SettingsDropdown.tsx` | 22 | `disabled?: boolean` | 🟢 Low |
| `PreRecordingScreen.tsx` | 19 | `isLoading?: boolean` | 🟢 Low |
| `RecordingInProgressScreen.tsx` | 23 | `isProcessing?: boolean` | 🟡 Medium |
| `PostRecordingReviewScreen.tsx` | 25 | `isSubmitting?: boolean` | 🟢 Low |
| `AppModeContext.tsx` | 43 | `isEmbedded: boolean` | 🟢 Derived |
| `AppModeContext.tsx` | 44 | `isStandalone: boolean` | 🟢 Derived |
| `CaptureApp.tsx` | 38 | `isConnected` (state) | 🟢 Internal |
| `CaptureApp.tsx` | 39 | `isCrossTabConnected` (state) | 🟢 Internal |

### 2.2 Recommendations

**A. VideoPlayer - Create State Enum**

```typescript
// CURRENT
interface VideoPlayerProps {
  showControls?: boolean;
  // Internal state
  const [isPlaying, setIsPlaying] = useState(false);
  const [hasError, setHasError] = useState(false);
}

// RECOMMENDED
type VideoPlayerState = 'idle' | 'loading' | 'playing' | 'paused' | 'error';

interface VideoPlayerProps {
  controlsVisibility?: 'visible' | 'hidden' | 'auto';
}

// Internal state consolidated
const [playerState, setPlayerState] = useState<VideoPlayerState>('idle');
```

**B. RecordingInProgressScreen - Explicit States**

```typescript
// CURRENT
<RecordingInProgressScreen isProcessing={true} />

// RECOMMENDED - More explicit
<RecordingInProgressScreen status="processing" />
<RecordingInProgressScreen status="recording" />
```

---

## 3. Components with 5+ Props

**Total: 5 components**

### 3.1 ErrorScreen (7 props) 🟡

**File:** `apps/recorder/src/components/recording/ErrorScreen.tsx`
**Lines:** 14-28

```typescript
interface ErrorScreenProps {
  title: string;
  subtitle: string;
  preferences?: RecordingPreferences;
  primaryActionText?: string;
  onPrimaryAction?: () => void;
  secondaryActionText?: string;
  onSecondaryAction?: () => void;
}
```

**RECOMMENDED - Group action props:**

```typescript
interface ErrorScreenProps {
  title: string;
  subtitle: string;
  preferences?: RecordingPreferences;
  primaryAction?: {
    text: string;
    onAction: () => void;
  };
  secondaryAction?: {
    text: string;
    onAction: () => void;
  };
}
```

### 3.2 PostRecordingReviewScreen (7 props) 🟡

**File:** `apps/recorder/src/components/recording/PostRecordingReviewScreen.tsx`
**Lines:** 19-33

```typescript
interface PostRecordingReviewScreenProps {
  videoUrl: string;
  preferences: RecordingPreferences;
  isSubmitting?: boolean;
  errorMessage?: string | null;
  onSubmit?: () => void;
  onStartOver?: () => void;
  settingsElement?: ReactNode;
}
```

**RECOMMENDED - Group related props:**

```typescript
interface PostRecordingReviewScreenProps {
  video: {
    url: string;
    preferences: RecordingPreferences;
  };
  status: 'idle' | 'submitting' | 'error';
  errorMessage?: string | null;
  actions: {
    onSubmit?: () => void;
    onStartOver?: () => void;
  };
  settingsElement?: ReactNode;
}
```

### 3.3 VerificationErrorScreen (8 props) 🟡

**File:** `apps/recorder/src/components/verification/VerificationErrorScreen.tsx`
**Lines:** 13-31

```typescript
interface VerificationErrorScreenProps {
  title: string;
  subtitle: string;
  workspaceLogoUrl?: string | null;
  workspaceName?: string | null;
  primaryColor?: string;
  primaryActionText?: string;
  onPrimaryAction?: () => void;
  secondaryActionText?: string;
  onSecondaryAction?: () => void;
}
```

**RECOMMENDED - Use compound pattern:**

```typescript
interface VerificationErrorScreenProps {
  title: string;
  subtitle: string;
  workspace?: {
    logoUrl: string | null;
    name: string | null;
    primaryColor?: string;
  };
  primaryAction?: ActionConfig;
  secondaryAction?: ActionConfig;
}

interface ActionConfig {
  text: string;
  onAction: () => void;
}
```

### 3.4 RecordingInProgressScreen (7 props) 🟡

**File:** `apps/recorder/src/components/recording/RecordingInProgressScreen.tsx`
**Lines:** 18-26

```typescript
interface RecordingInProgressScreenProps {
  title?: string;
  subtitle?: string;
  mediaStream?: MediaStream | null;
  preferences: RecordingPreferences;
  isProcessing?: boolean;
  onStopRecording?: () => void;
  settingsElement?: ReactNode;
}
```

### 3.5 RecordingScreenLayout (5 props) 🟢

**File:** `apps/recorder/src/components/recording/RecordingScreenLayout.tsx`
**Lines:** 10-16

```typescript
interface RecordingScreenLayoutProps {
  children: ReactNode;
  workspaceLogoUrl?: string | null;
  workspaceName?: string | null;
  footnoteText?: string | null;
  topRightElement?: ReactNode;
}
```

This is acceptable - layout props are naturally grouped.

---

## 4. Callback Props Analysis

**Total: 20+ callbacks across components**

### 4.1 Complete Callback Inventory

| Component | File | Callbacks | Count |
|-----------|------|-----------|-------|
| VideoPlayer | `recording/VideoPlayer.tsx` | onError, onLoadStart, onCanPlay | 3 |
| SettingsDropdown | `recording/SettingsDropdown.tsx` | onBlurToggle | 1 |
| PreRecordingScreen | `recording/PreRecordingScreen.tsx` | onStartRecording | 1 |
| RecordingInProgressScreen | `recording/RecordingInProgressScreen.tsx` | onStopRecording | 1 |
| PostRecordingReviewScreen | `recording/PostRecordingReviewScreen.tsx` | onSubmit, onStartOver | 2 |
| ErrorScreen | `recording/ErrorScreen.tsx` | onPrimaryAction, onSecondaryAction | 2 |
| VerificationErrorScreen | `verification/VerificationErrorScreen.tsx` | onPrimaryAction, onSecondaryAction | 2 |
| RecordingApp | `recording/RecordingApp.tsx` | Multiple internal handlers | 5+ |
| CaptureApp | `capture/CaptureApp.tsx` | Cross-tab callbacks | 5+ |

### 4.2 Recommendations

The callback patterns are well-structured. Consider using the Actions Object pattern for consistency:

```typescript
// Group related callbacks
interface RecordingActions {
  start: (mode: RecordingMode) => void;
  stop: () => void;
  submit: () => void;
  startOver: () => void;
}

// Pass as single prop
<RecordingScreen actions={recordingActions} />
```

---

## 5. Context Provider Analysis

### 5.1 AppModeContext (Single Provider)

**File:** `apps/recorder/src/contexts/AppModeContext.tsx`

**Interface (Lines 41-56):**

```typescript
interface AppModeContextValue {
  mode: AppMode;
  isEmbedded: boolean;
  isStandalone: boolean;
  bridge: RecorderBridgeConnection | null;
  verificationBridge: VerificationBridgeConnection | null;
  crossTab: CrossTabCoordinator | null;
  crossTabLockStatus: CrossTabLockStatus;
  crossTabError: string | null;
  blockingTabId: string | null;
}
```

**Provider Props (Lines 66-81):**

```typescript
interface AppModeProviderProps {
  children: ReactNode;
  actionType: AppActionType;
  crossTabWorkerURL?: string;
}
```

### 5.2 Context Architecture (Good Pattern ✅)

```
AppModeProvider
├── mode detection (useMemo)
├── bridge connections (useRef + useEffect)
├── lock status state (useState)
└── Context value (useMemo)
```

**Derived Hooks (All well-designed):**
- `useAppMode()` - Primary hook (Line 276)
- `useIsEmbedded()` - Boolean convenience (Line 288)
- `useIsStandalone()` - Boolean convenience (Line 296)
- `useBridge()` - Bridge accessor (Line 304)
- `useVerificationBridge()` - Verification bridge (Line 312)
- `useCrossTab()` - Cross-tab coordinator (Line 321)
- `useCrossTabLock()` - Lock status (Line 329)

### 5.3 Recommendation - Follows State/Actions/Meta Pattern

The context already follows a good pattern. Could be enhanced with explicit separation:

```typescript
interface AppModeContextValue {
  state: {
    mode: AppMode;
    isEmbedded: boolean;
    isStandalone: boolean;
    crossTabLockStatus: CrossTabLockStatus;
    crossTabError: string | null;
    blockingTabId: string | null;
  };
  connections: {
    bridge: RecorderBridgeConnection | null;
    verificationBridge: VerificationBridgeConnection | null;
    crossTab: CrossTabCoordinator | null;
  };
}
```

---

## 6. State Management Analysis

### 6.1 useState Usage by File

| File | useState Count | Variables |
|------|----------------|-----------|
| `useRecordingFlow.ts` | 15 | flowState, error, sessionInfo, recordingMode, recordingBlob, thumbnailBlob, videoUrl, uploadProgress, requesterName, mediaStream, isProcessing, maxRecordingDurationMs, maxRecordingSize, recordingDurationMs, preferences, countdownSeconds, uploadState |
| `useVerificationFlow.ts` | 4 | flowState, error, result, domainUrl |
| `AppModeContext.tsx` | 3 | lockStatus, lockError, blockingTabId |
| `CaptureApp.tsx` | 5 | isConnected, isCrossTabConnected, error, recorderState |
| `RecordingApp.tsx` | 4 | linkToken, action, workspaceIds, blurOverride |
| `VerificationApp.tsx` | 2 | token, workspaceIds |
| `VideoPlayer.tsx` | 2 | isPlaying, hasError |
| `PreRecordingScreen.tsx` | 1 | selectedMode |
| `RecordingInProgressScreen.tsx` | 1 | isLocalProcessing |

### 6.2 Heavy State in useRecordingFlow (15 useState calls)

**File:** `apps/recorder/src/hooks/useRecordingFlow.ts`
**Lines:** 87-107

This is a complex hook managing the entire recording flow. Consider:

1. **Using useReducer for related state:**

```typescript
// CURRENT - 15 separate useState calls
const [flowState, setFlowState] = useState<FlowState>('init');
const [error, setError] = useState<Error | null>(null);
const [sessionInfo, setSessionInfo] = useState<SessionInfo | null>(null);
// ... 12 more

// RECOMMENDED - Group related state
interface RecordingState {
  flow: FlowState;
  error: Error | null;
  session: SessionInfo | null;
  recording: {
    mode: RecordingMode | null;
    blob: Blob | null;
    thumbnailBlob: Blob | null;
    videoUrl: string | null;
    durationMs: number;
  };
  upload: {
    progress: number;
    state: UploadState;
  };
  config: {
    maxDurationMs: number;
    maxSize: number;
    countdownSeconds: number;
  };
  media: {
    stream: MediaStream | null;
    isProcessing: boolean;
  };
}

const [state, dispatch] = useReducer(recordingReducer, initialState);
```

### 6.3 useRef Usage (Good Pattern ✅)

| File | Refs | Purpose |
|------|------|---------|
| `useRecordingFlow.ts` | 6 | sessionRef, mediaCaptureRef, beforeUnloadHandlerRef, countdownIntervalRef, autoStopTimerRef, stopRecordingRef |
| `AppModeContext.tsx` | 3 | bridgeRef, verificationBridgeRef, crossTabRef |
| `RecordingApp.tsx` | 3 | prevFlowStateRef, sessionReportedRef, prevBlurStateRef |
| `CaptureApp.tsx` | 2 | bridgeRef, crossTabRef |

All refs are properly used for cleanup tracking and DOM references.

---

## 7. Custom Hooks Analysis

### 7.1 useRecordingFlow

**File:** `apps/recorder/src/hooks/useRecordingFlow.ts`
**Lines:** 78-825

```typescript
export function useRecordingFlow(
  linkToken: string,
  action: string,
  workspaceIds: string[]
): {
  state: RecordingFlowState;
  actions: RecordingFlowActions;
  sessionInfo: SessionInfo | null;
}
```

**Exports:**
- `FlowState` type
- `SessionInfo` type
- `RecordingFlowState` type
- `RecordingFlowActions` type
- `useRecordingFlow` hook

**Complexity:** HIGH (825 lines)

**Recommendation:** Consider splitting into smaller hooks:
- `useRecordingSession` - Session management
- `useMediaCapture` - Media stream handling
- `useUploadFlow` - Upload state management

### 7.2 useVerificationFlow

**File:** `apps/recorder/src/hooks/useVerificationFlow.ts`
**Lines:** 60-196

```typescript
export function useVerificationFlow(
  token: string,
  workspaceIds: string[],
  bridge: VerificationBridgeConnection | null
): {
  state: VerificationState;
  actions: VerificationFlowActions;
}
```

**Complexity:** MEDIUM (136 lines) - Well-sized

### 7.3 Context-Derived Hooks (Good Pattern ✅)

All 7 hooks in `AppModeContext.tsx` are well-designed convenience hooks:

```typescript
// Single source hook
useAppMode()

// Derived convenience hooks
useIsEmbedded()      // Returns boolean
useIsStandalone()    // Returns boolean
useBridge()          // Returns connection or null
useVerificationBridge()
useCrossTab()
useCrossTabLock()    // Returns object with status
```

---

## 8. useEffect Analysis

**Total: 17+ useEffect calls**

### 8.1 Effect Patterns by File

| File | Effects | Purpose |
|------|---------|---------|
| `AppModeContext.tsx` | 2 | Bridge connections |
| `useRecordingFlow.ts` | 4 | Session init, beforeUnload, auto-stop timer, ref sync |
| `useVerificationFlow.ts` | 1 | Verification trigger |
| `VideoPlayer.tsx` | 1 | Event listeners |
| `RecordingInProgressScreen.tsx` | 2 | Video stream, processing state |
| `PreRecordingCountdownScreen.tsx` | 1 | Video stream |
| `RecordingApp.tsx` | 4 | URL params, state reporting, blur broadcasting, session init |
| `VerificationApp.tsx` | 2 | URL params, completion reporting |
| `CaptureApp.tsx` | 2 | Bridge init, status logging |
| `VerifiedScreen.tsx` | 1 | Auto-close timer |
| `PostRecordingAckScreen.tsx` | 1 | Auto-close timer |

### 8.2 Cleanup Patterns (Good ✅)

All effects with subscriptions properly clean up:

```typescript
// Example from AppModeContext.tsx
useEffect(() => {
  let mounted = true;

  async function connectAll() {
    if (!mounted) return;
    // ...
  }

  connectAll();

  return () => {
    mounted = false;
    bridge.disconnect();
    crossTab.disconnect();
  };
}, [isEmbedded, isRecording]);
```

---

## 9. Component Architecture

### 9.1 Recording Flow Components

```
RecordingApp.tsx (Orchestrator)
├── PreRecordingScreen
│   └── RecordingScreenLayout
├── PreRecordingCountdownScreen
│   └── RecordingScreenLayout
├── RecordingInProgressScreen
│   ├── RecordingScreenLayout
│   └── SettingsDropdown
├── PostRecordingReviewScreen
│   ├── RecordingScreenLayout
│   └── VideoPlayer
├── PostRecordingUploadScreen
│   ├── RecordingScreenLayout
│   └── VideoPlayer
├── PostRecordingAckScreen
│   └── RecordingScreenLayout
└── ErrorScreen
    └── RecordingScreenLayout
```

### 9.2 Verification Flow Components

```
VerificationApp.tsx (Orchestrator)
├── VerificationScreenLayout
├── VerifyingScreen
├── VerifiedScreen
└── VerificationErrorScreen
```

### 9.3 Capture Flow Components

```
CaptureApp.tsx (Orchestrator)
└── (Headless - no UI, manages cross-tab communication)
```

---

## 10. Priority Action Items

### Immediate (P0)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 1 | Migrate useContext to use() | `AppModeContext.tsx` | Low |

### Short-term (P1)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 2 | Group ErrorScreen action props | `ErrorScreen.tsx` | Low |
| 3 | Group VerificationErrorScreen props | `VerificationErrorScreen.tsx` | Low |
| 4 | Consolidate VideoPlayer state | `VideoPlayer.tsx` | Low |
| 5 | Add status enum to RecordingInProgressScreen | `RecordingInProgressScreen.tsx` | Low |

### Long-term (P2)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 6 | Refactor useRecordingFlow to useReducer | `useRecordingFlow.ts` | High |
| 7 | Split useRecordingFlow into smaller hooks | `useRecordingFlow.ts` | High |
| 8 | Restructure AppModeContext with state/connections | `AppModeContext.tsx` | Medium |

---

## 11. What's Working Well ✅

1. **No forwardRef** - Already React 19 compatible in this regard
2. **Single Context Provider** - Clean, centralized state management
3. **Derived Hooks** - Well-designed convenience hooks from context
4. **Cleanup Patterns** - All effects properly clean up subscriptions
5. **Ref Usage** - Correct use of refs for cleanup tracking
6. **Component Hierarchy** - Clear separation between flows
7. **Type Safety** - All interfaces properly typed

---

## 12. Files Reference

### Files Requiring Changes

```
apps/recorder/src/
├── contexts/
│   └── AppModeContext.tsx (useContext → use())
├── components/
│   ├── recording/
│   │   ├── ErrorScreen.tsx (prop grouping)
│   │   ├── VideoPlayer.tsx (state consolidation)
│   │   └── RecordingInProgressScreen.tsx (status enum)
│   └── verification/
│       └── VerificationErrorScreen.tsx (prop grouping)
└── hooks/
    └── useRecordingFlow.ts (useReducer refactor)
```

### Well-Designed Files (Reference)

```
apps/recorder/src/
├── contexts/
│   └── AppModeContext.tsx (provider pattern, derived hooks)
├── hooks/
│   └── useVerificationFlow.ts (good size, clear interface)
└── components/
    └── recording/
        └── RecordingScreenLayout.tsx (clean layout component)
```

---

## 13. Storybook Coverage

The recorder app has Storybook stories for key components:

| Component | Story File |
|-----------|------------|
| VideoPlayer | `VideoPlayer.stories.tsx` |
| PreRecordingScreen | `PreRecordingScreen.stories.tsx` |
| RecordingInProgressScreen | `RecordingInProgressScreen.stories.tsx` |
| PostRecordingReviewScreen | `PostRecordingReviewScreen.stories.tsx` |

This enables visual testing of composition patterns.

---

*Generated by Claude Code using Vercel Composition Patterns Skill*
