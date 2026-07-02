# NSC Plan Review Measures — v2.1 Patch

Fixes: event window anchored to the plan quarter (kills the 2023/blank/reversed-dates bugs),
weekday reference widened from single SPLY date to PY same-quarter same-day-class,
reviewer gating (blank [Reviewer] → no line comments, overall = not approved),
and the final Approved / Not Approved decision measure.

> **Model check first (the £45bn bug):** the same-date-range peak returned ~£45–48bn because the
> actuals fact aggregated ALL participants — the plan row's participant filter is not reaching the
> actuals table. Confirm both `Fact_Quarterly_Plans` and the actuals fact relate to the SAME
> institution dimension (your Institution ID slicer sitting at "All" is the tell). If they use two
> different dims, bridge them — no measure below can compensate for that.

---

## 1. Applicable Reference Peak | PY Comparable Days (widened)

Replaces the single-date SAMEPERIODLASTYEAR with the whole previous-year same quarter,
restricted to the same day class. Stable per day class, matches the procedure and the
participants' justification basis.

```dax
Applicable Reference Peak | PY Comparable Days =
-- PY same-quarter peak restricted to the SAME day class (peak vs non-peak),
-- evaluated over the WHOLE comparable quarter, not the single equivalent date.
VAR IsPeakDayFlag     = SELECTEDVALUE ( 'Date'[Peak Day Flag] )
VAR QuarterStartDate  =
    CALCULATE ( MAX ( 'Fact_Quarterly_Plans'[Quarter Start] ), ALLSELECTED ( 'Fact_Quarterly_Plans' ) )
VAR QuarterEndDate    = EOMONTH ( QuarterStartDate, 3 )
VAR PYQuarterStart    = EDATE ( QuarterStartDate, -12 )
VAR PYQuarterEnd      = EDATE ( QuarterEndDate, -12 )
RETURN
    CALCULATE (
        [Peak Position],
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= PYQuarterStart
            && 'Date'[Date] <= PYQuarterEnd,
        'Date'[Peak Day Flag] = IsPeakDayFlag
    )
```

If `Peak Day Flag` text varies year to year (e.g. contains dates), classify on blank vs non-blank
instead: `ISBLANK ( 'Date'[Peak Day Flag] ) = ISBLANK ( IsPeakDayFlag )`.

---

## 2. Plan Line Review Comment (event block fixed + reviewer gating)

```dax
Plan Line Review Comment =
VAR ReviewerName = [Reviewer]

VAR PlannedNSCForLine      = SELECTEDVALUE ( 'Fact_Quarterly_Plans'[Proposed NSC] )
VAR LineJustification      = SELECTEDVALUE ( 'Fact_Quarterly_Plans'[Justification] )
VAR CurrentHolidayName     = SELECTEDVALUE ( 'Date'[Working_Day_Before_Holiday_Flag] )
VAR IsEventLine            = NOT ISBLANK ( CurrentHolidayName )
VAR GrowthUpliftMultiplier = 1 + [Payment Growth %]

-- Anchor everything to the plan quarter (fixes wrong-year and spanning-window bugs)
VAR QuarterStartDate =
    CALCULATE ( MAX ( 'Fact_Quarterly_Plans'[Quarter Start] ), ALLSELECTED ( 'Fact_Quarterly_Plans' ) )
VAR QuarterEndDate   = EOMONTH ( QuarterStartDate, 3 )

VAR MissingJustificationNote =
    IF (
        ISBLANK ( LineJustification ),
        " No justification provided for this entry — request the Participant's reasoning before approval.",
        ""
    )

------------------------------------------------------------------
-- WEEKDAY / NON-EVENT LINES
------------------------------------------------------------------
VAR ComparableReferencePeak = [Applicable Reference Peak | PY Comparable Days]
VAR WeekdayRequiredCover    = ComparableReferencePeak * GrowthUpliftMultiplier
VAR WeekdayHeadroom         = PlannedNSCForLine - WeekdayRequiredCover
VAR WeekdayComment =
    IF (
        WeekdayHeadroom >= 0,
        "Sufficient against NSC + Growth.",
        "Insufficient against NSC + Growth — shortfall of c"
            & FORMAT ( - WeekdayHeadroom, "£#,##0,," ) & "m vs required "
            & FORMAT ( WeekdayRequiredCover, "£#,##0" )
            & " (PY same-quarter, same day-class peak of "
            & FORMAT ( ComparableReferencePeak, "£#,##0" )
            & " + " & FORMAT ( [Payment Growth %], "0.00%" ) & " growth). Query with participant."
    )

------------------------------------------------------------------
-- EVENT LINES — window restricted to THIS quarter's instance
------------------------------------------------------------------
VAR CurrentHolidayDates =
    CALCULATETABLE (
        VALUES ( 'Date'[Date] ),
        REMOVEFILTERS ( 'Date' ),
        'Date'[Working_Day_Before_Holiday_Flag] = CurrentHolidayName,
        'Date'[Date] >= QuarterStartDate
            && 'Date'[Date] <= QuarterEndDate
    )
VAR EventWindowStart = MINX ( CurrentHolidayDates, 'Date'[Date] )
VAR EventWindowEnd   = MAXX ( CurrentHolidayDates, 'Date'[Date] )

-- (a) Same-named holiday, previous year — restricted to a ±60-day band around
--     the anniversary so a same-named event elsewhere in the year can't leak in
VAR PYAnniversary = EDATE ( EventWindowStart, -12 )
VAR PreviousYearHolidayDates =
    CALCULATETABLE (
        VALUES ( 'Date'[Date] ),
        REMOVEFILTERS ( 'Date' ),
        'Date'[Working_Day_Before_Holiday_Flag] = CurrentHolidayName,
        'Date'[Date] >= PYAnniversary - 60
            && 'Date'[Date] <= PYAnniversary + 60
    )
VAR PYNamedHolidayPeak =
    CALCULATE ( [Peak Position], REMOVEFILTERS ( 'Date' ), PreviousYearHolidayDates )
VAR PYNamedHolidayPeakDate =
    VAR DailyPeaks =
        CALCULATETABLE (
            ADDCOLUMNS ( PreviousYearHolidayDates, "@DailyPeak", [Peak Position] ),
            REMOVEFILTERS ( 'Date' )
        )
    RETURN MAXX ( TOPN ( 1, DailyPeaks, [@DailyPeak], DESC ), 'Date'[Date] )
VAR PYNamedWindowStart = MINX ( PreviousYearHolidayDates, 'Date'[Date] )
VAR PYNamedWindowEnd   = MAXX ( PreviousYearHolidayDates, 'Date'[Date] )
VAR PYNamedEventFound  = NOT ISBLANK ( PYNamedHolidayPeak )

-- (b) Same calendar date range, previous year
VAR PYSameDateRangeStart = EDATE ( EventWindowStart, -12 )
VAR PYSameDateRangeEnd   = EDATE ( EventWindowEnd, -12 )
VAR PYSameDateRangePeak =
    CALCULATE (
        [Peak Position],
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= PYSameDateRangeStart
            && 'Date'[Date] <= PYSameDateRangeEnd
    )
VAR PYSameDateRangePeakDate =
    VAR SameDateDailyPeaks =
        CALCULATETABLE (
            ADDCOLUMNS (
                CALCULATETABLE (
                    VALUES ( 'Date'[Date] ),
                    REMOVEFILTERS ( 'Date' ),
                    'Date'[Date] >= PYSameDateRangeStart
                        && 'Date'[Date] <= PYSameDateRangeEnd
                ),
                "@DailyPeak", [Peak Position]
            ),
            REMOVEFILTERS ( 'Date' )
        )
    RETURN MAXX ( TOPN ( 1, SameDateDailyPeaks, [@DailyPeak], DESC ), 'Date'[Date] )

VAR GoverningEventPeak      = MAX ( PYNamedHolidayPeak, PYSameDateRangePeak )
VAR EventRequiredCover      = GoverningEventPeak * GrowthUpliftMultiplier
VAR EventSurplusOrShortfall = PlannedNSCForLine - EventRequiredCover

VAR NamedEventSentence =
    IF (
        PYNamedEventFound,
        FORMAT ( YEAR ( PYNamedWindowStart ), "0" ) & " " & CurrentHolidayName
            & " period " & FORMAT ( PYNamedWindowStart, "dd MMM" )
            & "–" & FORMAT ( PYNamedWindowEnd, "dd MMM" )
            & " — max usage was on " & FORMAT ( PYNamedHolidayPeakDate, "dd MMM" )
            & " at " & FORMAT ( PYNamedHolidayPeak, "£#,##0.00" ) & ". ",
        "No same-named event found in the previous year — comparison based on the equivalent calendar dates only. "
    )

VAR EventComment =
    NamedEventSentence
        & "Same dates " & FORMAT ( PYSameDateRangeStart, "dd MMM" )
        & "–" & FORMAT ( PYSameDateRangeEnd, "dd MMM yyyy" )
        & " max usage peaked on " & FORMAT ( PYSameDateRangePeakDate, "dd MMM" )
        & " at " & FORMAT ( PYSameDateRangePeak, "£#,##0.00" ) & ". "
        & "With growth of " & FORMAT ( [Payment Growth %], "0.00%" )
        & " the required cover is " & FORMAT ( EventRequiredCover, "£#,##0" )
        & ", therefore " & FORMAT ( PlannedNSCForLine, "£#,##0,," ) & "m "
        & IF (
            EventSurplusOrShortfall >= 0,
            "would be sufficient (surplus of c"
                & FORMAT ( EventSurplusOrShortfall, "£#,##0,," ) & "m).",
            "would NOT be sufficient — shortfall of c"
                & FORMAT ( - EventSurplusOrShortfall, "£#,##0,," ) & "m. Query with participant."
        )

RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( ReviewerName ), BLANK (),          -- not yet reviewed by an SPE: no line commentary
        ISBLANK ( PlannedNSCForLine ), BLANK (),
        ( IF ( IsEventLine, EventComment, WeekdayComment ) & MissingJustificationNote )
    )
```

---

## 3. Plan Review Decision (new — final Approved / Not Approved)

```dax
Plan Review Decision =
VAR ReviewerName = [Reviewer]
VAR RAGStatus    = [Plan Review RAG]
RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( ReviewerName ), "Not Approved",
        RAGStatus = "RED", "Not Approved",
        "Approved"
    )
```

---

## 4. Plan Review Commentary (reviewer gate at the top)

Add this immediately after the existing VAR block, replacing the current RETURN with:

```dax
VAR ReviewerName = [Reviewer]
RETURN
    IF (
        ISBLANK ( ReviewerName ),
        "Not approved — this plan has not been reviewed by an SPE.",
        -- ...existing full RETURN expression unchanged from v2...
    )
```

---

## Notes

- Both PY lookups are now anchored: the named-event lookup to a ±60-day band around the
  anniversary of THIS quarter's event window, the same-dates lookup via EDATE(-12) on both ends,
  which also guarantees start ≤ end (fixes "24–22 May").
- The weekday reference change means every non-event line in a quarter of the same day class gets
  the SAME required cover — verdicts will no longer flip line-to-line on single-date noise. If you
  want the old behaviour retained for comparison, keep the previous measure as
  `Applicable Reference Peak | PY Same Date` in the display folder.
- Reviewer gating assumes a `[Reviewer]` measure returning blank when unreviewed. If Reviewer is a
  column on the tracker table, wrap it: `Reviewer = SELECTEDVALUE ( 'Review_Tracker'[Reviewer] )`.
- The £45bn magnitude is a MODEL issue: verify the actuals fact and the plans fact share one
  institution dimension so participant context survives `REMOVEFILTERS ( 'Date' )`.






# NSC Plan Review Measures — v2 (aligned to your existing model + procedure + sample output)

Uses YOUR names: `Fact_Quarterly_Plans[Proposed NSC]`, `[New NSC]`, `[Quarter Start]`,
`Dim_Institutions[Participant]`, `'Date'[Peak Day Flag]` (text), `'Date'[Working_Day_Before_Holiday_Flag]`,
and your measures `[Peak Position]`, `[Usage Ratio]`, `[Payment Growth %]`, `[Previous Year Peak NSC Usage]`,
`[Previous Year Peak NSC Usage Date]`, `[Starting NSC]`, `[Proposed Peak NSC Value]`, `[MNSC]`.

## What changed and why

| Measure | Change |
|---|---|
| `Governing Reference Peak` | **FIXED overreach.** Growth uplift now applies ONLY to the PY peak (procedure step 11). Trailing 4M and CYTD become raw floors — they already embody recent growth. |
| `MNSC Comparison Narrative` | **NEW — was missing.** Step 10: peak within 10% of MNSC, non-peak above 50% of MNSC. Handles the new-Participant blank-MNSC case. |
| `NST SNST Trigger Narrative` | **NEW — was missing.** States trigger values and whether last year's comparable peak would have breached them. |
| `Plan Line Review Comment` | **REWRITTEN.** Routes event lines to sample-style event wording ("2025 Easter period... therefore £570m would be sufficient"), weekday lines to the short "Sufficient against NSC + Growth" form, and appends a step-12 blank-justification flag. No procedure-step references in output. |
| `Plan Review RAG` | Kept your logic; RequiredCover now uses the corrected governing measure; MNSC peak breach added as an AMBER floor. |
| `Plan Review Commentary` | Kept your structure and phrasing rotation; inserts the two new narratives; growth attribution corrected in wording. |
| Unchanged | `Peak Position | Current Year To Date`, `Peak Position | Trailing 4 Months Pre-Quarter`, `Applicable Plan NSC`, `Applicable Reference Peak | PY Comparable Days`, `Typical Peak Days Narrative`, `Holiday Event Comparison Narrative` (page-level version). |

---

## 1. Governing Reference Peak (revised)

```dax
Governing Reference Peak =
-- The single number the maintained/peak plan values must beat.
-- Growth uplift applies ONLY to the previous-year comparable peak (procedure step 11).
-- Trailing 4 months and current-year-to-date are raw floors: recent usage already
-- embodies growth, so uplifting them would double-count.
VAR GrowthUpliftMultiplier = 1 + [Payment Growth %]
VAR CandidateRequirements =
    {
        [Previous Year Peak NSC Usage] * GrowthUpliftMultiplier,
        [Peak Position | Trailing 4 Months Pre-Quarter],
        [Peak Position | Current Year To Date]
    }
RETURN
    MAXX ( CandidateRequirements, [Value] )
```

---

## 2. MNSC Comparison Narrative (new — procedure step 10)

```dax
MNSC Comparison Narrative =
VAR MNSCValue           = [MNSC]
VAR ProposedPeakNSC     = [Proposed Peak NSC Value]
VAR ProposedNonPeakNSC  = [Proposed Non-Peak NSC Value]
VAR PeakPctOfMNSC       = DIVIDE ( ProposedPeakNSC, MNSCValue )
VAR NonPeakPctOfMNSC    = DIVIDE ( ProposedNonPeakNSC, MNSCValue )

VAR NewParticipantNote =
    "No MNSC available for this Participant — new Participant FPS Stress Testing Charts "
        & "take time to fill with historic payment processing data, so the MNSC and PCV "
        & "calculation may be distorted. Review the Extra Data tab of the FPS Stress Testing Chart."

VAR PeakLine =
    "Proposed peak NSC of " & FORMAT ( ProposedPeakNSC, "£#,##0" )
        & " is " & FORMAT ( PeakPctOfMNSC, "0%" ) & " of the MNSC ("
        & FORMAT ( MNSCValue, "£#,##0" ) & ")"
        & IF (
            PeakPctOfMNSC >= 0.90,
            " — within the 10% tolerance.",
            " — more than 10% below the MNSC. Verify against the latest FPS Stress Testing Chart and query with participant."
        )

VAR NonPeakLine =
    "Proposed non-peak NSC of " & FORMAT ( ProposedNonPeakNSC, "£#,##0" )
        & " is " & FORMAT ( NonPeakPctOfMNSC, "0%" ) & " of the MNSC"
        & IF (
            NonPeakPctOfMNSC > 0.50,
            " — above the 50% floor.",
            " — at or below the 50% floor. Verify against the latest FPS Stress Testing Chart and query with participant."
        )

RETURN
    IF (
        ISBLANK ( MNSCValue ),
        NewParticipantNote,
        PeakLine & UNICHAR ( 10 ) & NonPeakLine
    )
```

If you don't yet have `[Proposed Non-Peak NSC Value]`:

```dax
Proposed Non-Peak NSC Value =
CALCULATE (
    MIN ( 'Fact_Quarterly_Plans'[New NSC] ),
    ALLSELECTED ( 'Fact_Quarterly_Plans' )
)
```

---

## 3. NST SNST Trigger Narrative (new)

```dax
NST SNST Trigger Narrative =
VAR MaintainedNSC        = [Starting NSC]
VAR ParticipantNSTPct    = [Participant Proposed NST %]      -- e.g. 0.85; swap for your measure/column
VAR PayUKSNSTPct         = 0.75                              -- pre-populated per procedure
VAR NSTTriggerValue      = MaintainedNSC * ParticipantNSTPct
VAR SNSTTriggerValue     = MaintainedNSC * PayUKSNSTPct
VAR PYComparablePeak     = [Previous Year Peak NSC Usage]

VAR TriggerBreachText =
    SWITCH (
        TRUE (),
        PYComparablePeak > NSTTriggerValue,
            "Last year's peak of " & FORMAT ( PYComparablePeak, "£#,##0" )
                & " would have breached the NST trigger — expect settlement-risk alerts on comparable days unless intraday increases are actioned in line with the plan.",
        PYComparablePeak > SNSTTriggerValue,
            "Last year's peak of " & FORMAT ( PYComparablePeak, "£#,##0" )
                & " sits between the SNST and NST triggers — monitor on comparable days.",
        "Last year's peak of " & FORMAT ( PYComparablePeak, "£#,##0" )
            & " sits below both triggers."
    )

RETURN
    "Participant proposed NST of " & FORMAT ( ParticipantNSTPct, "0%" )
        & " gives a settlement-risk trigger of " & FORMAT ( NSTTriggerValue, "£#,##0" )
        & "; Pay.UK SNST of " & FORMAT ( PayUKSNSTPct, "0%" )
        & " gives " & FORMAT ( SNSTTriggerValue, "£#,##0" ) & ". "
        & TriggerBreachText
```

---

## 4. Plan Line Review Comment (rewritten — matches sample column H wording)

```dax
Plan Line Review Comment =
VAR PlannedNSCForLine      = SELECTEDVALUE ( 'Fact_Quarterly_Plans'[Proposed NSC] )
VAR LineJustification      = SELECTEDVALUE ( 'Fact_Quarterly_Plans'[Justification] )
VAR CurrentHolidayName     = SELECTEDVALUE ( 'Date'[Working_Day_Before_Holiday_Flag] )
VAR IsEventLine            = NOT ISBLANK ( CurrentHolidayName )
VAR GrowthUpliftMultiplier = 1 + [Payment Growth %]

-- Step 12: blank-entry flag (appended to whichever comment applies)
VAR MissingJustificationNote =
    IF (
        ISBLANK ( LineJustification ),
        " No justification provided for this entry — request the Participant's reasoning before approval.",
        ""
    )

------------------------------------------------------------------
-- WEEKDAY / NON-EVENT LINES: comparable-day-class PY peak + growth
------------------------------------------------------------------
VAR ComparableReferencePeak = [Applicable Reference Peak | PY Comparable Days]
VAR WeekdayRequiredCover    = ComparableReferencePeak * GrowthUpliftMultiplier
VAR WeekdayHeadroom         = PlannedNSCForLine - WeekdayRequiredCover
VAR WeekdayComment =
    IF (
        WeekdayHeadroom >= 0,
        "Sufficient against NSC + Growth.",
        "Insufficient against NSC + Growth — shortfall of c"
            & FORMAT ( - WeekdayHeadroom, "£#,##0,," ) & "m vs required "
            & FORMAT ( WeekdayRequiredCover, "£#,##0" ) & ". Query with participant."
    )

------------------------------------------------------------------
-- EVENT LINES: named event PY peak + same-dates PY peak, sample style
------------------------------------------------------------------
VAR CurrentHolidayDates =
    CALCULATETABLE (
        VALUES ( 'Date'[Date] ),
        ALLSELECTED ( 'Date' ),
        'Date'[Working_Day_Before_Holiday_Flag] = CurrentHolidayName
    )
VAR EventWindowStart = MINX ( CurrentHolidayDates, 'Date'[Date] )
VAR EventWindowEnd   = MAXX ( CurrentHolidayDates, 'Date'[Date] )

-- (a) Same-named holiday, previous year
VAR PreviousYearHolidayDates =
    CALCULATETABLE (
        VALUES ( 'Date'[Date] ),
        REMOVEFILTERS ( 'Date' ),
        'Date'[Working_Day_Before_Holiday_Flag] = CurrentHolidayName,
        'Date'[Year] = YEAR ( EventWindowStart ) - 1
    )
VAR PYNamedHolidayPeak =
    CALCULATE ( [Peak Position], REMOVEFILTERS ( 'Date' ), PreviousYearHolidayDates )
VAR PYNamedHolidayPeakDate =
    VAR DailyPeaks =
        CALCULATETABLE (
            ADDCOLUMNS ( PreviousYearHolidayDates, "@DailyPeak", [Peak Position] ),
            REMOVEFILTERS ( 'Date' )
        )
    RETURN MAXX ( TOPN ( 1, DailyPeaks, [@DailyPeak], DESC ), 'Date'[Date] )
VAR PYNamedWindowStart = MINX ( PreviousYearHolidayDates, 'Date'[Date] )
VAR PYNamedWindowEnd   = MAXX ( PreviousYearHolidayDates, 'Date'[Date] )

-- (b) Same calendar date range, previous year (holiday dates shift)
VAR PYSameDateRangeStart =
    DATE ( YEAR ( EventWindowStart ) - 1, MONTH ( EventWindowStart ), DAY ( EventWindowStart ) )
VAR PYSameDateRangeEnd =
    DATE ( YEAR ( EventWindowEnd ) - 1, MONTH ( EventWindowEnd ), DAY ( EventWindowEnd ) )
VAR PYSameDateRangePeak =
    CALCULATE (
        [Peak Position],
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= PYSameDateRangeStart
            && 'Date'[Date] <= PYSameDateRangeEnd
    )
VAR PYSameDateRangePeakDate =
    VAR SameDateDailyPeaks =
        CALCULATETABLE (
            ADDCOLUMNS (
                FILTER (
                    ALL ( 'Date'[Date] ),
                    'Date'[Date] >= PYSameDateRangeStart
                        && 'Date'[Date] <= PYSameDateRangeEnd
                ),
                "@DailyPeak", [Peak Position]
            ),
            REMOVEFILTERS ( 'Date' )
        )
    RETURN MAXX ( TOPN ( 1, SameDateDailyPeaks, [@DailyPeak], DESC ), 'Date'[Date] )

-- Governing requirement for this event
VAR GoverningEventPeak       = MAX ( PYNamedHolidayPeak, PYSameDateRangePeak )
VAR EventRequiredCover       = GoverningEventPeak * GrowthUpliftMultiplier
VAR EventSurplusOrShortfall  = PlannedNSCForLine - EventRequiredCover

VAR EventComment =
    FORMAT ( YEAR ( EventWindowStart ) - 1, "0" ) & " " & CurrentHolidayName
        & " period from " & FORMAT ( PYNamedWindowStart, "dd" )
        & " to " & FORMAT ( PYNamedWindowEnd, "dd MMMM" )
        & " — max usage was on " & FORMAT ( PYNamedHolidayPeakDate, "dd MMMM" )
        & " at " & FORMAT ( PYNamedHolidayPeak, "£#,##0.00" )
        & "; with a growth rate of " & FORMAT ( [Payment Growth %], "0.00%" )
        & " the required cover is " & FORMAT ( EventRequiredCover, "£#,##0" ) & ". "
        & "Same dates " & FORMAT ( PYSameDateRangeStart, "dd" )
        & "–" & FORMAT ( PYSameDateRangeEnd, "dd MMMM yyyy" )
        & " max usage peaked on " & FORMAT ( PYSameDateRangePeakDate, "dd" )
        & " at " & FORMAT ( PYSameDateRangePeak, "£#,##0.00" ) & ", therefore "
        & FORMAT ( PlannedNSCForLine, "£#,##0,," ) & "m "
        & IF (
            EventSurplusOrShortfall >= 0,
            "would be sufficient (surplus of c"
                & FORMAT ( EventSurplusOrShortfall, "£#,##0,," ) & "m).",
            "would NOT be sufficient — shortfall of c"
                & FORMAT ( - EventSurplusOrShortfall, "£#,##0,," ) & "m. Query with participant."
        )

RETURN
    IF (
        ISBLANK ( PlannedNSCForLine ),
        BLANK (),
        IF ( IsEventLine, EventComment, WeekdayComment ) & MissingJustificationNote
    )
```

---

## 5. Plan Review RAG (revised)

```dax
Plan Review RAG =
VAR StartingNSCValue = [Starting NSC]
VAR MaxPlannedNSC    = [Proposed Peak NSC Value]
VAR RequiredCover    = [Governing Reference Peak]
VAR MNSCValue        = [MNSC]
VAR PeakPctOfMNSC    = DIVIDE ( MaxPlannedNSC, MNSCValue )
RETURN
    SWITCH (
        TRUE (),
        MaxPlannedNSC < RequiredCover, "RED",                      -- even the highest planned cap fails
        NOT ISBLANK ( MNSCValue ) && PeakPctOfMNSC < 0.90, "AMBER", -- step 10 peak/MNSC tolerance breached
        StartingNSCValue < RequiredCover, "AMBER",                 -- maintained level fails, increases cover it
        "GREEN"
    )
```

---

## 6. Plan Review Commentary (revised — your structure, corrected content)

```dax
Plan Review Commentary =
VAR ParticipantName   = SELECTEDVALUE ( 'Dim_Institutions'[Participant], "the participant" )
VAR StartingNSCValue  = [Starting NSC]
VAR MaxPlannedNSC     = [Proposed Peak NSC Value]
VAR GrowthRatePct     = [Payment Growth %]

VAR PYQuarterPeak     = [Previous Year Peak NSC Usage]
VAR PYQuarterPeakDate = [Previous Year Peak NSC Usage Date]
VAR Trailing4MPeak    = [Peak Position | Trailing 4 Months Pre-Quarter]
VAR CurrentYearPeak   = [Peak Position | Current Year To Date]
VAR RequiredCover     = [Governing Reference Peak]

VAR StartingHeadroom  = StartingNSCValue - RequiredCover
VAR MaxPlanHeadroom   = MaxPlannedNSC - RequiredCover
VAR RAGStatus         = [Plan Review RAG]
VAR LineBreak         = UNICHAR ( 10 )

-- Deterministic phrasing rotation: feels generative, fully reproducible.
VAR PhrasingSeed =
    MOD ( LEN ( ParticipantName ) + MONTH ( MIN ( 'Date'[Date] ) ), 3 )
VAR OpeningPhrase =
    SWITCH ( PhrasingSeed,
        0, "Automated plan review for ",
        1, "Plan robustness assessment — ",
        2, "Validation summary for "
    )

RETURN
    OpeningPhrase & ParticipantName & ":" & LineBreak & LineBreak
        & "Max usage in the same quarter last year was on "
        & FORMAT ( PYQuarterPeakDate, "dd MMMM yyyy" ) & " at "
        & FORMAT ( PYQuarterPeak, "£#,##0.00" )
        & "; against a growth rate of " & FORMAT ( GrowthRatePct, "0.00%" )
        & " this requires " & FORMAT ( PYQuarterPeak * ( 1 + GrowthRatePct ), "£#,##0" ) & "."
        & " The trailing 4 months peaked at " & FORMAT ( Trailing4MPeak, "£#,##0.00" )
        & ", and the highest position so far this year is " & FORMAT ( CurrentYearPeak, "£#,##0.00" )
        & " (recent peaks taken as-is — growth is applied to the prior-year comparison only)."
        & " The governing requirement is " & FORMAT ( RequiredCover, "£#,##0" ) & "."
        & LineBreak & LineBreak
        & SWITCH (
            RAGStatus,
            "GREEN",
                "The maintained NSC of " & FORMAT ( StartingNSCValue, "£#,##0" )
                    & " covers the requirement with headroom of c"
                    & FORMAT ( StartingHeadroom, "£#,##0,," ) & "m. GREEN — no action required.",
            "AMBER",
                "This provides a RED flag for the maintained NSC value of "
                    & FORMAT ( StartingNSCValue, "£#,##0" )
                    & " (compared to previous year peak + growth). However, whilst the overall maintained amount is less, there are sufficient increases within the plan to cover the peaks (up to "
                    & FORMAT ( MaxPlannedNSC, "£#,##0" )
                    & ", headroom c" & FORMAT ( MaxPlanHeadroom, "£#,##0,," ) & "m). AMBER — monitor.",
            "RED",
                "RED FLAG: even the highest planned NSC of " & FORMAT ( MaxPlannedNSC, "£#,##0" )
                    & " falls short of the requirement by c"
                    & FORMAT ( RequiredCover - MaxPlannedNSC, "£#,##0,," )
                    & "m. Plan must be queried with the participant before approval."
        )
        & LineBreak & LineBreak
        & [MNSC Comparison Narrative]
        & LineBreak & LineBreak
        & [NST SNST Trigger Narrative]
        & LineBreak & LineBreak
        & [Typical Peak Days Narrative]
        & LineBreak & LineBreak
        & [Holiday Event Comparison Narrative]
```

---

## Sanity check against your USBE sample

- PY peak £357,040,792.30 × (1 + 19.10%) = **£425.3m** required → maintained £400m below it, but max planned £570m covers → **AMBER**, and the AMBER wording reproduces the sample comment near-verbatim.
- Easter line: named event window peak £357.0m → required £425.3m vs planned £570m → "therefore £570m would be sufficient (surplus of c£145m)" plus the same-dates comparison (02–07 April 2025, peaked 07 at £227.7m) — matching the sample's two-part structure.
- Weekday £160m lines → non-peak comparable-day-class check → "Sufficient against NSC + Growth." matching column H.
- Non-peak £160m is 46% of MNSC £347.4m → the MNSC narrative now surfaces "at or below the 50% floor — verify against the latest FPS Stress Testing Chart", which your current suite silently ignored.
