# Dash App - Vercel Composition Patterns Audit

**App:** apps/dash
**Framework:** Next.js 15.4.5 + React 19.1.1
**Date:** January 30, 2026
**Auditor:** Claude Code

---

## Executive Summary

The Dash app is a Next.js application with 130+ components, 20+ custom hooks, and Zustand-based state management. This audit identifies specific composition pattern issues and provides actionable recommendations.

| Category | Score | Issues Found |
|----------|-------|--------------|
| Boolean Props | 6/10 | 106+ components with boolean props |
| forwardRef Usage | 9/10 | Only 2 instances (good) |
| useContext Usage | 10/10 | None found (uses Zustand) |
| Prop Complexity | 5/10 | 10+ components with 7+ props |
| Callback Props | 6/10 | Heavy callback passing patterns |
| State Lifting | 7/10 | Some opportunities for improvement |

---

## 1. forwardRef Usage (React 19 Migration Required)

**Total Instances: 2**

### 1.1 URLInput Component

**File:** `apps/dash/src/components/url-input.tsx`
**Line:** 49

```typescript
// CURRENT (React 18 pattern)
export const URLInput = React.forwardRef<HTMLInputElement, URLInputProps>(
  (
    { value, onChange, placeholder, className, error, id, onKeyDown, disabled, onBlur },
    ref
  ) => {
    // ...
  }
);
```

**RECOMMENDED (React 19 pattern):**
```typescript
export function URLInput({
  value,
  onChange,
  placeholder,
  className,
  error,
  id,
  onKeyDown,
  disabled,
  onBlur,
  ref,
}: URLInputProps & { ref?: React.Ref<HTMLInputElement> }) {
  // ...
}
```

### 1.2 DomainInput Component

**File:** `apps/dash/src/components/domain-input.tsx`
**Line:** 46

```typescript
// CURRENT (React 18 pattern)
export const DomainInput = React.forwardRef<HTMLInputElement, DomainInputProps>(
  (
    { value, onChange, placeholder, className, error, id, onKeyDown, disabled, onBlur },
    ref
  ) => {
    // ...
  }
);
```

**RECOMMENDED (React 19 pattern):**
```typescript
export function DomainInput({
  value,
  onChange,
  placeholder,
  className,
  error,
  id,
  onKeyDown,
  disabled,
  onBlur,
  ref,
}: DomainInputProps & { ref?: React.Ref<HTMLInputElement> }) {
  // ...
}
```

---

## 2. Boolean Props Analysis

**Total Components with Boolean Props: 106+**

### 2.1 High-Impact Boolean Props (Multiple booleans in one component)

| Component | File | Boolean Props | Count | Severity |
|-----------|------|---------------|-------|----------|
| VideoOverlayProps | `player/components/VideoOverlay.tsx` | `hasStarted`, `hasError` | 2 | 🟡 Medium |
| StreamControlsProps | `player/StreamControls.tsx` | `hasStarted`, `showControls`, `hasError` | 3 | 🔴 High |
| PlanHeaderProps | `settings/workspace-plan-section.tsx` | `isUrgent`, `hasPlanId` | 2 | 🟡 Medium |
| BillingActionButtonProps | `settings/workspace-plan-section.tsx` | `hasPlanId`, `isUrgent` | 2 | 🟡 Medium |
| LinkPreviewProps | `links/link-preview.tsx` | `isReusable`, `canDelete` | 2 | 🟡 Medium |
| EmailInputAreaProps | `modals/invite-modal.tsx` | `hasError` | 1 | 🟢 Low |
| DinoErrorProps | `shared/dino-error.tsx` | `showRetry`, `showGoBack` | 2 | 🟡 Medium |

### 2.2 Loading State Boolean Props

These are acceptable patterns for async states:

| Component | File | Prop |
|-----------|------|------|
| DeleteConfirmationModalProps | `modals/delete-confirmation-modal.tsx:19` | `isLoading` |
| StatsTabProps | `admin/workspace-detail/stats-tab.tsx:11` | `isLoading` |
| OverviewTabProps | `admin/workspace-detail/overview-tab.tsx:22` | `isLoading` |
| InviteLinkSectionProps | `modals/invite-modal.tsx:142` | `isLoading` |
| LinksTabProps | `links/links-tab.tsx:7` | `isLoading` |
| PaginatedLinksTabProps | `links/paginated-links-tab.tsx:13` | `isLoading` |
| LeaveWorkspaceModalProps | `modals/leave-workspace-modal.tsx:22` | `isLoading` |
| ClearCacheModalProps | `modals/clear-cache-modal.tsx:20` | `isLoading` |
| BillingEmailVerificationModalProps | `modals/billing-email-verification-modal.tsx:29` | `isLoading` |
| DeleteAccountModalProps | `modals/delete-account-modal.tsx:17` | `isLoading` |
| TeamMemberActionModalProps | `modals/team-member-action-modal.tsx:26` | `isPending` |

### 2.3 Recommendations for Boolean Props

**A. StreamControls - Create Explicit Variants**

```typescript
// CURRENT
<StreamControls
  hasStarted={hasStarted}
  showControls={showControls}
  hasError={hasError}
/>

// RECOMMENDED - Use discriminated union
type PlayerState =
  | { status: 'idle' }
  | { status: 'playing'; showControls: boolean }
  | { status: 'error'; errorMessage: string };

<StreamControls state={playerState} />
```

**B. DinoError - Create Explicit Variants**

```typescript
// CURRENT
<DinoError showRetry={true} showGoBack={false} />

// RECOMMENDED
<DinoError.WithRetry onRetry={handleRetry} />
<DinoError.WithGoBack onGoBack={handleGoBack} />
<DinoError.WithBoth onRetry={handleRetry} onGoBack={handleGoBack} />
```

---

## 3. High-Prop Components (7+ Props)

**Total: 10+ components identified**

### 3.1 CreateDomainModalProps (10 props) 🔴

**File:** `apps/dash/src/components/modals/create-domain-modal.tsx`
**Lines:** 42-53

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

**RECOMMENDED - Extract to Provider:**

```typescript
// Create a DomainActionsContext
interface DomainActionsContextValue {
  state: {
    loading: boolean;
    savingSettings: boolean;
  };
  actions: {
    create: (domainUrl: string) => Promise<CreatedDomain | null>;
    startVerification: (domainId: string) => Promise<string | null>;
    pollVerification: (domainId: string) => Promise<boolean>;
    fetchSettings: (domainId: string) => Promise<DomainRecordingPreferences>;
    saveSettings: (domainId: string, settings: DomainRecordingPreferences) => void;
  };
  meta: {
    workspaceId: string;
  };
}

// Simplified modal props
interface CreateDomainModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
}

// Usage
<DomainActionsProvider workspaceId={workspaceId}>
  <CreateDomainModal open={open} onOpenChange={setOpen} />
</DomainActionsProvider>
```

### 3.2 DomainConfigureModalProps (10 props) 🔴

**File:** `apps/dash/src/components/modals/domain-configure-modal.tsx`
**Lines:** 26-37

```typescript
interface DomainConfigureModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  domainUrl: string;
  domainId: string;
  verified: boolean;
  settings: DomainRecordingPreferences;
  onSave: (settings: DomainRecordingPreferences) => void;
  onVerify: () => void;
  onSendInstructions: () => void;
  loading?: boolean;
}
```

**RECOMMENDED - Group related props:**

```typescript
interface DomainConfigureModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  domain: {
    id: string;
    url: string;
    verified: boolean;
    settings: DomainRecordingPreferences;
  };
  actions: {
    onSave: (settings: DomainRecordingPreferences) => void;
    onVerify: () => void;
    onSendInstructions: () => void;
  };
  loading?: boolean;
}
```

### 3.3 BillingDetailsModalProps (10+ props) 🔴

**File:** `apps/dash/src/components/modals/billing-details-modal.tsx`
**Lines:** 16-31

Complex nested subscription object with 9+ properties.

### 3.4 DeleteConfirmationModalProps (8 props) 🟡

**File:** `apps/dash/src/components/modals/delete-confirmation-modal.tsx`
**Lines:** 13-28

```typescript
interface DeleteConfirmationModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  itemName: string;
  itemType: string;
  onConfirm: () => void;
  isLoading?: boolean;
  description?: string;
  counts?: { /* nested */ };
}
```

### 3.5 DomainModalsManagerProps (8+ props) 🟡

**File:** `apps/dash/src/components/settings/domain-modals-manager.tsx`
**Lines:** 20-28+

Multiple modal state props passed through.

### 3.6 PlanHeaderProps (7 props) 🟡

**File:** `apps/dash/src/components/settings/workspace-plan-section.tsx`
**Lines:** 63-71

```typescript
interface PlanHeaderProps {
  planName: string;
  statusMessage: string;
  isUrgent: boolean;
  hasPlanId: boolean;
  accessStatus: string;
  onShowBillingDetails: () => void;
  onShowPlanPortal: () => void;
}
```

---

## 4. Callback Props Analysis

**Total Components with Multiple Callbacks: 20+**

### 4.1 Heavy Callback Passing (4+ callbacks)

| Component | File | Callbacks | Count |
|-----------|------|-----------|-------|
| CreateDomainModalProps | `modals/create-domain-modal.tsx` | onOpenChange, onCreateDomain, onStartVerification, onPollVerification, onFetchSettings, onSaveSettings | 6 |
| DomainConfigureModalProps | `modals/domain-configure-modal.tsx` | onOpenChange, onSave, onVerify, onSendInstructions | 4 |
| LinkPreviewProps | `links/link-preview.tsx` | onShare, onView, onEdit, onDelete | 4 |
| DomainModalsManagerProps | `settings/domain-modals-manager.tsx` | onCloseVerification, onCloseEdit, onStartVerify, + more | 8+ |
| DomainListProps | `settings/domain-list.tsx` | onConfigure, onEdit, onDelete | 3 |

### 4.2 Recommendations

**Use Actions Object Pattern:**

```typescript
// CURRENT
<LinkPreview
  onShare={handleShare}
  onView={handleView}
  onEdit={handleEdit}
  onDelete={handleDelete}
/>

// RECOMMENDED
<LinkPreview
  actions={{
    share: handleShare,
    view: handleView,
    edit: handleEdit,
    delete: handleDelete,
  }}
/>
```

---

## 5. State Management Analysis

### 5.1 Provider Architecture (Good Pattern ✅)

**File:** `apps/dash/src/components/shared/providers.tsx`

```
Providers (root)
├── QueryClientProvider (TanStack React Query)
├── NuqsAdapter (URL state)
├── SentryUserProvider (error tracking)
│   ├── ImpersonationBanner
│   └── WorkspaceProvider
│       └── useWorkspaceStore (Zustand)
└── Toaster (notifications)
```

### 5.2 Zustand Store (Good Pattern ✅)

**File:** `apps/dash/src/stores/workspace.ts`

```typescript
interface WorkspaceState {
  workspaceId: string | null;
  setWorkspace: (workspaceId: string) => void;
  clearWorkspace: () => void;
}
```

Uses devtools and persist middleware correctly.

### 5.3 Components with Multiple useState (Candidates for useReducer)

| Component | File | useState Count | Recommendation |
|-----------|------|----------------|----------------|
| ProjectsTab | `admin/workspace-detail/projects-tab.tsx` | 5 | Use useReducer |
| OverviewTab | `admin/workspace-detail/overview-tab.tsx` | 3+ | Consider extraction |
| MembersTab | `admin/workspace-detail/members-tab.tsx` | 3+ | Consider extraction |
| StreamPlayer | `player/index.tsx` | 4 | Already uses custom hook ✅ |
| LinkPreview | `links/link-preview.tsx` | 2 | Modal provider pattern |

### 5.4 State Lifting Opportunities

**A. Log Entry Expansion State**

Files affected:
- `recordings/network-entry.tsx` (isExpanded)
- `recordings/console-log-entry.tsx` (isExpanded, isObjectExpanded)
- `recordings/backend-log-entry.tsx` (isExpanded)

**RECOMMENDED:** Create `useExpandable` hook or lift to parent with controlled expansion.

**B. Modal States in LinkPreview**

```typescript
// CURRENT - State in component
const [isEditModalOpen, setIsEditModalOpen] = useState(false);
const [isShareModalOpen, setIsShareModalOpen] = useState(false);

// RECOMMENDED - Use modal provider
const { openModal } = useModalContext();
openModal('editLink', { linkId });
```

---

## 6. Custom Hooks Analysis

**Total Hooks: 20+**

### 6.1 Well-Designed Hooks ✅

| Hook | File | Pattern |
|------|------|---------|
| useWorkspace | `hooks/use-workspace.ts` | React Query wrapper |
| useWorkspaceInvite | `hooks/use-workspace-invite.ts` | Invite management |
| usePagination | `hooks/use-pagination.ts` | Pagination logic |
| useDebouncedValue | `hooks/use-debounced-value.ts` | Utility |
| useVerificationFlow | `hooks/use-verification-flow.ts` | Uses useReducer ✅ |
| useDomainSectionState | `hooks/use-domain-section-state.ts` | Uses useReducer ✅ |

### 6.2 Hooks Using useReducer (Good Pattern ✅)

1. **useVerificationFlow** (`hooks/use-verification-flow.ts:102`)
2. **useDomainSectionState** (`hooks/use-domain-section-state.ts:183`)

---

## 7. Modal Components Audit

**Total Modals: 37 files in `apps/dash/src/components/modals/`**

### 7.1 Modal Prop Patterns

| Modal | Props Count | Severity |
|-------|-------------|----------|
| CreateDomainModal | 10 | 🔴 High |
| DomainConfigureModal | 10 | 🔴 High |
| BillingDetailsModal | 10+ | 🔴 High |
| DeleteConfirmationModal | 8 | 🟡 Medium |
| PlanChangePreviewModal | 5+ | 🟡 Medium |
| InviteModal | 2 | 🟢 Low |
| LeaveWorkspaceModal | 5 | 🟡 Medium |
| EditLinkModal | 6 | 🟡 Medium |
| CreateLinkModal | 4 | 🟢 Low |
| CreateProjectModal | 3 | 🟢 Low |

### 7.2 Recommended Modal Pattern

Create a shared modal compound component:

```typescript
// packages/ui/src/components/modal.tsx
const Modal = {
  Root: ModalRoot,
  Header: ModalHeader,
  Title: ModalTitle,
  Description: ModalDescription,
  Body: ModalBody,
  Footer: ModalFooter,
  Actions: ModalActions,
  CloseButton: ModalCloseButton,
};

// Usage
<Modal.Root open={open} onOpenChange={onOpenChange}>
  <Modal.Header>
    <Modal.Title>Create Domain</Modal.Title>
    <Modal.Description>Add a new domain for recording</Modal.Description>
  </Modal.Header>
  <Modal.Body>
    {/* Form content */}
  </Modal.Body>
  <Modal.Footer>
    <Modal.Actions>
      <Button variant="secondary">Cancel</Button>
      <Button>Create</Button>
    </Modal.Actions>
  </Modal.Footer>
</Modal.Root>
```

---

## 8. Player Components Audit

**Directory:** `apps/dash/src/components/player/`

### 8.1 Component Structure (Good Pattern ✅)

```
player/
├── index.tsx (StreamPlayer)
├── StreamControls.tsx
├── hooks/
│   └── useVideoPlayer.ts
├── components/
│   ├── VideoOverlay.tsx
│   ├── VideoProgressBar.tsx
│   ├── PlaybackControls.tsx
│   ├── QualityControl.tsx
│   ├── SpeedControl.tsx
│   ├── VolumeControlInline.tsx
│   └── ControlDropdown.tsx
└── types.ts
```

### 8.2 Issues Found

**A. StreamControls Boolean Props (3 booleans)**

```typescript
interface StreamControlsProps {
  hasStarted?: boolean;  // ⚠️
  showControls?: boolean;  // ⚠️
  hasError?: boolean;  // ⚠️
}
```

**B. VideoOverlay Boolean Props (2 booleans)**

```typescript
interface VideoOverlayProps {
  hasStarted: boolean;  // ⚠️
  hasError: boolean;  // ⚠️
}
```

### 8.3 Recommendations

```typescript
// Use player state enum
type PlayerStatus = 'idle' | 'loading' | 'playing' | 'paused' | 'error';

interface StreamControlsProps {
  player: Player | null;
  markers?: VideoMarker[];
  onFullscreen?: () => void;
  duration?: number;
  status: PlayerStatus;  // Single state instead of multiple booleans
  showControls?: boolean;  // Keep this one - it's UI visibility
}
```

---

## 9. Priority Action Items

### Immediate (P0)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 1 | Migrate URLInput forwardRef | 1 file | Low |
| 2 | Migrate DomainInput forwardRef | 1 file | Low |
| 3 | Extract DomainActionsProvider | 1 new file | Medium |

### Short-term (P1)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 4 | Refactor StreamControls boolean props | 2 files | Medium |
| 5 | Group DomainConfigureModal props | 1 file | Low |
| 6 | Create Modal compound component | 1 new file | Medium |
| 7 | Extract useExpandable hook for log entries | 3 files | Low |

### Long-term (P2)

| # | Action | Files | Effort |
|---|--------|-------|--------|
| 8 | Refactor ProjectsTab to useReducer | 1 file | Medium |
| 9 | Create ModalProvider for LinkPreview | 2 files | Medium |
| 10 | Standardize all modal prop patterns | 37 files | High |

---

## 10. Files Reference

### Components with Issues

```
apps/dash/src/components/
├── domain-input.tsx (forwardRef)
├── url-input.tsx (forwardRef)
├── modals/
│   ├── create-domain-modal.tsx (10 props)
│   ├── domain-configure-modal.tsx (10 props)
│   ├── billing-details-modal.tsx (10+ props)
│   └── delete-confirmation-modal.tsx (8 props)
├── player/
│   ├── StreamControls.tsx (3 boolean props)
│   └── components/VideoOverlay.tsx (2 boolean props)
├── settings/
│   ├── workspace-plan-section.tsx (boolean props)
│   └── domain-modals-manager.tsx (8+ props)
├── links/
│   └── link-preview.tsx (4 callbacks, modal state)
└── shared/
    └── dino-error.tsx (2 boolean props)
```

### Well-Designed Components (Reference)

```
apps/dash/src/components/
├── shared/providers.tsx (good provider composition)
├── modals/invite-modal.tsx (clean 2-prop interface)
└── player/hooks/useVideoPlayer.ts (good state management)

apps/dash/src/hooks/
├── use-verification-flow.ts (uses useReducer)
└── use-domain-section-state.ts (uses useReducer)

apps/dash/src/stores/
└── workspace.ts (good Zustand pattern)
```

---

*Generated by Claude Code using Vercel Composition Patterns Skill*
