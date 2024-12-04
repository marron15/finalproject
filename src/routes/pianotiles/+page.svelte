<script lang="ts">
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';

    interface Stage {
        id: number;
        title: string;
        song: string;
        url: string;
    }

    interface Tile {
        column: number;
        y: number;
        clicked: boolean;
        timestamp: number;
        id: number;
    }

    let selectedStage: Stage | null = null;
    let isLoading = true;
    let score = 0;
    let gameStarted = false;
    let gameOver = false;
    let tiles: Tile[] = [];
    let speed = 3;
    let animationFrame: number | null = null;
    let lastTimestamp = 0;
    let audio: HTMLAudioElement;
    let startTime = 0;
    let lastClickTime = 0;

    // Define rhythm patterns (in milliseconds)
    const rhythmPatterns: Record<number, number[]> = {
        1: [
            0, 800, 1600, 2400,    // First measure
            3200, 4000, 4800, 5600, // Second measure
            6400, 7200, 8000, 8800, // Third measure
            9600, 10400, 11200, 12000 // Fourth measure
        ]
    };

    function createTile(timestamp: number): Tile {
        const column = Math.floor(Math.random() * 4);
        return {
            column,
            y: -150,
            clicked: false,
            timestamp,
            id: Date.now() + Math.random()
        };
    }

    function startGame(): void {
        if (gameStarted) return;
        gameStarted = true;
        gameOver = false;
        score = 0;
        speed = 3;
        tiles = [];
        startTime = performance.now();
        lastTimestamp = startTime;
        
        // Start playing the song
        if (selectedStage?.url && audio) {
            try {
                audio.currentTime = 0;
                const playPromise = audio.play();
                if (playPromise !== undefined) {
                    playPromise.catch(error => {
                        console.error('Audio playback error:', error);
                    });
                }
            } catch (error) {
                console.error('Error playing audio:', error);
            }
        }
        
        animate();
    }

    function animate(timestamp = 0): void {
        if (gameOver) return;
        
        const deltaTime = timestamp - lastTimestamp;
        const gameTime = timestamp - startTime;
        lastTimestamp = timestamp;

        // Move existing tiles down with smoother motion
        tiles = tiles.map(tile => ({
            ...tile,
            y: tile.y + Math.min(speed * deltaTime / 16, 20) // Cap maximum movement
        }));

        // Generate new tiles based on rhythm pattern
        if (selectedStage?.id && rhythmPatterns[selectedStage.id]) {
            const pattern = rhythmPatterns[selectedStage.id];
            const currentBeat = pattern.find(beat => 
                Math.abs((gameTime % 12000) - (beat % 12000)) < 20 && // 12000ms = pattern length
                !tiles.some(tile => Math.abs(tile.timestamp - beat) < 20)
            );

            if (currentBeat !== undefined) {
                const newTile = createTile(currentBeat);
                tiles = [...tiles, newTile];
            }
        }

        // Only check for missed tiles that are WAY past the bottom
        const bottomLine = window.innerHeight + 100;
        const failedTiles = tiles.filter(tile => 
            !tile.clicked && 
            tile.y > bottomLine + 200 // Super forgiving bottom line
        );

        if (failedTiles.length > 0) {
            endGame();
            return;
        }

        // Clean up tiles that are way off screen
        tiles = tiles.filter(tile => 
            tile.y < window.innerHeight + 400 || // Keep unclicked tiles longer
            (tile.clicked && tile.y < window.innerHeight + 500) // Keep clicked tiles even longer
        );

        if (!gameOver) {
            animationFrame = requestAnimationFrame(animate);
        }
    }

    function handleTileClick(tile: Tile, event: MouseEvent): void {
        if (!gameStarted || gameOver) return;
        
        event.preventDefault();
        event.stopPropagation();

        const target = event.target as HTMLElement;
        const gameContainer = target.closest('.game-container');
        if (!gameContainer) return;
        
        const containerRect = gameContainer.getBoundingClientRect();
        const clickY = event.clientY - containerRect.top;
        
        // Get tiles in the clicked column
        const columnTiles = tiles.filter(t => 
            !t.clicked && 
            t.column === tile.column &&
            Math.abs(t.y - clickY) <= 200
        );

        if (columnTiles.length === 0) return;
        
        // Find the closest tile to the click position
        const closestTile = columnTiles.reduce((closest, current) => {
            const closestDist = Math.abs(closest.y - clickY);
            const currentDist = Math.abs(current.y - clickY);
            return currentDist < closestDist ? current : closest;
        });

        const clickTolerance = 150;
        if (Math.abs(closestTile.y - clickY) <= clickTolerance) {
            tiles = tiles.map(t => {
                if (t === closestTile) {
                    return { ...t, clicked: true };
                }
                return t;
            });
            
            score++;
            // Increase speed every 15 points
            if (score % 15 === 0) {
                speed += 0.3;
            }
            return;
        }
        
        if (clickY < closestTile.y - clickTolerance * 2) {
            endGame();
        }
    }

    function endGame(): void {
        gameOver = true;
        gameStarted = false;
        if (animationFrame !== null) {
            cancelAnimationFrame(animationFrame);
        }
        if (audio) {
            audio.pause();
            audio.currentTime = 0;
        }
    }

    onMount(() => {
        const stageData = localStorage.getItem('selectedStage');
        if (!stageData) {
            goto('/mainmenu');
            return;
        }
        selectedStage = JSON.parse(stageData);
        isLoading = false;

        // Initialize audio
        if (selectedStage?.url) {
            audio = new Audio(selectedStage.url);
            audio.addEventListener('canplaythrough', () => {
                console.log('Audio loaded and ready to play');
            });
            audio.addEventListener('error', (e) => {
                console.error('Audio error:', e);
            });
            audio.load();
        }

        return () => {
            if (animationFrame !== null) {
                cancelAnimationFrame(animationFrame);
            }
            if (audio) {
                audio.pause();
                audio.currentTime = 0;
            }
        };
    });
</script>

<main>
    {#if isLoading}
        <div class="loading">Loading...</div>
    {:else}
        <div class="game-header">
            <h1>{selectedStage?.title || 'Piano Tiles'}</h1>
            <p class="song-info">{selectedStage?.song}</p>
            <div class="score">Score: {score}</div>
        </div>

        <div class="game-container">
            {#if !gameStarted}
                <button class="start-button" onclick={startGame}>START!</button>
            {/if}
            {#if gameOver}
                <div class="game-over">
                    <p>Game Over! Score: {score}</p>
                    <button class="restart-button" onclick={startGame}>
                        Try Again
                    </button>
                    <button class="back-button" onclick={() => goto('/mainmenu')}>
                        Back to Menu
                    </button>
                </div>
            {/if}
            <div 
                class="tiles-container"
                role="presentation"
                onclick={(e) => {
                    if (!gameStarted || gameOver) return;
                    
                    // Get click coordinates relative to container
                    const container = e.currentTarget as HTMLElement;
                    const rect = container.getBoundingClientRect();
                    const clickX = e.clientX - rect.left;
                    const clickY = e.clientY - rect.top;

                    // Find if we clicked on any unclicked black tile
                    const clickedOnTile = tiles.some(tile => {
                        if (tile.clicked) return false;
                        
                        const tileLeft = tile.column * (rect.width / 4);
                        const tileRight = tileLeft + (rect.width / 4);
                        const tileTop = tile.y;
                        const tileBottom = tile.y + 140;

                        return clickX >= tileLeft && 
                               clickX <= tileRight && 
                               clickY >= tileTop && 
                               clickY <= tileBottom;
                    });

                    // If clicked outside any valid tile, end game
                    if (!clickedOnTile) {
                        endGame();
                    }
                }}
            >
                {#each tiles as tile}
                    <button
                        type="button"
                        class="tile {tile.clicked ? 'clicked' : ''}"
                        style="left: {tile.column * 25}%; top: {tile.y}px; height: 140px;"
                        onmousedown={(e) => {
                            e.stopPropagation(); // Prevent container click when clicking tile
                            handleTileClick(tile, e);
                        }}
                        onclick={(e) => e.preventDefault()}
                        disabled={tile.clicked}
                        aria-label="Piano tile column {tile.column + 1}"
                    ></button>
                {/each}
            </div>
        </div>
    {/if}
</main>

<style>
    main {
        display: flex;
        flex-direction: column;
        align-items: center;
        min-height: 100vh;
        background: linear-gradient(180deg, #2c3e50 0%, #3498db 100%);
        color: white;
        padding: 20px;
    }

    .loading {
        font-size: 1.5em;
        margin-top: 40px;
    }

    .game-header {
        text-align: center;
        margin-bottom: 30px;
    }

    h1 {
        font-size: 2.5em;
        margin-bottom: 10px;
    }

    .song-info {
        font-size: 1.2em;
        color: #e0e0e0;
        margin-bottom: 10px;
    }

    .score {
        font-size: 1.5em;
        font-weight: bold;
        color: #2ecc71;
    }

    .game-container {
        background: rgba(0, 0, 0, 0.5);
        border-radius: 10px;
        width: 400px;
        height: 600px;
        position: relative;
        overflow: hidden;
        margin: 20px 0;
    }

    .tiles-container {
        width: 100%;
        height: 100%;
        position: relative;
        touch-action: none;
        padding: 8px 0;
        user-select: none; /* Prevent text selection */
    }

    .tile {
        position: absolute;
        width: 25%;
        background: #000;
        border: none;
        cursor: pointer;
        padding: 0;
        margin: 0;
        border: 1px solid rgba(255, 255, 255, 0.1);
        touch-action: none;
        -webkit-tap-highlight-color: transparent;
        user-select: none;
        z-index: 1;
    }

    .tile:active {
        background: #333;
    }

    .tile.clicked {
        background: #2ecc71;
        cursor: default;
    }

    /* Improve hover feedback */
    @media (hover: hover) {
        .tile:hover:not(.clicked):not(:active) {
            background: #222;
        }
    }

    .start-button {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        padding: 20px 40px;
        font-size: 24px;
        background: #2ecc71;
        z-index: 10;
    }

    .game-over {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        background: rgba(231, 76, 60, 0.9);
        padding: 20px;
        border-radius: 10px;
        text-align: center;
        z-index: 10;
        display: flex;
        flex-direction: column;
        gap: 10px;
    }

    .restart-button {
        background: #2ecc71;
    }

    .restart-button:hover {
        transform: scale(1.05);
        background: #27ae60;
    }

    .back-button {
        background: #e74c3c;
    }

    .back-button:hover {
        transform: scale(1.05);
        background: #c0392b;
    }

    button {
        padding: 15px 30px;
        font-size: 18px;
        color: white;
        border: none;
        border-radius: 5px;
        cursor: pointer;
        transition: transform 0.2s, background-color 0.2s;
    }
</style>
  
  