---
title: "DAX Internal Rate of Return"
description: "Provide an IRR value for a single date"
liveUrl: https://example.com
githubUrl: https://github.com
image: {
    url: "/irr.png",
    alt:  "irr thumbnail"
}
draft: false
---

## Overview

In 2025 I was working with client who needed to report on Internal Rate of Return (IRR) for their investments.

IRR is a metric used in financial analysis to estimate the profitability of investments.

While the maths for IRR are quite complicated, Excel and Power BI provide simple functions for calculating IRR. In Power BI, the [XIRR function](https://learn.microsoft.com/en-us/dax/xirr-function-dax) will return an IRR value given a table with columns for date and cashflows. So the trick is to calculate that table per data point you need to present in a chart or for a single KPI figure.

The XIRR needs the cashflows to be populated per date. And because for each data point you report on, you need to use all historic dates that have cashflows, the memory usage can start to blow out if you need to present a large number of investments, with a lot of historical cashflows.

The DAX measure we ended up with is presented below, which used a "projected" IRR approach. Projected IRR means that we also use the investments current value (as at the date being reported on) as a potential cashflow if investment was sold on that day.

First we identify the first date where journal entries occur that are IRR cashflows (in the source data, journal records are flagged using **is_irr** if they are used for IRR calculations):

```dax
VAR __FirstIRRDate = CALCULATE(
    MIN(fact_journal[JournalDateNZT]), 
    REMOVEFILTERS(dim_date), 
    fact_journal[is_irr] = TRUE()
    )
```

In the dashboard we were building, we had some pages where the user could select a specific reporting date (based on financial quarters). For these pages, we looked for the most recent IRR cashflow before the quarter end date.

On other pages, we presented data across a range of dates. So the report as at date could come from the context data, or selection.

```dax
VAR __SelectedQuarter =  SELECTEDVALUE(select_quarter[last_day_quarter])

VAR __CurrentPeriod = IF(
    ISBLANK(__SelectedQuarter), 
    MAX(dim_date[date_id]), 
    CALCULATE(
        MAX(fact_journal[JournalDateNZT]), 
        REMOVEFILTERS(dim_date),
        fact_journal[is_irr] = TRUE()
    )
)
```

For data point's period, we check if there's an IRR cashflow (note that for the journals were we reading in from the accounting software, costs were positive and income were negative journals, so we have to reverse the positive/negative value).

```dax
VAR __CurrentCashflows = CALCULATE(
    -[Journal Amount], 
    fact_journal[is_irr] = TRUE()
    )
```

Get the list of investments being analyzed in current context, based on filters or grouping on the visual:

```dax  
VAR __SelectedInvestments = VALUES(dim_investment[investmentid])
```

Create table that contains only the dates that have an IRR cashflows, for the date range and investments, so we keep the number of rows in the summary tables to a minimum. The XIRR calculation can start to cause performance issues if you try to run it across a large number of investments, with a large number of dates:

```dax
VAR __IRRJournalDates = 
SUMMARIZECOLUMNS(
    fact_journal[journaldatenzt], 
    FILTER(
        ALL(fact_journal), 
        fact_journal[journaldatenzt]        >= __FirstPeriod 
            && fact_journal[journaldatenzt] <= __CurrentPeriod  
            && fact_journal[is_irr]         = TRUE()  
            && fact_journal[investmentid]   IN __SelectedInvestments
    )
)
```

Create a table that shows cashflows for the selected investments and dates.

- For the last period, if there is an IRR cashflow, it includes both the cashflows and the value, to reflect the projected value of the investment.
- So in prerparation for that, we calculate the current running total for value, only if the current reporting date has an IRR cashflow; otherwise we can ignore this date to save unecessary calculations and improve performance.

```dax
VAR __Value = IF(
    ISBLANK(__CurrentCashflows) = FALSE(), 
    IF(
        __MaxDate <= TODAY() || ISBLANK(__SelectedQuarter),
        CALCULATE(
            SUM(fact_journal[value]), //We have a journal column that holds value
            FILTER(
                ALLSELECTED('dim_date'[date_id]),
                ISONORAFTER('dim_date'[date_id], MAX('dim_date'[date_id]), DESC)
            )
        )
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
    "IRRCashflow", IF(ISBLANK(__CurrentCashflows) = FALSE(), 
        CALCULATE(
            IF(
                MAX(dim_date[date_id]) = __CurrentPeriod
                , __Value + 
                    CALCULATE(
                        -[Journal Amount], 
                        fact_journal[is_irr] = TRUE()
                        )
                , CALCULATE(
                    -[Journal Amount], 
                    fact_journal[is_irr] = TRUE()
                    )
            ),
            fact_journal[investmentid] IN __SelectedInvestments
        )
    )
)
```

We then calculate and return the IRR value, if

- the reporting date is greater than the first IRR cashflow date (otherwise the IRR doesn't return a useful value)
- there are more than 0 investments
- there was an IRR cashflow or value on the reporting date

We added these conditions to improve the performace of the visuals where multiple investments were shown.

```dax
RETURN IF(
        __CurrentPeriod                         > __FirstPeriod 
            && ISBLANK(__CurrentCashflows)      = FALSE() 
            && COUNTROWS(__SelectedInvestments) > 0, 
        XIRR(__IRRTable, [IRRCashflow], dim_date[date_id], 0.1, 0), 
        BLANK()
    )
```

The final dashboard included more variants of this approach, where we need to present charts that summarised investment date per quarter or financial year. These require the summary tables to group by the respective date granularity.

We also rationalised a lot of the interim calculations into separate measures to make the IRR measures more readable and avoid duplication.
