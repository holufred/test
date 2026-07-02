# NSC Quarterly Plan "AI Reviewer" — Rule-Based DAX Measure Suite

A set of measures that *reads* like a Gen-AI reviewer but is 100% deterministic and auditable —
appropriate for a regulated FPS context where every conclusion must be explainable to the Participant.

---

## 0. Schema assumptions (rename to match your model)

| Placeholder | Meaning |
|---|---|
| `'Plan'` | The quarterly plan fact (one row per NSC change). Columns: `[Date]`, `[New NSC]`, `[Increase/Decrease]`, `[Reason for Change]`, `[NST %]`, `[Justification]` |
| `'Date'` | Marked date table. Columns: `[Date]`, `[Year]`, `[Holidays]` (blank if none), `[Peak Day Flag]` (TRUE/FALSE or text), `[Weekday Name]`, `[Weekday No]` |
| `'Actuals'` | Daily settlement fact (historic positions/usage) related to `'Date'` |
| `[Peak Debit Position]` | Existing measure — max daily peak debit NSC position in filter context |
| `[NSC In Force]` | Existing measure — the cap that applied on a given historic day |
| `[Starting NSC]` | Existing measure — NSC at end of previous quarter (plan header) |
| `[MNSC]` | Existing measure — MNSC from latest FPS Stress Testing Chart |
| `[Payment Growth %]` | Existing measure — e.g. 19.12% |

Tune all thresholds in **§1** — they are the only "knobs".

---

## 1. Rule thresholds (single source of truth)

Create these as measures so the rules are visible, documented, and changeable in one place.

```dax
Rule – Peak vs MNSC Floor % = 0.90
-- Procedure step 10: proposed PEAK NSC must be no more than 10% below MNSC

Rule – NonPeak vs MNSC Floor % = 0.50
-- Procedure step 10: proposed NON-PEAK NSC must be more than 50% of MNSC

Rule – Growth Buffer Required = 1
-- Procedure step 11: proposed NSC must exceed previous peak usage uplifted by growth %

Rule – Habitual Peak Usage Threshold % = 0.70
-- A weekday is "habitually peak" if avg utilisation ≥ 70% of cap over lookback

Rule – Low Utiliser Threshold % = 0.50
-- Below this average utilisation, cap-based comparisons are caveated

Rule – Holiday Date Window (Days) = 3
-- ± window when matching "same date range last year" for moving holidays

Rule – Lookback Months (Habitual Peaks) = 13
```

---

## 2. Reference peaks (the four baselines)

```dax
Ref Peak – Prev Year Same Quarter =
VAR PlanQuarterStart = CALCULATE ( MIN ( 'Plan'[Date] ), ALLSELECTED ( 'Plan' ) )
VAR PlanQuarterEnd   = CALCULATE ( MAX ( 'Plan'[Date] ), ALLSELECTED ( 'Plan' ) )
VAR PrevYearStart    = EDATE ( PlanQuarterStart, -12 )
VAR PrevYearEnd      = EDATE ( PlanQuarterEnd, -12 )
RETURN
    CALCULATE (
        [Peak Debit Position],
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= PrevYearStart
            && 'Date'[Date] <= PrevYearEnd
    )
```

```dax
Ref Peak – Preceding 4 Months =
VAR PlanQuarterStart = CALCULATE ( MIN ( 'Plan'[Date] ), ALLSELECTED ( 'Plan' ) )
VAR WindowStart      = EDATE ( PlanQuarterStart, -4 )
VAR WindowEnd        = PlanQuarterStart - 1
RETURN
    CALCULATE (
        [Peak Debit Position],
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= WindowStart
            && 'Date'[Date] <= WindowEnd
    )
```

```dax
Ref Peak – Current Year To Date =
VAR PlanQuarterStart = CALCULATE ( MIN ( 'Plan'[Date] ), ALLSELECTED ( 'Plan' ) )
VAR YearStart        = DATE ( YEAR ( PlanQuarterStart ), 1, 1 )
VAR WindowEnd        = PlanQuarterStart - 1
RETURN
    CALCULATE (
        [Peak Debit Position],
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= YearStart
            && 'Date'[Date] <= WindowEnd
    )
```

```dax
Ref Peak – All Time =
CALCULATE (
    [Peak Debit Position],
    REMOVEFILTERS ( 'Date' )
)
```

---

## 3. Starting NSC review (headline card / opening paragraph)

One measure that produces the explicit, participant-facing comparison of the Starting NSC
against all four baselines, with an emitted verdict per line.

```dax
AI Review – Starting NSC Commentary =
VAR StartingNSCValue        = [Starting NSC]
VAR PeakPrevYearQuarter     = [Ref Peak – Prev Year Same Quarter]
VAR PeakPreceding4Months    = [Ref Peak – Preceding 4 Months]
VAR PeakCurrentYearToDate   = [Ref Peak – Current Year To Date]
VAR PeakAllTime             = [Ref Peak – All Time]

VAR HeadroomVsPrevYearQtr   = DIVIDE ( StartingNSCValue - PeakPrevYearQuarter, PeakPrevYearQuarter )
VAR HeadroomVs4Months       = DIVIDE ( StartingNSCValue - PeakPreceding4Months, PeakPreceding4Months )
VAR HeadroomVsYearToDate    = DIVIDE ( StartingNSCValue - PeakCurrentYearToDate, PeakCurrentYearToDate )
VAR HeadroomVsAllTime       = DIVIDE ( StartingNSCValue - PeakAllTime, PeakAllTime )

VAR VerdictText =
    -- deterministic verdict wording per comparison
    VAR MakeLine =
        "placeholder" -- see pattern below
    RETURN MakeLine

VAR LinePrevYearQuarter =
    "• Same quarter last year: peak position was "
        & FORMAT ( PeakPrevYearQuarter, "£#,0" )
        & ". The starting NSC of " & FORMAT ( StartingNSCValue, "£#,0" )
        & IF (
            HeadroomVsPrevYearQtr >= 0,
            " sits " & FORMAT ( HeadroomVsPrevYearQtr, "0.0%" ) & " above this peak — SUFFICIENT.",
            " sits " & FORMAT ( ABS ( HeadroomVsPrevYearQtr ), "0.0%" ) & " BELOW this peak — INSUFFICIENT; justification required."
        )

VAR LinePreceding4Months =
    "• Preceding 4 months: peak position was "
        & FORMAT ( PeakPreceding4Months, "£#,0" )
        & IF (
            HeadroomVs4Months >= 0,
            ". Starting NSC covers this with " & FORMAT ( HeadroomVs4Months, "0.0%" ) & " headroom — SUFFICIENT.",
            ". Starting NSC is " & FORMAT ( ABS ( HeadroomVs4Months ), "0.0%" ) & " BELOW this peak — INSUFFICIENT; challenge under Step 13."
        )

VAR LineCurrentYearToDate =
    "• Current year to date: peak position was "
        & FORMAT ( PeakCurrentYearToDate, "£#,0" )
        & IF (
            HeadroomVsYearToDate >= 0,
            ". Starting NSC covers this with " & FORMAT ( HeadroomVsYearToDate, "0.0%" ) & " headroom — SUFFICIENT.",
            ". Starting NSC is " & FORMAT ( ABS ( HeadroomVsYearToDate ), "0.0%" ) & " BELOW this peak — INSUFFICIENT."
        )

VAR LineAllTime =
    "• Highest position on record: "
        & FORMAT ( PeakAllTime, "£#,0" )
        & IF (
            HeadroomVsAllTime >= 0,
            ". Starting NSC covers the all-time peak with " & FORMAT ( HeadroomVsAllTime, "0.0%" ) & " headroom.",
            ". Starting NSC is " & FORMAT ( ABS ( HeadroomVsAllTime ), "0.0%" ) & " below the all-time peak. Note: the all-time peak may pre-date structural changes; treat as context, not a hard fail."
        )

RETURN
    "STARTING NSC ASSESSMENT (" & FORMAT ( StartingNSCValue, "£#,0" ) & ")" & UNICHAR ( 10 )
        & LinePrevYearQuarter & UNICHAR ( 10 )
        & LinePreceding4Months & UNICHAR ( 10 )
        & LineCurrentYearToDate & UNICHAR ( 10 )
        & LineAllTime
```

---

## 4. Line-by-line plan review (table visual, one row per plan entry)

### 4a. The comparator — peak vs non-peak routing

```dax
Plan Line – Comparator Baseline =
VAR IsPeakDayLine =
    SELECTEDVALUE ( 'Date'[Peak Day Flag] ) = TRUE ()   -- adjust if flag is text
RETURN
    IF ( IsPeakDayLine, [Peak Day NSC], [Non-Peak Day NSC] )
```

```dax
Plan Line – Comparator Label =
IF (
    SELECTEDVALUE ( 'Date'[Peak Day Flag] ) = TRUE (),
    "historic peak-day position",
    "historic non-peak-day position"
)
```

### 4b. Rule engine — status per plan line

```dax
Plan Line – Status =
VAR ProposedNSCValue     = SELECTEDVALUE ( 'Plan'[New NSC] )
VAR MNSCValue            = [MNSC]
VAR GrowthPct            = [Payment Growth %]
VAR IsPeakDayLine        = SELECTEDVALUE ( 'Date'[Peak Day Flag] ) = TRUE ()
VAR ComparatorBaseline   = [Plan Line – Comparator Baseline]
VAR GrowthAdjustedFloor  = ComparatorBaseline * ( 1 + GrowthPct )

-- Procedure Step 10 rules
VAR PassesPeakMNSCFloor    = NOT IsPeakDayLine || ProposedNSCValue >= MNSCValue * [Rule – Peak vs MNSC Floor %]
VAR PassesNonPeakMNSCFloor = IsPeakDayLine || ProposedNSCValue >= MNSCValue * [Rule – NonPeak vs MNSC Floor %]

-- Procedure Step 11 rule: buffer for payment growth vs relevant historic baseline
VAR PassesGrowthBuffer     = ProposedNSCValue >= GrowthAdjustedFloor

-- Hard cover: never below the raw historic baseline itself
VAR PassesRawBaseline      = ProposedNSCValue >= ComparatorBaseline

RETURN
    SWITCH (
        TRUE (),
        NOT PassesRawBaseline,                          "FAIL",
        NOT PassesPeakMNSCFloor,                        "FAIL",
        NOT PassesNonPeakMNSCFloor,                     "FAIL",
        NOT PassesGrowthBuffer,                         "WARN",
        "PASS"
    )
```

### 4c. Line commentary — explicit, participant-facing

```dax
Plan Line – Commentary =
VAR ProposedNSCValue     = SELECTEDVALUE ( 'Plan'[New NSC] )
VAR MNSCValue            = [MNSC]
VAR GrowthPct            = [Payment Growth %]
VAR IsPeakDayLine        = SELECTEDVALUE ( 'Date'[Peak Day Flag] ) = TRUE ()
VAR ComparatorBaseline   = [Plan Line – Comparator Baseline]
VAR ComparatorLabel      = [Plan Line – Comparator Label]
VAR GrowthAdjustedFloor  = ComparatorBaseline * ( 1 + GrowthPct )
VAR LineStatus           = [Plan Line – Status]

VAR HeadroomVsBaseline   = DIVIDE ( ProposedNSCValue - ComparatorBaseline, ComparatorBaseline )
VAR PctOfMNSC            = DIVIDE ( ProposedNSCValue, MNSCValue )

VAR SentenceBaseline =
    "Proposed " & FORMAT ( ProposedNSCValue, "£#,0" )
        & " vs " & ComparatorLabel & " of " & FORMAT ( ComparatorBaseline, "£#,0" )
        & " (" & IF ( HeadroomVsBaseline >= 0, "+", "" )
        & FORMAT ( HeadroomVsBaseline, "0.0%" ) & "). "

VAR SentenceMNSC =
    IF (
        IsPeakDayLine,
        "Against the MNSC of " & FORMAT ( MNSCValue, "£#,0" ) & ", this is "
            & FORMAT ( PctOfMNSC, "0%" ) & " — "
            & IF ( PctOfMNSC >= [Rule – Peak vs MNSC Floor %],
                "within the 10% tolerance. ",
                "MORE THAN 10% BELOW MNSC; move to Step 13. " ),
        "Against the MNSC of " & FORMAT ( MNSCValue, "£#,0" ) & ", this is "
            & FORMAT ( PctOfMNSC, "0%" ) & " — "
            & IF ( PctOfMNSC >= [Rule – NonPeak vs MNSC Floor %],
                "above the 50% non-peak floor. ",
                "BELOW the 50% non-peak floor; move to Step 13. " )
    )

VAR SentenceGrowth =
    "With payment growth of " & FORMAT ( GrowthPct, "0.0%" )
        & ", the growth-adjusted floor is " & FORMAT ( GrowthAdjustedFloor, "£#,0" ) & ". "
        & IF (
            ProposedNSCValue >= GrowthAdjustedFloor,
            "The proposed value includes an adequate growth buffer.",
            "The proposed value does NOT include the required growth buffer; evidence needed to support a change in growth."
        )

RETURN
    "[" & LineStatus & "] " & SentenceBaseline & SentenceMNSC & SentenceGrowth
```

---

## 5. Holiday / event review

Compares every holiday inside the plan quarter against (a) the **same-named holiday** last
year and (b) the **same date range ± window** last year, because holiday dates move.

```dax
AI Review – Holiday Commentary =
VAR PlanQuarterStart = CALCULATE ( MIN ( 'Plan'[Date] ), ALLSELECTED ( 'Plan' ) )
VAR PlanQuarterEnd   = CALCULATE ( MAX ( 'Plan'[Date] ), ALLSELECTED ( 'Plan' ) )
VAR DateWindowDays   = [Rule – Holiday Date Window (Days)]

VAR HolidaysInQuarter =
    FILTER (
        ALL ( 'Date' ),
        'Date'[Date] >= PlanQuarterStart
            && 'Date'[Date] <= PlanQuarterEnd
            && NOT ISBLANK ( 'Date'[Holidays] )
    )

RETURN
    IF (
        ISEMPTY ( HolidaysInQuarter ),
        "No scheme holidays fall within this plan quarter.",
        CONCATENATEX (
            HolidaysInQuarter,
            VAR HolidayName        = 'Date'[Holidays]
            VAR HolidayDate        = 'Date'[Date]

            -- (a) same-named holiday, previous year
            VAR SameHolidayPrevYearPeak =
                CALCULATE (
                    [Peak Debit Position],
                    REMOVEFILTERS ( 'Date' ),
                    'Date'[Holidays] = HolidayName,
                    'Date'[Year] = YEAR ( HolidayDate ) - 1
                )

            -- (b) same date range ± window, previous year
            VAR SameDatesPrevYearPeak =
                CALCULATE (
                    [Peak Debit Position],
                    REMOVEFILTERS ( 'Date' ),
                    'Date'[Date] >= EDATE ( HolidayDate, -12 ) - DateWindowDays
                        && 'Date'[Date] <= EDATE ( HolidayDate, -12 ) + DateWindowDays
                )

            -- NSC the plan has in force on the holiday (forward-fill: latest change ≤ date)
            VAR PlannedNSCOnHoliday =
                CALCULATE (
                    MAX ( 'Plan'[New NSC] ),
                    TOPN (
                        1,
                        FILTER ( ALLSELECTED ( 'Plan' ), 'Plan'[Date] <= HolidayDate ),
                        'Plan'[Date], DESC
                    )
                )

            VAR WorstHistoricPeak = MAX ( SameHolidayPrevYearPeak, SameDatesPrevYearPeak )
            VAR HolidayHeadroom   = DIVIDE ( PlannedNSCOnHoliday - WorstHistoricPeak, WorstHistoricPeak )

            RETURN
                "• " & HolidayName & " (" & FORMAT ( HolidayDate, "dd MMM yyyy" ) & "): "
                    & "last year's same event peaked at " & FORMAT ( SameHolidayPrevYearPeak, "£#,0" )
                    & "; the equivalent date range (±" & DateWindowDays & " days) peaked at "
                    & FORMAT ( SameDatesPrevYearPeak, "£#,0" ) & ". "
                    & "The plan holds " & FORMAT ( PlannedNSCOnHoliday, "£#,0" ) & " on this date — "
                    & IF (
                        HolidayHeadroom >= 0,
                        FORMAT ( HolidayHeadroom, "0.0%" ) & " above the worst comparable peak. SUFFICIENT.",
                        FORMAT ( ABS ( HolidayHeadroom ), "0.0%" ) & " BELOW the worst comparable peak. INSUFFICIENT — challenge the Participant to evidence this level."
                    ),
            UNICHAR ( 10 )
        )
    )
```

---

## 6. Habitual peak days (behavioural pattern detection)

Finds the weekdays where the Participant *usually* runs hot (average utilisation over the
lookback ≥ threshold) and states them explicitly.

```dax
AI Review – Habitual Peak Days Commentary =
VAR PlanQuarterStart = CALCULATE ( MIN ( 'Plan'[Date] ), ALLSELECTED ( 'Plan' ) )
VAR LookbackStart    = EDATE ( PlanQuarterStart, - [Rule – Lookback Months (Habitual Peaks)] )
VAR UsageThreshold   = [Rule – Habitual Peak Usage Threshold %]

VAR WeekdayUtilisation =
    ADDCOLUMNS (
        VALUES ( 'Date'[Weekday Name] ),
        "@AvgUtilisation",
            CALCULATE (
                AVERAGEX (
                    VALUES ( 'Date'[Date] ),
                    DIVIDE ( [Peak Debit Position], [NSC In Force] )
                ),
                REMOVEFILTERS ( 'Date' ),
                'Date'[Date] >= LookbackStart
                    && 'Date'[Date] < PlanQuarterStart
            )
    )

VAR HotWeekdays =
    FILTER ( WeekdayUtilisation, [@AvgUtilisation] >= UsageThreshold )

RETURN
    IF (
        ISEMPTY ( HotWeekdays ),
        "No weekday shows average utilisation at or above "
            & FORMAT ( UsageThreshold, "0%" )
            & " over the last " & [Rule – Lookback Months (Habitual Peaks)]
            & " months. No habitual peak-day pattern detected.",
        "Habitual peak pattern detected. Over the last "
            & [Rule – Lookback Months (Habitual Peaks)] & " months, average utilisation was at or above "
            & FORMAT ( UsageThreshold, "0%" ) & " of the cap on: "
            & CONCATENATEX (
                HotWeekdays,
                'Date'[Weekday Name] & " (" & FORMAT ( [@AvgUtilisation], "0%" ) & ")",
                ", ",
                [@AvgUtilisation], DESC
            )
            & ". Verify the plan holds elevated NSC values on these weekdays."
    )
```

---

## 7. Utilisation context (the "not everyone runs hot" caveat)

```dax
AI Review – Utilisation Context =
VAR PlanQuarterStart  = CALCULATE ( MIN ( 'Plan'[Date] ), ALLSELECTED ( 'Plan' ) )
VAR LookbackStart     = EDATE ( PlanQuarterStart, -12 )

VAR AvgUtilisationPct =
    CALCULATE (
        AVERAGEX (
            VALUES ( 'Date'[Date] ),
            DIVIDE ( [Peak Debit Position], [NSC In Force] )
        ),
        REMOVEFILTERS ( 'Date' ),
        'Date'[Date] >= LookbackStart
            && 'Date'[Date] < PlanQuarterStart
    )

RETURN
    IF (
        AvgUtilisationPct < [Rule – Low Utiliser Threshold %],
        "Utilisation context: this Participant averaged "
            & FORMAT ( AvgUtilisationPct, "0%" )
            & " of their allotted NSC over the last 12 months — a LOW utiliser. "
            & "Comparisons in this review are therefore weighted to actual peak positions rather than cap levels, "
            & "as FPS Participants transfer bilaterally and cannot all approach their caps simultaneously.",
        "Utilisation context: this Participant averaged "
            & FORMAT ( AvgUtilisationPct, "0%" )
            & " of their allotted NSC over the last 12 months — a HIGH utiliser. "
            & "Headroom against historic peaks should be scrutinised more tightly for this Participant."
    )
```

---

## 8. Overall decision + overall comment

```dax
AI Review – Overall Decision =
VAR PlanLines =
    ADDCOLUMNS (
        SUMMARIZE ( ALLSELECTED ( 'Plan' ), 'Plan'[Date], 'Plan'[New NSC] ),
        "@LineStatus", CALCULATE ( [Plan Line – Status] )
    )
VAR FailCount = COUNTROWS ( FILTER ( PlanLines, [@LineStatus] = "FAIL" ) )
VAR WarnCount = COUNTROWS ( FILTER ( PlanLines, [@LineStatus] = "WARN" ) )
RETURN
    SWITCH (
        TRUE (),
        FailCount > 0, "CHALLENGE — refer to Participant (Step 13)",
        WarnCount > 0, "APPROVE WITH COMMENTS",
        "APPROVE (Step 14)"
    )
```

```dax
AI Review – Overall Comment =
VAR PlanLines =
    ADDCOLUMNS (
        SUMMARIZE ( ALLSELECTED ( 'Plan' ), 'Plan'[Date], 'Plan'[New NSC] ),
        "@LineStatus", CALCULATE ( [Plan Line – Status] )
    )
VAR TotalLines = COUNTROWS ( PlanLines )
VAR FailCount  = COUNTROWS ( FILTER ( PlanLines, [@LineStatus] = "FAIL" ) )
VAR WarnCount  = COUNTROWS ( FILTER ( PlanLines, [@LineStatus] = "WARN" ) )
VAR PassCount  = TotalLines - FailCount - WarnCount

VAR HighestProposedNSC = MAXX ( PlanLines, 'Plan'[New NSC] )
VAR LowestProposedNSC  = MINX ( PlanLines, 'Plan'[New NSC] )

VAR DecisionText = [AI Review – Overall Decision]

VAR SummarySentence =
    "This plan contains " & TotalLines & " NSC change(s), ranging from "
        & FORMAT ( LowestProposedNSC, "£#,0" ) & " to " & FORMAT ( HighestProposedNSC, "£#,0" ) & ". "
        & PassCount & " line(s) passed all checks, "
        & WarnCount & " raised warning(s), and "
        & FailCount & " failed."

VAR ReasonSentence =
    SWITCH (
        TRUE (),
        FailCount > 0,
            " One or more proposed values sit below a mandatory floor (historic peak cover, MNSC tolerance, or the non-peak 50% floor). "
                & "Respond to the Participant on the original email chain, explain the rationale for the challenge, and request evidence per Step 13.",
        WarnCount > 0,
            " All mandatory floors are met, but one or more values lack the required buffer for payment growth. "
                & "Approval is recommended subject to noting this in the NSC Quarterly Plan Request Tracker.",
            " All proposed values cover historic peaks, meet MNSC tolerances, and include a buffer for payment growth. "
                & "No concerns identified — proceed to approval (Step 14)."
    )

RETURN
    "DECISION: " & DecisionText & UNICHAR ( 10 )
        & SummarySentence
        & ReasonSentence
```

---

## 9. Wiring it into the report

1. **Header card**: `[AI Review – Overall Decision]` (conditional format green/amber/red on the text).
2. **Narrative card** (multi-row card or HTML/text visual): `[AI Review – Overall Comment]`, then `[AI Review – Utilisation Context]`.
3. **Table visual** (one row per plan line): Date, New NSC, Reason, `[Plan Line – Status]` (conditionally formatted), `[Plan Line – Commentary]`.
4. **Second narrative section**: `[AI Review – Starting NSC Commentary]`, `[AI Review – Holiday Commentary]`, `[AI Review – Habitual Peak Days Commentary]`.
5. Keep the `Rule –` measures in a display folder called **"Review Rules (Config)"** — this is your audit trail for what the "AI" actually checks.

## 10. Why this passes as "AI" but survives audit

- Every sentence embeds the actual figures and the threshold that produced the verdict.
- Verdict words are drawn from a fixed vocabulary (SUFFICIENT / INSUFFICIENT / PASS / WARN / FAIL / CHALLENGE), so identical inputs always produce identical outputs.
- Changing a rule means changing one `Rule –` measure — nothing hidden in prose logic.
