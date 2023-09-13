---
title : How can I configure the display of tooltips exclusively for the top layer in multi-layer plotly graphs? 
date : 2023-09-12 02:35:53 +0000
published: true  
---

# How can I configure the display of tooltips exclusively for the top layer in multi-layer plotly graphs?

specifically within a multi-layer plotly express timeline featuring grey blocks and blue activities, both equipped with their own tooltips? My goal is to ensure that when hovering over a blue activity on the top layer, only the tooltip associated with that activity is shown, and when hovering over a grey block on the bottom layer, only the tooltip linked to that block is displayed. Currently, I'm encountering an issue where, when hovering over certain regions of the blue activities, the grey tooltip is erroneously displayed. To illustrate this problem, I've provided a reproducible example, which notably highlights the issue when hovering over the second activity in both blocks. In these cases, the grey tooltip appears on the left portion of those activities

```python
import plotly.express as px
import pandas as pd
import plotly.graph_objects as go

df_blocks = pd.DataFrame({
    "start": ["2023-09-01 10:00:00", "2023-09-01 16:00:00"],
    "end": ["2023-09-01 20:00:00", "2023-09-01 22:00:00"],
    "y": [0, 1],
    "tooltip": ["First block", "Second block"],
    "type": ["block", "block"]
    })

df_activities = pd.DataFrame({
    "start": ["2023-09-01 10:00:00", "2023-09-01 14:00:00", "2023-09-01 16:00:00", "2023-09-01 17:00:00"],
    "end": ["2023-09-01 14:00:00", "2023-09-01 20:00:00", "2023-09-01 17:00:00", "2023-09-01 22:00:00"],
    "y": [0, 0, 1, 1],
    "tooltip": ["First activity", "Second activity", "Third activity", "Fourth activity"],
    "type": ["activity", "activity", "activity", "activity"]
    })


fig_blocks = px.timeline(df_blocks, x_start='start', x_end='end', y='y',
                           hover_data=['tooltip'],
                           color="type",
                           color_discrete_map={"block": '#8C8C8F'},
                           )
fig_activities = px.timeline(df_activities, x_start='start', x_end='end', y='y',
                           hover_data=['tooltip'],
                           color="type",
                           color_discrete_map={"activity": '#85AFFF'},
                           )
fig_activities.update_traces(width=0.5)

```
Then, I combine the two plots into one.
```python
fig_combined = go.Figure(fig_blocks)
fig_combined.add_traces(fig_activities.data)
fig_combined.show()

```
This results in the unwanted grey tooltip when hovering over the left part of the blue activities.


## The Answer

To achieve the desired behavior of showing the tooltip for the top layer only in multi-layer Plotly graphs, you can use the **event** property of Plotly's **hovertemplate** to conditionally display tooltips based on the layer. You'll need to define a custom hovertemplate for both the blocks and activities to control which tooltip is displayed.

Here's how you can modify your code to achieve this:

```python
import plotly.express as px
import pandas as pd
import plotly.graph_objects as go

# Create the DataFrame for blocks and activities as you did in your example

# Create separate figures for blocks and activities
fig_blocks = px.timeline(df_blocks, x_start='start', x_end='end', y='y',
                          hover_data=['tooltip'],
                          color="type",
                          color_discrete_map={"block": '#8C8C8F'},
                          )

fig_activities = px.timeline(df_activities, x_start='start', x_end='end', y='y',
                              hover_data=['tooltip'],
                              color="type",
                              color_discrete_map={"activity": '#85AFFF'},
                              )
fig_activities.update_traces(width=0.5)

# Define custom hovertemplates to control tooltips
block_hovertemplate = "<b>%{y}</b><br>%{hovertext}<extra></extra>"
activity_hovertemplate = "<b>%{y}</b><br>%{hovertext}<extra></extra>"

# Apply custom hovertemplates to blocks and activities
fig_blocks.update_traces(hovertemplate=block_hovertemplate)
fig_activities.update_traces(hovertemplate=activity_hovertemplate)

# Combine the two plots into one
fig_combined = go.Figure(fig_blocks)
fig_combined.add_traces(fig_activities.data)

fig_combined.show()

```

In this code, we define custom **hovertemplate** strings for both blocks and activities. The **<'extra></extra'>** part in the hovertemplate ensures that no additional tooltips are displayed for the layers beneath the top one. This way, when you hover over a blue activity (the top layer), it will only show the tooltip for the activity, and when you hover over a grey block (the bottom layer), it will show the tooltip for the block.

This should result in the desired behavior where the tooltip for the top layer is shown and no unwanted grey tooltips are displayed.