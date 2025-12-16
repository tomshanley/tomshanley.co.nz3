---
title: "Spirals"
description: "Spiral heatmaps for displaying periodic, continuous datasets."
image: {
  url: "/spiral-heatmap.jpg",
  alt:  "Spiral Heatmap"
}
draft: false
---

## Overview

Spiral heatmaps are great for displaying periodic datasets, and you want to show a continuous series without visual breaks at certain points (for example at the end and beginning of the calendar year).

The chart below is an example of a spiral heatmap built using my [d3-spiral-heatmap plug-in](https://github.com/tomshanley/d3-spiral-heatmap). The chart shows the extent of the sea ice in the northern hemisphere, with low sea ice (shown in red) starting earlier in July, and lasting longer into October and November.

<iframe width="95%" height="900" frameborder="0"
  src="https://observablehq.com/embed/@tomshanley/spiral-heatmap-northern-hemisphere-sea-ice-extent-1978-to-2?cell=chart&cell=legend"></iframe>

The spiral plug-in can also make more traditional heatmaps; for example the following chart shows car production in Germany. The data is from the [Makeover Monday series](https://www.makeovermonday.co.uk/).

<iframe width="95%" height="731" frameborder="0"
  src="https://observablehq.com/embed/@tomshanley/spiral-heatmap?cell=chart&cell=colourLegend"></iframe>