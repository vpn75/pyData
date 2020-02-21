<style>
.vega-actions a {
margin-right: 12px;
color: #757575;
font-weight: normal;
font-size: 13px;
}
.error {
color: red;
}
</style>
<script src="https://cdn.jsdelivr.net/npm//vega@3.3.1"></script>
<script src="https://cdn.jsdelivr.net/npm//vega-lite@2.4.3"></script>
<script src="https://cdn.jsdelivr.net/npm//vega-embed@3.11"></script>

# Arsenal - Squad Profile Analysis

This is an analytics exercise to look at distribution of minutes across the different age-groups in the squad. 

Arsenal are entering a rebuilding phase and have a promising crop of academy players and other young talents. 

I'm interested to see if the squad is trending younger and if we are giving our youth talent enough game-time to properly develop them.

I did a similar [exercise](https://github.com/vpn75/squad_age_profile/blob/master/squad_age_profile_analysis.md) using R and I wanted to try and recreate it in Python. I will also be using a newer data-viz library, [Altair](https://altair-viz.github.io/getting_started/overview.html) that I've been interested in learning.


```python
#Package imports
import requests
from bs4 import BeautifulSoup
import pandas as pd
import numpy as np
import altair as alt
import re
import json
```

We'll start by scraping the squad data for Arsenal in the Premier League this season.


```python
url = 'http://us.soccerway.com/teams/england/arsenal-fc/660/squad/'

html = requests.get(url).text

soup = BeautifulSoup(html, 'html.parser')

data = []

table = soup.find_all('table')[0]
trs = table.find_all('tr')[1:] #Skip header row

for tr in trs:
    player = [c.text.strip() for c in tr.find_all('td')[2:8]]
    player.pop(1) #Remove 2nd list element containing nationality flag
    data.append(player)

```

Now that we have our scraped data, we'll convert it to a Pandas dataframe.


```python
df = pd.DataFrame(data)
df.columns = ['player', 'age', 'position', 'minutes', 'appearances']
```

We'll need to convert some of the columns to numeric for our analysis.


```python
#Convert cols to numeric
cols = ['age','minutes','appearances']
df[cols] = df[cols].apply(pd.to_numeric)

#Confirm dtype conversion
df.dtypes
```




    player         object
    age             int64
    position       object
    minutes         int64
    appearances     int64
    dtype: object



Let's filter out any players that did not feature in the league.


```python
df = df[df.appearances > 0]
```

We'll do some more data-cleaning by converting the **position** column to a categorical value and re-classify them to make it more readable.

Also noticed that Maitland-Niles was classified as a Midfielder for some reason when he had made most of his appearances at RB so changed his position to Defense.

We'll also update name of defender, Sokratis to improve display in our final viz.


```python
df.loc[df.player == 'A. Maitland-Niles', ['position']] = 'D'
df['position'] = df['position'].map({'A':'Forward','G':'Goalie','D':'Defense', 'M':'Midfield'}).astype('category')
df['player'] = df['player'].str.replace('S. Papastathopoulos','Sokratis')

df.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>player</th>
      <th>age</th>
      <th>position</th>
      <th>minutes</th>
      <th>appearances</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>B. Leno</td>
      <td>27</td>
      <td>Goalie</td>
      <td>2340</td>
      <td>26</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Bellerín</td>
      <td>24</td>
      <td>Defense</td>
      <td>533</td>
      <td>6</td>
    </tr>
    <tr>
      <th>4</th>
      <td>K. Tierney</td>
      <td>22</td>
      <td>Defense</td>
      <td>299</td>
      <td>5</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Sokratis</td>
      <td>31</td>
      <td>Defense</td>
      <td>1607</td>
      <td>18</td>
    </tr>
    <tr>
      <th>6</th>
      <td>R. Holding</td>
      <td>24</td>
      <td>Defense</td>
      <td>86</td>
      <td>2</td>
    </tr>
    <tr>
      <th>7</th>
      <td>S. Mustafi</td>
      <td>27</td>
      <td>Defense</td>
      <td>530</td>
      <td>7</td>
    </tr>
    <tr>
      <th>8</th>
      <td>C. Chambers</td>
      <td>25</td>
      <td>Defense</td>
      <td>1103</td>
      <td>14</td>
    </tr>
    <tr>
      <th>9</th>
      <td>David Luiz</td>
      <td>32</td>
      <td>Defense</td>
      <td>2006</td>
      <td>23</td>
    </tr>
    <tr>
      <th>10</th>
      <td>S. Kolašinac</td>
      <td>26</td>
      <td>Defense</td>
      <td>1089</td>
      <td>16</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Dani Ceballos</td>
      <td>23</td>
      <td>Midfield</td>
      <td>745</td>
      <td>12</td>
    </tr>
    <tr>
      <th>16</th>
      <td>M. Özil</td>
      <td>31</td>
      <td>Midfield</td>
      <td>1278</td>
      <td>16</td>
    </tr>
    <tr>
      <th>17</th>
      <td>L. Torreira</td>
      <td>24</td>
      <td>Midfield</td>
      <td>1357</td>
      <td>23</td>
    </tr>
    <tr>
      <th>18</th>
      <td>A. Maitland-Niles</td>
      <td>22</td>
      <td>Defense</td>
      <td>1211</td>
      <td>14</td>
    </tr>
    <tr>
      <th>19</th>
      <td>J. Willock</td>
      <td>20</td>
      <td>Midfield</td>
      <td>576</td>
      <td>19</td>
    </tr>
    <tr>
      <th>20</th>
      <td>M. Guendouzi</td>
      <td>20</td>
      <td>Midfield</td>
      <td>1582</td>
      <td>21</td>
    </tr>
    <tr>
      <th>21</th>
      <td>G. Xhaka</td>
      <td>27</td>
      <td>Midfield</td>
      <td>1728</td>
      <td>20</td>
    </tr>
    <tr>
      <th>22</th>
      <td>B. Saka</td>
      <td>18</td>
      <td>Midfield</td>
      <td>1014</td>
      <td>16</td>
    </tr>
    <tr>
      <th>25</th>
      <td>A. Lacazette</td>
      <td>28</td>
      <td>Forward</td>
      <td>1287</td>
      <td>19</td>
    </tr>
    <tr>
      <th>26</th>
      <td>P. Aubameyang</td>
      <td>30</td>
      <td>Forward</td>
      <td>2125</td>
      <td>24</td>
    </tr>
    <tr>
      <th>27</th>
      <td>N. Pépé</td>
      <td>24</td>
      <td>Forward</td>
      <td>1432</td>
      <td>22</td>
    </tr>
    <tr>
      <th>28</th>
      <td>R. Nelson</td>
      <td>20</td>
      <td>Forward</td>
      <td>440</td>
      <td>10</td>
    </tr>
    <tr>
      <th>29</th>
      <td>E. Nketiah</td>
      <td>20</td>
      <td>Forward</td>
      <td>102</td>
      <td>3</td>
    </tr>
    <tr>
      <th>30</th>
      <td>Martinelli</td>
      <td>18</td>
      <td>Forward</td>
      <td>656</td>
      <td>14</td>
    </tr>
  </tbody>
</table>
</div>



Next, we'll add a new column that calculates the percentage of total league-minutes played per player.


```python
gp = df.appearances.max()

df['mp_pct'] = round(df['minutes']/(gp*90), 2)
df.head()
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>player</th>
      <th>age</th>
      <th>position</th>
      <th>minutes</th>
      <th>appearances</th>
      <th>mp_pct</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>B. Leno</td>
      <td>27</td>
      <td>Goalie</td>
      <td>2340</td>
      <td>26</td>
      <td>1.00</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Bellerín</td>
      <td>24</td>
      <td>Defense</td>
      <td>533</td>
      <td>6</td>
      <td>0.23</td>
    </tr>
    <tr>
      <th>4</th>
      <td>K. Tierney</td>
      <td>22</td>
      <td>Defense</td>
      <td>299</td>
      <td>5</td>
      <td>0.13</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Sokratis</td>
      <td>31</td>
      <td>Defense</td>
      <td>1607</td>
      <td>18</td>
      <td>0.69</td>
    </tr>
    <tr>
      <th>6</th>
      <td>R. Holding</td>
      <td>24</td>
      <td>Defense</td>
      <td>86</td>
      <td>2</td>
      <td>0.04</td>
    </tr>
  </tbody>
</table>
</div>



I wasn't happy with the way the player names looked. 

Some had first initial but others like Bellerin, just had their lastname. 

Let's split out the name so we can separate the last name from the first initial.


```python
tdf = df.reset_index(drop=True).copy()

tdf[['fname','lname']] = tdf.player.str.split(expand=True)

tdf.head()
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>player</th>
      <th>age</th>
      <th>position</th>
      <th>minutes</th>
      <th>appearances</th>
      <th>mp_pct</th>
      <th>fname</th>
      <th>lname</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>B. Leno</td>
      <td>27</td>
      <td>Goalie</td>
      <td>2340</td>
      <td>26</td>
      <td>1.00</td>
      <td>B.</td>
      <td>Leno</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Bellerín</td>
      <td>24</td>
      <td>Defense</td>
      <td>533</td>
      <td>6</td>
      <td>0.23</td>
      <td>Bellerín</td>
      <td>None</td>
    </tr>
    <tr>
      <th>2</th>
      <td>K. Tierney</td>
      <td>22</td>
      <td>Defense</td>
      <td>299</td>
      <td>5</td>
      <td>0.13</td>
      <td>K.</td>
      <td>Tierney</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Sokratis</td>
      <td>31</td>
      <td>Defense</td>
      <td>1607</td>
      <td>18</td>
      <td>0.69</td>
      <td>Sokratis</td>
      <td>None</td>
    </tr>
    <tr>
      <th>4</th>
      <td>R. Holding</td>
      <td>24</td>
      <td>Defense</td>
      <td>86</td>
      <td>2</td>
      <td>0.04</td>
      <td>R.</td>
      <td>Holding</td>
    </tr>
  </tbody>
</table>
</div>



OK, looking better but we can see we have an issue players that did not have first initials. They have NA values for lname now so let's clean them up.

For the players with nulls in lname, we'll just replace with the value from **player**.


```python
idx = tdf[tdf.lname.isna()].index.values

for val in idx:
    tdf.at[val,'lname'] = tdf.at[val, 'player']
    
tdf.drop('fname', axis=1, inplace=True)
tdf
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>player</th>
      <th>age</th>
      <th>position</th>
      <th>minutes</th>
      <th>appearances</th>
      <th>mp_pct</th>
      <th>lname</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>B. Leno</td>
      <td>27</td>
      <td>Goalie</td>
      <td>2340</td>
      <td>26</td>
      <td>1.00</td>
      <td>Leno</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Bellerín</td>
      <td>24</td>
      <td>Defense</td>
      <td>533</td>
      <td>6</td>
      <td>0.23</td>
      <td>Bellerín</td>
    </tr>
    <tr>
      <th>2</th>
      <td>K. Tierney</td>
      <td>22</td>
      <td>Defense</td>
      <td>299</td>
      <td>5</td>
      <td>0.13</td>
      <td>Tierney</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Sokratis</td>
      <td>31</td>
      <td>Defense</td>
      <td>1607</td>
      <td>18</td>
      <td>0.69</td>
      <td>Sokratis</td>
    </tr>
    <tr>
      <th>4</th>
      <td>R. Holding</td>
      <td>24</td>
      <td>Defense</td>
      <td>86</td>
      <td>2</td>
      <td>0.04</td>
      <td>Holding</td>
    </tr>
    <tr>
      <th>5</th>
      <td>S. Mustafi</td>
      <td>27</td>
      <td>Defense</td>
      <td>530</td>
      <td>7</td>
      <td>0.23</td>
      <td>Mustafi</td>
    </tr>
    <tr>
      <th>6</th>
      <td>C. Chambers</td>
      <td>25</td>
      <td>Defense</td>
      <td>1103</td>
      <td>14</td>
      <td>0.47</td>
      <td>Chambers</td>
    </tr>
    <tr>
      <th>7</th>
      <td>David Luiz</td>
      <td>32</td>
      <td>Defense</td>
      <td>2006</td>
      <td>23</td>
      <td>0.86</td>
      <td>Luiz</td>
    </tr>
    <tr>
      <th>8</th>
      <td>S. Kolašinac</td>
      <td>26</td>
      <td>Defense</td>
      <td>1089</td>
      <td>16</td>
      <td>0.47</td>
      <td>Kolašinac</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Dani Ceballos</td>
      <td>23</td>
      <td>Midfield</td>
      <td>745</td>
      <td>12</td>
      <td>0.32</td>
      <td>Ceballos</td>
    </tr>
    <tr>
      <th>10</th>
      <td>M. Özil</td>
      <td>31</td>
      <td>Midfield</td>
      <td>1278</td>
      <td>16</td>
      <td>0.55</td>
      <td>Özil</td>
    </tr>
    <tr>
      <th>11</th>
      <td>L. Torreira</td>
      <td>24</td>
      <td>Midfield</td>
      <td>1357</td>
      <td>23</td>
      <td>0.58</td>
      <td>Torreira</td>
    </tr>
    <tr>
      <th>12</th>
      <td>A. Maitland-Niles</td>
      <td>22</td>
      <td>Defense</td>
      <td>1211</td>
      <td>14</td>
      <td>0.52</td>
      <td>Maitland-Niles</td>
    </tr>
    <tr>
      <th>13</th>
      <td>J. Willock</td>
      <td>20</td>
      <td>Midfield</td>
      <td>576</td>
      <td>19</td>
      <td>0.25</td>
      <td>Willock</td>
    </tr>
    <tr>
      <th>14</th>
      <td>M. Guendouzi</td>
      <td>20</td>
      <td>Midfield</td>
      <td>1582</td>
      <td>21</td>
      <td>0.68</td>
      <td>Guendouzi</td>
    </tr>
    <tr>
      <th>15</th>
      <td>G. Xhaka</td>
      <td>27</td>
      <td>Midfield</td>
      <td>1728</td>
      <td>20</td>
      <td>0.74</td>
      <td>Xhaka</td>
    </tr>
    <tr>
      <th>16</th>
      <td>B. Saka</td>
      <td>18</td>
      <td>Midfield</td>
      <td>1014</td>
      <td>16</td>
      <td>0.43</td>
      <td>Saka</td>
    </tr>
    <tr>
      <th>17</th>
      <td>A. Lacazette</td>
      <td>28</td>
      <td>Forward</td>
      <td>1287</td>
      <td>19</td>
      <td>0.55</td>
      <td>Lacazette</td>
    </tr>
    <tr>
      <th>18</th>
      <td>P. Aubameyang</td>
      <td>30</td>
      <td>Forward</td>
      <td>2125</td>
      <td>24</td>
      <td>0.91</td>
      <td>Aubameyang</td>
    </tr>
    <tr>
      <th>19</th>
      <td>N. Pépé</td>
      <td>24</td>
      <td>Forward</td>
      <td>1432</td>
      <td>22</td>
      <td>0.61</td>
      <td>Pépé</td>
    </tr>
    <tr>
      <th>20</th>
      <td>R. Nelson</td>
      <td>20</td>
      <td>Forward</td>
      <td>440</td>
      <td>10</td>
      <td>0.19</td>
      <td>Nelson</td>
    </tr>
    <tr>
      <th>21</th>
      <td>E. Nketiah</td>
      <td>20</td>
      <td>Forward</td>
      <td>102</td>
      <td>3</td>
      <td>0.04</td>
      <td>Nketiah</td>
    </tr>
    <tr>
      <th>22</th>
      <td>Martinelli</td>
      <td>18</td>
      <td>Forward</td>
      <td>656</td>
      <td>14</td>
      <td>0.28</td>
      <td>Martinelli</td>
    </tr>
  </tbody>
</table>
</div>



## Building the viz

OK, now the names look good and we're ready to build our squad-profile viz.

In the past, I have been reluctant to use Python for data visualization because I was not a fan of matplotlib.

Coming more from the Rstats world, I missed the declarative syntax of the awesome ggplot library.

So, I was really happy when I came across the Altair package which is just a Python wrapper for [Vega-lite](https://vega.github.io/vega-lite/)

It has an elegant, declarative syntax that utilizes the "grammar of graphics" style that I loved from ggplot2.

I also find the default visual output to be more aesthetically pleasing than matplotlib.

## Viz components

Our squad profile viz will consist of 3 components:

 * Scatter plot of players with Age on **x-axis** and % played of max league minutes on **y-axis**
 * Player name labels 
 * Shaded region indicating peak-age bracket
 
We'll start by creating our base-layer that creates our scatter plot. 

I added some customization to the **y-axis** to display in percent-format. 

I also tweaked the range of the axis-ticks to improve display.


```python
base = alt.Chart(tdf).encode(
    alt.X('age:Q',
          scale=alt.Scale(domain=[16, 34]),
          axis=alt.Axis(values=[16,18,20,22,24,26,28,30,32,34,36]),
          title='Player Age'
         ),
    alt.Y('mp_pct:Q',
        axis=alt.Axis(format='%', 
        title='% played of max league minutes',
        values=[.1,.2,.3,.4,.5,.6,.7,.8,.9,1]),
        scale=alt.Scale(domain=[0, 1.01]),
        )
)
```

Next, we'll add the marks that will create our scatter-plot. I've added a color legend based on player position using the categories we created earlier. 


```python
points = base.mark_circle(size=120).encode(
    color=alt.Color(
        'position',
        title='Position', 
        scale=alt.Scale(domain=(['Goalie','Defense','Midfield','Forward']))
    )
)
```

Next, comes the player-name labels. I used the lastname rather than the playername with first initial for cleaner display.


```python
text = base.mark_text(
    align='left',
    baseline='middle',
    dx=10, # Nudges text to right so it doesn't appear on top of the bar
    font='Menlo'
).encode(
    text='lname'
)
```

This layer will create the shaded rectangular region that will indicate the peak-age group. 

One thing I discovered was that I couldn't use age-values from the **x-axis** in the encoding specification. 

Rather, I had to pick the values based on the overall width of the final viz that you will see specified further down. 

This was a bit counter-intuitive and worked differently from ggplot.

I defined the peak-age range as 24-30 years old.


```python
rect = alt.Chart(tdf).mark_rect(fill='green').encode(
    x=alt.value(266),
    x2=alt.value(468),
    opacity=alt.value(0.01)
)
```

Finally, we'll bring all the layers together and add our final customizations including font and font-size.


```python
final_viz = (rect+points+text)

final_viz.properties(
    width=600, 
    height=450, 
    title='Arsenal - Squad Profile Analysis'
).configure_title(
        fontSize=18, 
        orient='top', 
        anchor='start', 
        offset=10, 
        font='Menlo'
).configure_axis(
        titleFont='Menlo',
        titleFontSize=12
).configure_legend(
        titleFont='Menlo',
        labelFont='Menlo'
).display(renderer='svg')
```


<div id="vis"></div>
  <script>
(function(vegaEmbed) {
var spec = {"config": {"view": {"width": 400, "height": 300}, "mark": {"tooltip": null}}, "layer": [{"mark": {"type": "rect", "fill": "green"}, "encoding": {"opacity": {"value": 0.01}, "x": {"value": 266}, "x2": {"value": 468}}}, {"mark": {"type": "circle", "size": 120}, "encoding": {"color": {"type": "nominal", "field": "position", "scale": {"domain": ["Goalie", "Defense", "Midfield", "Forward"]}, "title": "Position"}, "x": {"type": "quantitative", "axis": {"values": [16, 18, 20, 22, 24, 26, 28, 30, 32, 34, 36]}, "field": "age", "scale": {"domain": [16, 34]}, "title": "Player Age"}, "y": {"type": "quantitative", "axis": {"format": "%", "title": "% played of max league minutes", "values": [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1]}, "field": "mp_pct", "scale": {"domain": [0, 1.01]}}}}, {"mark": {"type": "text", "align": "left", "baseline": "middle", "dx": 10, "font": "Menlo"}, "encoding": {"text": {"type": "nominal", "field": "lname"}, "x": {"type": "quantitative", "axis": {"values": [16, 18, 20, 22, 24, 26, 28, 30, 32, 34, 36]}, "field": "age", "scale": {"domain": [16, 34]}, "title": "Player Age"}, "y": {"type": "quantitative", "axis": {"format": "%", "title": "% played of max league minutes", "values": [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1]}, "field": "mp_pct", "scale": {"domain": [0, 1.01]}}}}], "data": {"name": "data-92e15a500a04a9596cba551744fecce8"}, "$schema": "https://vega.github.io/schema/vega-lite/v3.4.0.json", "datasets": {"data-92e15a500a04a9596cba551744fecce8": [{"player": "B. Leno", "age": 27, "position": "Goalie", "minutes": 2340, "appearances": 26, "mp_pct": 1.0, "lname": "Leno"}, {"player": "Beller\u00edn", "age": 24, "position": "Defense", "minutes": 533, "appearances": 6, "mp_pct": 0.23, "lname": "Beller\u00edn"}, {"player": "K. Tierney", "age": 22, "position": "Defense", "minutes": 299, "appearances": 5, "mp_pct": 0.13, "lname": "Tierney"}, {"player": "Sokratis", "age": 31, "position": "Defense", "minutes": 1607, "appearances": 18, "mp_pct": 0.69, "lname": "Sokratis"}, {"player": "R. Holding", "age": 24, "position": "Defense", "minutes": 86, "appearances": 2, "mp_pct": 0.04, "lname": "Holding"}, {"player": "S. Mustafi", "age": 27, "position": "Defense", "minutes": 530, "appearances": 7, "mp_pct": 0.23, "lname": "Mustafi"}, {"player": "C. Chambers", "age": 25, "position": "Defense", "minutes": 1103, "appearances": 14, "mp_pct": 0.47, "lname": "Chambers"}, {"player": "David Luiz", "age": 32, "position": "Defense", "minutes": 2006, "appearances": 23, "mp_pct": 0.86, "lname": "Luiz"}, {"player": "S. Kola\u0161inac", "age": 26, "position": "Defense", "minutes": 1089, "appearances": 16, "mp_pct": 0.47, "lname": "Kola\u0161inac"}, {"player": "Dani Ceballos", "age": 23, "position": "Midfield", "minutes": 745, "appearances": 12, "mp_pct": 0.32, "lname": "Ceballos"}, {"player": "M. \u00d6zil", "age": 31, "position": "Midfield", "minutes": 1278, "appearances": 16, "mp_pct": 0.55, "lname": "\u00d6zil"}, {"player": "L. Torreira", "age": 24, "position": "Midfield", "minutes": 1357, "appearances": 23, "mp_pct": 0.58, "lname": "Torreira"}, {"player": "A. Maitland-Niles", "age": 22, "position": "Defense", "minutes": 1211, "appearances": 14, "mp_pct": 0.52, "lname": "Maitland-Niles"}, {"player": "J. Willock", "age": 20, "position": "Midfield", "minutes": 576, "appearances": 19, "mp_pct": 0.25, "lname": "Willock"}, {"player": "M. Guendouzi", "age": 20, "position": "Midfield", "minutes": 1582, "appearances": 21, "mp_pct": 0.68, "lname": "Guendouzi"}, {"player": "G. Xhaka", "age": 27, "position": "Midfield", "minutes": 1728, "appearances": 20, "mp_pct": 0.74, "lname": "Xhaka"}, {"player": "B. Saka", "age": 18, "position": "Midfield", "minutes": 1014, "appearances": 16, "mp_pct": 0.43, "lname": "Saka"}, {"player": "A. Lacazette", "age": 28, "position": "Forward", "minutes": 1287, "appearances": 19, "mp_pct": 0.55, "lname": "Lacazette"}, {"player": "P. Aubameyang", "age": 30, "position": "Forward", "minutes": 2125, "appearances": 24, "mp_pct": 0.91, "lname": "Aubameyang"}, {"player": "N. P\u00e9p\u00e9", "age": 24, "position": "Forward", "minutes": 1432, "appearances": 22, "mp_pct": 0.61, "lname": "P\u00e9p\u00e9"}, {"player": "R. Nelson", "age": 20, "position": "Forward", "minutes": 440, "appearances": 10, "mp_pct": 0.19, "lname": "Nelson"}, {"player": "E. Nketiah", "age": 20, "position": "Forward", "minutes": 102, "appearances": 3, "mp_pct": 0.04, "lname": "Nketiah"}, {"player": "Martinelli", "age": 18, "position": "Forward", "minutes": 656, "appearances": 14, "mp_pct": 0.28, "lname": "Martinelli"}]}};
var embedOpt = {"mode": "vega-lite"};

function showError(el, error){
el.innerHTML = ('<div class="error" style="color:red;">'
+ '<p>JavaScript Error: ' + error.message + '</p>'
+ "<p>This usually means there's a typo in your chart specification. "
+ "See the javascript console for the full traceback.</p>"
+ '</div>');
throw error;
}
const el = document.getElementById('vis');
vegaEmbed("#vis", spec, embedOpt)
  .catch(error => showError(el, error));
})(vegaEmbed);
</script>

## Thoughts

As my first foray into Altair, I was pretty happy with how the final viz turned out. 

I wish Altair supported something similar to the **ggrepl** package in R that could more intelligently space out the labels to avoid overlap.

Another source of frustration was the inability to add a subtitle. 

Apparently Vega-lite was patched recently to support this, but I could not get it to work in my environment for some reason in spite of confirming I had the latest versions of Altair/Vega


## Bonus analysis

Let's continue our analysis on the dataset.

We can look at the average age by position.


```python
tdf.groupby('position').age.mean()
```




    position
    Defense     25.888889
    Forward     23.333333
    Goalie      27.000000
    Midfield    23.285714
    Name: age, dtype: float64



Let's create another viz that shows distribution of minutes by player.

We'll create a sorted bar-chart. 

As a bonus, we'll take advantage of Altair's interactive capabilities by adding a tooltip that appears as you hover over the player bar to display additional information.


```python

chart = alt.Chart(df).mark_bar(color='firebrick').encode(
    x='minutes:Q',
    y=alt.Y(
        'player',
        sort=alt.SortField(field="minutes", order='descending')
    ),
    tooltip = ['player', 'age', 'appearances', 'minutes']
)

chart.properties(
    title={
        'text':'Arsenal league minutes by player - Through Matchday 25'
    }
).display(renderer='svg')

```
<div id="vis"></div>
<script>
(function(vegaEmbed) {
var spec = {"config": {"view": {"width": 400, "height": 300}, "mark": {"tooltip": null}}, "data": {"name": "data-549f4a1ea1f81d17ce047298969b09fc"}, "mark": {"type": "bar", "color": "firebrick"}, "encoding": {"tooltip": [{"type": "nominal", "field": "player"}, {"type": "quantitative", "field": "age"}, {"type": "quantitative", "field": "appearances"}, {"type": "quantitative", "field": "minutes"}], "x": {"type": "quantitative", "field": "minutes"}, "y": {"type": "nominal", "field": "player", "sort": {"field": "minutes", "order": "descending"}}}, "$schema": "https://vega.github.io/schema/vega-lite/v3.4.0.json", "datasets": {"data-549f4a1ea1f81d17ce047298969b09fc": [{"player": "B. Leno", "age": 27, "position": "Goalie", "minutes": 2340, "appearances": 26, "mp_pct": 1.0}, {"player": "Beller\u00edn", "age": 24, "position": "Defense", "minutes": 533, "appearances": 6, "mp_pct": 0.23}, {"player": "K. Tierney", "age": 22, "position": "Defense", "minutes": 299, "appearances": 5, "mp_pct": 0.13}, {"player": "Sokratis", "age": 31, "position": "Defense", "minutes": 1607, "appearances": 18, "mp_pct": 0.69}, {"player": "R. Holding", "age": 24, "position": "Defense", "minutes": 86, "appearances": 2, "mp_pct": 0.04}, {"player": "S. Mustafi", "age": 27, "position": "Defense", "minutes": 530, "appearances": 7, "mp_pct": 0.23}, {"player": "C. Chambers", "age": 25, "position": "Defense", "minutes": 1103, "appearances": 14, "mp_pct": 0.47}, {"player": "David Luiz", "age": 32, "position": "Defense", "minutes": 2006, "appearances": 23, "mp_pct": 0.86}, {"player": "S. Kola\u0161inac", "age": 26, "position": "Defense", "minutes": 1089, "appearances": 16, "mp_pct": 0.47}, {"player": "Dani Ceballos", "age": 23, "position": "Midfield", "minutes": 745, "appearances": 12, "mp_pct": 0.32}, {"player": "M. \u00d6zil", "age": 31, "position": "Midfield", "minutes": 1278, "appearances": 16, "mp_pct": 0.55}, {"player": "L. Torreira", "age": 24, "position": "Midfield", "minutes": 1357, "appearances": 23, "mp_pct": 0.58}, {"player": "A. Maitland-Niles", "age": 22, "position": "Defense", "minutes": 1211, "appearances": 14, "mp_pct": 0.52}, {"player": "J. Willock", "age": 20, "position": "Midfield", "minutes": 576, "appearances": 19, "mp_pct": 0.25}, {"player": "M. Guendouzi", "age": 20, "position": "Midfield", "minutes": 1582, "appearances": 21, "mp_pct": 0.68}, {"player": "G. Xhaka", "age": 27, "position": "Midfield", "minutes": 1728, "appearances": 20, "mp_pct": 0.74}, {"player": "B. Saka", "age": 18, "position": "Midfield", "minutes": 1014, "appearances": 16, "mp_pct": 0.43}, {"player": "A. Lacazette", "age": 28, "position": "Forward", "minutes": 1287, "appearances": 19, "mp_pct": 0.55}, {"player": "P. Aubameyang", "age": 30, "position": "Forward", "minutes": 2125, "appearances": 24, "mp_pct": 0.91}, {"player": "N. P\u00e9p\u00e9", "age": 24, "position": "Forward", "minutes": 1432, "appearances": 22, "mp_pct": 0.61}, {"player": "R. Nelson", "age": 20, "position": "Forward", "minutes": 440, "appearances": 10, "mp_pct": 0.19}, {"player": "E. Nketiah", "age": 20, "position": "Forward", "minutes": 102, "appearances": 3, "mp_pct": 0.04}, {"player": "Martinelli", "age": 18, "position": "Forward", "minutes": 656, "appearances": 14, "mp_pct": 0.28}]}};
var embedOpt = {"mode": "vega-lite"};

function showError(el, error){
el.innerHTML = ('<div class="error" style="color:red;">'
+ '<p>JavaScript Error: ' + error.message + '</p>'
+ "<p>This usually means there's a typo in your chart specification. "
+ "See the javascript console for the full traceback.</p>"
+ '</div>');
throw error;
}
const el = document.getElementById('vis');
vegaEmbed("#vis", spec, embedOpt)
.catch(error => showError(el, error));
})(vegaEmbed);
</script>


It could also be interesting to see the distribution of minutes across different age-groups in the squad.

We'll drop the goalie position since only Leno has featured for Arsenal in the league this season.


```python
df2 = df.copy()

def age_group(df):
    
    if df.age >= 16 and df.age < 24:
        df['age_group'] = 'Youth'
    elif df.age >= 24 and df.age <= 30:
        df['age_group'] = 'Prime'
    else: 
        df['age_group'] = 'Senior'
    return df

#df2.assign(age_group=age_group(df))
df2 = df2.apply(age_group, axis=1)
df2.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>player</th>
      <th>age</th>
      <th>position</th>
      <th>minutes</th>
      <th>appearances</th>
      <th>mp_pct</th>
      <th>age_group</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>B. Leno</td>
      <td>27</td>
      <td>Goalie</td>
      <td>2340</td>
      <td>26</td>
      <td>1.00</td>
      <td>Prime</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Bellerín</td>
      <td>24</td>
      <td>Defense</td>
      <td>533</td>
      <td>6</td>
      <td>0.23</td>
      <td>Prime</td>
    </tr>
    <tr>
      <th>4</th>
      <td>K. Tierney</td>
      <td>22</td>
      <td>Defense</td>
      <td>299</td>
      <td>5</td>
      <td>0.13</td>
      <td>Youth</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Sokratis</td>
      <td>31</td>
      <td>Defense</td>
      <td>1607</td>
      <td>18</td>
      <td>0.69</td>
      <td>Senior</td>
    </tr>
    <tr>
      <th>6</th>
      <td>R. Holding</td>
      <td>24</td>
      <td>Defense</td>
      <td>86</td>
      <td>2</td>
      <td>0.04</td>
      <td>Prime</td>
    </tr>
  </tbody>
</table>
</div>



We can visualize this information using a stacked bar-chart and take advantage of Altair's powerful aggregation capabilities.


```python
df2.drop

alt.Chart(df2, title='Minutes distribution by age group').mark_bar().encode(
    x=alt.X('total_minutes:Q', title='Total Minutes Played'),
    y=alt.Y(
        'position:O', 
        title='',
        sort=['Goalie','Forward','Midfield','Defense']
    ),
    color=alt.Color(
        'age_group:N', 
        legend=alt.Legend(title='Age Group'),
        scale=alt.Scale(domain=(['Youth','Prime','Senior']))
    )
).transform_aggregate(
    total_minutes='sum(minutes)',
    groupby=['position','age_group']
).display()

```

<div id="vis"></div>
<script>
(function(vegaEmbed) {
var spec = {"config": {"view": {"width": 400, "height": 300}, "mark": {"tooltip": null}}, "data": {"name": "data-42f0fb27cade2c46fca14ccb96f56799"}, "mark": "bar", "encoding": {"color": {"type": "nominal", "field": "age_group", "legend": {"title": "Age Group"}, "scale": {"domain": ["Youth", "Prime", "Senior"]}}, "x": {"type": "quantitative", "field": "total_minutes", "title": "Total Minutes Played"}, "y": {"type": "ordinal", "field": "position", "sort": ["Goalie", "Forward", "Midfield", "Defense"], "title": ""}}, "title": "Minutes distribution by age group", "transform": [{"aggregate": [{"op": "sum", "field": "minutes", "as": "total_minutes"}], "groupby": ["position", "age_group"]}], "$schema": "https://vega.github.io/schema/vega-lite/v3.4.0.json", "datasets": {"data-42f0fb27cade2c46fca14ccb96f56799": [{"player": "B. Leno", "age": 27, "position": "Goalie", "minutes": 2340, "appearances": 26, "mp_pct": 1.0, "age_group": "Prime"}, {"player": "Beller\u00edn", "age": 24, "position": "Defense", "minutes": 533, "appearances": 6, "mp_pct": 0.23, "age_group": "Prime"}, {"player": "K. Tierney", "age": 22, "position": "Defense", "minutes": 299, "appearances": 5, "mp_pct": 0.13, "age_group": "Youth"}, {"player": "Sokratis", "age": 31, "position": "Defense", "minutes": 1607, "appearances": 18, "mp_pct": 0.69, "age_group": "Senior"}, {"player": "R. Holding", "age": 24, "position": "Defense", "minutes": 86, "appearances": 2, "mp_pct": 0.04, "age_group": "Prime"}, {"player": "S. Mustafi", "age": 27, "position": "Defense", "minutes": 530, "appearances": 7, "mp_pct": 0.23, "age_group": "Prime"}, {"player": "C. Chambers", "age": 25, "position": "Defense", "minutes": 1103, "appearances": 14, "mp_pct": 0.47, "age_group": "Prime"}, {"player": "David Luiz", "age": 32, "position": "Defense", "minutes": 2006, "appearances": 23, "mp_pct": 0.86, "age_group": "Senior"}, {"player": "S. Kola\u0161inac", "age": 26, "position": "Defense", "minutes": 1089, "appearances": 16, "mp_pct": 0.47, "age_group": "Prime"}, {"player": "Dani Ceballos", "age": 23, "position": "Midfield", "minutes": 745, "appearances": 12, "mp_pct": 0.32, "age_group": "Youth"}, {"player": "M. \u00d6zil", "age": 31, "position": "Midfield", "minutes": 1278, "appearances": 16, "mp_pct": 0.55, "age_group": "Senior"}, {"player": "L. Torreira", "age": 24, "position": "Midfield", "minutes": 1357, "appearances": 23, "mp_pct": 0.58, "age_group": "Prime"}, {"player": "A. Maitland-Niles", "age": 22, "position": "Defense", "minutes": 1211, "appearances": 14, "mp_pct": 0.52, "age_group": "Youth"}, {"player": "J. Willock", "age": 20, "position": "Midfield", "minutes": 576, "appearances": 19, "mp_pct": 0.25, "age_group": "Youth"}, {"player": "M. Guendouzi", "age": 20, "position": "Midfield", "minutes": 1582, "appearances": 21, "mp_pct": 0.68, "age_group": "Youth"}, {"player": "G. Xhaka", "age": 27, "position": "Midfield", "minutes": 1728, "appearances": 20, "mp_pct": 0.74, "age_group": "Prime"}, {"player": "B. Saka", "age": 18, "position": "Midfield", "minutes": 1014, "appearances": 16, "mp_pct": 0.43, "age_group": "Youth"}, {"player": "A. Lacazette", "age": 28, "position": "Forward", "minutes": 1287, "appearances": 19, "mp_pct": 0.55, "age_group": "Prime"}, {"player": "P. Aubameyang", "age": 30, "position": "Forward", "minutes": 2125, "appearances": 24, "mp_pct": 0.91, "age_group": "Prime"}, {"player": "N. P\u00e9p\u00e9", "age": 24, "position": "Forward", "minutes": 1432, "appearances": 22, "mp_pct": 0.61, "age_group": "Prime"}, {"player": "R. Nelson", "age": 20, "position": "Forward", "minutes": 440, "appearances": 10, "mp_pct": 0.19, "age_group": "Youth"}, {"player": "E. Nketiah", "age": 20, "position": "Forward", "minutes": 102, "appearances": 3, "mp_pct": 0.04, "age_group": "Youth"}, {"player": "Martinelli", "age": 18, "position": "Forward", "minutes": 656, "appearances": 14, "mp_pct": 0.28, "age_group": "Youth"}]}};
var embedOpt = {"mode": "vega-lite"};

function showError(el, error){
el.innerHTML = ('<div class="error" style="color:red;">'
+ '<p>JavaScript Error: ' + error.message + '</p>'
+ "<p>This usually means there's a typo in your chart specification. "
+ "See the javascript console for the full traceback.</p>"
+ '</div>');
throw error;
}
const el = document.getElementById('vis');
vegaEmbed("#vis", spec, embedOpt)
.catch(error => showError(el, error));
})(vegaEmbed);

</script>

We can see Arsenal's youth are getting the most run at midfield with the Forward line being led by prime attackers, Aubameyang and Lacazette.

The defense is where Arsenal's more senior players have seen the most minutes. Over the coming seasons, Arsenal should look to giving more minutes to prime defenders.

They should trend younger at the position next season with the return of William Saliba from loan plus a likely big-budget CB signing.
