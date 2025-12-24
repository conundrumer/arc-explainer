# Re-ARC Dataset Generation & Verification Interface

**Author:** Claude Code using Sonnet 4.5
**Date:** 2025-12-24
**Purpose:** Plan for implementing a stateless re-arc dataset generation and verification interface

## Overview

Create a web interface that allows users to:
1. Generate re-arc datasets on demand
2. Submit solutions for verification
3. Verification works via XOR of task IDs to regenerate the seed (stateless, no DB needed)

## Data Structures

```typescript
// Core types
type IOGrid = {
  input: number[][];
  output: number[][];
};

type Challenge = {
  train: IOGrid[];
  test: IOGrid[];
};

type ChallengeDataset = {
  [taskId: string]: Challenge;
};

type Submission = {
  [taskId: string]: {
    attempt_1: number[][];
    attempt_2: number[][];
  }[];
};
```

## Architecture

### Frontend Components

**1. ReArcGeneratorPage** (`client/src/pages/ReArcGeneratorPage.tsx`)
- Input controls for generation parameters:
  - Number of tasks (e.g., 5, 10, 20)
  - Difficulty level (if re-arc supports this)
  - Any other re-arc generation parameters
- "Generate Dataset" button
- Display generated task IDs
- Download button for challenges.json
- Show the derived seed (for transparency/debugging)

**2. ReArcVerifierPage** (`client/src/pages/ReArcVerifierPage.tsx`)
- File upload for submission.json
- "Verify Submission" button
- Results display:
  - Per-task scoring
  - Overall score
  - Which attempts were correct
  - Visual grid comparison (optional but nice)

### Backend Services

**1. Re-ARC Python Integration** (`server/services/reArc/`)

Files needed:
- `reArcService.ts` - Main service orchestrating Python calls
- `reArcGenerator.py` - Python script for dataset generation
- `reArcVerifier.py` - Python script for verification

**2. Seed Management** (`server/utils/reArcSeed.ts`)
```typescript
// Convert task IDs to seed via XOR
function taskIdsToSeed(taskIds: string[]): number;

// Generate task IDs from seed (for regeneration)
function seedToTaskIds(seed: number, count: number): string[];
```

**3. API Endpoints** (`server/controllers/reArcController.ts`)

```typescript
// POST /api/rearc/generate
// Body: { taskCount: number, difficulty?: string, ...otherParams }
// Response: { challenges: ChallengeDataset, seed: number, taskIds: string[] }

// POST /api/rearc/verify
// Body: { submission: Submission }
// Response: {
//   score: number,
//   taskResults: { [taskId: string]: TaskVerificationResult },
//   regeneratedSeed: number
// }
```

## Following Existing Patterns

**Controller Pattern (see `server/controllers/`):**
- Export functions wrapped with `asyncHandler` middleware
- Use `formatResponse` utility for consistent API responses
- Import from repositories/services, don't do DB queries directly

**Service Pattern (see `server/services/poetiq/`):**
- Main service file orchestrates Python subprocess calls
- Use `child_process.spawn` for Python scripts
- Handle stdout/stderr streaming for progress updates

**Frontend Patterns:**
- Use TanStack Query hooks for API calls (see existing pages)
- Reuse shadcn/ui components (Card, Button, Table, etc.)
- Follow PageLayout wrapper pattern for consistent styling
- Use Wouter for routing (`<Route path="/rearc/..." component={...} />`)

**Navigation (AppNavigation.tsx):**
- Add dropdown menu item or direct link
- Include icon, title, description
- Follow discriminated union pattern (NavLink | NavDropdown)

## Implementation Steps

### Phase 1: Research & Planning
- [ ] Review re-arc library documentation (https://github.com/michaelhodel/re-arc)
- [ ] Understand generation parameters and API
- [ ] Get user approval on key decisions (task IDs, dataset size, etc.)
- [ ] Verify XOR seed approach works with re-arc

### Phase 2: Core Infrastructure
- [ ] Create type definitions in `shared/types.ts`
- [ ] Implement XOR seed logic in `server/utils/reArcSeed.ts`
- [ ] Create Python wrapper scripts for re-arc library
- [ ] Implement `reArcService.ts` with generation and verification methods
- [ ] Test Python integration with simple script

### Phase 3: Backend API
- [ ] Create `reArcController.ts` with generation endpoint
- [ ] Create verification endpoint
- [ ] Add routes to `server/routes.ts` (follow existing pattern)
- [ ] Test with curl/Postman
- [ ] Add error handling and validation

### Phase 4: Frontend - Generator
- [ ] Create `ReArcGeneratorPage.tsx` using PageLayout
- [ ] Add form controls for generation parameters (use shadcn/ui)
- [ ] Create DatasetDownloadButton component
- [ ] Add route to `App.tsx`
- [ ] Test generation flow end-to-end

### Phase 5: Frontend - Verifier
- [ ] Create `ReArcVerifierPage.tsx`
- [ ] Create SubmissionUploader component (file input)
- [ ] Create VerificationResults component (table/cards)
- [ ] Add visual grid comparison (reuse existing ARC grid rendering)
- [ ] Add route to `App.tsx`
- [ ] Test verification flow end-to-end

### Phase 6: Navigation & Polish
- [ ] Add nav items to AppNavigation.tsx (decide: dropdown or direct links)
- [ ] Error handling for invalid submissions
- [ ] Loading states during generation/verification (use shadcn/ui Skeleton)
- [ ] Validation of submission format
- [ ] User documentation/help text
- [ ] Git commit with detailed message

## Technical Decisions to Make

### 1. Task ID Format
**Options:**
- Random hex strings (e.g., `a1b2c3d4`)
- UUIDs
- Sequential IDs with random prefix

**Recommendation:** Use short hex strings (8 chars) for readability
- Easy to type/share
- Sufficient entropy for seed generation
- Human-readable in URLs/debugging

### 2. XOR Seed Algorithm
**Considerations:**
- Task IDs need to be converted to numbers first
- Should we hash task IDs before XOR? (prevents manipulation)
- Endianness and bit width

**Proposed approach:**
```typescript
function taskIdsToSeed(taskIds: string[]): number {
  let seed = 0;
  for (const id of taskIds) {
    // Convert hex string to number, XOR accumulate
    const idNum = parseInt(id, 16);
    seed ^= idNum;
  }
  return seed >>> 0; // Ensure unsigned 32-bit
}
```

### 3. Re-ARC Generation Parameters
**Need to determine:**
- What parameters does re-arc support?
- Should users control these, or use sensible defaults?
- How many train/test pairs per task?

**Action:** Review re-arc docs and expose key parameters

### 4. Verification Scoring
**Questions:**
- Score per task? (0.0 to 1.0 if any attempt correct)
- Overall score? (average or sum?)
- Should we match ARC-AGI scoring exactly?

**Proposed:** Match ARC-AGI scoring (see CLAUDE.md):
- Task score = 1.0 if ANY attempt correct, 0.0 otherwise
- Overall score = (solved tasks) / (total tasks)

## Integration with Existing Webapp

### Files to Create/Modify

**Backend:**
```
server/
├── controllers/
│   └── reArcController.ts          # NEW - API endpoints
├── services/
│   └── reArc/
│       ├── reArcService.ts         # NEW - Main service
│       ├── reArcGenerator.py       # NEW - Python generation script
│       └── reArcVerifier.py        # NEW - Python verification script
├── utils/
│   └── reArcSeed.ts                # NEW - Seed XOR logic
└── routes.ts                        # MODIFY - Add re-arc routes
```

**Frontend:**
```
client/src/
├── pages/
│   ├── ReArcGeneratorPage.tsx      # NEW - Generation UI
│   └── ReArcVerifierPage.tsx       # NEW - Verification UI
├── components/
│   ├── reArc/
│   │   ├── DatasetDownloadButton.tsx   # NEW
│   │   ├── SubmissionUploader.tsx      # NEW
│   │   └── VerificationResults.tsx     # NEW
│   └── layout/
│       └── AppNavigation.tsx        # MODIFY - Add nav items
└── App.tsx                          # MODIFY - Add routes
```

**Shared:**
```
shared/
└── types.ts                         # MODIFY - Add ReARC types
```

## Open Questions

1. **Dataset size:** How many tasks should a default dataset have? (Suggest: 10)
2. **Re-ARC parameters:** What generation params should be exposed to users?
3. **Caching:** Should we cache generated datasets temporarily? (Probably no, given stateless design)
4. **Rate limiting:** Should we limit generation requests? (Depends on compute cost)
5. **Grid visualization:** Reuse existing ARC grid rendering, or build custom for re-arc?
6. **Download format:** Just JSON, or also support other formats?
7. **Reverse engineering prevention:** Should we add salt/HMAC to prevent seed guessing?

## Security Considerations

- Validate submission structure before processing
- Limit file upload size
- Rate limit generation/verification endpoints
- Sanitize task IDs (only allow hex chars)
- Prevent Python code injection in generation params

## User Flow Examples

### Generation Flow
1. User visits `/rearc/generate`
2. Selects "10 tasks"
3. Clicks "Generate Dataset"
4. System generates task IDs: `[a1b2c3d4, e5f6a7b8, ...]`
5. XOR → seed: `123456789`
6. Calls re-arc with seed
7. Returns challenges.json
8. User downloads and shares with solvers

### Verification Flow
1. User visits `/rearc/verify`
2. Uploads `submission.json`
3. System extracts task IDs from keys
4. XOR → seed: `123456789`
5. Regenerates dataset using seed
6. Compares submission attempts to ground truth
7. Displays:
   - Task a1b2c3d4: ✓ (attempt_2 correct)
   - Task e5f6a7b8: ✗ (both attempts wrong)
   - Overall: 1/10 = 0.1

## Next Steps

1. **Clarify re-arc integration details** - What's the Python API?
2. **Prototype seed XOR logic** - Verify it's deterministic
3. **Review re-arc documentation** - Understand generation parameters
4. **Decide on task ID format** - Get user approval
5. **Start with backend** - Python integration first, then REST API
6. **Build frontend** - Once backend is solid

## Notes

- This design is completely stateless (no DB needed) ✓
- Seed derivation from task IDs is deterministic ✓
- Users can share just the challenge file ✓
- Verification works from submission alone ✓
- Clean separation between generation and verification ✓

---

**Questions for User:**
1. How many tasks per dataset by default?
2. Should users control re-arc generation parameters, or use defaults?
3. Do you want visual grid comparison in verification results?
4. Should we add any salt/HMAC to prevent seed reverse engineering?
5. What re-arc generation parameters are most important to expose?
