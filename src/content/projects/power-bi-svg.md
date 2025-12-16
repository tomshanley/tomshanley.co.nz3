---
title: "Power BI SVG charts"
description: "Mini charts using SVG generated from Power BI DAX measures"
liveUrl: https://example.com
githubUrl: https://github.com
image: {
    url: "/svg-bar-udf.png",
    alt:  "svg-bar-udf thumbnail"
}
draft: true
---

## Overview

Power BI DAX measures can be used to generate SVG images, which can be displayed within Power BI table visuals. These provide designers with the flexibility to create visuals ideally suited to the client's needs.

## Example 1 - progress bars

In 2024 I worked with a client who needed to display current information on their work orders to the staff working on the production lines. The display had to highlight how work orders were tracking in terms of:

- the quantity produced, as a percentage of the planned quantity
- compare quantity produced percentage to time logged to the work order, again, as a percentage to the planned time, or "standard" time

We decided that simple progress bar would be ideal for this, and I set about making a visual that would show:

1. Show the percentage of time logged, compared to the planned time (ie 100%)
2. Highligh orders, using colour and length, that were:
    1. in danger of running out of time before the required quanitity was made
    2. not completed, but had gone over their planned time
    3. complete, and had gone over their planned time
    4. OK: either their actual hours were tracking in line with the manufactured quantity, or they had completed within their planned time.

To ensure the bars have a consistent scale, we decided to cap the lengths of the bars to 150% (ie, where actual time was 150% of the planned time), so the SVG width would be the same. Any orders where actual time was greater than 150% would be signified by an arrow.

### Example bars

Work order is OK
<img src="/SVG/progress-bar-live-ok.svg" style="height:31px !important;">

Work order where actual hours is greater than the actual produced, but the time is still less than total planned
<img src="/SVG/progress-bar-live-under100.svg" style="height:31px !important;">

Work order where actual hours is greater than the actual produced, and the time is greater than total planned
<img src="/SVG/progress-bar-live-over100.svg" style="height:31px !important;">

Work order where actual hours is greater than the actual produced, the time is greater than total planned, and the actual time is greater than 150% of planned time
<img src="/SVG/progress-bar-live-over150.svg" style="height:31px !important;">

A completed work order where actual hours is greater than the actual produced, the time is greater than total planned, and the actual time is greater than 150% of planned time
<img src="/SVG/progress-bar-completed-over150.svg" style="height:31px !important;">

### The code

The key measures are the percentages of actual time recorded and actual quantity built, as a percentage of their planned counterparts:

```dax
Actual Time Pct Standard Time = 
DIVIDE(
    [Actual Time Sum],  
    [Planned Time Sum]
    )

Built Pct = 
DIVIDE(
    [Actual Built Sum], 
    [Planned Quantity Sum], 
    0)
```

Using the percentages, we assign a classification per work order:

```dax
Bar Classification = SWITCH(TRUE(),
    [Actual Time Pct Standard Time] > 1 
        && [Built Pct] < 1,                         "Not Complete - Time Over", 
    [Actual Time Pct Standard Time] > 1 
        && [Built Pct] >= 1,                        "Complete - Time Over",
    [Built Pct] >= 1,                               "Complete - OK",
    [Actual Time Pct Standard Time] > [Built Pct] , "Time > Built",
    [Actual Time Pct Standard Time] <= [Built Pct], "Time <= Built", 
    BLANK()
)
```

For each classification, we choose an appropriate colour for the progress bar:

```dax
Bar Colour Hex = SWITCH([Bar Classification],
    "Not Complete - Time Over", "#BE3A34",  //red
    "Complete - OK", "#007749",             //green
    "Complete - Time Over", "#EFB9B9",      //light red
    "Time > Built", "#E87722",              //orange
    "Time <= Built", "#007749",             //green
    "#999")                                 //grey, not used
```

Then to the fun part - making the SVG image.

- The target bar, that's used a comparison for the progress bar, is always set to 100%, and for the SVG rect, we'd decided that 100 pixels would be a nice width for the display - so that made the maths nice and easy.
- While the cut off for the bar was set to 150% (1.5), if the actual time percetnage was greater than 150%, we cut the bar's length to 140% so that there was space for the triangle arrow.
- The progress bar' height is used to create the Path element for the triangle arrow.
- A padding of 1 pixel was added the positions of the elements so there is a little white space around the visual.

```dax
Actual Time Pct SVG = 

VAR __actualPct = IF(
    [Actual Time Pct Standard Time] > 1.5, 
    1.4 * 100, 
    [Actual Time Pct Standard Time] * 100
    )

VAR __padding = 1
VAR __targetBarHeight = 30
VAR __progressBarHeight = 16
VAR __progressBarY = __padding 
    + ((__targetBarHeight - __progressBarHeight)/2)

VAR __triangleH = __progressBarHeight/2
VAR __triangleO = IF([Actual Time Pct Standard Time] > 1.5, 1, 0)

VAR __targetBarColour = "#DDD"

RETURN if(ISBLANK([Actual Time Pct Standard Time]), BLANK(),
"data:image/svg+xml;utf8,<svg 
xmlns:dc='http://purl.org/dc/elements/1.1/'
   xmlns:cc='http://creativecommons.org/ns#'
   xmlns:svg='http://www.w3.org/2000/svg'
   xmlns='http://www.w3.org/2000/svg'>
  <rect 
    width='100' 
    height='" & __targetBarHeight &"' 
    y='" & __padding & "' 
    style='fill:" & __targetBarColour & ";' 
    />
  <rect 
    x='0' 
    width='" & __actualPct & "' 
    height='" & __progressBarHeight & "' 
    y='" & __progressBarY & "' 
    style='fill: " & [Bar Colour Hex] & ";' 
    />
  <path 
    d='M" & __actualPct + 2 & "," & __progressBarY 
    & " v" & __progressBarHeight 
    & " l" &  __progressBarHeight/2 & ",-" & __triangleH 
    & " z' 
    style='fill: " & [Bar Colour Hex] & " ; 
        opacity: " & __triangleO & ";' 
    />
</svg>")
```

So while the final visuals look simple (and they needed to be so that they could be understood at a glance), the thought and coding to make them can be complex. But with SVG elements, we are not constrained by default Power BI table elements or visuals.

## DAX UDFs and DaxLib.SVG

In 2025 Microsoft released an updated to Power BI that Data Analysis Expressions (DAX) user-defined functions (UDFs), which let you package reusable, parameterized DAX logic into your models making your DAX code easier to write, maintain, and share.

To help Power BI developers share and re-use UDFs, [DAX Lib](https://daxlib.org/) provides a _centralized, open repository of reusable DAX functions and libraries that you can instantly integrate into your Fabric, Power BI, or Analysis Services models_.

[DaxLib.SVG](https://daxlib.org/package/daxlib.svg/) is collection of functions that help making SVG visuals in Power BI.

Recently I have been using this libary to make little charts in tables.

### Mini pie charts

I needed to show a metric as a percentage of the whole. I decided to extend the DaxLib.SVG circle elements to use [this approach](https://dieterpeirs.com/articles/simple-svg-pie-charts/) to making pie segments using a path's stroke-dasharray.

I adapted DaxLib.SVG's approach for compound elements (ie charts such as line charts or heatmap) to make the pie chart.

The function takes a **segmentRef** variable for the value of the segment, and **totalRef** for the value of the whole pie (inclusive of the segment). There were many ways to define this (for example, the function could just take the percentage value of the segment). But I felt my approach could allow for more flexibility in future projects.

```dax
DEFINE
	/// Creates a Pie compound SVG Visual for a numeric axis
	/// width          	INT64           	width of the compound
	/// height         	INT64           	height of the compound
	/// paddingX		DOUBLE				horizontal padding percentage 
	/// paddingY		DOUBLE				vertical padding percentage
	/// segmentRef     	NUMERIC EXPR    	measure to evaluate for segment
    /// totalRef     	NUMERIC EXPR    	measure to evaluate for the total
	/// segmentColor    STRING          	pie segment Hex  color i.e "#01B8AA"
    /// totalColor    	STRING          	remaining pie Hex color i.e "#afafaf"
	FUNCTION DaxLib.SVG.Compound.Pie = ( 
		width: INT64,
		height: INT64,
		paddingX: DOUBLE,
		paddingY: DOUBLE,
		segmentRef: NUMERIC EXPR,
		totalRef: NUMERIC EXPR,
		segmentColor: STRING,
		totalColor: STRING
	) =>

		// Apply padding to dimensions
		VAR _Width = 		width * (1 - IF(ISBLANK(paddingX), 0, paddingX))
		VAR _Height = 		height * (1 - IF(ISBLANK(paddingY), 0, paddingY))

		VAR _R =  DIVIDE(MIN(_Width, _Height), 2)
		VAR _Cx = DIVIDE(width, 2)
		VAR _Cy = DIVIDE(height, 2)
		
		VAR _SegmentPct = DIVIDE(segmentRef, totalRef)
		VAR _SegmentCircumference = 2 * PI() * DIVIDE(_R, 2)
		VAR _Dash = _SegmentCircumference * _SegmentPct
		VAR _Gap = _SegmentCircumference - _Dash
		VAR _DashArray = "stroke-dasharray='" & _Dash & " " & _Gap & "' "

		// Base Circle
		VAR _BaseCircle =
			DaxLib.SVG.Element.Circle(
				_Cx, // cx
				_Cy, // cy
				_R,  // r
				DaxLib.SVG.Attr.Shapes(
					totalColor, // fill
					BLANK(),    // fillOpacity
					BLANK(),    // fillRule
					BLANK(),    // stroke
					BLANK(),    // strokeWidth
					BLANK(),    // strokeOpacity
					BLANK()     // opacity
				),
				BLANK()         // transforms
			)
		
		//Segment
		VAR _SegmentCircle = 
			DaxLib.SVG.Element.Circle(
				_Cx,            // cx
				_Cy,            // cy
				DIVIDE(_R, 2),  // r
				DaxLib.SVG.Attr.Shapes(
					BLANK(), 	    // fill
					0,              // fillOpacity
					BLANK(),        // fillRule
					segmentColor,   // stroke
					_R,             // strokeWidth
					BLANK(),        // strokeOpacity
					BLANK()         // opacity
				),
				DaxLib.SVG.Transforms(
					BLANK(),
					"-90, " & _Cx & ", " & _Cy,
					BLANK(),
					BLANK(),
					BLANK()
				)
			)
		
		//insert the dash array to the segment element
		VAR _Segment = SUBSTITUTE(
            _SegmentCircle, 
            "stroke-width", 
            _DashArray & "stroke-width" 
            )

		// Return both elements
		RETURN _BaseCircle & IF(ISBLANK(segmentRef) = FALSE(), _Segment)
```


### Mini bar charts

