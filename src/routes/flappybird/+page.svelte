<script lang="ts">
    import { onMount, onDestroy } from 'svelte';
    import { goto } from '$app/navigation';
    
    let canvas: HTMLCanvasElement;
    let ctx: CanvasRenderingContext2D;
    let isMobile = false;
    let canvasWidth = 800;
    let canvasHeight = 600;
    let bird = {
        x: 50,
        y: 150,
        velocity: 0,
        gravity: 0.12,
        jumpForce: -3.2,
        size: 22,
        maxVelocity: 3.5,
        rotation: 0,
        wingOffset: 0,
        wingSpeed: 0.15,
        hitboxSize: 18,
        airResistance: 0.99
    };
    
    let pipes: Array<{x: number, gap: number}> = [];
    let gameOver = false;
    let score = 0;
    let animationFrame: number;
    let lastJumpTime = 0;
    let countdown = 3;
    let isCountingDown = true;
    let previousCountdown = 0;
    let isPaused = false;
    let gameTimer = 0;
    let gameTimerInterval: number;
    let isResumingCountdown = false;
    let resumeCountdown = 0;
    let mainMenuBtnHovered = false;
    
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
        if (!gameOver && !isPaused) {
            const currentTime = Date.now();
            if (currentTime - lastJumpTime >= JUMP_COOLDOWN) {
                bird.velocity = bird.jumpForce;
                bird.rotation = -0.4;
                lastJumpTime = currentTime;
            }
        }
    }
    
    function updateBirdPhysics() {
        bird.velocity += bird.gravity;
        
        if (bird.velocity > 0) {
            bird.velocity *= bird.airResistance;
        }
        
        bird.velocity = Math.min(bird.velocity, bird.maxVelocity);
        
        bird.y += bird.velocity;
        
        const targetRotation = bird.velocity * 0.08;
        bird.rotation += (targetRotation - bird.rotation) * 0.1;
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
        if (!ctx) return;
        
        ctx.save();
        ctx.translate(bird.x, bird.y);
        
        // Smooth rotation for more natural movement
        const targetRotation = bird.velocity * 0.08;
        bird.rotation += (targetRotation - bird.rotation) * 0.1;
        ctx.rotate(bird.rotation);
        
        // Wing animation
        bird.wingOffset = Math.sin(Date.now() * bird.wingSpeed) * 8;
        
        // Body shadow for depth
        ctx.fillStyle = '#e67e22';
        ctx.beginPath();
        ctx.ellipse(2, 2, bird.size * 0.7, bird.size * 0.5, 0, 0, Math.PI * 2);
        ctx.fill();
        
        // Main body - thinner and more oval
        const bodyGradient = ctx.createRadialGradient(0, 0, 0, 0, 0, bird.size);
        bodyGradient.addColorStop(0, '#ffd54f');  // Lighter center
        bodyGradient.addColorStop(1, '#ffb300');  // Darker edge
        ctx.fillStyle = bodyGradient;
        ctx.beginPath();
        ctx.ellipse(0, 0, bird.size * 0.7, bird.size * 0.5, 0, 0, Math.PI * 2);
        ctx.fill();
        
        // Wings with gradient - medium size
        const wingGradient = ctx.createLinearGradient(-15, -10, 5, 10);
        wingGradient.addColorStop(0, '#ffb300');
        wingGradient.addColorStop(1, '#ffa000');
        ctx.fillStyle = wingGradient;
        
        // Top wing - medium size
        ctx.beginPath();
        ctx.ellipse(-bird.size * 0.3, -bird.size * 0.3 + bird.wingOffset/2, 
                    bird.size * 0.4, bird.size * 0.25, Math.PI/4, 0, Math.PI * 2);
        ctx.fill();
        
        // Bottom wing - medium size
        ctx.beginPath();
        ctx.ellipse(-bird.size * 0.3, bird.size * 0.1 + bird.wingOffset, 
                    bird.size * 0.4, bird.size * 0.25, -Math.PI/4, 0, Math.PI * 2);
        ctx.fill();
        
        // Beak - adjusted for thinner body
        const beakGradient = ctx.createLinearGradient(bird.size * 0.4, 0, bird.size * 0.8, 0);
        beakGradient.addColorStop(0, '#ff7043');
        beakGradient.addColorStop(1, '#ff5722');
        ctx.fillStyle = beakGradient;
        ctx.beginPath();
        ctx.moveTo(bird.size * 0.4, -bird.size * 0.1);
        ctx.lineTo(bird.size * 0.8, 0);
        ctx.lineTo(bird.size * 0.4, bird.size * 0.1);
        ctx.closePath();
        ctx.fill();
        
        // Eye - adjusted position
        ctx.fillStyle = '#fff';
        ctx.beginPath();
        ctx.arc(bird.size * 0.3, -bird.size * 0.1, bird.size * 0.15, 0, Math.PI * 2);
        ctx.fill();
        
        // Pupil
        ctx.fillStyle = '#2c3e50';
        ctx.beginPath();
        ctx.arc(bird.size * 0.35, -bird.size * 0.1, bird.size * 0.08, 0, Math.PI * 2);
        ctx.fill();
        
        // Eye shine
        ctx.fillStyle = '#fff';
        ctx.beginPath();
        ctx.arc(bird.size * 0.25, -bird.size * 0.15, bird.size * 0.05, 0, Math.PI * 2);
        ctx.fill();
        
        // Chest/belly highlight - thinner
        const chestGradient = ctx.createRadialGradient(-2, 2, 0, -2, 2, bird.size * 0.6);
        chestGradient.addColorStop(0, '#fff9c4');
        chestGradient.addColorStop(1, '#ffd54f');
        ctx.fillStyle = chestGradient;
        ctx.beginPath();
        ctx.ellipse(-bird.size * 0.1, bird.size * 0.1, bird.size * 0.5, bird.size * 0.3, 0, 0, Math.PI * 2);
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
    
    function drawCountdown() {
        if (!ctx) return;
        
        // First draw the game state in the background
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = '#87CEEB';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        
        // Draw game elements
        drawBird();
        for (let pipe of pipes) {
            ctx.fillStyle = '#2ecc71';
            ctx.fillRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
            ctx.fillRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
        }
        
        // Draw semi-transparent overlay
        const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
        gradient.addColorStop(0, 'rgba(0, 0, 0, 0.6)');
        gradient.addColorStop(0.5, 'rgba(0, 0, 0, 0.5)');
        gradient.addColorStop(1, 'rgba(0, 0, 0, 0.6)');
        ctx.fillStyle = gradient;
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        
        // Draw countdown number with enhanced styling
        if (countdown > 0) {
            // Shadow for depth
            ctx.shadowColor = '#000';
            ctx.shadowBlur = 15;
            
            // Glow effect
            ctx.fillStyle = '#fff';
            ctx.font = 'bold 72px Arial';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(countdown.toString(), canvas.width/2, canvas.height/2);
            
            // Draw instruction text
            ctx.shadowBlur = 5;
            ctx.font = 'bold 24px Arial';
            ctx.fillText(isMobile ? 'Tap to jump' : 'Click or press Space to jump', canvas.width/2, canvas.height/2 + 60);
            
            // Reset shadow
            ctx.shadowBlur = 0;
        }
    }
    
    async function startCountdown() {
        isCountingDown = true;
        countdown = 3;
        previousCountdown = 0;
        
        // Initialize game state but don't start physics
        bird.y = 150;
        bird.velocity = 0;
        pipes = [];
        score = 0;
        
        // Create initial pipes
        createPipe();
        
        while (countdown > 0) {
            drawCountdown();
            await new Promise(resolve => setTimeout(resolve, 1000));
            countdown--;
        }
        
        isCountingDown = false;
        gameLoop();
    }
    
    function formatTime(seconds: number): string {
        const minutes = Math.floor(seconds / 60);
        const remainingSeconds = seconds % 60;
        return `${minutes}:${remainingSeconds.toString().padStart(2, '0')}`;
    }
    
    function startTimer() {
        gameTimer = 0;
        gameTimerInterval = window.setInterval(() => {
            if (!isPaused && !gameOver && !isCountingDown) {
                gameTimer++;
            }
        }, 1000);
    }
    
    function stopTimer() {
        if (gameTimerInterval) {
            clearInterval(gameTimerInterval);
        }
    }
    
    async function startResumeCountdown() {
        if (gameOver) return;
        
        isResumingCountdown = true;
        resumeCountdown = 3;
        
        while (resumeCountdown > 0) {
            // Draw the current game state
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = '#87CEEB';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            
            // Draw game elements in background
            drawBird();
            for (let pipe of pipes) {
                ctx.fillStyle = '#2ecc71';
                ctx.fillRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
                ctx.fillRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
            }
            
            // Draw semi-transparent overlay
            ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            
            // Draw countdown number with shadow and stroke
            ctx.shadowColor = '#000';
            ctx.shadowBlur = 15;
            ctx.strokeStyle = '#fff';
            ctx.lineWidth = 2;
            ctx.fillStyle = '#fff';
            ctx.font = 'bold 72px Arial';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(resumeCountdown.toString(), canvas.width/2, canvas.height/2);
            ctx.strokeText(resumeCountdown.toString(), canvas.width/2, canvas.height/2);
            
            // Draw "Get Ready!" text
            ctx.font = 'bold 28px Arial';
            ctx.fillText('Get Ready!', canvas.width/2, canvas.height/2 + 60);
            
            ctx.shadowBlur = 0;
            
            await new Promise(resolve => setTimeout(resolve, 1000));
            resumeCountdown--;
        }
        
        isResumingCountdown = false;
        isPaused = false;
        animationFrame = requestAnimationFrame(gameLoop);
    }
    
    function drawResumeCountdown() {
        if (!ctx) return;
        ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = '#fff';
        ctx.font = '72px Poppins';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(resumeCountdown.toString(), canvas.width/2, canvas.height/2);
        ctx.font = '24px Poppins';
        ctx.fillText('Get Ready!', canvas.width/2, canvas.height/2 + 60);
    }
    
    function togglePause() {
        if (!gameOver && !isCountingDown && !isResumingCountdown) {
            if (isPaused) {
                startResumeCountdown();
            } else {
                isPaused = true;
                if (animationFrame) {
                    cancelAnimationFrame(animationFrame);
                }
                drawPauseOverlay();
            }
        }
    }
    
    function drawPauseButton() {
        if (!ctx) return;
        
        // Draw button background with gradient
        const gradient = ctx.createLinearGradient(10, 10, 10, 50);
        gradient.addColorStop(0, 'rgba(0, 0, 0, 0.7)');
        gradient.addColorStop(1, 'rgba(0, 0, 0, 0.8)');
        
        // Draw rounded rectangle for button
        ctx.fillStyle = gradient;
        ctx.beginPath();
        ctx.roundRect(10, 10, 40, 40, 8);
        ctx.fill();
        
        // Draw pause/play icon with shadow
        ctx.shadowColor = 'rgba(0, 0, 0, 0.3)';
        ctx.shadowBlur = 5;
        ctx.fillStyle = '#fff';
        
        if (isPaused) {
            // Draw play triangle
            ctx.beginPath();
            ctx.moveTo(22, 20);
            ctx.lineTo(22, 40);
            ctx.lineTo(42, 30);
            ctx.closePath();
            ctx.fill();
        } else {
            // Draw pause bars
            ctx.fillRect(22, 20, 6, 20);
            ctx.fillRect(32, 20, 6, 20);
        }
        ctx.shadowBlur = 0;
    }
    
    function drawPauseOverlay() {
        if (!ctx || !isPaused || gameOver) return;
        
        // Create semi-transparent overlay
        ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        
        // Draw PAUSED text with shadow and white stroke
        ctx.shadowColor = '#000';
        ctx.shadowBlur = 20;
        ctx.strokeStyle = '#fff';
        ctx.lineWidth = 3;
        ctx.fillStyle = '#fff';
        ctx.font = 'bold 72px Arial';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText('PAUSED', canvas.width/2, canvas.height/2 - 100);
        ctx.strokeText('PAUSED', canvas.width/2, canvas.height/2 - 100);
        
        // Draw Resume button (play icon)
        const resumeBtnY = canvas.height/2;
        
        // Resume button shadow
        ctx.fillStyle = 'rgba(0, 0, 0, 0.4)';
        ctx.beginPath();
        ctx.roundRect(canvas.width/2 - 28, resumeBtnY - 28, 56, 56, 12);
        ctx.fill();
        
        // Resume button background
        const btnGradient = ctx.createLinearGradient(0, resumeBtnY - 30, 0, resumeBtnY + 30);
        btnGradient.addColorStop(0, '#4CAF50');
        btnGradient.addColorStop(1, '#45a049');
        ctx.fillStyle = btnGradient;
        ctx.beginPath();
        ctx.roundRect(canvas.width/2 - 30, resumeBtnY - 30, 60, 60, 12);
        ctx.fill();
        
        // Draw play icon
        ctx.fillStyle = '#fff';
        ctx.beginPath();
        ctx.moveTo(canvas.width/2 - 10, resumeBtnY - 15);
        ctx.lineTo(canvas.width/2 - 10, resumeBtnY + 15);
        ctx.lineTo(canvas.width/2 + 15, resumeBtnY);
        ctx.closePath();
        ctx.fill();
        
        // Draw Return to Main Menu button
        const menuBtnY = resumeBtnY + 100;
        const menuBtnWidth = 230;
        const menuBtnX = canvas.width/2 - menuBtnWidth/2;
        
        // Menu button shadow
        ctx.fillStyle = 'rgba(0, 0, 0, 0.4)';
        ctx.beginPath();
        ctx.roundRect(menuBtnX + 2, menuBtnY - 23, menuBtnWidth - 4, 50 - 4, 12);
        ctx.fill();
        
        // Menu button background with gradient
        const menuBtnGradient = ctx.createLinearGradient(0, menuBtnY - 25, 0, menuBtnY + 25);
        menuBtnGradient.addColorStop(0, '#e74c3c');
        menuBtnGradient.addColorStop(1, '#c0392b');
        ctx.fillStyle = menuBtnGradient;
        ctx.beginPath();
        ctx.roundRect(menuBtnX, menuBtnY - 25, menuBtnWidth, 50, 12);
        ctx.fill();
        
        // Menu button border
        ctx.strokeStyle = 'rgba(255, 255, 255, 0.2)';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.roundRect(menuBtnX, menuBtnY - 25, menuBtnWidth, 50, 12);
        ctx.stroke();
        
        // Menu button text with shadow
        ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
        ctx.shadowBlur = 8;
        ctx.fillStyle = '#fff';
        ctx.font = 'bold 22px Arial';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText('Return to Main Menu', canvas.width/2, menuBtnY);
        
        // Reset shadow
        ctx.shadowBlur = 0;
    }
    
    function goToMainMenu() {
        // Clean up game resources
        stopTimer();
        if (animationFrame) {
            cancelAnimationFrame(animationFrame);
        }
        // Navigate to main menu
        goto('/');
    }
    
    function handleClick(event: MouseEvent) {
        const rect = canvas.getBoundingClientRect();
        const x = event.clientX - rect.left;
        const y = event.clientY - rect.top;

        if (gameOver) {
            // Main menu button hitbox
            const menuBtnY = canvas.height/2 + 120;
            if (x >= canvas.width/2 - 80 && x <= canvas.width/2 + 80 &&
                y >= menuBtnY - 20 && y <= menuBtnY + 20) {
                goToMainMenu();
                return;
            }
            
            // Restart game hitbox
            if (y < menuBtnY - 30) {
                startGame();
            }
        } else if (isPaused) {
            const resumeBtnY = canvas.height/2;
            // Resume button hitbox (icon)
            if (x >= canvas.width/2 - 30 && x <= canvas.width/2 + 30 &&
                y >= resumeBtnY - 30 && y <= resumeBtnY + 30) {
                togglePause();
                return;
            }
            
            // Return to Main Menu button hitbox
            const menuBtnY = resumeBtnY + 100;
            const menuBtnWidth = 220;
            if (x >= canvas.width/2 - menuBtnWidth/2 && x <= canvas.width/2 + menuBtnWidth/2 &&
                y >= menuBtnY - 25 && y <= menuBtnY + 25) {
                goToMainMenu();
                return;
            }
        }
        
        // Pause button hitbox
        if (x >= 10 && x <= 50 && y >= 10 && y <= 50) {
            if (!isResumingCountdown && !gameOver) {
                togglePause();
            }
            return;
        }
        
        // Jump only if not paused and not in countdown
        if (!isPaused && !isResumingCountdown && !gameOver) {
            jump();
        }
    }
    
    function handleMouseMove(event: MouseEvent) {
        if (!gameOver) return;
        
        const rect = canvas.getBoundingClientRect();
        const x = event.clientX - rect.left;
        const y = event.clientY - rect.top;
        
        const menuBtnY = canvas.height/2 + 80;
        mainMenuBtnHovered = (
            x >= canvas.width/2 - 80 && 
            x <= canvas.width/2 + 80 &&
            y >= menuBtnY - 20 && 
            y <= menuBtnY + 20
        );
    }
    
    function gameLoop() {
        if (!ctx || gameOver) return;
        if (isCountingDown || isResumingCountdown) return;
        
        // If paused, only draw the pause overlay and return
        if (isPaused) {
            drawPauseOverlay();
            return;
        }
        
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        // Fill background
        ctx.fillStyle = '#87CEEB';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        
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
        
        // Draw score in the top right corner
        ctx.fillStyle = '#000';
        ctx.font = 'bold 28px Poppins';
        ctx.textAlign = 'right';
        ctx.fillText(`Score: ${score}`, canvas.width - 20, 37);
        
        // Draw pause button
        drawPauseButton();
        
        if (gameOver) {
            stopTimer();
            // Gradient overlay
            const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
            gradient.addColorStop(0, 'rgba(0, 0, 0, 0.7)');
            gradient.addColorStop(0.5, 'rgba(0, 0, 0, 0.6)');
            gradient.addColorStop(1, 'rgba(0, 0, 0, 0.7)');
            ctx.fillStyle = gradient;
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            
            // Game Over text with shadow and glow
            ctx.shadowColor = '#e74c3c';
            ctx.shadowBlur = 20;
            ctx.fillStyle = '#ff6b6b';
            ctx.font = 'bold 72px Arial';
            ctx.textAlign = 'center';
            ctx.fillText('Game Over!', canvas.width/2, canvas.height/2 - 40);
            
            // Reset shadow
            ctx.shadowBlur = 0;
            
            // Click to restart text with glow animation
            const glowIntensity = (Math.sin(Date.now() * 0.003) + 1) / 2;
            ctx.fillStyle = `rgba(255, 255, 255, ${0.7 + glowIntensity * 0.3})`;
            ctx.font = 'bold 28px Arial';
            ctx.fillText('Click to restart', canvas.width/2, canvas.height/2 + 40);
            
            // Main Menu button with enhanced styling
            const menuBtnY = canvas.height/2 + 120;
            
            // Button outer glow
            if (mainMenuBtnHovered) {
                ctx.shadowColor = '#e74c3c';
                ctx.shadowBlur = 15;
            }
            
            // Button shadow
            ctx.fillStyle = 'rgba(0, 0, 0, 0.4)';
            ctx.beginPath();
            ctx.roundRect(canvas.width/2 - 78, menuBtnY - 18, 156, 46, 10);
            ctx.fill();
            
            // Button gradient
            const btnGradient = ctx.createLinearGradient(0, menuBtnY - 20, 0, menuBtnY + 20);
            if (mainMenuBtnHovered) {
                btnGradient.addColorStop(0, '#ff6b6b');
                btnGradient.addColorStop(0.5, '#e74c3c');
                btnGradient.addColorStop(1, '#c0392b');
            } else {
                btnGradient.addColorStop(0, '#e74c3c');
                btnGradient.addColorStop(0.5, '#d63031');
                btnGradient.addColorStop(1, '#c0392b');
            }
            
            // Button background with rounded corners
            ctx.fillStyle = btnGradient;
            ctx.beginPath();
            ctx.roundRect(canvas.width/2 - 80, menuBtnY - 20, 160, 40, 10);
            ctx.fill();
            
            // Button border
            ctx.strokeStyle = mainMenuBtnHovered ? '#fff' : 'rgba(255, 255, 255, 0.5)';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.roundRect(canvas.width/2 - 80, menuBtnY - 20, 160, 40, 10);
            ctx.stroke();
            
            // Button text with enhanced shadow
            ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
            ctx.shadowBlur = mainMenuBtnHovered ? 8 : 4;
            ctx.shadowOffsetY = 2;
            ctx.fillStyle = '#fff';
            ctx.font = 'bold 22px Arial';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText('Main Menu', canvas.width/2, menuBtnY);
            
            // Reset shadow effects
            ctx.shadowBlur = 0;
            ctx.shadowOffsetY = 0;
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
            isPaused = false;
            startTimer();
            startCountdown();
        }
    }
    
    function handleKeydown(e: KeyboardEvent) {
        if (e.code === 'Space') {
            e.preventDefault();
            if (gameOver) {
                startGame();
            } else if (!isResumingCountdown) {
                if (isPaused) {
                    togglePause();
                } else {
                    jump();
                }
            }
        } else if (e.code === 'Escape') {
            e.preventDefault();
            if (!gameOver && !isResumingCountdown) {
                togglePause();
            }
        }
    }
    
    onMount(() => {
        ctx = canvas.getContext('2d')!;
        adjustForMobile();
        startCountdown();
        
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
        stopTimer();
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
        on:click={handleClick}
        on:mousemove={handleMouseMove}
        tabindex="0"
    />
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

    @media (max-width: 768px) {
        .game-container {
            padding: 10px;
        }
    }
</style>
