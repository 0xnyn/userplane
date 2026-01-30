# Recorder App - Composition Pattern Improvements

**App:** apps/recorder
**Based on:** Vercel Composition Patterns Audit
**Date:** January 30, 2026

---

## Table of Contents

1. [React 19 Migration](#1-react-19-migration)
2. [Reduce Boolean Prop Proliferation](#2-reduce-boolean-prop-proliferation)
3. [Group High-Prop Components](#3-group-high-prop-components)
4. [Refactor useRecordingFlow](#4-refactor-userecordingflow)
5. [Enhance AppModeContext](#5-enhance-appmodecontext)
6. [Create Action Prop Patterns](#6-create-action-prop-patterns)

---

## 1. React 19 Migration

### 1.1 Migrate useContext to use()

**File:** `apps/recorder/src/contexts/AppModeContext.tsx`

**Current Implementation (Line 11, 277):**
```typescript
import {
  createContext,
  type ReactNode,
  useContext,  // ❌ Legacy React 18 pattern
  useEffect,
  useMemo,
  useRef,
  useState,
} from 'react';

// Line 277
export function useAppMode(): AppModeContextValue {
  const context = useContext(AppModeContext);
  if (!context) {
    throw new Error('useAppMode must be used within an AppModeProvider');
  }
  return context;
}
```

**Improved Implementation:**
```typescript
import {
  createContext,
  type ReactNode,
  use,  // ✅ React 19 pattern
  useEffect,
  useMemo,
  useRef,
  useState,
} from 'react';

// Line 277
export function useAppMode(): AppModeContextValue {
  const context = use(AppModeContext);
  if (!context) {
    throw new Error('useAppMode must be used within an AppModeProvider');
  }
  return context;
}
```

**Benefits of `use()`:**
- Can be called conditionally inside components
- Works seamlessly with React Suspense
- Better TypeScript inference
- Cleaner API for async operations

### 1.2 Update Derived Hooks

All derived hooks in `AppModeContext.tsx` will automatically benefit from the change since they call `useAppMode()` internally:

```typescript
// These all work automatically after the migration
export function useIsEmbedded(): boolean {
  return useAppMode().isEmbedded;
}

export function useIsStandalone(): boolean {
  return useAppMode().isStandalone;
}

export function useBridge(): RecorderBridgeConnection | null {
  return useAppMode().bridge;
}

// ... etc
```

---

## 2. Reduce Boolean Prop Proliferation

### 2.1 Refactor VideoPlayer State

**File:** `apps/recorder/src/components/recording/VideoPlayer.tsx`

**Current Implementation (Lines 13-42):**
```typescript
interface VideoPlayerProps {
  videoUrl: string;
  autoPlay?: boolean;
  poster?: string;
  muted?: boolean;
  className?: string;
  showControls?: boolean;  // ❌ Boolean prop
  onError?: (error: Error) => void;
  onLoadStart?: () => void;
  onCanPlay?: () => void;
}

// Internal state
const [isPlaying, setIsPlaying] = useState(false);  // ❌ Boolean state
const [hasError, setHasError] = useState(false);    // ❌ Boolean state
```

**Improved Implementation:**
```typescript
// Define explicit player states
type VideoPlayerStatus = 'idle' | 'loading' | 'ready' | 'playing' | 'paused' | 'error';

type ControlsVisibility = 'visible' | 'hidden' | 'auto';

interface VideoPlayerProps {
  videoUrl: string;
  autoPlay?: boolean;
  poster?: string;
  muted?: boolean;
  className?: string;
  controlsVisibility?: ControlsVisibility;  // ✅ Explicit options
  onError?: (error: Error) => void;
  onLoadStart?: () => void;
  onCanPlay?: () => void;
}

export function VideoPlayer({
  videoUrl,
  autoPlay = false,
  poster,
  muted = false,
  className,
  controlsVisibility = 'auto',
  onError,
  onLoadStart,
  onCanPlay,
}: VideoPlayerProps) {
  // Single state for player status
  const [status, setStatus] = useState<VideoPlayerStatus>('idle');

  // Derive booleans from status
  const isPlaying = status === 'playing';
  const hasError = status === 'error';
  const isLoading = status === 'loading';
  const showControls =
    controlsVisibility === 'visible' ||
    (controlsVisibility === 'auto' && status !== 'playing');

  // Event handlers update status
  const handlePlay = useCallback(() => setStatus('playing'), []);
  const handlePause = useCallback(() => setStatus('paused'), []);
  const handleError = useCallback((e: Error) => {
    setStatus('error');
    onError?.(e);
  }, [onError]);

  // ... rest of implementation
}
```

### 2.2 Refactor RecordingInProgressScreen

**File:** `apps/recorder/src/components/recording/RecordingInProgressScreen.tsx`

**Current (Line 23):**
```typescript
interface RecordingInProgressScreenProps {
  title?: string;
  subtitle?: string;
  mediaStream?: MediaStream | null;
  preferences: RecordingPreferences;
  isProcessing?: boolean;  // ❌ Boolean
  onStopRecording?: () => void;
  settingsElement?: ReactNode;
}
```

**Improved:**
```typescript
type RecordingStatus = 'recording' | 'processing' | 'stopping';

interface RecordingInProgressScreenProps {
  title?: string;
  subtitle?: string;
  mediaStream?: MediaStream | null;
  preferences: RecordingPreferences;
  status?: RecordingStatus;  // ✅ Explicit status
  onStopRecording?: () => void;
  settingsElement?: ReactNode;
}

// Usage
<RecordingInProgressScreen
  status="processing"  // Instead of isProcessing={true}
  // ...
/>
```

### 2.3 Refactor SettingsDropdown

**File:** `apps/recorder/src/components/recording/SettingsDropdown.tsx`

**Current (Lines 17-22):**
```typescript
interface SettingsDropdownProps {
  isBlurEnabled: boolean;
  onBlurToggle: (enabled: boolean) => void;
  disabled?: boolean;
}
```

**Improved - Use settings object:**
```typescript
interface PrivacySettings {
  blurEnabled: boolean;
}

interface SettingsDropdownProps {
  settings: PrivacySettings;
  onSettingsChange: (settings: Partial<PrivacySettings>) => void;
  disabled?: boolean;
}

// Usage
<SettingsDropdown
  settings={{ blurEnabled: true }}
  onSettingsChange={(changes) => updateSettings(changes)}
/>

// Extensible for future settings
interface PrivacySettings {
  blurEnabled: boolean;
  audioEnabled?: boolean;  // Easy to add
  webcamEnabled?: boolean; // Easy to add
}
```

---

## 3. Group High-Prop Components

### 3.1 Refactor ErrorScreen

**File:** `apps/recorder/src/components/recording/ErrorScreen.tsx`

**Current (Lines 14-28):**
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

**Improved - Group action props:**
```typescript
interface ActionConfig {
  text: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary';
}

interface ErrorScreenProps {
  title: string;
  subtitle: string;
  preferences?: RecordingPreferences;
  primaryAction?: ActionConfig;
  secondaryAction?: ActionConfig;
}

// Usage
<ErrorScreen
  title="Recording Failed"
  subtitle="Something went wrong"
  primaryAction={{
    text: 'Try Again',
    onClick: handleRetry,
  }}
  secondaryAction={{
    text: 'Go Back',
    onClick: handleGoBack,
  }}
/>
```

### 3.2 Refactor VerificationErrorScreen

**File:** `apps/recorder/src/components/verification/VerificationErrorScreen.tsx`

**Current (Lines 13-31):**
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

**Improved - Use compound prop groups:**
```typescript
interface WorkspaceBranding {
  logoUrl: string | null;
  name: string | null;
  primaryColor?: string;
}

interface ActionConfig {
  text: string;
  onClick: () => void;
}

interface VerificationErrorScreenProps {
  title: string;
  subtitle: string;
  workspace?: WorkspaceBranding;
  primaryAction?: ActionConfig;
  secondaryAction?: ActionConfig;
}

// Usage
<VerificationErrorScreen
  title="Verification Failed"
  subtitle="Unable to verify domain"
  workspace={{
    logoUrl: session.workspaceLogoUrl,
    name: session.workspaceName,
    primaryColor: '#3B82F6',
  }}
  primaryAction={{
    text: 'Try Again',
    onClick: handleRetry,
  }}
/>
```

### 3.3 Refactor PostRecordingReviewScreen

**File:** `apps/recorder/src/components/recording/PostRecordingReviewScreen.tsx`

**Current (Lines 19-33):**
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

**Improved:**
```typescript
type SubmitStatus = 'idle' | 'submitting' | 'error';

interface VideoData {
  url: string;
  preferences: RecordingPreferences;
}

interface ReviewActions {
  onSubmit: () => void;
  onStartOver: () => void;
}

interface PostRecordingReviewScreenProps {
  video: VideoData;
  submitStatus: SubmitStatus;
  errorMessage?: string | null;
  actions: ReviewActions;
  settingsElement?: ReactNode;
}

// Usage
<PostRecordingReviewScreen
  video={{
    url: videoUrl,
    preferences: recordingPreferences,
  }}
  submitStatus={isSubmitting ? 'submitting' : 'idle'}
  errorMessage={error?.message}
  actions={{
    onSubmit: handleSubmit,
    onStartOver: handleStartOver,
  }}
  settingsElement={<SettingsDropdown />}
/>
```

---

## 4. Refactor useRecordingFlow

### 4.1 Replace Multiple useState with useReducer

**File:** `apps/recorder/src/hooks/useRecordingFlow.ts`

**Current (Lines 87-107) - 15 useState calls:**
```typescript
const [flowState, setFlowState] = useState<FlowState>('init');
const [error, setError] = useState<Error | null>(null);
const [sessionInfo, setSessionInfo] = useState<SessionInfo | null>(null);
const [recordingMode, setRecordingMode] = useState<RecordingMode | null>(null);
const [recordingBlob, setRecordingBlob] = useState<Blob | null>(null);
const [thumbnailBlob, setThumbnailBlob] = useState<Blob | null>(null);
const [videoUrl, setVideoUrl] = useState<string | null>(null);
const [uploadProgress, setUploadProgress] = useState(0);
const [requesterName, setRequesterName] = useState<string | null>(null);
const [mediaStream, setMediaStream] = useState<MediaStream | null>(null);
const [isProcessing, setIsProcessing] = useState(false);
const [maxRecordingDurationMs, setMaxRecordingDurationMs] = useState(0);
const [maxRecordingSize, setMaxRecordingSize] = useState(0);
const [recordingDurationMs, setRecordingDurationMs] = useState(0);
const [preferences, setPreferences] = useState<RecordingPreferences | null>(null);
const [countdownSeconds, setCountdownSeconds] = useState(3);
const [uploadState, setUploadState] = useState<UploadState>('idle');
```

**Improved - Use useReducer with grouped state:**

```typescript
// State types
interface RecordingFlowState {
  // Flow control
  flowState: FlowState;
  error: Error | null;

  // Session
  session: SessionInfo | null;
  requesterName: string | null;

  // Recording
  recording: {
    mode: RecordingMode | null;
    blob: Blob | null;
    thumbnailBlob: Blob | null;
    videoUrl: string | null;
    durationMs: number;
    isProcessing: boolean;
  };

  // Media
  media: {
    stream: MediaStream | null;
  };

  // Upload
  upload: {
    state: UploadState;
    progress: number;
  };

  // Configuration
  config: {
    maxDurationMs: number;
    maxSize: number;
    countdownSeconds: number;
    preferences: RecordingPreferences | null;
  };
}

// Actions
type RecordingAction =
  | { type: 'SET_FLOW_STATE'; payload: FlowState }
  | { type: 'SET_ERROR'; payload: Error | null }
  | { type: 'SET_SESSION'; payload: { session: SessionInfo; requesterName: string | null } }
  | { type: 'SET_RECORDING_MODE'; payload: RecordingMode }
  | { type: 'SET_MEDIA_STREAM'; payload: MediaStream | null }
  | { type: 'START_RECORDING' }
  | { type: 'STOP_RECORDING'; payload: { blob: Blob; thumbnailBlob: Blob; durationMs: number } }
  | { type: 'SET_VIDEO_URL'; payload: string }
  | { type: 'START_UPLOAD' }
  | { type: 'UPDATE_UPLOAD_PROGRESS'; payload: number }
  | { type: 'UPLOAD_COMPLETE' }
  | { type: 'UPLOAD_ERROR'; payload: Error }
  | { type: 'SET_CONFIG'; payload: Partial<RecordingFlowState['config']> }
  | { type: 'SET_PROCESSING'; payload: boolean }
  | { type: 'RESET' };

// Reducer
function recordingReducer(state: RecordingFlowState, action: RecordingAction): RecordingFlowState {
  switch (action.type) {
    case 'SET_FLOW_STATE':
      return { ...state, flowState: action.payload };

    case 'SET_ERROR':
      return { ...state, error: action.payload };

    case 'SET_SESSION':
      return {
        ...state,
        session: action.payload.session,
        requesterName: action.payload.requesterName,
      };

    case 'SET_RECORDING_MODE':
      return {
        ...state,
        recording: { ...state.recording, mode: action.payload },
      };

    case 'SET_MEDIA_STREAM':
      return {
        ...state,
        media: { stream: action.payload },
      };

    case 'STOP_RECORDING':
      return {
        ...state,
        recording: {
          ...state.recording,
          blob: action.payload.blob,
          thumbnailBlob: action.payload.thumbnailBlob,
          durationMs: action.payload.durationMs,
          isProcessing: false,
        },
      };

    case 'START_UPLOAD':
      return {
        ...state,
        upload: { state: 'uploading', progress: 0 },
      };

    case 'UPDATE_UPLOAD_PROGRESS':
      return {
        ...state,
        upload: { ...state.upload, progress: action.payload },
      };

    case 'UPLOAD_COMPLETE':
      return {
        ...state,
        upload: { state: 'complete', progress: 100 },
      };

    case 'SET_CONFIG':
      return {
        ...state,
        config: { ...state.config, ...action.payload },
      };

    case 'SET_PROCESSING':
      return {
        ...state,
        recording: { ...state.recording, isProcessing: action.payload },
      };

    case 'RESET':
      return initialState;

    default:
      return state;
  }
}

// Initial state
const initialState: RecordingFlowState = {
  flowState: 'init',
  error: null,
  session: null,
  requesterName: null,
  recording: {
    mode: null,
    blob: null,
    thumbnailBlob: null,
    videoUrl: null,
    durationMs: 0,
    isProcessing: false,
  },
  media: {
    stream: null,
  },
  upload: {
    state: 'idle',
    progress: 0,
  },
  config: {
    maxDurationMs: 0,
    maxSize: 0,
    countdownSeconds: 3,
    preferences: null,
  },
};

// Hook implementation
export function useRecordingFlow(
  linkToken: string,
  action: string,
  workspaceIds: string[]
) {
  const [state, dispatch] = useReducer(recordingReducer, initialState);

  // ... rest of implementation using dispatch instead of multiple setters

  const actions = {
    setFlowState: (flowState: FlowState) => dispatch({ type: 'SET_FLOW_STATE', payload: flowState }),
    setError: (error: Error | null) => dispatch({ type: 'SET_ERROR', payload: error }),
    // ... other actions
  };

  return { state, actions, sessionInfo: state.session };
}
```

### 4.2 Split useRecordingFlow into Smaller Hooks

**Alternative approach - Composition of smaller hooks:**

```typescript
// apps/recorder/src/hooks/useRecordingSession.ts
export function useRecordingSession(linkToken: string, action: string, workspaceIds: string[]) {
  const [session, setSession] = useState<SessionInfo | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // Session initialization logic
  useEffect(() => {
    // ... fetch session
  }, [linkToken, action, workspaceIds]);

  return { session, error, isLoading };
}

// apps/recorder/src/hooks/useMediaCapture.ts
export function useMediaCapture(mode: RecordingMode | null) {
  const [stream, setStream] = useState<MediaStream | null>(null);
  const [isCapturing, setIsCapturing] = useState(false);
  const mediaCaptureRef = useRef<MediaCapture | null>(null);

  const startCapture = useCallback(async () => {
    // ... start capture
  }, [mode]);

  const stopCapture = useCallback(async () => {
    // ... stop capture and return blob
  }, []);

  return { stream, isCapturing, startCapture, stopCapture };
}

// apps/recorder/src/hooks/useRecordingUpload.ts
export function useRecordingUpload() {
  const [uploadState, setUploadState] = useState<UploadState>('idle');
  const [progress, setProgress] = useState(0);

  const upload = useCallback(async (blob: Blob, metadata: UploadMetadata) => {
    // ... upload logic
  }, []);

  return { uploadState, progress, upload };
}

// Compose in main hook
export function useRecordingFlow(linkToken: string, action: string, workspaceIds: string[]) {
  const session = useRecordingSession(linkToken, action, workspaceIds);
  const capture = useMediaCapture(recordingMode);
  const upload = useRecordingUpload();

  // Coordinate between hooks
  // ...

  return {
    session,
    capture,
    upload,
    // ... combined state and actions
  };
}
```

---

## 5. Enhance AppModeContext

### 5.1 Restructure with State/Connections Pattern

**File:** `apps/recorder/src/contexts/AppModeContext.tsx`

**Current (Lines 41-56):**
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

**Improved - Clear separation:**
```typescript
interface AppModeState {
  mode: AppMode;
  isEmbedded: boolean;
  isStandalone: boolean;
  lock: {
    status: CrossTabLockStatus;
    error: string | null;
    blockingTabId: string | null;
  };
}

interface AppModeConnections {
  bridge: RecorderBridgeConnection | null;
  verificationBridge: VerificationBridgeConnection | null;
  crossTab: CrossTabCoordinator | null;
}

interface AppModeContextValue {
  state: AppModeState;
  connections: AppModeConnections;
}

// Provider implementation
export function AppModeProvider({ children, actionType, crossTabWorkerURL }: AppModeProviderProps) {
  // ... existing logic

  const value: AppModeContextValue = {
    state: {
      mode,
      isEmbedded,
      isStandalone: mode === 'standalone',
      lock: {
        status: lockStatus,
        error: lockError,
        blockingTabId,
      },
    },
    connections: {
      bridge: isEmbedded ? bridgeRef.current : null,
      verificationBridge: isEmbedded ? verificationBridgeRef.current : null,
      crossTab: isEmbedded ? crossTabRef.current : null,
    },
  };

  return (
    <AppModeContext value={value}>
      {children}
    </AppModeContext>
  );
}

// Updated hooks
export function useAppModeState() {
  return useAppMode().state;
}

export function useAppModeConnections() {
  return useAppMode().connections;
}

export function useCrossTabLock() {
  const { lock } = useAppMode().state;
  return {
    ...lock,
    isAcquired: lock.status === 'acquired',
    isBlocked: lock.status === 'blocked',
    isPending: lock.status === 'pending' || lock.status === 'connecting',
  };
}
```

---

## 6. Create Action Prop Patterns

### 6.1 Define Reusable Action Types

**New File:** `apps/recorder/src/types/actions.ts`

```typescript
// Reusable action configuration type
export interface ActionConfig {
  text: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary' | 'destructive';
  loading?: boolean;
  disabled?: boolean;
}

// Common action sets
export interface ConfirmCancelActions {
  confirm: ActionConfig;
  cancel: ActionConfig;
}

export interface SubmitStartOverActions {
  submit: ActionConfig;
  startOver: ActionConfig;
}

export interface RetryGoBackActions {
  retry: ActionConfig;
  goBack: ActionConfig;
}
```

### 6.2 Create Action Button Component

**New File:** `apps/recorder/src/components/shared/ActionButton.tsx`

```typescript
import type { ActionConfig } from '@/types/actions';
import { Button } from '@workspace/ui';

interface ActionButtonProps extends ActionConfig {}

export function ActionButton({
  text,
  onClick,
  variant = 'primary',
  loading = false,
  disabled = false,
}: ActionButtonProps) {
  return (
    <Button
      variant={variant}
      onClick={onClick}
      disabled={disabled || loading}
    >
      {loading ? 'Loading...' : text}
    </Button>
  );
}

// Action group component
interface ActionGroupProps {
  primary?: ActionConfig;
  secondary?: ActionConfig;
  layout?: 'horizontal' | 'vertical';
}

export function ActionGroup({ primary, secondary, layout = 'horizontal' }: ActionGroupProps) {
  return (
    <div className={layout === 'horizontal' ? 'flex gap-3' : 'flex flex-col gap-2'}>
      {secondary && (
        <ActionButton {...secondary} variant={secondary.variant ?? 'secondary'} />
      )}
      {primary && (
        <ActionButton {...primary} variant={primary.variant ?? 'primary'} />
      )}
    </div>
  );
}
```

### 6.3 Update Error Screens to Use Action Pattern

```typescript
// ErrorScreen.tsx
import { ActionGroup, type ActionConfig } from '@/components/shared/ActionButton';

interface ErrorScreenProps {
  title: string;
  subtitle: string;
  preferences?: RecordingPreferences;
  primaryAction?: ActionConfig;
  secondaryAction?: ActionConfig;
}

export function ErrorScreen({
  title,
  subtitle,
  preferences,
  primaryAction,
  secondaryAction,
}: ErrorScreenProps) {
  return (
    <RecordingScreenLayout preferences={preferences}>
      <div className="text-center">
        <h1>{title}</h1>
        <p>{subtitle}</p>

        <ActionGroup
          primary={primaryAction}
          secondary={secondaryAction}
        />
      </div>
    </RecordingScreenLayout>
  );
}
```

---

## Implementation Checklist

### Phase 1: Quick Wins (1 day)
- [ ] Migrate `useContext` to `use()` in AppModeContext
- [ ] Group ErrorScreen action props
- [ ] Group VerificationErrorScreen props
- [ ] Create ActionConfig type

### Phase 2: State Improvements (2-3 days)
- [ ] Refactor VideoPlayer with status enum
- [ ] Refactor RecordingInProgressScreen status
- [ ] Create ActionButton and ActionGroup components
- [ ] Restructure AppModeContext with state/connections

### Phase 3: Hook Refactoring (1 week)
- [ ] Implement useReducer in useRecordingFlow
- [ ] Or split useRecordingFlow into smaller hooks
- [ ] Add comprehensive tests for state transitions

### Phase 4: Polish (2-3 days)
- [ ] Update all screens to use new patterns
- [ ] Update Storybook stories
- [ ] Document patterns in component README

---

## Testing Strategy

### Unit Tests for Reducer

```typescript
describe('recordingReducer', () => {
  it('should transition to recording state', () => {
    const state = recordingReducer(initialState, { type: 'START_RECORDING' });
    expect(state.flowState).toBe('recording');
  });

  it('should handle upload progress', () => {
    const state = recordingReducer(
      { ...initialState, upload: { state: 'uploading', progress: 0 } },
      { type: 'UPDATE_UPLOAD_PROGRESS', payload: 50 }
    );
    expect(state.upload.progress).toBe(50);
  });

  // ... more tests
});
```

### Integration Tests for Flows

```typescript
describe('Recording Flow', () => {
  it('should complete full recording flow', async () => {
    const { result } = renderHook(() => useRecordingFlow('token', 'record', ['ws1']));

    // Start recording
    act(() => result.current.actions.startRecording('screen'));

    // Wait for state transition
    await waitFor(() => expect(result.current.state.flowState).toBe('recording'));

    // Stop recording
    act(() => result.current.actions.stopRecording());

    // Verify state
    expect(result.current.state.recording.blob).not.toBeNull();
  });
});
```

---

*Implementation guide based on Vercel Composition Patterns Audit*
