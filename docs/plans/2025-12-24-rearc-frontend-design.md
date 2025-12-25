# Re-ARC Frontend Design

**Author:** Claude Code using Sonnet 4.5
**Date:** 2025-12-25
**Purpose:** Self-service page for generating and verifying re-ARC datasets

## Page: `/rearc`

### Single-page design with 4 sections:
1. Header - What is this?
2. How it Works - Instructions
3. Generate Dataset - Create challenge
4. Verify Solution - Test submission

---

## Section 1: Header

**Title:** Re-ARC Challenge Verifier

**Subtitle:** Generate and verify ARC puzzle datasets

---

## Section 2: How It Works

Collapsible section with steps:

1. **Generate**: Click "Generate Dataset" to create unique ARC puzzles
2. **Download**: Save the challenges file
3. **Verify**: Upload your solutions to get your score

**Technical note (collapsible detail):**
The task IDs encode a seed. Verification regenerates the same puzzles from that seed to check your solutions.

---

## Section 3: Generate Dataset

**Card with:**
- Title: "Generate Challenge Dataset"
- Description: "Creates unique ARC puzzles selected by difficulty"
- Warning: "Each generation is unique. Save the file - you can't regenerate the same one!"
- Button: "Generate Dataset"

**Generation states:**
1. **Idle**: Show generate button
2. **In Progress**:
   - Hide button
   - Show progress bar
   - Display progress updates from API
3. **Complete**:
   - Show download button for challenges.json
   - No other information displayed

**API requirement:**
- Generation endpoint must support progress reporting
- Stream progress events during generation

---

## Section 4: Verify Solution

**Card with:**
- Title: "Verify Your Submission"
- Description: "Upload your submission to check your score"

**Submission format guide (collapsible):**
Show expected JSON structure

**Upload interface:**
- Drag-and-drop zone
- "Drop submission.json here" text
- File picker button as fallback

**Verification states:**
1. **Idle**: Show drop zone
2. **Validating Structure**:
   - Quick check of JSON format
   - No progress bar (fast)
   - Show specific structural errors if invalid
3. **Regenerating Dataset**:
   - Hide upload interface
   - Show progress bar
   - Display progress from API
4. **Scoring**:
   - Continue progress bar
   - "Comparing solutions..."
5. **Complete**:
   - Display score (percentage)
   - Display solved/total count
   - If message decoded, show it

**API requirement:**
- Verification endpoint must support progress reporting
- Stream events for: validation → regeneration → scoring

---

## Progress Bar Implementation

**Required for both generation and verification**

Progress updates from API:
- Generation: per-task completion (e.g., "Generating task 47/400")
- Verification: stages (validating → regenerating → scoring)

Progress bar shows:
- Current percentage
- Current stage description
- No time estimates

---

## Error Handling

**Generation errors:**
- Display error message inline
- Show "Try Again" button

**Verification errors:**

1. **Invalid JSON**
   - "Invalid JSON format. Please check your file syntax."

2. **Wrong structure**
   - Be specific about what's wrong:
     - "Missing required field: attempt_1"
     - "Expected array of attempts, got object"
     - "Task [task_id] has 3 attempts, expected 2"

3. **Seed recovery failure**
   - "Could not verify submission. This may mean:"
     - "- Wrong dataset (task IDs don't match)"
     - "- Corrupted submission file"
     - "- Modified task IDs"

4. **Grid format errors**
   - "Invalid grid format for task [task_id], attempt [N]"
   - "Expected 2D array of integers"

All errors:
- Display inline in the verify section
- Keep upload interface visible
- Allow immediate retry

---

## Layout Structure

```
Header
  ↓
How It Works (collapsible)
  ↓
Generate Dataset
  - Generate button (or progress bar)
  - Download button (when complete)
  ↓
Verify Solution
  - Upload interface (or progress bar)
  - Results (when complete)
```

---

## Component Breakdown

**Needed components:**
- Card containers (shadcn/ui)
- Collapsible sections (shadcn/ui)
- Progress bars (shadcn/ui)
- File upload with drag-and-drop
- Alert/error display (shadcn/ui)

**NOT needed:**
- Toast notifications
- Navigation between states
- Separate pages
- Statistics display
- Marketing copy

---

## API Requirements

### Generation Endpoint

**Request:** `POST /api/rearc/generate`
- No body

**Response:** Server-Sent Events (SSE) stream
```
event: progress
data: {"stage": "generating", "current": 47, "total": 400}

event: progress
data: {"stage": "generating", "current": 48, "total": 400}

...

event: complete
data: {<Dataset JSON>}
```

### Verification Endpoint

**Request:** `POST /api/rearc/verify`
- Body: `{ submission: Submission }`

**Response:** Server-Sent Events (SSE) stream
```
event: progress
data: {"stage": "validating"}

event: progress
data: {"stage": "regenerating", "current": 47, "total": 400}

event: progress
data: {"stage": "scoring"}

event: complete
data: {"score": 0.875, "solvedTasks": 350, "totalTasks": 400, "message": "..."}
```

**Errors:**
```
event: error
data: {"message": "Invalid JSON format. Please check your file syntax."}
```

---

## User Experience Flow

**Happy path:**
1. User arrives at page
2. Clicks "Generate Dataset"
3. Watches progress bar fill
4. Downloads challenges.json
5. (Solves puzzles offline)
6. Returns to page
7. Drags submission.json onto upload zone
8. Watches progress through validation → regeneration → scoring
9. Sees final score

**Error path:**
1. User uploads invalid submission
2. Sees specific error about structure
3. Fixes submission
4. Uploads again immediately
5. Success

---

## Open Questions

1. Should we show task count anywhere? (Currently removed all numbers)
2. Generation time estimate to show user?
3. Maximum file size for submissions?
4. Should decode message be prominent or subtle?
5. Download filename: `challenges.json` or include timestamp?

