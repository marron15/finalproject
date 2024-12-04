<script lang="ts">
    import type { PageData } from './$types';
    import { goto } from '$app/navigation';

    let { data }: { data: PageData } = $props();

    interface Stage {
      id: number;
      title: string;
      song: string;
      url: string;
    }

    const stages: Stage[] = [
      {
        id: 1,
        title: "Stage 1",
        song: "Chipi Chipi Chapa",
        url: "/audio/1 HOUR LOOP of chipi chipi chapa chapa trend!.mp3"
      },
      {
        id: 2,
        title: "Stage 2",
        song: "Coming Soon",
        url: ""
      },
      {
        id: 3,
        title: "Stage 3",
        song: "Coming Soon",
        url: ""
      }
    ];

    function selectStage(stage: Stage) {
      // Store the selected stage information (you might want to use a store)
      localStorage.setItem('selectedStage', JSON.stringify(stage));
      goto('/pianotiles');
    }
</script>

<main>
  <h1>Select a Stage</h1>
  
  <div class="stages-container">
    {#each stages as stage}
      <div class="stage-card">
        <h2>{stage.title}</h2>
        <p class="song-title">{stage.song}</p>
        <button 
          onclick={() => selectStage(stage)}
          class:disabled={!stage.url}
          disabled={!stage.url}
        >
          Play Stage
        </button>
      </div>
    {/each}
  </div>

  <button class="back-button" onclick={() => goto('/')}>Back to Home</button>
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    background: linear-gradient(180deg, #4a90e2 0%, #7e57c2 100%);
    color: white;
    padding: 20px;
  }

  h1 {
    font-size: 2.5em;
    margin-bottom: 40px;
  }

  .stages-container {
    display: flex;
    gap: 30px;
    flex-wrap: wrap;
    justify-content: center;
    max-width: 1200px;
  }

  .stage-card {
    background: rgba(255, 255, 255, 0.1);
    padding: 20px;
    border-radius: 10px;
    text-align: center;
    backdrop-filter: blur(5px);
    width: 300px;
    transition: transform 0.2s;
  }

  .stage-card:hover {
    transform: translateY(-5px);
  }

  .song-title {
    font-size: 1.1em;
    margin: 10px 0;
    color: #e0e0e0;
  }

  button {
    padding: 15px 30px;
    font-size: 18px;
    background: #2ecc71;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    transition: transform 0.2s, background-color 0.2s;
  }

  button:hover:not(.disabled) {
    transform: scale(1.05);
    background: #27ae60;
  }

  button.disabled {
    background: #95a5a6;
    cursor: not-allowed;
  }

  .back-button {
    margin-top: 30px;
    background: #e74c3c;
  }

  .back-button:hover {
    background: #c0392b;
  }
</style>