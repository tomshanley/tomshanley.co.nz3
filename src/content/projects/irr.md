---
title: "DAX Internal Rate of Return"
description: "Provide an IRR value for a single date"
liveUrl: https://example.com
githubUrl: https://github.com
image: {
    url: "/irr.png",
    alt:  "irr thumbnail"
}
draft: true
---

## Overview

In 2025 I was working with client who needed to report on Internal Rate of Return (IRR) for their investments.

IRR is a metric used in financial analysis to estimate the profitability of investments.

While the maths for IRR are quite complicated, Excel and Power BI provide simple functions for calculating IRR. In Power BI, the [XIRR function](https://learn.microsoft.com/en-us/dax/xirr-function-dax) will return an IRR value given a table with columns for date and cashflows. So the trick is to calculate that table per data point you need to present in a chart or for a single KPI figure.

The XIRR needs the cashflows to be populated per date. And for each data point you present you need all historic dates, the memory usage can start to blow out for large number of investments and dates.

The overall DAX calculation we ended up using was this, which we used for reporting at a financial quarter and for "projected" IRR. 

- For reporting at quarter end, if there cashflows on or after that date, we calculate IRR as at the last day of the quarter. Otherwise, if the last cashflow is before the end of the quarter, we report IRR as at the date of the last cashflow.
- IRR is sensitive to the dates used, so it helps to provide IRR on the cashflow dates if possible, and if it is clear to the dashboard user.
- Projected IRR means that we use the investments current value (as at the date being reported on) as a potential cashflow if investment was sold on that day.

First I identify the first and last date where journal entries occur that relate to IRR (in the source data, journal records are flagged using **is_irr** if they are used for IRR calculations):

```dax
IRR (Selected Qtr) = 
VAR __FirstIRRDate = CALCULATE(
    MIN(fact_journal[JournalDateNZT]), 
    REMOVEFILTERS(dim_date), 
    fact_journal[is_irr] = TRUE()
    )

VAR __LatestIRRDate = CALCULATE(
    MAX(fact_journal[JournalDateNZT]), 
    REMOVEFILTERS(dim_date), 
    fact_journal[is_irr] = TRUE()
    )
```

For data point's period (ie quarter), we get the start and end dates that the quarter covers:

```dax
VAR __StartPeriod = MIN(dim_date[date_id])
VAR __EndPeriod = MAX(dim_date[date_id])
```

For data point's period (ie quarter), we check if there's an IRR cashflow or the current running total for the investments value.

```dax
VAR __CurrTransaction = IF(
    [IRR Cashflow Is Blank], 
    [Value RT], 
    [IRR Cashflow]
    )
```

Get the list of investments being analyzed right now (based on filters or grouping on the visual):

```dax  
VAR __SelectedInvestments = VALUES(dim_investment[investmentid])
```

We get today's date for use later:

```dax
VAR __LocalToday = TODAY()
```

If the last IRR transaction is before the end of the current period, then we use a different, simpler, IRR calculation that uses per day records from the journals.

If the most recent IRR transaction date is before or equal to the current reporting period, and today’s date is within the reporting range, then: 

Return the final IRR as of the last actual transaction. 

This avoids projection if you already have final numbers. 

```dax
VAR __LatestIRR = IF(
    __LatestIRRDate <= __EndPeriod
    && __LocalToday >= __StartPeriod, 
        CALCULATE(
            [IRR (Projected, Day)], 
            dim_date[date_id] <= __LatestIRRDate
        )
    )
```

Create a table that shows cash flows for each investment, within the reporting period.

For the last period, it includes both value and projected cash flows to reflect expected future value.

```dax
VAR __IRRTable =  
ADDCOLUMNS (
    CALCULATETABLE (
        SUMMARIZE (
            FILTER (
                'dim_date',
                dim_date[date_id] >= __FirstIRRDate 
                    && dim_date[date_id] <= __EndPeriod
            ),
            dim_date[last_day_quarter] 
             
        ),
        REMOVEFILTERS ( dim_date[last_day_quarter]  )
      
    ),
    "IRRCashflow", CALCULATE(
        IF(
            MAX(dim_date[last_day_quarter]) = __EndPeriod, 
            [Value RT] + [IRR Cashflow], 
            [IRR Cashflow]
        ),
        fact_journal[investmentid] IN __SelectedInvestments
        )
)
```

Return Either the Projected IRR or the Latest Known IRR

Depending on the situation:

- If the report includes future periods and there are valid transactions:
- Calculate a projected IRR using quarterly cash flows (XIRR).
- If not: Just return the last known IRR from historical data.

```dax
RETURN IF(
    ISBLANK(__EndPeriod) = FALSE() 
        && ISBLANK(__FirstIRRDate) = FALSE() 
        && __LatestIRRDate > __EndPeriod, 
    XIRR(__IRRTable, [IRRCashflow], dim_date[last_day_quarter], 0.1, 0),
    __LatestIRR
)
```

The following is the equivalent measure where IRR is calculated using the daily records from the journal tables. The journal records are filters to only those that are

1. Within the selected date range
2. Flagged as IRR related cashflows
3. Related to the investments being reported on


```dax
IRR (Projected, Day) = 
VAR __FirstPeriod = [First Period With IRR Journal]
VAR __SelectedQuarter =  [Selected Quarter End]
VAR __CurrPeriod = IF(
    ISBLANK(__SelectedQuarter), 
    [Max Date], 
    [Last Period With IRR Journal]
    )
VAR __CurrTransaction = [IRR Cashflow]

VAR __SelectedInvestments = CALCULATETABLE(
    VALUES(dim_investment[investmentid]), 
    dim_investment[actualexitdate_filled] > __SelectedQuarter
    )

VAR __CountInvestments = COUNTROWS(__SelectedInvestments)

VAR __Value = IF(ISBLANK(__CurrTransaction) = FALSE(), [Value RT])

VAR __IRRJournalDates = 
SUMMARIZECOLUMNS(
    fact_journal[journaldatenzt], 
    FILTER(
        ALL(fact_journal), 
        fact_journal[journaldatenzt] >= __FirstPeriod 
            && fact_journal[journaldatenzt] <= __CurrPeriod  
            && fact_journal[is_irr] = TRUE()  
            && fact_journal[investmentid] IN __SelectedInvestments
    )
)

VAR __IRRTable = 
ADDCOLUMNS (
    CALCULATETABLE (
        SUMMARIZE (
            FILTER (
                'dim_date',
                dim_date[date_id] IN __IRRJournalDates
            ),
             dim_date[date_id] 
        ),
        REMOVEFILTERS ( 'dim_date'[date_id] )
    ),
    "IRRCashflow", IF(ISBLANK(__CurrTransaction) = FALSE(), 
        CALCULATE(
            IF(
                MAX(dim_date[date_id]) = __CurrPeriod
                , __Value + [IRR Cashflow]
                , [IRR Cashflow]
            ),
            fact_journal[investmentid] IN __SelectedInvestments
        )
    )
)

RETURN IF(
            __CurrPeriod > __FirstPeriod 
            && ISBLANK(__CurrTransaction) = FALSE() 
            && __CountInvestments > 0, 
            XIRR(__IRRTable, [IRRCashflow], dim_date[date_id], 0.1, 0), 
            BLANK()
        )
```
