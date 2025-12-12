---
title: "DAX Internal Rate of Return"
description: "Provide an IRR value for a single date"
liveUrl: https://example.com
githubUrl: https://github.com
image: {
url: "/formsync.webp",
alt:  "FormSync thumbnail"
}
---

## Overview

In 2025 I was working with client who needed to report on Internal Rate of Return (IRR) for their investments.

IRR is a simple metric used in financial analysis to estimate the profitability of investments.

While the maths for IRR are quite complicated, fortunately Excel and Power BI provide simple functions for calculating IRR.

In Power BI, the [XIRR function](https://learn.microsoft.com/en-us/dax/xirr-function-dax) will return an IRR value given a table with columns for date and cashflows.

The trick then is calculate the table that used as an input.

1. Identify the Timeframe to Use 

__FirstIRRDate and __LatestIRRDate: 

Find the first and last date where IRR-related entries exist in the data (i.e. IRR journal entries). 

__StartPeriod and __EndPeriod: 

Set the date range of the current report view — from the earliest to the latest date shown in the visual/report. 

__LocalToday: 

Get today’s date for comparison. 
 

2. Pull Investment Data 

__SelectedInvestments: 

Get the list of investments being analyzed right now (based on filters or grouping on the visual). 

__CurrTransaction: 

Pull the current cash flow or value for IRR calculations (could be actuals or projections). 
 

3. Check If a Final IRR Can Be Used 

__LatestIRR: 

If the most recent IRR transaction date is before or equal to the current reporting period, and today’s date is within the reporting range, then: 

Return the final IRR as of the last actual transaction. 

This avoids projection if you already have final numbers. 
 

4. Build a Table of Cash Flows by Quarter 

__IRRTable: 

Create a table that shows cash flows for each investment, only within the reporting period. 

For the last period, it includes both MOIC (Multiple on Invested Capital) and projected cash flows to reflect expected future value. 
 

5. Return Either the Projected IRR or the Latest Known IRR

Depending on the situation:

If the report includes future periods and there are valid transactions:

Calculate a projected IRR using quarterly cash flows (XIRR).
If not: Just return the last known IRR from historical data.
