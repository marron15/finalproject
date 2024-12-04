<script>
  import { goto } from '$app/navigation';
</script>

<main>
  <h1>Welcome to Piano Tiles</h1>
  <button on:click={() => goto('/pianotiles')}>Play Game</button>
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100vh;
    background: linear-gradient(180deg, #4a90e2 0%, #7e57c2 100%);
    color: white;
  }

  button {
    padding: 15px 30px;
    font-size: 20px;
    background: #2ecc71;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    margin-top: 20px;
    transition: transform 0.2s;
  }

  button:hover {
    transform: scale(1.05);
  }

  h1 {
    font-size: 2.5em;
    margin-bottom: 20px;
  }
</style>

