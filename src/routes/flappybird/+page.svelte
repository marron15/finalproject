<script lang="ts">
    import { onMount, onDestroy } from 'svelte';
    
    let canvas: HTMLCanvasElement;
    let ctx: CanvasRenderingContext2D;
    let isMobile = false;
    let canvasWidth = 800;
    let canvasHeight = 600;
    let bird = {
        x: 50,
        y: 150,
        velocity: 0,
        gravity: 0.15,
        jumpForce: -4,
        size: 20,
        maxVelocity: 4,
        rotation: 0,
        wingOffset: 0,
        wingSpeed: 0.15,
        hitboxSize: 16,
        airResistance: 0.98
    };
    
    let pipes: Array<{x: number, gap: number}> = [];
    let gameOver = false;
    let score = 0;
    let animationFrame: number;
    let lastJumpTime = 0;
    
    const PIPE_SPEED = 1.5;
    const PIPE_WIDTH = 50;
    const PIPE_GAP = 180;
    const JUMP_COOLDOWN = 150;
    const COLLISION_BUFFER = 5;
    const MIN_VELOCITY = 0.5;
    
    function adjustForMobile() {
        isMobile = window.innerWidth < 768;
        canvasWidth = isMobile ? window.innerWidth - 40 : 800;
        canvasHeight = isMobile ? window.innerHeight - 200 : 600;
        
        if (canvas) {
            canvas.width = canvasWidth;
            canvas.height = canvasHeight;
            
            // Adjust game parameters for mobile
            if (isMobile) {
                bird.gravity = 0.12;
                bird.jumpForce = -3.5;
                bird.size = Math.min(20, canvasWidth / 30);
                bird.hitboxSize = bird.size * 0.8;
            }
        }
    }
    
    function handleResize() {
        adjustForMobile();
    }
    
    function jump() {
        if (!gameOver) {
            const currentTime = Date.now();
            if (currentTime - lastJumpTime >= JUMP_COOLDOWN) {
                bird.velocity = bird.jumpForce;
                bird.rotation = -0.5;
                lastJumpTime = currentTime;
            }
        }
    }
    
    function updateBirdPhysics() {
        bird.velocity += bird.gravity;
        
        if (bird.velocity > MIN_VELOCITY) {
            bird.velocity *= bird.airResistance;
        }
        
        bird.velocity = Math.min(bird.velocity, bird.maxVelocity);
        
        bird.y += bird.velocity;
    }
    
    function createPipe() {
        const gap = Math.random() * 200 + 100;
        pipes.push({
            x: canvas.width,
            gap
        });
        pipes = pipes;
    }
    
    function drawBird() {
        ctx.save();
        ctx.translate(bird.x, bird.y);
        
        const targetRotation = bird.velocity * 0.1;
        bird.rotation += (targetRotation - bird.rotation) * 0.1;
        ctx.rotate(bird.rotation);
        
        // Body shadow
        ctx.fillStyle = '#e67e22';
        ctx.beginPath();
        ctx.ellipse(2, 2, bird.size, bird.size * 0.8, 0, 0, Math.PI * 2);
        ctx.fill();
        
        // Main body
        ctx.fillStyle = '#f1c40f';
        ctx.beginPath();
        ctx.ellipse(0, 0, bird.size, bird.size * 0.8, 0, 0, Math.PI * 2);
        ctx.fill();
        
        // Wing animation
        bird.wingOffset = Math.sin(Date.now() * bird.wingSpeed) * 8;
        
        // Wings with gradient
        const wingGradient = ctx.createLinearGradient(-15, -10, 5, 10);
        wingGradient.addColorStop(0, '#f39c12');
        wingGradient.addColorStop(1, '#e67e22');
        ctx.fillStyle = wingGradient;
        
        // Top wing
        ctx.beginPath();
        ctx.ellipse(-5, -10 + bird.wingOffset/2, 12, 6, Math.PI/4, 0, Math.PI * 2);
        ctx.fill();
        
        // Bottom wing
        ctx.beginPath();
        ctx.ellipse(-5, 10 + bird.wingOffset, 12, 6, -Math.PI/4, 0, Math.PI * 2);
        ctx.fill();
        
        // Beak
        ctx.fillStyle = '#e74c3c';
        ctx.beginPath();
        ctx.moveTo(bird.size - 2, -4);
        ctx.lineTo(bird.size + 8, 0);
        ctx.lineTo(bird.size - 2, 4);
        ctx.fill();
        
        // Eye white
        ctx.fillStyle = '#fff';
        ctx.beginPath();
        ctx.arc(bird.size/2, -bird.size/4, 4, 0, Math.PI * 2);
        ctx.fill();
        
        // Eye pupil
        ctx.fillStyle = '#2c3e50';
        ctx.beginPath();
        ctx.arc(bird.size/2 + 1, -bird.size/4, 2, 0, Math.PI * 2);
        ctx.fill();
        
        // Chest/belly highlight
        ctx.fillStyle = '#f9e076';
        ctx.beginPath();
        ctx.ellipse(-2, 2, bird.size * 0.6, bird.size * 0.5, 0, 0, Math.PI * 2);
        ctx.fill();
        
        ctx.restore();
    }
    
    function checkCollision(pipe: {x: number, gap: number}): boolean {
        const hitboxX = bird.x;
        const hitboxY = bird.y;
        const hitboxSize = bird.hitboxSize;

        const leftPipe = pipe.x + COLLISION_BUFFER;
        const rightPipe = pipe.x + PIPE_WIDTH - COLLISION_BUFFER;
        const topPipe = pipe.gap - PIPE_GAP/2 + COLLISION_BUFFER;
        const bottomPipe = pipe.gap + PIPE_GAP/2 - COLLISION_BUFFER;

        const withinPipeX = hitboxX + hitboxSize > leftPipe && 
                           hitboxX - hitboxSize < rightPipe;
        
        const hitsTopPipe = hitboxY - hitboxSize < topPipe;
        const hitsBottomPipe = hitboxY + hitboxSize > bottomPipe;

        return withinPipeX && (hitsTopPipe || hitsBottomPipe);
    }
    
    function gameLoop() {
        if (!ctx || gameOver) return;
        
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        updateBirdPhysics();
        
        drawBird();
        
        for (let i = pipes.length - 1; i >= 0; i--) {
            const pipe = pipes[i];
            pipe.x -= PIPE_SPEED;
            
            ctx.fillStyle = '#2ecc71';
            ctx.fillRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
            ctx.fillRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
            
            if (checkCollision(pipe)) {
                gameOver = true;
            }
            
            if (pipe.x + PIPE_WIDTH < 0) {
                pipes.splice(i, 1);
                score++;
                pipes = pipes;
            }
        }
        
        if (bird.y - bird.size < COLLISION_BUFFER || 
            bird.y + bird.size > canvas.height - COLLISION_BUFFER) {
            gameOver = true;
        }
        
        if (pipes.length === 0 || pipes[pipes.length - 1].x < canvas.width - 300) {
            createPipe();
        }
        
        ctx.fillStyle = '#000';
        ctx.font = '24px Arial';
        ctx.fillText(`Score: ${score}`, 10, 30);
        
        if (gameOver) {
            ctx.fillStyle = '#e74c3c';
            ctx.font = '48px Arial';
            ctx.fillText('Game Over!', canvas.width/2 - 100, canvas.height/2);
            ctx.font = '24px Arial';
            ctx.fillText('Click to restart', canvas.width/2 - 60, canvas.height/2 + 40);
        } else {
            animationFrame = requestAnimationFrame(gameLoop);
        }
    }
    
    function startGame() {
        if (gameOver) {
            gameOver = false;
            bird.y = 150;
            bird.velocity = 0;
            pipes = [];
            score = 0;
            animationFrame = requestAnimationFrame(gameLoop);
        }
    }
    
    function handleKeydown(e: KeyboardEvent) {
        if (e.code === 'Space') {
            e.preventDefault();
            gameOver ? startGame() : jump();
        }
    }
    
    onMount(() => {
        ctx = canvas.getContext('2d')!;
        adjustForMobile();
        animationFrame = requestAnimationFrame(gameLoop);
        
        window.addEventListener('keydown', handleKeydown);
        window.addEventListener('resize', handleResize);
        window.addEventListener('touchstart', (e) => {
            e.preventDefault();
            gameOver ? startGame() : jump();
        }, { passive: false });
    });
    
    onDestroy(() => {
        if (animationFrame) {
            cancelAnimationFrame(animationFrame);
        }
        window.removeEventListener('keydown', handleKeydown);
        window.removeEventListener('resize', handleResize);
    });
</script>

<svelte:head>
    <title>Flappy Bird</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
</svelte:head>

<div class="game-container">
    <canvas
        bind:this={canvas}
        on:click={gameOver ? startGame : jump}
        tabindex="0"
    />
    <div class="instructions">
        {isMobile ? 'Tap to jump/start' : 'Click or press Space to jump/start'}
    </div>
</div>

<style>
    .game-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        padding: 20px;
        background-color: #f0f0f0;
        min-height: 100vh;
        touch-action: manipulation;
    }
    
    canvas {
        border: 2px solid #333;
        border-radius: 8px;
        background-color: #87CEEB;
        cursor: pointer;
        max-width: 100%;
        touch-action: none;
    }
    
    .instructions {
        margin-top: 20px;
        font-size: 18px;
        color: #333;
        text-align: center;
        padding: 10px;
    }

    @media (max-width: 768px) {
        .game-container {
            padding: 10px;
        }
        
        .instructions {
            font-size: 16px;
        }
    }
</style>
