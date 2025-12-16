---
title: "Sankey Plus"
description: "Enhanced sankey visual that handles circular flows, improves link routing and show arrows."
image: {
    url: "/sankey-thumb.jpg",
    alt:  "sankey thumbnail"
}
draft: false
---

## Overview

In 2022 I published the [Sankey Plus library](https://github.com/tomshanley/sankey-plus), which improved upon my previous [Sankey Circular library](https://github.com/tomshanley/d3-sankey-circular).

The new Sankey Plus library introduced the following new features, and was incorporates feedback and suggestions from [Ralph Spandl](https://x.com/ralph_spandl) as he integrated this libarary with Google Data Studio:

- the concept of virtual nodes and links, which allows for nice routing of links around other nodes
- arrows on the links to show direction
- a new API that allows for easy sankey generation and drawing within a specified DOM element.

You can see the new features in action in this [Observable notebook](https://observablehq.com/@tomshanley/introducing-sankey-plus).
