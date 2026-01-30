# Dash App - Composition Pattern Improvements

**App:** apps/dash
**Based on:** Vercel Composition Patterns Audit
**Date:** January 30, 2026

---

## Table of Contents

1. [React 19 Migration](#1-react-19-migration)
2. [Reduce Boolean Prop Proliferation](#2-reduce-boolean-prop-proliferation)
3. [Extract Domain Actions Provider](#3-extract-domain-actions-provider)
4. [Create Modal Compound Component](#4-create-modal-compound-component)
5. [Consolidate Player State](#5-consolidate-player-state)
6. [Extract Reusable Hooks](#6-extract-reusable-hooks)

---

## 1. React 19 Migration

### 1.1 Migrate URLInput forwardRef

**File:** `apps/dash/src/components/url-input.tsx`

**Current Implementation (Lines 49-95):**
```typescript
export const URLInput = React.forwardRef<HTMLInputElement, URLInputProps>(
  (
    { value, onChange, placeholder, className, error, id, onKeyDown, disabled, onBlur },
    ref
  ) => {
    const internalRef = useRef<HTMLInputElement>(null);

    useImperativeHandle(ref, () => internalRef.current!, []);

    // ... rest of implementation
  }
);

URLInput.displayName = 'URLInput';
```

**Improved Implementation:**
```typescript
interface URLInputProps {
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
  className?: string;
  error?: boolean;
  id?: string;
  onKeyDown?: (e: React.KeyboardEvent<HTMLInputElement>) => void;
  disabled?: boolean;
  onBlur?: () => void;
  ref?: React.Ref<HTMLInputElement>;
}

export function URLInput({
  value,
  onChange,
  placeholder = 'Enter URL',
  className,
  error,
  id,
  onKeyDown,
  disabled,
  onBlur,
  ref,
}: URLInputProps) {
  const internalRef = useRef<HTMLInputElement>(null);

  // Merge refs if needed
  const mergedRef = useMergedRef(ref, internalRef);

  // ... rest of implementation using mergedRef

  return (
    <Input
      ref={mergedRef}
      // ... other props
    />
  );
}

// Helper hook for merging refs
function useMergedRef<T>(...refs: (React.Ref<T> | undefined)[]) {
  return useCallback((node: T) => {
    refs.forEach((ref) => {
      if (typeof ref === 'function') {
        ref(node);
      } else if (ref && 'current' in ref) {
        (ref as React.MutableRefObject<T>).current = node;
      }
    });
  }, refs);
}
```

### 1.2 Migrate DomainInput forwardRef

**File:** `apps/dash/src/components/domain-input.tsx`

Apply the same pattern as URLInput above.

---

## 2. Reduce Boolean Prop Proliferation

### 2.1 Refactor StreamControls

**File:** `apps/dash/src/components/player/StreamControls.tsx`

**Current Implementation (Lines 20-28):**
```typescript
interface StreamControlsProps {
  player: Player | null;
  markers?: VideoMarker[];
  onFullscreen?: () => void;
  duration?: number;
  hasStarted?: boolean;   // ❌ Boolean
  showControls?: boolean; // ❌ Boolean
  hasError?: boolean;     // ❌ Boolean
}
```

**Improved Implementation:**
```typescript
// Define explicit player states
type PlayerStatus = 'idle' | 'loading' | 'playing' | 'paused' | 'error';

interface StreamControlsProps {
  player: Player | null;
  markers?: VideoMarker[];
  onFullscreen?: () => void;
  duration?: number;
  status: PlayerStatus;
  controlsVisible?: boolean; // Keep this - it's UI visibility, not state
}

// Usage
const StreamControls: React.FC<StreamControlsProps> = ({
  player,
  markers = [],
  onFullscreen,
  duration: videoDuration = 0,
  status,
  controlsVisible = false,
}) => {
  const showBigPlayButton = status === 'idle';
  const showErrorOverlay = status === 'error';
  const showControls = controlsVisible || status === 'idle';

  return (
    <>
      {/* Video overlay based on status */}
      <VideoOverlay status={status} onPlay={handleBigPlayClick} />

      {/* Control Bar - hidden when error */}
      {status !== 'error' && (
        <div
          className="custom-video-controls"
          data-show-controls={showControls}
          data-initial-state={status === 'idle'}
        >
          {/* ... controls */}
        </div>
      )}
    </>
  );
};
```

### 2.2 Refactor VideoOverlay

**File:** `apps/dash/src/components/player/components/VideoOverlay.tsx`

**Current:**
```typescript
interface VideoOverlayProps {
  hasStarted: boolean;
  hasError: boolean;
  duration: number;
  onPlay: () => void;
}
```

**Improved:**
```typescript
type OverlayStatus = 'idle' | 'playing' | 'error';

interface VideoOverlayProps {
  status: OverlayStatus;
  duration: number;
  onPlay: () => void;
  errorMessage?: string;
}

// Or use compound components for different overlays
const VideoOverlay = {
  Idle: VideoOverlayIdle,
  Playing: VideoOverlayPlaying,
  Error: VideoOverlayError,
};

// Usage
{status === 'idle' && <VideoOverlay.Idle duration={duration} onPlay={onPlay} />}
{status === 'error' && <VideoOverlay.Error message={errorMessage} />}
```

### 2.3 Refactor DinoError

**File:** `apps/dash/src/components/shared/dino-error.tsx`

**Current:**
```typescript
interface DinoErrorProps {
  showRetry?: boolean;
  showGoBack?: boolean;
  onRetry?: () => void;
  onGoBack?: () => void;
}
```

**Improved - Use explicit variants:**
```typescript
// Base props
interface DinoErrorBaseProps {
  title?: string;
  message?: string;
}

// Variant-specific components
interface DinoErrorWithRetryProps extends DinoErrorBaseProps {
  onRetry: () => void;
}

interface DinoErrorWithGoBackProps extends DinoErrorBaseProps {
  onGoBack: () => void;
}

interface DinoErrorWithBothProps extends DinoErrorBaseProps {
  onRetry: () => void;
  onGoBack: () => void;
}

// Export as compound component
export const DinoError = {
  Basic: DinoErrorBasic,
  WithRetry: DinoErrorWithRetry,
  WithGoBack: DinoErrorWithGoBack,
  WithBoth: DinoErrorWithBoth,
};

// Usage
<DinoError.WithRetry onRetry={handleRetry} />
<DinoError.WithBoth onRetry={handleRetry} onGoBack={handleGoBack} />
```

---

## 3. Extract Domain Actions Provider

### 3.1 Create DomainActionsContext

**New File:** `apps/dash/src/contexts/domain-actions-context.tsx`

```typescript
import { createContext, use, type ReactNode } from 'react';
import type { CreatedDomain, DomainRecordingPreferences } from '@/types';

// Define the context interface following state/actions/meta pattern
interface DomainActionsState {
  loading: boolean;
  savingSettings: boolean;
}

interface DomainActions {
  create: (domainUrl: string) => Promise<CreatedDomain | null>;
  startVerification: (domainId: string) => Promise<string | null>;
  pollVerification: (domainId: string) => Promise<boolean>;
  fetchSettings: (domainId: string) => Promise<DomainRecordingPreferences>;
  saveSettings: (domainId: string, settings: DomainRecordingPreferences) => void;
}

interface DomainActionsMeta {
  workspaceId: string;
}

interface DomainActionsContextValue {
  state: DomainActionsState;
  actions: DomainActions;
  meta: DomainActionsMeta;
}

const DomainActionsContext = createContext<DomainActionsContextValue | null>(null);

// Provider component
interface DomainActionsProviderProps {
  children: ReactNode;
  workspaceId: string;
}

export function DomainActionsProvider({
  children,
  workspaceId
}: DomainActionsProviderProps) {
  // Use your existing hooks for domain operations
  const {
    createDomain,
    startVerification,
    pollVerification,
    isCreating,
  } = useWorkspaceDomains(workspaceId);

  const {
    fetchSettings,
    saveSettings,
    isSaving,
  } = useWorkspaceDomainSettings(workspaceId);

  const value: DomainActionsContextValue = {
    state: {
      loading: isCreating,
      savingSettings: isSaving,
    },
    actions: {
      create: createDomain,
      startVerification,
      pollVerification,
      fetchSettings,
      saveSettings,
    },
    meta: {
      workspaceId,
    },
  };

  return (
    <DomainActionsContext value={value}>
      {children}
    </DomainActionsContext>
  );
}

// Hook to consume context
export function useDomainActions(): DomainActionsContextValue {
  const context = use(DomainActionsContext);
  if (!context) {
    throw new Error('useDomainActions must be used within DomainActionsProvider');
  }
  return context;
}

// Convenience hooks
export function useDomainActionsState() {
  return useDomainActions().state;
}

export function useDomainActionsActions() {
  return useDomainActions().actions;
}
```

### 3.2 Refactor CreateDomainModal

**File:** `apps/dash/src/components/modals/create-domain-modal.tsx`

**Current (10 props):**
```typescript
interface CreateDomainModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onCreateDomain: (domainUrl: string) => Promise<CreatedDomain | null>;
  onStartVerification: (domainId: string) => Promise<string | null>;
  onPollVerification: (domainId: string) => Promise<boolean>;
  onFetchSettings: (domainId: string) => Promise<DomainRecordingPreferences>;
  onSaveSettings: (domainId: string, settings: DomainRecordingPreferences) => void;
  workspaceId: string;
  loading?: boolean;
  savingSettings?: boolean;
}
```

**Improved (2 props):**
```typescript
interface CreateDomainModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
}

export function CreateDomainModal({ open, onOpenChange }: CreateDomainModalProps) {
  // Get everything from context
  const { state, actions, meta } = useDomainActions();

  // Internal step state
  const [step, setStep] = useState<ModalStep>('create');
  const [createdDomain, setCreatedDomain] = useState<CreatedDomain | null>(null);

  const handleCreate = async (domainUrl: string) => {
    const domain = await actions.create(domainUrl);
    if (domain) {
      setCreatedDomain(domain);
      setStep('verify');
    }
  };

  // ... rest of implementation using actions from context
}

// Usage in parent
<DomainActionsProvider workspaceId={workspaceId}>
  <CreateDomainModal open={open} onOpenChange={setOpen} />
</DomainActionsProvider>
```

### 3.3 Refactor DomainConfigureModal

Apply the same pattern - consume context instead of prop drilling.

---

## 4. Create Modal Compound Component

### 4.1 Create Base Modal Components

**New File:** `packages/ui/src/components/modal.tsx`

```typescript
import * as React from 'react';
import * as DialogPrimitive from '@radix-ui/react-dialog';
import { X } from 'lucide-react';
import { cn } from '../lib/utils';

// Context for modal state
interface ModalContextValue {
  onClose: () => void;
}

const ModalContext = React.createContext<ModalContextValue | null>(null);

function useModal() {
  const context = React.use(ModalContext);
  if (!context) {
    throw new Error('Modal components must be used within Modal.Root');
  }
  return context;
}

// Root component
interface ModalRootProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  children: React.ReactNode;
}

function ModalRoot({ open, onOpenChange, children }: ModalRootProps) {
  return (
    <ModalContext value={{ onClose: () => onOpenChange(false) }}>
      <DialogPrimitive.Root open={open} onOpenChange={onOpenChange}>
        {children}
      </DialogPrimitive.Root>
    </ModalContext>
  );
}

// Content wrapper
interface ModalContentProps {
  children: React.ReactNode;
  className?: string;
  size?: 'sm' | 'md' | 'lg' | 'xl';
}

function ModalContent({ children, className, size = 'md' }: ModalContentProps) {
  const sizeClasses = {
    sm: 'max-w-sm',
    md: 'max-w-md',
    lg: 'max-w-lg',
    xl: 'max-w-xl',
  };

  return (
    <DialogPrimitive.Portal>
      <DialogPrimitive.Overlay className="fixed inset-0 bg-black/50" />
      <DialogPrimitive.Content
        className={cn(
          'fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2',
          'bg-white rounded-lg shadow-xl',
          'w-full p-6',
          sizeClasses[size],
          className
        )}
      >
        {children}
      </DialogPrimitive.Content>
    </DialogPrimitive.Portal>
  );
}

// Header
interface ModalHeaderProps {
  children: React.ReactNode;
  icon?: React.ReactNode;
  showCloseButton?: boolean;
}

function ModalHeader({ children, icon, showCloseButton = true }: ModalHeaderProps) {
  const { onClose } = useModal();

  return (
    <div className="flex items-start justify-between mb-4">
      <div className="flex items-center gap-3">
        {icon && <div className="flex-shrink-0">{icon}</div>}
        <div>{children}</div>
      </div>
      {showCloseButton && (
        <button onClick={onClose} className="text-gray-400 hover:text-gray-600">
          <X className="h-5 w-5" />
        </button>
      )}
    </div>
  );
}

// Title
function ModalTitle({ children, className }: { children: React.ReactNode; className?: string }) {
  return (
    <DialogPrimitive.Title className={cn('text-lg font-semibold', className)}>
      {children}
    </DialogPrimitive.Title>
  );
}

// Description
function ModalDescription({ children, className }: { children: React.ReactNode; className?: string }) {
  return (
    <DialogPrimitive.Description className={cn('text-sm text-gray-500', className)}>
      {children}
    </DialogPrimitive.Description>
  );
}

// Body
function ModalBody({ children, className }: { children: React.ReactNode; className?: string }) {
  return <div className={cn('py-4', className)}>{children}</div>;
}

// Footer
function ModalFooter({ children, className }: { children: React.ReactNode; className?: string }) {
  return (
    <div className={cn('flex items-center justify-end gap-3 pt-4 border-t', className)}>
      {children}
    </div>
  );
}

// Close button (uses context)
function ModalClose({ children }: { children: React.ReactNode }) {
  const { onClose } = useModal();
  return (
    <button onClick={onClose} type="button">
      {children}
    </button>
  );
}

// Export as compound component
export const Modal = {
  Root: ModalRoot,
  Content: ModalContent,
  Header: ModalHeader,
  Title: ModalTitle,
  Description: ModalDescription,
  Body: ModalBody,
  Footer: ModalFooter,
  Close: ModalClose,
};
```

### 4.2 Example Modal Usage

```typescript
import { Modal } from '@workspace/ui';
import { Globe } from 'lucide-react';

function CreateDomainModal({ open, onOpenChange }: CreateDomainModalProps) {
  const { state, actions } = useDomainActions();

  return (
    <Modal.Root open={open} onOpenChange={onOpenChange}>
      <Modal.Content size="md">
        <Modal.Header icon={<Globe className="h-6 w-6 text-blue-500" />}>
          <Modal.Title>Add New Domain</Modal.Title>
          <Modal.Description>
            Add a domain to start recording user sessions
          </Modal.Description>
        </Modal.Header>

        <Modal.Body>
          <form onSubmit={handleSubmit}>
            <DomainInput
              value={domainUrl}
              onChange={setDomainUrl}
              placeholder="example.com"
            />
          </form>
        </Modal.Body>

        <Modal.Footer>
          <Modal.Close>
            <Button variant="secondary">Cancel</Button>
          </Modal.Close>
          <Button onClick={handleCreate} loading={state.loading}>
            Add Domain
          </Button>
        </Modal.Footer>
      </Modal.Content>
    </Modal.Root>
  );
}
```

---

## 5. Consolidate Player State

### 5.1 Create Player State Machine

**New File:** `apps/dash/src/components/player/hooks/usePlayerState.ts`

```typescript
import { useReducer, useCallback } from 'react';

// Define all possible player states
type PlayerState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'ready'; duration: number }
  | { status: 'playing'; currentTime: number; duration: number }
  | { status: 'paused'; currentTime: number; duration: number }
  | { status: 'error'; error: Error };

// Actions
type PlayerAction =
  | { type: 'LOAD_START' }
  | { type: 'LOADED'; duration: number }
  | { type: 'PLAY' }
  | { type: 'PAUSE' }
  | { type: 'TIME_UPDATE'; currentTime: number }
  | { type: 'ERROR'; error: Error }
  | { type: 'RESET' };

function playerReducer(state: PlayerState, action: PlayerAction): PlayerState {
  switch (action.type) {
    case 'LOAD_START':
      return { status: 'loading' };

    case 'LOADED':
      return { status: 'ready', duration: action.duration };

    case 'PLAY':
      if (state.status === 'ready') {
        return { status: 'playing', currentTime: 0, duration: state.duration };
      }
      if (state.status === 'paused') {
        return { status: 'playing', currentTime: state.currentTime, duration: state.duration };
      }
      return state;

    case 'PAUSE':
      if (state.status === 'playing') {
        return { status: 'paused', currentTime: state.currentTime, duration: state.duration };
      }
      return state;

    case 'TIME_UPDATE':
      if (state.status === 'playing' || state.status === 'paused') {
        return { ...state, currentTime: action.currentTime };
      }
      return state;

    case 'ERROR':
      return { status: 'error', error: action.error };

    case 'RESET':
      return { status: 'idle' };

    default:
      return state;
  }
}

export function usePlayerState() {
  const [state, dispatch] = useReducer(playerReducer, { status: 'idle' });

  const actions = {
    loadStart: useCallback(() => dispatch({ type: 'LOAD_START' }), []),
    loaded: useCallback((duration: number) => dispatch({ type: 'LOADED', duration }), []),
    play: useCallback(() => dispatch({ type: 'PLAY' }), []),
    pause: useCallback(() => dispatch({ type: 'PAUSE' }), []),
    timeUpdate: useCallback((currentTime: number) => dispatch({ type: 'TIME_UPDATE', currentTime }), []),
    error: useCallback((error: Error) => dispatch({ type: 'ERROR', error }), []),
    reset: useCallback(() => dispatch({ type: 'RESET' }), []),
  };

  // Derived state helpers
  const isIdle = state.status === 'idle';
  const isLoading = state.status === 'loading';
  const isPlaying = state.status === 'playing';
  const isPaused = state.status === 'paused';
  const hasError = state.status === 'error';
  const isReady = state.status !== 'idle' && state.status !== 'loading' && state.status !== 'error';

  return {
    state,
    actions,
    isIdle,
    isLoading,
    isPlaying,
    isPaused,
    hasError,
    isReady,
  };
}
```

---

## 6. Extract Reusable Hooks

### 6.1 Create useExpandable Hook

**New File:** `apps/dash/src/hooks/use-expandable.ts`

```typescript
import { useState, useCallback } from 'react';

interface UseExpandableOptions {
  defaultExpanded?: boolean;
  onExpand?: () => void;
  onCollapse?: () => void;
}

interface UseExpandableReturn {
  isExpanded: boolean;
  toggle: () => void;
  expand: () => void;
  collapse: () => void;
}

export function useExpandable(options: UseExpandableOptions = {}): UseExpandableReturn {
  const { defaultExpanded = false, onExpand, onCollapse } = options;
  const [isExpanded, setIsExpanded] = useState(defaultExpanded);

  const expand = useCallback(() => {
    setIsExpanded(true);
    onExpand?.();
  }, [onExpand]);

  const collapse = useCallback(() => {
    setIsExpanded(false);
    onCollapse?.();
  }, [onCollapse]);

  const toggle = useCallback(() => {
    setIsExpanded((prev) => {
      const next = !prev;
      if (next) {
        onExpand?.();
      } else {
        onCollapse?.();
      }
      return next;
    });
  }, [onExpand, onCollapse]);

  return { isExpanded, toggle, expand, collapse };
}
```

### 6.2 Update Log Entry Components

**File:** `apps/dash/src/components/recordings/network-entry.tsx`

```typescript
// Before
const [isExpanded, setIsExpanded] = useState(false);
const [showDetails, setShowDetails] = useState(false);

// After
import { useExpandable } from '@/hooks/use-expandable';

const { isExpanded, toggle: toggleExpanded } = useExpandable();
const { isExpanded: showDetails, toggle: toggleDetails } = useExpandable();
```

Apply the same pattern to:
- `console-log-entry.tsx`
- `backend-log-entry.tsx`

---

## Implementation Checklist

### Phase 1: Quick Wins (1-2 days)
- [ ] Migrate URLInput forwardRef
- [ ] Migrate DomainInput forwardRef
- [ ] Create useExpandable hook
- [ ] Update log entry components

### Phase 2: State Improvements (3-5 days)
- [ ] Create DomainActionsContext
- [ ] Refactor CreateDomainModal
- [ ] Refactor DomainConfigureModal
- [ ] Create usePlayerState hook

### Phase 3: Component Architecture (1-2 weeks)
- [ ] Create Modal compound component
- [ ] Refactor StreamControls with status enum
- [ ] Refactor VideoOverlay variants
- [ ] Create DinoError variants

### Phase 4: Standardization (2-3 weeks)
- [ ] Apply Modal pattern to all 37 modals
- [ ] Document component patterns
- [ ] Create Storybook stories for new patterns

---

*Implementation guide based on Vercel Composition Patterns Audit*
