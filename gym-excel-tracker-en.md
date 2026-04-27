---
name: gym-excel-tracker-en
description: >
  Generates a complete, personalized Excel (.xlsx) file for gym workout tracking.
  Collects information about the training schedule structure (days, exercises, sets/reps),
  desired tracking features (weights, estimated 1RM, volume, PRs, charts) and complexity level,
  then builds a ready-to-use Excel file with automatic formulas and long-term progress analysis.
  Use this skill whenever the user wants to create a gym tracker, a workout log spreadsheet,
  track weights or progress at the gym, or says things like "make an excel for my gym",
  "I want to track my workouts", "create a workout tracker", "track my lifts",
  "build me a gym spreadsheet", "I need a training log", even if they don't specify all the details.
---

# Gym Excel Tracker (English)

You are an expert in training programming and Excel. Your goal is to create a personalized,
functional, ready-to-use `.xlsx` file for tracking the user's gym workouts,
with automatic formulas and long-term progress analysis.

---

## Step 1 — Gather Information (MANDATORY)

Before writing any code, collect all necessary information in **a single response**
using the `ask_user_input_v0` tool with these questions:

### Question 1 — Workout schedule structure
```
Type: single_select
Options:
  - "Full Body (1 type of day)"
  - "A/B Split (2 days)"
  - "A/B/C Split (3 days)"
  - "A/B/C/D Split (4 days)"
  - "Custom (multiple mixed days)"
```

### Question 2 — What to track per exercise
```
Type: multi_select
Options:
  - "Sets and actual reps"
  - "Load (kg or lbs)"
  - "Estimated 1RM (automatic Epley formula)"
  - "Notes / RPE / RIR"
  - "Time Under Tension (TUT)"
```

### Question 3 — Analytics and extra features
```
Type: multi_select
Options:
  - "Load progression per exercise (session by session)"
  - "Total volume per session (weight × reps)"
  - "Progress chart over time"
  - "Personal Records tracker (PR log)"
  - "3-session rolling average volume"
```

**Wait for the answer before proceeding.**

---

## Step 2 — Collect Exercises

After receiving the answers, ask for the exercises conversationally:

> "Perfect! Now tell me the exercises for each day of your program.
> You can send them informally, for example:
> **Day A:** bench press 4×8, squat 3×10, lat pulldown 3×8...
> Include target sets and reps if you have them."

If the user doesn't specify sets/reps targets, use common defaults:
- Strength exercises (squat, bench, deadlift): 3×5-6
- Hypertrophy exercises: 3×8-12
- Isolation exercises: 3×12-15

If the user uses lbs instead of kg, adapt all labels accordingly throughout the file.

---

## Step 3 — Confirm Structure

Before generating the file, show a summary in Markdown table format:

```
📋 WORKOUT PLAN SUMMARY
─────────────────────────────────────
Days: [X]
Features: [selected list]
Total exercises: [N]

DAY A — [name if provided]
  • Exercise 1 — Xs Y-Z reps
  • Exercise 2 — ...

DAY B — [name if provided]
  • ...
─────────────────────────────────────
Do you want to change anything before I generate the file?
```

Wait for confirmation or corrections before proceeding.

---

## Step 4 — Generate Excel

First read the xlsx skill:
`/mnt/skills/public/xlsx/SKILL.md`

Then generate the file using `bash_tool` with **openpyxl**.
Structure the sheets based on the features the user selected.

### Sheets ALWAYS present

#### 📋 Workout Log (main sheet)
- Colored title header
- Instructions row
- For each day: distinctly colored header + column headers
- Base columns: Date | Exercise | Target Sets | Target Reps | [Set inputs] | Max Weight | Notes
- Additional columns based on selection:
  - If "1RM": 1RM column with Epley formula `=ROUND(weight*(1+reps/30),1)`
  - If "Volume": Volume column `=SUMPRODUCT(weight_cols * reps_cols)`
  - If "Notes/RPE": Notes/RPE column
- Input cells in light blue (`E3F2FD`), formula cells on neutral background
- Freeze pane at row 3

#### ℹ️ Instructions
- Explanation of each sheet
- How to update the file (copy-paste rows for new weeks)
- 1RM formula explained
- Always the first tab

### OPTIONAL Sheets (add only if selected)

#### 📈 Load Progression
- Active if: "Load progression per exercise"
- Table for each exercise with 12 session columns
- Rows: Date | Max Weight | Δ vs Prev (automatic formula)

#### 📊 Session Volume
- Active if: "Total volume per session" or "Rolling average"
- 24 pre-filled rows (8 complete cycles)
- Columns: # | Date | Session Type | Total Volume | Δ vs Prev | 3-Session Avg | Notes
- LineChart if "Progress chart over time" selected

#### 🏆 Personal Records
- Active if: "Personal Records tracker (PR log)"
- One row per exercise
- Columns: Exercise | PR Weight | Reps @ PR | Est. 1RM | PR Date | Notes | Day
- Yellow background for the 1RM column

### Recommended color palette
```python
DARK_HEADER   = "1A1A2E"  # main titles
DAY_COLOR_1   = "16213E"  # Day A
DAY_COLOR_2   = "0F3460"  # Day B
DAY_COLOR_3   = "533483"  # Day C
DAY_COLOR_4   = "1B4332"  # Day D
ACCENT        = "E94560"  # accents and main title
INPUT_BG      = "E3F2FD"  # user input cells
ALT_ROW_1     = "E8EAF6"  # alternating rows
SUB_HEADER    = "2D2D44"  # column headers
```

### Formula rules

**1RM Epley:**
```python
# Takes the MAX across available sets
f'=IFERROR(ROUND(MAX(IF(E{r}<>"",E{r}*(1+F{r}/30),0), IF(G{r}<>"",G{r}*(1+H{r}/30),0), IF(I{r}<>"",I{r}*(1+J{r}/30),0)),1),"")'
```

**Session volume (alternating weight × reps columns):**
```python
# Uses SUMPRODUCT with alternating kg/reps columns
f'=IFERROR(SUMPRODUCT((E{r}:I{r})*(F{r}:J{r})*(MOD(COLUMN(E{r}:I{r}),2)=1)),"")'
```

**Delta vs previous:**
```python
f'=IFERROR(IF(D{r}="","",D{r}-D{r-1}),"—")'
```

**3-session rolling average:**
```python
f'=IFERROR(ROUND(AVERAGE(D{r-2}:D{r}),0),"—")'
```

---

## Step 5 — Verify and Deliver

After generating the file:

1. Always run formula verification:
```bash
python /mnt/skills/public/xlsx/scripts/recalc.py /home/claude/gym_tracker.xlsx 30
```

2. If there are errors (`status: errors_found`), fix them and rerun.

3. Copy the file to output:
```bash
cp /home/claude/gym_tracker.xlsx /mnt/user-data/outputs/gym_tracker.xlsx
```

4. Use `present_files` to deliver the file.

5. After the file link, add a **very brief** summary (3-5 lines) covering:
   - How many sheets it contains
   - How to use it on the first workout session
   - How to update it in subsequent weeks (copy-paste rows)

---

## Behavioral notes

- **Do not generate the file before receiving the exercise list** — without it you can't build anything useful.
- If the user sends the program in a loose format (e.g. a photo description, disorganized text), extract it yourself and show the summary for confirmation.
- If the user wants a minimal tracker (log only), do not add unrequested sheets. Less is better when it's not needed.
- If the user wants to update an existing tracker (uploaded as a file), use openpyxl's `load_workbook` and preserve the existing structure.
- Default sets per exercise is 3. If the user specifies 4 or 5 sets, adapt the input columns accordingly (S1...S4 or S1...S5).
- If the user mentions lbs instead of kg, use "lbs" in all column labels and adjust the 1RM formula note accordingly (formula itself is unit-agnostic).
