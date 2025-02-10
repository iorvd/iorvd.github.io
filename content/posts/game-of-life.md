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

<script>

let dimension = 20;

let playing = false;

let grid = new Array(dimension);
let nextGrid = new Array(dimension);

let timer;
let reproductionTime = 100;

function initializeGrids() {
    for (let i = 0; i < dimension; i++) {
        grid[i] = new Array(dimension);
        nextGrid[i] = new Array(dimension);
    }
}

function resetGrids() {
    for (let i = 0; i < dimension; i++) {
        for (let j = 0; j < dimension; j++) {
            grid[i][j] = 0;
            nextGrid[i][j] = 0;
        }
    }
}

function copyAndResetGrid() {
    for (let i = 0; i < dimension; i++) {
        for (let j = 0; j < dimension; j++) {
            grid[i][j] = nextGrid[i][j];
            nextGrid[i][j] = 0;
        }
    }
}

function initialize() {
    createTable();
    initializeGrids();
    resetGrids();
    setupControlButtons();
}

function createTable() {
    let gridContainer = document.getElementById('gridContainer');
    if (!gridContainer) {
        console.error("Problem: No div for the grid table!");
    }

    let table = document.createElement("table");

    for (let i = 0; i < dimension; i++) {
        let tr = document.createElement("tr");
        for (let j = 0; j < dimension; j++) {
            let cell = document.createElement("td");
            cell.setAttribute("id", i + "_" + j);
            cell.setAttribute("class", "dead");
            cell.onclick = cellClickHandler;
            tr.appendChild(cell);
        }
        table.appendChild(tr);
    }
    gridContainer.appendChild(table);
}

function cellClickHandler() {
    const axes = this.id.split("_");
    const x = axes[0];
    const y = axes[1];

    let classes = this.getAttribute("class");
    if (classes.indexOf("live") > -1) {
        this.setAttribute("class", "dead");
        grid[x][y] = 0;
    } else {
        this.setAttribute("class", "live");
        grid[x][y] = 1;
    }
}

function updateView() {
    for (let i = 0; i < dimension; i++) {
        for (let j = 0; j < dimension; j++) {
            let cell = document.getElementById(i + "_" + j);
            if (grid[i][j] === 0) {
                cell.setAttribute("class", "dead");
            } else {
                cell.setAttribute("class", "live");
            }
        }
    }
}

function setupControlButtons() {
    let startButton = document.getElementById('start');
    startButton.onclick = startButtonHandler;

    let clearButton = document.getElementById('clear');
    clearButton.onclick = clearButtonHandler;
}

function clearButtonHandler() {
    console.log("Clear the game: stop playing, clear the grid");

    playing = false;
    let startButton = document.getElementById('start');
    startButton.innerHTML = "Start";
    clearTimeout(timer);

    let cellsList = document.getElementsByClassName("live");
    // convert to array first, otherwise, you're working on a live node list
    // and the update doesn't work!
    let cells = [];
    for (let i = 0; i < cellsList.length; i++) {
        cells.push(cellsList[i]);
    }

    for (let i = 0; i < cells.length; i++) {
        cells[i].setAttribute("class", "dead");
    }
    resetGrids();
}

function startButtonHandler() {
    if (playing) {
        playing = false;
        this.innerHTML = "Continue";
        clearTimeout(timer);
    } else {
        playing = true;
        this.innerHTML = "Pause";
        play();
    }
}

// run the life game
function play() {
    computeNextGen();

    if (playing) {
        timer = setTimeout(play, reproductionTime);
    }
}

function computeNextGen() {
    for (let x = 0; x < dimension; x++) {
        for (let y = 0; y < dimension; y++) {
            const cellsAround = get_live_cells_around(grid, neighbours(x, y, grid.length));
            const cell = grid[x][y]

            if (cellsAround < 2 && cell === 1) {
                nextGrid[x][y] = 0;
            } else if ((cellsAround === 2 || cellsAround === 3) && cell === 1) {
                nextGrid[x][y] = 1;
            } else if (cellsAround > 3 && cell === 1) {
                nextGrid[x][y] = 0;
            } else if (cellsAround === 3 && cell === 0) {
                nextGrid[x][y] = 1;
            } else {
                nextGrid[x][y] = 0;
            }
        }
    }

    // copy NextGrid to grid, and reset nextGrid
    copyAndResetGrid();
    // copy all 1 values to "live" in the table
    updateView();
}

function get_live_cells_around(grid, neighbors) {
    let count = 0
    for (let i = 0; i < neighbors.length; i++) {
        if (grid[neighbors[i][0]][neighbors[i][1]] !== 0) {
            count++
        }
    }
    return count
}


function neighbours(x, y, dimension) {
    let nb = new (Array);
    let directions = [[-1, -1], [0, -1], [1, -1], [-1, 0], [1, 0], [-1, 1], [0, 1], [1, 1]]
    for (let i = 0; i < directions.length; i++) {
        let xoverflow = x + directions[i][0]
        let yoverflow = y + directions[i][1]
        if (!(xoverflow < 0 || yoverflow < 0 || xoverflow >= dimension || yoverflow >= dimension)) {
            nb.push(Array(xoverflow, yoverflow));
        }
    }

    return nb;
}

window.onload = initialize;

</script>
<br><br><br>