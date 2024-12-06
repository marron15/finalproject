<script>
  import { goto } from '$app/navigation';
  import { onMount } from 'svelte';
  let gameStarted = false;
  let btnHovered = false;

  function startGame() {
    gameStarted = true;
    goto('/flappybird');
  }

  // Arcade-style blinking effect for text
  let visible = true;
  onMount(() => {
    setInterval(() => {
      visible = !visible;
    }, 800);
  });
</script>

<main class="arcade-container">
  <div class="scanline"></div>
  <div class="crt-overlay"></div>
  
  <div class="content">
    <h1 class="arcade-title">FLAPPY BIRD</h1>
    <div class="bird-icon">
      <svg width="64" height="64" viewBox="0 0 64 64" fill="none">
        <path d="M32 16C32 16 45 20 45 32C45 44 32 48 32 48C32 48 19 44 19 32C19 20 32 16 32 16Z" 
              fill="#FFD700" 
              stroke="#FFA500" 
              stroke-width="2"/>
        <circle cx="36" cy="28" r="2" fill="#000"/>
        <path d="M40 32C40 32 42 33 44 32" stroke="#FFA500" stroke-width="2" stroke-linecap="round"/>
      </svg>
    </div>
    
    {#if !gameStarted}
      <button 
        on:click={startGame} 
        on:mouseenter={() => btnHovered = true}
        on:mouseleave={() => btnHovered = false}
        class="arcade-button"
        class:hovered={btnHovered}
      >
        <div class="button-content">
          <span class="play-icon">▶</span>
          <span class:blink={visible} class="press-start">PRESS START</span>
        </div>
      </button>
    {/if}

    <div class="instructions">
      <p>SPACE / CLICK / TAP TO JUMP</p>
      <p>ESC TO PAUSE</p>
    </div>
  </div>
</main>

<style>
  .arcade-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    background-color: #000;
    position: relative;
    overflow: hidden;
    font-family: 'Press Start 2P', monospace;
    color: #fff;
  }

  .content {
    z-index: 2;
    text-align: center;
    padding: 2rem;
  }

  .arcade-title {
    font-size: 3rem;
    margin-bottom: 1rem;
    color: #ffd700;
    text-shadow: 0 0 10px #ffd700,
                 0 0 20px #ffd700,
                 0 0 30px #ff4500;
    letter-spacing: 4px;
  }

  .bird-icon {
    font-size: 4rem;
    margin: 2rem 0;
    animation: float 2s ease-in-out infinite;
    filter: drop-shadow(0 0 10px rgba(255, 215, 0, 0.5));
  }

  .arcade-button {
    background: rgba(255, 69, 0, 0.2);
    border: 2px solid rgba(255, 255, 255, 0.4);
    border-radius: 50%;
    width: 150px;
    height: 150px;
    cursor: pointer;
    position: relative;
    transition: all 0.3s ease;
    backdrop-filter: blur(5px);
    -webkit-backdrop-filter: blur(5px);
  }

  .arcade-button:hover, .arcade-button.hovered {
    background: rgba(255, 69, 0, 0.3);
    border-color: rgba(255, 255, 255, 0.6);
    transform: scale(1.1);
    box-shadow: 0 0 30px rgba(255, 69, 0, 0.3);
  }

  .button-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100%;
  }

  .play-icon {
    font-size: 2.5rem;
    margin-bottom: 0.5rem;
    color: rgba(255, 255, 255, 0.9);
    text-shadow: 0 0 10px rgba(255, 69, 0, 0.5);
  }

  .press-start {
    font-size: 0.8rem;
    white-space: nowrap;
    color: rgba(255, 255, 255, 0.8);
    text-shadow: 0 0 5px rgba(255, 69, 0, 0.5);
  }

  .blink {
    opacity: 0;
  }

  .instructions {
    margin-top: 3rem;
    font-size: 0.8rem;
    color: rgba(255, 255, 255, 0.6);
    line-height: 1.8;
    text-shadow: 0 0 5px rgba(255, 69, 0, 0.3);
  }

  .instructions p {
    margin: 0.5rem 0;
  }

  /* Scanline effect */
  .scanline {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: linear-gradient(
      to bottom,
      transparent 50%,
      rgba(0, 0, 0, 0.1) 50%
    );
    background-size: 100% 4px;
    z-index: 1;
    pointer-events: none;
  }

  /* CRT overlay effect */
  .crt-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: radial-gradient(
      circle at center,
      transparent 30%,
      rgba(0, 0, 0, 0.2) 100%
    );
    pointer-events: none;
    z-index: 1;
  }

  @keyframes float {
    0%, 100% {
      transform: translateY(0) rotate(0deg);
    }
    50% {
      transform: translateY(-20px) rotate(5deg);
    }
  }

  @media (max-width: 768px) {
    .arcade-title {
      font-size: 2rem;
    }

    .arcade-button {
      width: 120px;
      height: 120px;
    }

    .play-icon {
      font-size: 2rem;
    }

    .press-start {
      font-size: 0.6rem;
    }

    .instructions {
      font-size: 0.7rem;
    }
  }

  /* Add Press Start 2P font */
  @font-face {
    font-family: 'Press Start 2P';
    font-style: normal;
    font-weight: 400;
    src: url('https://fonts.gstatic.com/s/pressstart2p/v14/e3t4euO8T-267oIAQAu6jDQyK3nVivM.woff2') format('woff2');
    font-display: swap;
  }
</style>
