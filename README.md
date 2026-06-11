# NSC Plan Review — Rule-Based "Gen AI" Narrative Engine (DAX)

A layered measure suite that reads like an AI reviewer but is 100% deterministic and auditable — exactly what you want for a regulatory oversight function. The output mirrors the style of your Column H comments ("Max Usage for 2025 was on 22 April at £357,040,792.30, against growth rate of 19.10%, provides a RED flag…").

---

## 0. Assumed model (rename to match yours)

| Object | Assumed name | Notes |
|---|---|---|
| Plan fact | `'NSC Plan'` | Columns: `[Participant]`, `[Effective Date]`, `[New NSC]`, `[Change Type]` |
| Date table | `'Date'` | Columns: `[Date]`, `[Year]`, `[Day Name]`, `[Day Number]`, `[Holiday]` (blank if none), `[Is Peak Day]` (TRUE/FALSE) |
| Existing measures | `[Peak Position]` | Daily peak settlement position |
| | `[NSC Usage %]` | % of allotted NSC actually used |
| | `[Peak Day NSC]`, `[Non-Peak Day NSC]` | Your existing planned-cap measures |

All measures assume the visual/page is filtered to **one participant and one plan quarter**. Variables are deliberately verbose-named per your preference.

---

## 1. Layer 1 — Atomic reference measures

```dax
Assumed Growth Rate % =
-- Hard-coded house assumption; swap for SELECTEDVALUE over a what-if
-- parameter table if reviewers need to stress-test it interactively.
0.1910
```

```dax
Max Daily Peak Position =
MAXX ( VALUES ( 'Date'[Date] ), [Peak Position] )
```

```dax
Plan Starting NSC =
-- The NSC in force at the start of the plan quarter (first plan line).
VAR FirstEffectiveDateInQuarter =
    CALCULATE ( MIN ( 'NSC Plan'[Effective Date] ) )
RETURN
    CALCULATE (
        MAX ( 'NSC Plan'[New NSC] ),
        'NSC Plan'[Effective Date] = FirstEffectiveDateInQuarter
    )
```

```dax
Max Planned NSC In Quarter =
-- Highest cap anywhere in the submitted plan (used for the
-- "sufficient increases within the plan to cover the peaks" mitigation).
CALCULATE ( MAX ( 'NSC Plan'[New NSC] ) )
```

### 1a. Previous-year same quarter

```dax
Peak Position | PY Same Quarter =
CALCULATE (
    [Max Daily Peak Position],
    SAMEPERIODLASTYEAR ( 'Date'[Date] )
)
```

```dax
Peak Position Date | PY Same Quarter =
-- The exact date of the PY peak, so the narrative can cite it explicitly.
VAR PreviousYearDailyPeaks =
    CALCULATETABLE (
        ADDCOLUMNS ( VALUES ( 'Date'[Date] ), "@DailyPeak", [Peak Position] ),
        SAMEPERIODLASTYEAR ( 'Date'[Date] )
    )
VAR TopPeakRow =
    TOPN ( 1, PreviousYearDailyPeaks, [@DailyPeak], DESC )
RETURN
    MAXX ( TopPeakRow, 'Date'[Date] )
```

### 1b. Trailing 4 months before the quarter

```dax
Peak Position | Trailing 4 Months Pre-Quarter =
VAR QuarterStartDate   = MIN ( 'Date'[Date] )
VAR LookbackStartDate  = EOMONTH ( QuarterStartDate, -5 ) + 1   -- 4 full calendar months
VAR LookbackEndDate    = QuarterStartDate - 1
RETURN
    CALCULATE (
        [Max Daily Peak Position],
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= LookbackStartDate
            && 'Date'[Date] <= LookbackEndDate
    )
```

### 1c. Highest so far in the current year

```dax
Peak Position | Current Year To Date =
VAR QuarterStartDate  = MIN ( 'Date'[Date] )
VAR CurrentYearStart  = DATE ( YEAR ( QuarterStartDate ), 1, 1 )
VAR EvaluationEndDate = MIN ( QuarterStartDate - 1, TODAY () )
RETURN
    CALCULATE (
        [Max Daily Peak Position],
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= CurrentYearStart
            && 'Date'[Date] <= EvaluationEndDate
    )
```

---

## 2. Layer 2 — Peak-day routing & governing requirement

```dax
Applicable Plan NSC =
-- Routes the comparison: peak-day flag → Peak Day NSC, else Non-Peak Day NSC.
VAR IsPeakDayFlag = SELECTEDVALUE ( 'Date'[Is Peak Day], FALSE )
RETURN
    IF ( IsPeakDayFlag, [Peak Day NSC], [Non-Peak Day NSC] )
```

```dax
Applicable Reference Peak | PY Comparable Days =
-- PY same-quarter peak restricted to the SAME day class (peak vs non-peak),
-- so a weekday line is never judged against a bank-holiday spike.
VAR IsPeakDayFlag = SELECTEDVALUE ( 'Date'[Is Peak Day], FALSE )
RETURN
    CALCULATE (
        [Max Daily Peak Position],
        SAMEPERIODLASTYEAR ( 'Date'[Date] ),
        'Date'[Is Peak Day] = IsPeakDayFlag
    )
```

```dax
Governing Reference Peak =
-- The single number the plan must beat: the worst of the three lookbacks,
-- uplifted by the growth assumption.
VAR GrowthUpliftMultiplier = 1 + [Assumed Growth Rate %]
VAR CandidateRequirements =
    {
        [Peak Position | PY Same Quarter] * GrowthUpliftMultiplier,
        [Peak Position | Trailing 4 Months Pre-Quarter] * GrowthUpliftMultiplier,
        [Peak Position | Current Year To Date] * GrowthUpliftMultiplier
    }
RETURN
    MAXX ( CandidateRequirements, [Value] )
```

> If house policy only applies the growth uplift to the PY figure (recent months arguably already embed growth), move the multiplier onto the first candidate only.

---

## 3. Layer 3 — Holiday / event comparison

Compares every holiday in the plan quarter against **(a)** the same-named holiday last year and **(b)** the same calendar date range last year (because Easter etc. moves), and takes the worse of the two as the requirement.

```dax
Holiday Event Comparison Narrative =
VAR GrowthUpliftMultiplier = 1 + [Assumed Growth Rate %]
VAR HolidaysInQuarter =
    FILTER (
        SUMMARIZE ( 'Date', 'Date'[Holiday] ),
        NOT ISBLANK ( 'Date'[Holiday] )
    )
RETURN
    CONCATENATEX (
        HolidaysInQuarter,
        VAR CurrentHolidayName = 'Date'[Holiday]
        VAR CurrentHolidayDates =
            CALCULATETABLE ( VALUES ( 'Date'[Date] ), 'Date'[Holiday] = CurrentHolidayName )
        VAR EventWindowStart = MINX ( CurrentHolidayDates, 'Date'[Date] )
        VAR EventWindowEnd   = MAXX ( CurrentHolidayDates, 'Date'[Date] )
        VAR PlannedNSCForEvent =
            CALCULATE ( MAX ( 'NSC Plan'[New NSC] ), 'Date'[Holiday] = CurrentHolidayName )

        -- (a) Same-named holiday in the previous year
        VAR PreviousYearHolidayDates =
            CALCULATETABLE (
                VALUES ( 'Date'[Date] ),
                REMOVEFILTERS ( 'Date' ),
                'Date'[Holiday] = CurrentHolidayName,
                'Date'[Year] = YEAR ( EventWindowStart ) - 1
            )
        VAR PYNamedHolidayPeak =
            CALCULATE (
                [Max Daily Peak Position],
                REMOVEFILTERS ( 'Date' ),
                PreviousYearHolidayDates
            )
        VAR PYNamedHolidayPeakDate =
            VAR DailyPeaks =
                CALCULATETABLE (
                    ADDCOLUMNS ( PreviousYearHolidayDates, "@DailyPeak", [Peak Position] ),
                    REMOVEFILTERS ( 'Date' )
                )
            RETURN MAXX ( TOPN ( 1, DailyPeaks, [@DailyPeak], DESC ), 'Date'[Date] )

        -- (b) Same calendar date range, previous year (holiday dates shift)
        VAR PYSameDateRangeStart =
            DATE ( YEAR ( EventWindowStart ) - 1, MONTH ( EventWindowStart ), DAY ( EventWindowStart ) )
        VAR PYSameDateRangeEnd =
            DATE ( YEAR ( EventWindowEnd ) - 1, MONTH ( EventWindowEnd ), DAY ( EventWindowEnd ) )
        VAR PYSameDateRangePeak =
            CALCULATE (
                [Max Daily Peak Position],
                REMOVEFILTERS ( 'Date' ),
                'Date'[Date] >= PYSameDateRangeStart
                    && 'Date'[Date] <= PYSameDateRangeEnd
            )

        -- Governing requirement for this event
        VAR GoverningEventPeak = MAX ( PYNamedHolidayPeak, PYSameDateRangePeak )
        VAR RequiredCoverWithGrowth = GoverningEventPeak * GrowthUpliftMultiplier
        VAR SurplusOrShortfall = PlannedNSCForEvent - RequiredCoverWithGrowth
        RETURN
            CurrentHolidayName & " ("
                & FORMAT ( EventWindowStart, "dd MMM" ) & "–" & FORMAT ( EventWindowEnd, "dd MMM yyyy" )
                & "): " & FORMAT ( YEAR ( EventWindowStart ) - 1, "0" )
                & " comparable event max usage was on " & FORMAT ( PYNamedHolidayPeakDate, "dd MMMM yyyy" )
                & " at " & FORMAT ( PYNamedHolidayPeak, "£#,##0.00" )
                & "; same calendar dates last year peaked at " & FORMAT ( PYSameDateRangePeak, "£#,##0.00" )
                & ". With growth rate of " & FORMAT ( [Assumed Growth Rate %], "0.00%" )
                & ", required cover is " & FORMAT ( RequiredCoverWithGrowth, "£#,##0" )
                & " against a planned NSC of " & FORMAT ( PlannedNSCForEvent, "£#,##0" )
                & IF (
                    SurplusOrShortfall >= 0,
                    ", giving a surplus of c" & FORMAT ( SurplusOrShortfall, "£#,##0,," ) & "m — therefore "
                        & FORMAT ( PlannedNSCForEvent, "£#,##0,," ) & "m would be sufficient.",
                    ", leaving a SHORTFALL of c" & FORMAT ( - SurplusOrShortfall, "£#,##0,," ) & "m — planned level requires review."
                ),
        UNICHAR ( 10 ) & UNICHAR ( 10 )
    )
```

---

## 4. Layer 4 — Behavioural pattern: usual peak days

Derives the participant's typical peak weekdays over the trailing 13 months from utilisation (and peak position), and flags habitual high-utilisation days — your "are the increases timed where this bank actually peaks?" check.

```dax
Typical Peak Days Narrative =
VAR QuarterStartDate  = MIN ( 'Date'[Date] )
VAR LookbackStartDate = EOMONTH ( QuarterStartDate, -14 ) + 1   -- 13 full months
VAR LookbackEndDate   = QuarterStartDate - 1
VAR WeekdayBehaviour =
    CALCULATETABLE (
        ADDCOLUMNS (
            SUMMARIZE ( 'Date', 'Date'[Day Name], 'Date'[Day Number] ),
            "@AvgUtilisationPct", [NSC Usage %],
            "@AvgDailyPeak", AVERAGEX ( VALUES ( 'Date'[Date] ), [Peak Position] )
        ),
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= LookbackStartDate
            && 'Date'[Date] <= LookbackEndDate
    )
VAR TopUtilisationWeekdays =
    TOPN ( 2, WeekdayBehaviour, [@AvgUtilisationPct], DESC )
VAR HighestAvgUtilisation =
    MAXX ( TopUtilisationWeekdays, [@AvgUtilisationPct] )
VAR WeekdayListText =
    CONCATENATEX (
        TopUtilisationWeekdays,
        'Date'[Day Name] & " (avg utilisation "
            & FORMAT ( [@AvgUtilisationPct], "0.0%" )
            & " of allotted NSC; avg peak " & FORMAT ( [@AvgDailyPeak], "£#,##0" ) & ")",
        " and ",
        [@AvgUtilisationPct], DESC
    )
RETURN
    "Over the trailing 13 months, usage has typically peaked on "
        & WeekdayListText & "."
        & IF (
            HighestAvgUtilisation > 0.85,
            " ATTENTION: average utilisation on the busiest day class exceeds the 85% NST — confirm planned increases align to these days.",
            " Planned increase timings should be sense-checked against these days."
        )
```

---

## 5. Layer 5 — Per-line review comment (your Column H equivalent)

Drop this on each plan row in a table visual. It routes peak vs non-peak automatically and writes either the familiar "Sufficient against NSC + Growth" or an explicit, quantified exception.

```dax
Plan Line Review Comment =
VAR PlannedNSCForLine = SELECTEDVALUE ( 'NSC Plan'[New NSC] )
VAR GrowthUpliftMultiplier = 1 + [Assumed Growth Rate %]
VAR ComparableReferencePeak = [Applicable Reference Peak | PY Comparable Days]
VAR RequiredCoverWithGrowth = ComparableReferencePeak * GrowthUpliftMultiplier
VAR HeadroomAmount = PlannedNSCForLine - RequiredCoverWithGrowth
RETURN
    IF (
        ISBLANK ( PlannedNSCForLine ) || ISBLANK ( ComparableReferencePeak ),
        BLANK (),
        IF (
            HeadroomAmount >= 0,
            "Sufficient against NSC + Growth (comparable-day PY peak "
                & FORMAT ( ComparableReferencePeak, "£#,##0" )
                & " + " & FORMAT ( [Assumed Growth Rate %], "0.0%" ) & " growth = "
                & FORMAT ( RequiredCoverWithGrowth, "£#,##0" )
                & "; headroom c" & FORMAT ( HeadroomAmount, "£#,##0,," ) & "m).",
            "INSUFFICIENT against NSC + Growth — shortfall of c"
                & FORMAT ( - HeadroomAmount, "£#,##0,," ) & "m vs required "
                & FORMAT ( RequiredCoverWithGrowth, "£#,##0" ) & ". Query with participant."
        )
    )
```

---

## 6. Layer 6 — Headline RAG + full plan commentary

```dax
Plan Review RAG =
VAR StartingNSC       = [Plan Starting NSC]
VAR MaxPlannedNSC     = [Max Planned NSC In Quarter]
VAR RequiredCover     = [Governing Reference Peak]
RETURN
    SWITCH (
        TRUE (),
        MaxPlannedNSC < RequiredCover, "RED",      -- even the highest planned cap fails
        StartingNSC   < RequiredCover, "AMBER",    -- maintained level fails, but increases cover it
        "GREEN"
    )
```

```dax
AI Plan Review Commentary =
VAR ParticipantName  = SELECTEDVALUE ( 'NSC Plan'[Participant], "the participant" )
VAR StartingNSC      = [Plan Starting NSC]
VAR MaxPlannedNSC    = [Max Planned NSC In Quarter]
VAR GrowthRatePct    = [Assumed Growth Rate %]

VAR PYQuarterPeak      = [Peak Position | PY Same Quarter]
VAR PYQuarterPeakDate  = [Peak Position Date | PY Same Quarter]
VAR Trailing4MPeak     = [Peak Position | Trailing 4 Months Pre-Quarter]
VAR CurrentYearPeak    = [Peak Position | Current Year To Date]
VAR RequiredCover      = [Governing Reference Peak]

VAR StartingHeadroom   = StartingNSC - RequiredCover
VAR MaxPlanHeadroom    = MaxPlannedNSC - RequiredCover
VAR RAGStatus          = [Plan Review RAG]
VAR LineBreak          = UNICHAR ( 10 )

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
    & ". The trailing 4 months peaked at " & FORMAT ( Trailing4MPeak, "£#,##0.00" )
    & ", and the highest position so far this year is " & FORMAT ( CurrentYearPeak, "£#,##0.00" )
    & ". Against a growth rate of " & FORMAT ( GrowthRatePct, "0.00%" )
    & ", the governing requirement is " & FORMAT ( RequiredCover, "£#,##0" ) & "."
    & LineBreak & LineBreak
    & SWITCH (
        RAGStatus,
        "GREEN",
            "The maintained NSC of " & FORMAT ( StartingNSC, "£#,##0" )
            & " covers the requirement with headroom of c"
            & FORMAT ( StartingHeadroom, "£#,##0,," ) & "m. GREEN — no action required.",
        "AMBER",
            "This provides a RED flag for the maintained NSC value of "
            & FORMAT ( StartingNSC, "£#,##0" )
            & " (compared to previous year peak + growth). However, whilst the overall maintained amount is less, there are sufficient increases within the plan ("
            & "up to " & FORMAT ( MaxPlannedNSC, "£#,##0" )
            & ", headroom c" & FORMAT ( MaxPlanHeadroom, "£#,##0,," ) & "m) to cover the peaks. AMBER — monitor.",
        "RED",
            "RED FLAG: even the highest planned NSC of " & FORMAT ( MaxPlannedNSC, "£#,##0" )
            & " falls short of the requirement by c"
            & FORMAT ( RequiredCover - MaxPlannedNSC, "£#,##0,," )
            & "m. Plan must be queried with the participant before approval."
    )
    & LineBreak & LineBreak
    & [Typical Peak Days Narrative]
    & LineBreak & LineBreak
    & [Holiday Event Comparison Narrative]
```

---

## 7. Implementation notes

1. **Visuals.** Put `[AI Plan Review Commentary]` in a card (new card visual handles multi-line text well) or a narrative text box; `[Plan Line Review Comment]` goes in the plan table next to each row; `[Plan Review RAG]` drives conditional formatting (RED/AMBER/GREEN icon or background).
2. **The "AI feel" without the AI.** The phrasing-rotation seed (`PhrasingSeed`) varies sentence openers per participant/quarter deterministically — same inputs always produce identical text, which is exactly what audit wants. You can extend the SWITCH to rotate mid-sentence connectors too.
3. **Holiday matching caveat.** Same-named matching relies on consistent labels in `'Date'[Holiday]` across years ("Easter", "May BH 2", etc.). If labels differ, add a normalised `Holiday Group` column in Power Query.
4. **Peak-day flag.** `Applicable Plan NSC` / `Applicable Reference Peak` assume `'Date'[Is Peak Day]` exists for both years. If the flag is derived per-participant, move it to a measure-based test instead of a column filter.
5. **Performance.** `Holiday Event Comparison Narrative` iterates holidays with nested CALCULATEs — fine at quarter granularity (3–4 events), but don't put it in a large matrix; keep it at participant/quarter level.
6. **Growth rate.** Replace the constant with a what-if parameter (`SELECTEDVALUE ( 'Growth Rate'[Growth Rate Value], 0.1910 )`) if reviewers want to stress-test sensitivity live — that interactivity adds a lot to the "intelligent assistant" impression.
7. **NST.** If you want the 85% NST woven in, add a sentence comparing `[NSC Usage %]` projections against the NST in the commentary — the `Typical Peak Days Narrative` already warns when historical utilisation breaches it.
