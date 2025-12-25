# Re-ARC Frontend Design

**Author:** Claude Code using Sonnet 4.5
**Date:** 2025-12-25
**Purpose:** Self-service page for generating and verifying re-ARC datasets

## Page: `/rearc`

### Layout

Single-page design with clear sections:
1. **Header** - What is this?
2. **How it Works** - Quick explanation
3. **Generate Dataset** - Create challenge
4. **Verify Solution** - Test your submission

---

## Section 1: Header

```
┌─────────────────────────────────────────────────────┐
│  Re-ARC Challenge Verifier                         │
│                                                     │
│  Generate cryptographically-verified ARC datasets  │
│  No account needed • No database • No trust        │
└─────────────────────────────────────────────────────┘
```

**Copy:**
> **Re-ARC Challenge Verifier**
>
> Generate cryptographically-verified ARC puzzle datasets and verify solutions without any centralized infrastructure. The dataset itself contains all the information needed for verification.

---

## Section 2: How It Works

**Collapsible card** (shadcn/ui Collapsible)

**Title:** "How does this work?"

**Content:**
1. **Generate**: Click "Generate Dataset" to create 400 unique ARC puzzles
2. **Download**: Save the `challenges.json` file
3. **Solve**: Write code to solve the puzzles (2 attempts per test)
4. **Format**: Create `submission.json` with your solutions
5. **Verify**: Upload your submission to get your score

**Key insight box:**
> 🔐 **Cryptographic Verification**
>
> The task IDs in the dataset encode a secret seed. When you submit solutions, we regenerate the exact same puzzles from that seed and verify your answers. No server-side storage needed.

---

## Section 3: Generate Dataset

**Card with prominent CTA**

```typescript
<Card>
  <CardHeader>
    <CardTitle>Generate Challenge Dataset</CardTitle>
    <CardDescription>
      Creates 400 unique ARC puzzles selected by difficulty
    </CardDescription>
  </CardHeader>

  <CardContent>
    <div className="space-y-4">
      <Alert>
        <Info className="h-4 w-4" />
        <AlertDescription>
          Each generation creates a unique dataset. Save the file - you can't regenerate the same one!
        </AlertDescription>
      </Alert>

      <Button
        onClick={handleGenerate}
        disabled={isGenerating}
        size="lg"
        className="w-full"
      >
        {isGenerating ? (
          <>
            <Loader2 className="mr-2 h-4 w-4 animate-spin" />
            Generating...
          </>
        ) : (
          <>
            <Download className="mr-2 h-4 w-4" />
            Generate Dataset
          </>
        )}
      </Button>
    </div>
  </CardContent>

  {generatedDataset && (
    <CardFooter className="flex-col gap-2">
      <div className="w-full p-4 bg-muted rounded-md">
        <p className="text-sm font-mono">
          Generated {Object.keys(generatedDataset).length} tasks
        </p>
        {decodedMessage && (
          <p className="text-sm text-muted-foreground">
            Message: {decodedMessage}
          </p>
        )}
      </div>
      <Button
        onClick={handleDownload}
        variant="outline"
        className="w-full"
      >
        <FileDown className="mr-2 h-4 w-4" />
        Download challenges.json
      </Button>
    </CardFooter>
  )}
</Card>
```

---

## Section 4: Verify Solution

**Card with drag-and-drop + file picker**

```typescript
<Card>
  <CardHeader>
    <CardTitle>Verify Your Submission</CardTitle>
    <CardDescription>
      Upload your submission.json to check your score
    </CardDescription>
  </CardHeader>

  <CardContent>
    <div className="space-y-4">
      {/* Submission Format Guide */}
      <Collapsible>
        <CollapsibleTrigger className="flex items-center gap-2 text-sm">
          <ChevronRight className="h-4 w-4" />
          Expected format for submission.json
        </CollapsibleTrigger>
        <CollapsibleContent>
          <pre className="mt-2 p-4 bg-muted rounded-md text-xs">
{`{
  "task_id_1": [
    {
      "attempt_1": [[0,1],[1,0]],
      "attempt_2": [[1,0],[0,1]]
    }
  ],
  "task_id_2": [ ... ]
}`}
          </pre>
        </CollapsibleContent>
      </Collapsible>

      {/* Drag and Drop Zone */}
      <div
        onDrop={handleDrop}
        onDragOver={handleDragOver}
        className={cn(
          "border-2 border-dashed rounded-lg p-8 text-center cursor-pointer",
          "hover:border-primary/50 transition-colors",
          isDragging && "border-primary bg-primary/5"
        )}
      >
        <Upload className="mx-auto h-12 w-12 text-muted-foreground mb-4" />
        <p className="text-sm font-medium mb-2">
          Drop submission.json here
        </p>
        <p className="text-xs text-muted-foreground mb-4">
          or click to browse
        </p>
        <input
          type="file"
          accept=".json"
          onChange={handleFileSelect}
          className="hidden"
          ref={fileInputRef}
        />
        <Button
          variant="outline"
          onClick={() => fileInputRef.current?.click()}
        >
          Choose File
        </Button>
      </div>

      {/* Verification Results */}
      {verificationResult && (
        <div className="space-y-4">
          <Separator />

          <div className="p-6 bg-muted rounded-lg text-center">
            <p className="text-sm text-muted-foreground mb-2">
              Your Score
            </p>
            <p className="text-4xl font-bold">
              {(verificationResult.score * 100).toFixed(1)}%
            </p>
            <p className="text-sm text-muted-foreground mt-2">
              {verificationResult.solvedTasks} / {verificationResult.totalTasks} tasks solved
            </p>
          </div>

          {verificationResult.message && (
            <Alert>
              <MessageSquare className="h-4 w-4" />
              <AlertTitle>Decoded Message</AlertTitle>
              <AlertDescription>
                {verificationResult.message}
              </AlertDescription>
            </Alert>
          )}
        </div>
      )}
    </div>
  </CardContent>
</Card>
```

---

## Component Structure

```typescript
// client/src/pages/ReArcPage.tsx
export default function ReArcPage() {
  const [generatedDataset, setGeneratedDataset] = useState<Dataset | null>(null);
  const [decodedMessage, setDecodedMessage] = useState<string | null>(null);
  const [verificationResult, setVerificationResult] = useState<VerificationResult | null>(null);
  const [isGenerating, setIsGenerating] = useState(false);
  const [isVerifying, setIsVerifying] = useState(false);

  const handleGenerate = async () => {
    setIsGenerating(true);
    try {
      const response = await fetch('/api/rearc/generate', { method: 'POST' });
      const dataset = await response.json();
      setGeneratedDataset(dataset);
      // Auto-download
      downloadJSON(dataset, 'challenges.json');
    } catch (error) {
      toast.error('Generation failed');
    } finally {
      setIsGenerating(false);
    }
  };

  const handleVerify = async (file: File) => {
    setIsVerifying(true);
    try {
      const submission = JSON.parse(await file.text());
      const response = await fetch('/api/rearc/verify', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ submission })
      });
      const result = await response.json();
      setVerificationResult(result);
    } catch (error) {
      toast.error('Verification failed');
    } finally {
      setIsVerifying(false);
    }
  };

  return (
    <PageLayout>
      <div className="container max-w-4xl mx-auto py-8 space-y-8">
        {/* Header */}
        <div className="text-center space-y-2">
          <h1 className="text-3xl font-bold">Re-ARC Challenge Verifier</h1>
          <p className="text-muted-foreground">
            Generate cryptographically-verified ARC datasets
          </p>
        </div>

        {/* How It Works */}
        <HowItWorksSection />

        {/* Generate */}
        <GenerateSection
          isGenerating={isGenerating}
          generatedDataset={generatedDataset}
          onGenerate={handleGenerate}
          onDownload={handleDownload}
        />

        {/* Verify */}
        <VerifySection
          isVerifying={isVerifying}
          verificationResult={verificationResult}
          onVerify={handleVerify}
        />
      </div>
    </PageLayout>
  );
}
```

---

## Visual Flow

```
┌────────────────────────────────────────┐
│  Re-ARC Challenge Verifier             │
│  Generate cryptographic datasets       │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│  [?] How It Works (collapsible)        │
│  1. Generate → 2. Solve → 3. Verify    │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│  Generate Challenge Dataset            │
│  ┌──────────────────────────────────┐  │
│  │ [Generate Dataset]               │  │
│  └──────────────────────────────────┘  │
│  ↓ (after generation)                  │
│  [Download challenges.json]            │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│  Verify Your Submission                │
│  ┌──────────────────────────────────┐  │
│  │  Drop submission.json here       │  │
│  │  [Choose File]                   │  │
│  └──────────────────────────────────┘  │
│  ↓ (after verification)                │
│  ┌──────────────────────────────────┐  │
│  │  Score: 87.5%                    │  │
│  │  350/400 tasks solved            │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

---

## Key UX Principles

1. **No assumptions**: Explain everything inline
2. **Progressive disclosure**: Collapsible sections for details
3. **Immediate feedback**: Show results inline, no navigation
4. **Error handling**: Clear error messages with recovery steps
5. **Responsive**: Works on mobile (drag-and-drop falls back to file picker)

---

## States to Handle

**Generation:**
- Idle (show button)
- Generating (spinner + disabled)
- Success (show download + stats)
- Error (toast + retry button)

**Verification:**
- Idle (show drop zone)
- Uploading (progress indicator)
- Validating (spinner)
- Success (show score)
- Error (show validation errors)

---

## Copy for Common Errors

**Invalid submission format:**
> ❌ Invalid submission format
>
> Expected JSON object with task IDs as keys. See the format guide above.

**Missing task IDs:**
> ❌ Submission contains unknown task IDs
>
> Make sure you're submitting solutions for the dataset you downloaded.

**Wrong attempt count:**
> ❌ Each task must have exactly 2 attempts
>
> Format: `{ "attempt_1": [[...]], "attempt_2": [[...]] }`
