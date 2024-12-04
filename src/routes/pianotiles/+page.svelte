<script lang="ts">
    import { onMount } from 'svelte';
  
    interface Tile {
      column: number;
      y: number;
      clicked: boolean;
    }
  
    let score = 0;
    let gameStarted = false;
    let gameOver = false;
    let tiles: Tile[] = [];
    let speed = 2;
    let animationFrame: number | null = null;
    let lastTimestamp = 0;
    let maxTiles = 4; // Maximum number of tiles on screen
  
    function createTile(): Tile {
      const column = Math.floor(Math.random() * 4);
      return {
        column,
        y: -150,
        clicked: false
      };
    }
  
    function startGame(): void {
      if (gameStarted) return;
      gameStarted = true;
      gameOver = false;
      score = 0;
      speed = 2;
      tiles = [createTile()];
      lastTimestamp = performance.now();
      animate();
    }
  
    function animate(timestamp = 0): void {
      const deltaTime = timestamp - lastTimestamp;
      lastTimestamp = timestamp;
  
      // Move tiles down
      tiles.forEach(tile => {
        tile.y += speed * deltaTime / 16;
      });
  
      // Add new tile if we have space and the last tile is far enough down
      if (tiles.length < maxTiles && (tiles.length === 0 || tiles[tiles.length - 1].y > 150)) {
        tiles = [...tiles, createTile()];
      }
  
      // Check for missed tiles or tiles that passed the bottom
      const bottomLine = window.innerHeight - 200; // Define bottom line position
      const failedTile = tiles.find(tile => {
        // Game over if an unclicked tile passes the bottom line
        if (!tile.clicked && tile.y > bottomLine) {
          return true;
        }
        return false;
      });
  
      if (failedTile) {
        endGame();
        return;
      }
  
      // Clean up tiles that are way past the bottom
      tiles = tiles.filter(tile => tile.y < window.innerHeight);
  
      if (!gameOver) {
        animationFrame = requestAnimationFrame(animate);
      }
    }
  
    function handleTileClick(tile: Tile): void {
      if (!gameStarted || gameOver || tile.clicked) return;
      
      // Find the lowest unclicked tile
      const lowestUnclickedTile = tiles
        .filter(t => !t.clicked)
        .sort((a, b) => b.y - a.y)[0];
      
      if (tile === lowestUnclickedTile) {
        tile.clicked = true;
        score++;
        speed += 0.1; // Increase speed as score increases
      } else {
        endGame();
      }
    }
  
    function endGame(): void {
      gameOver = true;
      gameStarted = false;
      if (animationFrame !== null) {
        cancelAnimationFrame(animationFrame);
      }
    }
  
    onMount(() => {
      return () => {
        if (animationFrame !== null) {
          cancelAnimationFrame(animationFrame);
        }
      };
    });
  </script>
  
  <main>
    <div class="game-container">
      <div class="score">Score: {score}</div>
      {#if !gameStarted}
        <button class="start-button" on:click={startGame}>START!</button>
      {/if}
      {#if gameOver}
        <div class="game-over">
          <p>Game Over! Score: {score}</p>
          <button class="restart-button" on:click={startGame}>Play Again</button>
        </div>
      {/if}
      <div 
        class="tiles-container"
        on:click|self={() => {
          if (gameStarted && !gameOver) endGame();
        }}
      >
        {#each tiles as tile}
          <button
            type="button"
            class="tile {tile.clicked ? 'clicked' : ''}"
            style="left: {tile.column * 25}%; top: {tile.y}px;"
            on:click|stopPropagation={() => handleTileClick(tile)}
            on:keydown={(e) => e.key === 'Enter' && handleTileClick(tile)}
            aria-label="Piano tile {tile.column + 1}"
          ></button>
        {/each}
      </div>
    </div>
  </main>
  
  <style>
    main {
      width: 100%;
      height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      background: linear-gradient(180deg, #4a90e2 0%, #7e57c2 100%);
    }
  
    .game-container {
      width: 400px;
      height: 80vh;
      position: relative;
      overflow: hidden;
      background: rgba(255, 255, 255, 0.1);
      border-radius: 10px;
    }
  
    .score {
      position: absolute;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      font-size: 24px;
      color: white;
      z-index: 10;
    }
  
    .start-button {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      padding: 15px 30px;
      font-size: 24px;
      background: #2ecc71;
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      z-index: 10;
    }
  
    .game-over {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      text-align: center;
      background: rgba(231, 76, 60, 0.9);
      padding: 20px;
      border-radius: 5px;
      z-index: 10;
    }
  
    .game-over p {
      font-size: 24px;
      color: white;
      margin: 0 0 20px 0;
    }
  
    .restart-button {
      padding: 10px 20px;
      font-size: 18px;
      background: #2ecc71;
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      transition: transform 0.2s;
    }
  
    .restart-button:hover {
      transform: scale(1.05);
    }
  
    .tiles-container {
      width: 100%;
      height: 100%;
      position: relative;
    }
  
    .tile {
      position: absolute;
      width: 25%;
      height: 150px;
      background: #000;
      cursor: pointer;
      transition: background-color 0.2s;
    }
  
    .tile.clicked {
      background: #2ecc71;
    }
  
    :global(body) {
      margin: 0;
      padding: 0;
      font-family: Arial, sans-serif;
    }
  </style>
  
  