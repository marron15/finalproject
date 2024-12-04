<script>
  import { goto } from '$app/navigation';
  let gameStarted = false;

  function startGame() {
    gameStarted = true;
    goto('/flappybird');
  }
</script>

<main class="container">
  <h1>Welcome to Flappy Bird</h1>
  {#if !gameStarted}
    <button on:click={startGame} class="start-button">
      Start Game
    </button>
  {/if}
</main>

<style>
  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    background-color: #87CEEB;
  }

  h1 {
    color: #333;
    margin-bottom: 2rem;
    font-size: 2.5rem;
  }

  .start-button {
    padding: 1rem 2rem;
    font-size: 1.25rem;
    background-color: #4CAF50;
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    transition: background-color 0.3s ease;
  }

  .start-button:hover {
    background-color: #45a049;
  }
</style>
