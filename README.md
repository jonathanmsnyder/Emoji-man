<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Emoji-Man: The Maze Game</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom CSS for Game Aesthetics */
        @import url('https://fonts.googleapis.com/css2?family=Bungee+Inline&display=swap');
        
        body {
            font-family: 'Inter', sans-serif;
            background-color: #1a1a2e; /* Deep space blue */
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: flex-start;
            min-height: 100vh;
            padding: 1rem 0;
            overflow-x: hidden;
            user-select: none;
            -webkit-user-select: none;
            -ms-user-select: none;
        }

        .game-title {
            font-family: 'Bungee Inline', cursive;
            color: #e94560; /* Neon Red/Pink */
            text-shadow: 0 0 10px #e94560, 0 0 20px #e94560;
            letter-spacing: 2px;
        }

        .game-container {
            border: 4px solid #0f3460;
            box-shadow: 0 0 30px rgba(15, 52, 96, 0.7);
            background-color: #0c183a; /* Darker blue background for the maze */
            border-radius: 12px;
            margin-top: 1rem;
            max-width: 95vw;
        }

        .maze-grid {
            display: grid;
            touch-action: none; 
        }
        
        .grid-cell {
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 1rem; 
            transition: background-color 0.1s;
        }

        .wall {
            background-color: #0f3460; 
            border: 1px solid #0f3460;
        }

        .power-pellet {
            animation: pulse 1s infinite alternate;
        }

        @keyframes pulse {
            from { transform: scale(1); opacity: 1; }
            to { transform: scale(1.4); opacity: 0.8; }
        }

        .player-emoji {
            transition: transform 0.1s ease-out;
            display: inline-block;
            animation: none;
        }
        .player-invulnerable {
             animation: flash 0.3s infinite alternate;
        }
        @keyframes flash {
            from { opacity: 1; }
            to { opacity: 0.3; }
        }

        .game-message {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.9);
            color: #fff;
            padding: 20px 40px;
            border-radius: 15px;
            text-align: center;
            z-index: 1000;
            border: 3px solid #e94560;
            box-shadow: 0 0 20px #e94560;
            display: none;
        }
        
        .control-button {
            transition: all 0.1s ease-in-out;
            box-shadow: 0 4px #16213e;
            transform: translateY(0);
        }

        .control-button:active {
            box-shadow: 0 2px #16213e;
            transform: translateY(2px);
        }
    </style>
</head>
<body>

    <!-- Game Title -->
    <h1 class="game-title text-3xl md:text-5xl mt-4 mb-4 font-bold">EMOJI-MAN</h1>
    
    <!-- Scoreboard and Status -->
    <div class="flex justify-between w-full max-w-lg p-2 text-white font-semibold">
        <span id="score" class="text-xl text-yellow-400">Score: 0</span>
        <span id="lives" class="text-xl text-red-500">Lives: 3</span>
    </div>

    <!-- Game Container (Maze) -->
    <div id="game-container" class="game-container">
        <div id="maze-grid" class="maze-grid">
            <!-- Maze cells will be injected here -->
        </div>
    </div>

    <!-- Mobile Touch Controls -->
    <div id="controls" class="p-4 mt-4 w-full max-w-sm flex flex-col items-center gap-2">
        <button id="up" class="control-button bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-6 rounded-lg w-1/3">⬆️ Up</button>
        <div class="flex justify-center gap-2 w-full">
            <button id="left" class="control-button bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-6 rounded-lg w-1/3">⬅️ Left</button>
            <button id="right" class="control-button bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-6 rounded-lg w-1/3">➡️ Right</button>
        </div>
        <button id="down" class="control-button bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-6 rounded-lg w-1/3">⬇️ Down</button>
    </div>

    <!-- Game Message Overlay -->
    <div id="game-message" class="game-message">
        <h2 id="message-text" class="text-3xl font-bold mb-4">Game Over!</h2>
        <p id="message-subtitle" class="text-xl mb-4">Tap Restart to play again.</p>
        <button id="restart-button" class="bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded-lg">Restart</button>
    </div>

    <!-- Instructions for better touch detection -->
    <div class="text-xs text-gray-400 mt-4 px-4 text-center">
        Tip: Swipe anywhere on the maze itself to move, or use the buttons!
    </div>
    
    <script>
        // Game Constants
        const EMOJIS = {
            WALL: '🧱',
            PELLET: '✨',
            POWER_PELLET: '⭐', 
            PLAYER: '😀',
            GHOST_NORMAL: ['😈', '👹', '👺'], 
            GHOST_FRIGHTENED: '🥶',
            EMPTY: ' '
        };

        const GRID_SIZE = 21; 
        const GAME_SPEED = 150; 
        const GHOST_MOVE_RATE = 2; // Ghosts move every 2 ticks (300ms)
        const GHOST_LOCK_TICKS = 20; 
        const INVULNERABILITY_TICKS = 10; 
        const FRIGHTENED_TICKS = 60; 

        const PLAYER_START = { row: 5, col: 10 };
        const GHOST_STARTS = [
            { row: 9, col: 10 }, 
            { row: 9, col: 9 },  
            { row: 9, col: 11 }  
        ]; 
        
        // --- COMPLEX 21x21 MAZE MAP ---
        // 0: Wall, 1: Pellet, P: Player Start, S: Power Pellet
        const INITIAL_MAZE_MAP = [
            '000000000000000000000',
            '011111111101111111110',
            '010010001000100010010',
            '0S01111111111111111S0', 
            '010010100000010100100',
            '0111111111P1111111110', 
            '000010001000100010000',
            '011111111111111111110',
            '010001000000000100010',
            '011101111111111011110', // Tunnel Row (9)
            '000100010001000100010',
            '011111111111111111110',
            '010001000000000100010',
            '011111111111111111110',
            '000010001000100010000',
            '011111111111111111110',
            '010010100000010100100',
            '0S01111111111111111S0', 
            '010010001000100010010',
            '011111111101111111110',
            '000000000000000000000',
        ];

        // Global Game State
        let maze = [];
        let player = {};
        let ghosts = [];
        let score = 0;
        let lives = 3;
        let totalPellets = 0;
        let initialPelletCount = 0;
        let gameLoopId = null;
        let gameTickCounter = 0; 
        let isFrightenedMode = false;
        let frightenedTimer = 0;
        let isGameOver = false;
        let ghostReleaseTimer = 0; 
        let invulnerabilityTimer = 0; 

        // --- Utility Functions ---

        /** Creates the initial state of the maze, player, and ghosts. */
        function createInitialState() {
            let pelletCount = 0;
            
            // Deep copy the maze map
            maze = INITIAL_MAZE_MAP.map(rowStr => rowStr.split(''));

            // Calculate pellet count, replacing 'P' with '1' for consistency in movement checks
            for (let r = 0; r < GRID_SIZE; r++) {
                for (let c = 0; c < GRID_SIZE; c++) {
                    if (maze[r][c] === '1' || maze[r][c] === 'P' || maze[r][c] === 'S') {
                        pelletCount++;
                        if (maze[r][c] === 'P') maze[r][c] = '1';
                    }
                }
            }
            
            initialPelletCount = pelletCount;
            totalPellets = pelletCount;

            // Initialize game state
            player = { 
                ...PLAYER_START, 
                direction: 'RIGHT', 
                nextDirection: 'RIGHT', 
                isInvulnerable: false 
            };
            
            // Start with only the first ghost
            ghosts = [
                {
                    id: 0,
                    row: GHOST_STARTS[0].row,
                    col: GHOST_STARTS[0].col,
                    homeRow: GHOST_STARTS[0].row,
                    homeCol: GHOST_STARTS[0].col,
                    direction: 'UP',
                    isEdible: false,
                    eaten: false,
                    isLocked: true, 
                    emoji: EMOJIS.GHOST_NORMAL[0]
                }
            ];
            
            score = 0;
            lives = 3;
            isGameOver = false;
            
            startGracePeriod();
            updateScoreboard();
        }
        
        /** Spawns a new ghost if conditions are met. */
        function spawnGhostIfReady() {
            const currentGhostCount = ghosts.length;
            const pelletsEaten = initialPelletCount - totalPellets;
            
            if (currentGhostCount >= EMOJIS.GHOST_NORMAL.length) return; 

            let shouldSpawn = false;
            // Spawn 2nd ghost when 1/3 of pellets are eaten
            if (currentGhostCount === 1 && pelletsEaten >= initialPelletCount / 3) {
                shouldSpawn = true;
            } 
            // Spawn 3rd ghost when 2/3 of pellets are eaten
            else if (currentGhostCount === 2 && pelletsEaten >= initialPelletCount * 2 / 3) {
                shouldSpawn = true;
            }

            if (shouldSpawn) {
                const newId = currentGhostCount;
                const startPos = GHOST_STARTS[newId] || GHOST_STARTS[0]; 
                
                ghosts.push({
                    id: newId,
                    row: startPos.row,
                    col: startPos.col,
                    homeRow: startPos.row,
                    homeCol: startPos.col,
                    direction: 'DOWN', 
                    isEdible: false,
                    eaten: false,
                    isLocked: true, 
                    emoji: EMOJIS.GHOST_NORMAL[newId]
                });
                // Briefly lock the new ghost
                ghostReleaseTimer = Math.max(ghostReleaseTimer, 10); 
            }
        }

        /** Draws the current state of the maze to the DOM. */
        function drawMaze() {
            const gridElement = document.getElementById('maze-grid');
            
            // CRITICAL FIX: Ensure the container size is calculated and set FIRST
            const containerSize = Math.min(window.innerWidth * 0.95, 600);
            gridElement.style.width = `${containerSize}px`;
            gridElement.style.height = `${containerSize}px`;
            
            const CELL_SIZE = containerSize / GRID_SIZE;

            if (gridElement.children.length === 0) {
                // Initialize grid structure and base styles only once
                gridElement.style.gridTemplateColumns = `repeat(${GRID_SIZE}, ${CELL_SIZE}px)`;
                gridElement.style.gridTemplateRows = `repeat(${GRID_SIZE}, ${CELL_SIZE}px)`;

                for (let r = 0; r < GRID_SIZE; r++) {
                    for (let c = 0; c < GRID_SIZE; c++) {
                        const cellDiv = document.createElement('div');
                        cellDiv.id = `cell-${r}-${c}`;
                        cellDiv.className = 'grid-cell';
                        gridElement.appendChild(cellDiv);
                    }
                }
            } else {
                 // For subsequent redraws (like resize), update grid dimensions
                 gridElement.style.gridTemplateColumns = `repeat(${GRID_SIZE}, ${CELL_SIZE}px)`;
                 gridElement.style.gridTemplateRows = `repeat(${GRID_SIZE}, ${CELL_SIZE}px)`;
            }
            
            // Re-render content
            for (let r = 0; r < GRID_SIZE; r++) {
                for (let c = 0; c < GRID_SIZE; c++) {
                    const cellDiv = document.getElementById(`cell-${r}-${c}`);
                    let content = EMOJIS.EMPTY;
                    let emojiClass = '';
                    let isPlayerHere = player.row === r && player.col === c;
                    const ghostHere = ghosts.find(g => g.row === r && g.col === c);

                    if (isPlayerHere) {
                        content = EMOJIS.PLAYER;
                        emojiClass = `player-emoji ${player.isInvulnerable ? 'player-invulnerable' : ''}`;
                    } else if (ghostHere) {
                        content = ghostHere.isEdible ? EMOJIS.GHOST_FRIGHTENED : ghostHere.emoji;
                    } else {
                        // Determine wall/pellet/power pellet/empty based on the maze map
                        if (maze[r][c] === '0') {
                            content = EMOJIS.WALL;
                            cellDiv.className = 'grid-cell wall';
                        } else if (maze[r][c] === 'S') {
                            content = EMOJIS.POWER_PELLET;
                            cellDiv.className = 'grid-cell';
                            emojiClass = 'power-pellet';
                        } else {
                            content = maze[r][c] === '1' ? EMOJIS.PELLET : EMOJIS.EMPTY;
                            cellDiv.className = 'grid-cell';
                        }
                    }

                    cellDiv.innerHTML = `<span class="${emojiClass}">${content}</span>`;
                    
                    // Apply rotation to player emoji
                    if (isPlayerHere) {
                        const playerSpan = cellDiv.querySelector('.player-emoji');
                        let rotation = 0;
                        switch (player.direction) {
                            case 'UP': rotation = 270; break;
                            case 'DOWN': rotation = 90; break;
                            case 'LEFT': rotation = 180; break;
                            case 'RIGHT': rotation = 0; break;
                        }
                        playerSpan.style.transform = `rotate(${rotation}deg)`;
                    }
                }
            }
        }
        
        /** Updates the Score and Lives display. */
        function updateScoreboard() {
            document.getElementById('score').textContent = `Score: ${score}`;
            document.getElementById('lives').textContent = `Lives: ${lives}`;
        }

        /** Shows a game message overlay. */
        function showMessage(text, subtitle = '') {
            document.getElementById('message-text').textContent = text;
            document.getElementById('message-subtitle').textContent = subtitle;
            document.getElementById('game-message').style.display = 'block';
        }

        /** Hides the game message overlay. */
        function hideMessage() {
            document.getElementById('game-message').style.display = 'none';
        }

        /** Calculates the new position based on direction. */
        function getNextPos(row, col, direction) {
            let nextRow = row;
            let nextCol = col;
            switch (direction) {
                case 'UP': nextRow--; break;
                case 'DOWN': nextRow++; break;
                case 'LEFT': nextCol--; break;
                case 'RIGHT': nextCol++; break;
            }
            // Handle wrapping (tunnel on row 9)
            if (nextCol < 0) nextCol = GRID_SIZE - 1;
            if (nextCol >= GRID_SIZE) nextCol = 0;
            
            return { nextRow, nextCol };
        }

        /** Checks if a position is a valid path (not a wall). */
        function isValidMove(row, col) {
            if (row < 0 || row >= GRID_SIZE || col < 0 || col >= GRID_SIZE) {
                // Allow wrapping only on the tunnel row (9 in this new maze)
                if (row === 9 && (col < 0 || col >= GRID_SIZE)) return true;
                return false;
            }
            return maze[row][col] !== '0';
        }

        // --- Player Logic ---

        function handlePlayerMovement() {
            let { nextRow, nextCol } = getNextPos(player.row, player.col, player.nextDirection);
            
            if (isValidMove(nextRow, nextCol)) {
                player.direction = player.nextDirection;
            } else {
                ({ nextRow, nextCol } = getNextPos(player.row, player.col, player.direction));
                if (!isValidMove(nextRow, nextCol)) return;
            }

            // Move Player
            player.row = nextRow;
            player.col = nextCol;

            // Handle Pellet Collection
            const currentCell = maze[player.row][player.col];

            if (currentCell === '1') { // Regular Pellet
                maze[player.row][player.col] = '2'; // Eaten
                score += 10;
                totalPellets--;
                spawnGhostIfReady();
            } else if (currentCell === 'S') { // Power Pellet
                maze[player.row][player.col] = '2'; // Eaten
                score += 50;
                totalPellets--; // Crucial: Decrement totalPellets here!
                
                // Activate frightened mode!
                isFrightenedMode = true;
                frightenedTimer = FRIGHTENED_TICKS;
                ghosts.forEach(g => {
                    g.isEdible = true;
                    // Reverse ghost direction when power pellet is eaten
                    switch(g.direction) {
                        case 'UP': g.direction = 'DOWN'; break;
                        case 'DOWN': g.direction = 'UP'; break;
                        case 'LEFT': g.direction = 'RIGHT'; break;
                        case 'RIGHT': g.direction = 'LEFT'; break;
                    }
                });
            }

            if (totalPellets === 0) {
                endGame(true);
            }
        }

        // --- Ghost Logic ---

        function handleGhostMovement() {
            // Ghosts only move every GHOST_MOVE_RATE ticks
            if (gameTickCounter % GHOST_MOVE_RATE !== 0) return;

            ghosts.forEach(ghost => {
                if (ghost.eaten || ghost.isLocked) return;

                const validMoves = ['UP', 'DOWN', 'LEFT', 'RIGHT'].filter(dir => {
                    const { nextRow, nextCol } = getNextPos(ghost.row, ghost.col, dir);
                    return isValidMove(nextRow, nextCol);
                });

                let newDirection;
                
                // Behavior depends on edibility
                if (ghost.isEdible) {
                    // Flee (move away from player) or Random
                    if (Math.random() < 0.7) {
                        newDirection = getFleeDirection(ghost, validMoves);
                    } else {
                        newDirection = validMoves[Math.floor(Math.random() * validMoves.length)];
                    }
                } else {
                    // Chase or Random
                    if (Math.random() < 0.7) {
                        newDirection = getChaseDirection(ghost, validMoves);
                    } else {
                        newDirection = validMoves[Math.floor(Math.random() * validMoves.length)];
                    }
                }
                
                // Final move
                if (newDirection) {
                    const { nextRow, nextCol } = getNextPos(ghost.row, ghost.col, newDirection);
                    if (isValidMove(nextRow, nextCol)) {
                        ghost.row = nextRow;
                        ghost.col = nextCol;
                        ghost.direction = newDirection;
                    }
                }
            });
        }
        
        /** Chooses a direction that minimizes distance to the player (Chase). */
        function getChaseDirection(ghost, validMoves) {
            let bestDist = Infinity;
            let bestDir = ghost.direction; 

            for (const dir of validMoves) {
                const { nextRow, nextCol } = getNextPos(ghost.row, ghost.col, dir);
                
                // Prevent turning 180 degrees
                const isOpposite = (ghost.direction === 'UP' && dir === 'DOWN') ||
                                   (ghost.direction === 'DOWN' && dir === 'UP') ||
                                   (ghost.direction === 'LEFT' && dir === 'RIGHT') ||
                                   (ghost.direction === 'RIGHT' && dir === 'LEFT');

                if (isOpposite && validMoves.length > 1) continue; 

                const dist = Math.abs(player.row - nextRow) + Math.abs(player.col - nextCol);

                if (dist < bestDist) {
                    bestDist = dist;
                    bestDir = dir;
                }
            }
            return bestDir;
        }

        /** Chooses a direction that maximizes distance from the player (Flee). */
        function getFleeDirection(ghost, validMoves) {
            let worstDist = -Infinity;
            let bestDir = ghost.direction; 

            for (const dir of validMoves) {
                const { nextRow, nextCol } = getNextPos(ghost.row, ghost.col, dir);
                
                // Prevent turning 180 degrees
                const isOpposite = (ghost.direction === 'UP' && dir === 'DOWN') ||
                                   (ghost.direction === 'DOWN' && dir === 'UP') ||
                                   (ghost.direction === 'LEFT' && dir === 'RIGHT') ||
                                   (ghost.direction === 'RIGHT' && dir === 'LEFT');

                if (isOpposite && validMoves.length > 1) continue; 

                const dist = Math.abs(player.row - nextRow) + Math.abs(player.col - nextCol);

                if (dist > worstDist) {
                    worstDist = dist;
                    bestDir = dir;
                }
            }
            return bestDir;
        }

        // --- Collision Logic ---

        function checkCollisions() {
            if (player.isInvulnerable) return;

            ghosts.forEach(ghost => {
                if (ghost.eaten) return;

                if (player.row === ghost.row && player.col === ghost.col) {
                    if (ghost.isEdible) {
                        // Player eats ghost!
                        score += 200;
                        ghost.eaten = true;
                        ghost.isEdible = false;
                        ghost.isLocked = true;
                        
                        // Teleport ghost back to start/pen
                        ghost.row = ghost.homeRow;
                        ghost.col = ghost.homeCol;
                        
                        // Ghost respawns and is unlocked after 3 seconds
                        setTimeout(() => {
                             ghost.eaten = false;
                             ghost.isLocked = false;
                        }, 3000); 
                    } else {
                        // Ghost eats player!
                        loseLife();
                    }
                }
            });
        }

        // --- Game Flow and Loop ---
        
        /** Resets player and ghost positions and starts the timers. */
        function resetPositionsAndTimers() {
            player = { 
                ...PLAYER_START, 
                direction: 'RIGHT', 
                nextDirection: 'RIGHT',
                isInvulnerable: true 
            };

            // Reset and lock ALL ghosts
            ghosts.forEach((g) => {
                g.row = g.homeRow;
                g.col = g.homeCol;
                g.eaten = false;
                g.isEdible = false;
                g.isLocked = true;
            });

            isFrightenedMode = false;
            frightenedTimer = 0;
            
            // Start Timers
            ghostReleaseTimer = GHOST_LOCK_TICKS; 
            invulnerabilityTimer = GHOST_LOCK_TICKS + INVULNERABILITY_TICKS;
            
            hideMessage(); 
            drawMaze();
        }

        /** Manages the startup/reset grace period. */
        function startGracePeriod() {
            if (gameLoopId) clearInterval(gameLoopId);
            
            resetPositionsAndTimers();

            showMessage("GET READY!", "Ghosts are locked for 3 seconds...");

            gameTickCounter = 0;
            gameLoopId = setInterval(gameTick, GAME_SPEED);
        }


        function loseLife() {
            if (isGameOver) return;
            
            lives--;
            updateScoreboard();

            if (lives <= 0) {
                endGame(false);
            } else {
                if (gameLoopId) clearInterval(gameLoopId);
                
                showMessage("Life Lost!", "Wait for reset...");

                setTimeout(() => {
                    startGracePeriod(); 
                }, 1000); 
            }
        }

        function endGame(won) {
            isGameOver = true;
            if (gameLoopId) clearInterval(gameLoopId);
            gameLoopId = null;

            if (won) {
                showMessage(`YOU WIN! 🏆`, `Final Score: ${score}`);
            } else {
                showMessage(`GAME OVER!`, `Final Score: ${score}`);
            }
        }

        function gameTick() {
            if (isGameOver) return;
            gameTickCounter++;

            // 1. Player Movement (Always allowed)
            handlePlayerMovement();
            
            // 2. Handle Timers
            if (ghostReleaseTimer > 0) {
                ghostReleaseTimer--;
                if (ghostReleaseTimer === 0) {
                    // Release all current ghosts
                    ghosts.forEach(g => g.isLocked = false);
                    document.getElementById('message-subtitle').textContent = "Ghosts Released! Go Go Go!";
                    setTimeout(hideMessage, 500); 
                } else {
                    const secs = Math.ceil(ghostReleaseTimer * GAME_SPEED / 1000);
                    document.getElementById('message-subtitle').textContent = `Ghosts locked. Moving in ${secs}s...`;
                }
            }

            if (invulnerabilityTimer > 0) {
                invulnerabilityTimer--;
                player.isInvulnerable = true;
                if (invulnerabilityTimer === 0) {
                    player.isInvulnerable = false;
                }
            }
            
            if (isFrightenedMode && frightenedTimer > 0) {
                frightenedTimer--;
                if (frightenedTimer === 0) {
                    isFrightenedMode = false;
                    ghosts.forEach(g => g.isEdible = false);
                }
            }

            // 3. Ghost Movement (Only runs if not locked and only every GHOST_MOVE_RATE)
            handleGhostMovement();

            // 4. Collision Check (Only runs if player is NOT invulnerable)
            checkCollisions();

            // 5. Draw
            drawMaze();
        }


        // --- Input Handlers ---

        function setPlayerDirection(direction) {
            if (isGameOver) return; 
            player.nextDirection = direction;
        }

        // Keyboard Controls
        document.addEventListener('keydown', (e) => {
            if (isGameOver && (e.key === 'r' || e.key === 'R' || e.key === ' ')) {
                startGame();
                return;
            }
            switch (e.key) {
                case 'ArrowUp': case 'w': setPlayerDirection('UP'); break;
                case 'ArrowDown': case 's': setPlayerDirection('DOWN'); break;
                case 'ArrowLeft': case 'a': setPlayerDirection('LEFT'); break;
                case 'ArrowRight': case 'd': setPlayerDirection('RIGHT'); break;
            }
        });
        
        // Touch/Swipe Controls for Mobile
        let touchStartX = 0;
        let touchStartY = 0;
        const SWIPE_THRESHOLD = 30; // Min distance for a swipe

        const mazeElement = document.getElementById('maze-grid');
        
        mazeElement.addEventListener('touchstart', (e) => {
            if (e.touches.length === 1) {
                touchStartX = e.touches[0].clientX;
                touchStartY = e.touches[0].clientY;
            }
        }, { passive: true });

        mazeElement.addEventListener('touchend', (e) => {
            if (isGameOver) return;
            
            const touch = e.changedTouches[0];
            if (!touch) return;

            const deltaX = touch.clientX - touchStartX;
            const deltaY = touch.clientY - touchStartY;
            
            if (Math.abs(deltaX) > Math.abs(deltaY)) {
                // Horizontal swipe
                if (Math.abs(deltaX) > SWIPE_THRESHOLD) {
                    setPlayerDirection(deltaX > 0 ? 'RIGHT' : 'LEFT');
                }
            } else {
                // Vertical swipe
                if (Math.abs(deltaY) > SWIPE_THRESHOLD) {
                    setPlayerDirection(deltaY > 0 ? 'DOWN' : 'UP');
                }
            }
        });
        
        // Button Controls
        document.getElementById('up').addEventListener('click', () => setPlayerDirection('UP'));
        document.getElementById('down').addEventListener('click', () => setPlayerDirection('DOWN'));
        document.getElementById('left').addEventListener('click', () => setPlayerDirection('LEFT'));
        document.getElementById('right').addEventListener('click', () => setPlayerDirection('RIGHT'));
        document.getElementById('restart-button').addEventListener('click', startGame);

        // --- Initialization ---

        function startGame() {
            createInitialState();
        }

        window.onload = function() {
            startGame();
            window.addEventListener('resize', drawMaze);
        };

    </script>

</body>
</html>
