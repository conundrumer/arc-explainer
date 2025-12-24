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

### Phase 1: Setup & Install
- [x] Review re-arc library documentation (https://github.com/michaelhodel/re-arc)
- [x] Understand generation parameters and API
- [x] Get user approval on key decisions (APPROVED: 400 tasks, 8-char hex IDs, diff_lb/diff_ub params)
- [ ] Install re-arc library (`pip install` or clone repo)
- [ ] Test re-arc generation with sample seed
- [ ] Verify XOR seed approach produces deterministic results

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

## Technical Implementation Details

### 1. Task ID Format (DECIDED)
**Using:** 8-character hex strings (e.g., `a1b2c3d4`)
- Matches ARC-AGI convention
- Easy to type/share
- Sufficient entropy for seed generation
- Human-readable in URLs/debugging

### 2. Seed & Task ID Generation (DECIDED)
**XOR Technique:** Generate 399 random IDs, then compute the 400th as XOR of all 399 + seed.
This ensures XORing all 400 IDs recovers the original seed.

**Implementation:**
```typescript
// Generate 400 task IDs from seed using XOR trick
function generateTaskIds(seed: number): string[] {
  const rng = seedRandom(seed); // Use seeded PRNG (e.g., seedrandom library)
  const ids: string[] = [];

  // Generate first 399 IDs randomly
  for (let i = 0; i < 399; i++) {
    const randomHex = Math.floor(rng() * 0xFFFFFFFF).toString(16).padStart(8, '0');
    ids.push(randomHex);
  }

  // 400th ID = XOR of all 399 IDs plus seed
  let xor = seed;
  for (const id of ids) {
    xor ^= parseInt(id, 16);
  }
  ids.push(xor.toString(16).padStart(8, '0'));

  return ids; // 400 IDs total
}

// XOR all 400 task IDs to recover the seed
function taskIdsToSeed(taskIds: string[]): number {
  let seed = 0;
  for (const id of taskIds) {
    seed ^= parseInt(id, 16);
  }
  return seed >>> 0; // Unsigned 32-bit
}
// Proof: id[0] ^ ... ^ id[398] ^ (id[0] ^ ... ^ id[398] ^ seed) = seed


// Create 1-to-1 mapping between original and generated IDs
function createTaskMapping(generatedIds: string[]): Map<string, string> {
  const originalIds = getOriginalArcTaskIds(); // Read from data/training/*.json
  const sortedOriginal = originalIds.sort(); // Alphabetical sort

  const mapping = new Map<string, string>();
  for (let i = 0; i < 400; i++) {
    mapping.set(generatedIds[i], sortedOriginal[i]); // generated → original
  }
  return mapping;
}
```

### 3. Train/Test Matching (DECIDED)
**Approach:**
- Read original task from `data/training/{original_id}.json`
- Get exact train/test counts (e.g., 5 train, 1 test)
- Call re-arc generator for that specific original task
- Generate **same number** of train and test examples
- No random splitting needed - exact 1-to-1 structure match

### 4. Verification Scoring (DECIDED)
**Match ARC-AGI scoring (see CLAUDE.md):**
- Task score = 1.0 if ANY attempt correct, 0.0 otherwise
- Overall score = (solved tasks) / (total tasks)
- Per-attempt tracking for detailed feedback

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

## Decisions (User Approved)

1. **Dataset size:** 400 tasks (matching ARC-AGI training set)
2. **Task ID format:** 8-char hex strings (e.g., `a1b2c3d4`) - same as ARC-AGI
3. **Train/test distribution:** Match ARC-AGI statistics:
   - Train examples: 2-5 per task (avg ~3)
   - Test examples: 1 per task
4. **Re-ARC parameters to expose:**
   - `diff_lb` (difficulty lower bound: 0-1)
   - `diff_ub` (difficulty upper bound: 0-1)
   - Generate ~10 examples per task, then randomly split into train/test
5. **Security:** None - plain XOR is fine for this use case
6. **Navigation:** Add to "Misc" dropdown menu (niche community tool)

## Re-ARC Integration Details

**Python API:**
```python
from main import generate_dataset

generate_dataset(
    path='output_dir',
    seed=123456789,      # Derived from XOR of task IDs
    n_examples=10,       # Generate 10 examples per task
    diff_lb=0.0,         # Difficulty lower bound
    diff_ub=1.0          # Difficulty upper bound
)
```

**Our approach (FINAL):**
1. User provides initial seed (or we generate one random seed)
2. PRNG generates **399** random task IDs (8-char hex)
3. Compute **400th ID** = `id[0] ^ id[1] ^ ... ^ id[398] ^ seed`
4. Sort original ARC-AGI task IDs alphabetically (007bbfb7, ..., ff805c23)
5. Create 1-to-1 mapping: `generated_ids[i] ↔ sorted_original[i]`
6. For each mapping:
   - Read original task structure (e.g., `007bbfb7.json`: 5 train, 1 test)
   - Use re-arc generator for that specific task
   - Generate same number of train/test examples using re-arc
   - Assign to the generated task ID
7. Output: `{[generated_id]: {train: [...re-arc variations...], test: [...]}, ...}`

**Verification approach:**
1. Extract all 400 task IDs from submission
2. XOR all 400 IDs together → recovers original seed (XOR trick!)
3. Regenerate 400 IDs using that seed (deterministic PRNG + XOR)
4. Recreate the 1-to-1 mapping with sorted original tasks
5. Regenerate dataset using re-arc with the same seed (deterministic)
6. Verify submission attempts against regenerated ground truth

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
