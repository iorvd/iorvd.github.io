---
title: "Conway's game of life"
date: "2025-02-02"
aliases: ["/game-of-life"]
---

# Rules

1. Any live cell with fewer than two live neighbours dies, as if by underpopulation.
2. Any live cell with two or three live neighbours lives on to the next generation.
3. Any live cell with more than three live neighbours dies, as if by overpopulation.
4. Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction.

# Figures

See examples of [basic patterns](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life#Examples_of_patterns) such as
still figures, oscillators and gliders.

# Playground

<link rel="stylesheet" type="text/css" media="screen" href="../../game_of_life.css"/>

<div id="gridContainer">

</div>

<div class="controls">
<button id="start"><span>Start</span></button>
<button id="clear"><span>Clear</span></button>
</div>

<script src="../../js/index.js"></script>

<br><br><br>