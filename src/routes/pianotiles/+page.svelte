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
    let targetScore = 50; // Score needed to win the stage
    let stageCompleted = false;

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
            timestamp
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
        const deltaTime = timestamp - lastTimestamp;
        const gameTime = timestamp - startTime;
        lastTimestamp = timestamp;

        // Move existing tiles down
        tiles.forEach(tile => {
            tile.y += speed * deltaTime / 16;
        });

        // Generate new tiles based on rhythm pattern
        if (selectedStage?.id && rhythmPatterns[selectedStage.id]) {
            const pattern = rhythmPatterns[selectedStage.id];
            const currentBeat = pattern.find(beat => 
                Math.abs((gameTime % 12000) - (beat % 12000)) < 20 && // 12000ms = pattern length
                !tiles.some(tile => Math.abs(tile.timestamp - beat) < 20)
            );

            if (currentBeat !== undefined) {
                tiles = [...tiles, createTile(currentBeat)];
            }
        }

        // Check for missed tiles with more forgiving bottom line
        const bottomLine = window.innerHeight - 150; // Less strict bottom line
        const failedTile = tiles.find(tile => !tile.clicked && tile.y > bottomLine);

        if (failedTile) {
            endGame();
            return;
        }

        // Remove tiles that are off screen
        tiles = tiles.filter(tile => tile.y < window.innerHeight + 150);

        if (!gameOver) {
            animationFrame = requestAnimationFrame(animate);
        }
    }

    function handleTileClick(tile: Tile, event: MouseEvent): void {
        if (!gameStarted || gameOver || tile.clicked) return;
        
        event.preventDefault();
        event.stopPropagation();
        
        // Get visible unclicked tiles in the same column with more forgiving bounds
        const columnTiles = tiles.filter(t => 
            !t.clicked && 
            t.column === tile.column && 
            t.y >= -200 && // More forgiving top bound
            t.y <= window.innerHeight - 50 // More forgiving bottom bound
        );

        // If no tiles in column, ignore click
        if (columnTiles.length === 0) return;
        
        // Sort tiles by Y position and get the lowest one
        const sortedTiles = columnTiles.sort((a, b) => b.y - a.y);
        const lowestTile = sortedTiles[0];
        
        // Add a tolerance range for clicking (30px up or down)
        const clickTolerance = 30;
        const isWithinRange = Math.abs(tile.y - lowestTile.y) <= clickTolerance;
        
        if (tile === lowestTile || (tile.column === lowestTile.column && isWithinRange)) {
            lowestTile.clicked = true; // Always click the lowest tile if within range
            score++;
            if (score % 15 === 0) {
                speed += 0.3;
            }
            if (score >= targetScore) {
                stageCompleted = true;
                endGame();
            }
        } else if (columnTiles.includes(tile) && !isWithinRange) {
            // Only end game if we clicked a wrong tile and it's not within range
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
                <div class="game-over" class:completed={stageCompleted}>
                    <p>
                        {#if stageCompleted}
                            Stage Complete!
                        {:else}
                            Game Over!
                        {/if}
                        Score: {score}
                    </p>
                    <button class="restart-button" onclick={startGame}>
                        {stageCompleted ? 'Play Again' : 'Try Again'}
                    </button>
                </div>
            {/if}
            <div 
                class="tiles-container"
                role="presentation"
                onclick={(e) => {
                    e.preventDefault();
                    if (gameStarted && !gameOver) endGame();
                }}
            >
                {#each tiles as tile}
                    <button
                        type="button"
                        class="tile {tile.clicked ? 'clicked' : ''}"
                        style="left: {tile.column * 25}%; top: {tile.y}px;"
                        onclick={(e) => {
                            e.preventDefault();
                            e.stopPropagation();
                            handleTileClick(tile, e);
                        }}
                        disabled={tile.clicked}
                        aria-label="Piano tile column {tile.column + 1}"
                    ></button>
                {/each}
            </div>
        </div>

        <div class="controls">
            <button class="back-button" onclick={() => goto('/mainmenu')}>Back to Menu</button>
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
        height: 140px;
        background: #000;
        border: none;
        cursor: pointer;
        transition: background-color 0.2s;
        padding: 0;
        margin: 0;
        border: 1px solid rgba(255, 255, 255, 0.1);
        margin-bottom: 15px;
        touch-action: manipulation; /* Optimize for touch */
        -webkit-tap-highlight-color: transparent; /* Remove tap highlight on mobile */
    }

    .tile:active {
        background: #333; /* Visual feedback when pressing */
    }

    .tile.clicked {
        background: #2ecc71;
        cursor: default;
    }

    /* Add hover effect only on devices that support hover */
    @media (hover: hover) {
        .tile:hover:not(.clicked) {
            background: #333;
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
    }

    .game-over.completed {
        background: rgba(46, 204, 113, 0.9); /* Green background for completion */
    }

    .controls {
        margin-top: 20px;
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

    .back-button {
        background: #e74c3c;
    }

    .back-button:hover {
        transform: scale(1.05);
        background: #c0392b;
    }

    .restart-button {
        background: #2ecc71;
    }

    .restart-button:hover {
        transform: scale(1.05);
        background: #27ae60;
    }
</style>
  
  