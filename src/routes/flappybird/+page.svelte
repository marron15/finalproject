<script lang="ts">
    import { onMount, onDestroy } from 'svelte';
    import { goto } from '$app/navigation';
    
    let canvas: HTMLCanvasElement;
    let ctx: CanvasRenderingContext2D;
    let isMobile = /Mobi|Android/i.test(navigator.userAgent); // Device detection
    let canvasWidth = 800;
    let canvasHeight = 600;
    let bird = {
        x: 50,
        y: 150,
        velocity: 0,
        gravity: 0.15,
        jumpForce: -4.0,
        size: 25,
        maxVelocity: 4.5,
        rotation: 0,
        wingOffset: 0,
        wingSpeed: 0.2,
        hitboxSize: 15,
        airResistance: 0.99
    };
    
    let pipes: Array<{x: number, gap: number}> = [];
    let gameOver = false;
    let score = 0;
    let highScore = 0;
    let showShareMenu = false;
    let animationFrame: number;
    let lastJumpTime = 0;
    let countdown = 3;
    let isCountingDown = true;
    let previousCountdown = 0;
    let isPaused = false;
    let gameTimer = 0;
    let gameTimerInterval: number | null = null; // Initialize with null
    let isResumingCountdown = false;
    let resumeCountdown = 0;
    let mainMenuBtnHovered = false;
    
    const PC_SPEED = 2.5; // Speed for PC
    const MOBILE_SPEED = 1.5; // Speed for mobile
    const PIPE_WIDTH = 50;
    const PIPE_GAP = 180;
    const JUMP_COOLDOWN = 150;
    const COLLISION_BUFFER = 5;
    const MIN_VELOCITY = 0.5;
    
    let clouds: Array<{x: number, y: number, width: number, speed: number}> = [];
    let buildings: Array<{x: number, width: number, height: number}> = [];
    let isNightMode = true;
    
    // Add new state variables for stars and building windows
    let stars: Array<{x: number, y: number, size: number}> = [];
    let buildingWindows: Array<Array<{x: number, y: number, isLit: boolean}>> = [];
    
    function adjustForMobile() {
        if (isMobile) {
            // Mobile settings (full screen portrait)
            canvasWidth = window.innerWidth; // Full width
            canvasHeight = window.innerHeight * 0.9; // 90% of viewport height to account for browser UI
        } else {
            // PC settings (landscape) - increased size for better visibility
            canvasWidth = Math.min(window.innerWidth * 0.6, 1980); // 60% of window width
            canvasHeight = Math.min(window.innerHeight * 0.7, 1080); // 70% of window height
        }
        
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
        chestGradient.addColorStop(0, '#fff9c4');  // Lighter center
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
            // Create gradient for pipes
            const pipeGradient = ctx.createLinearGradient(pipe.x, 0, pipe.x + PIPE_WIDTH, 0);
            pipeGradient.addColorStop(0, '#2ecc71');  // Base green
            pipeGradient.addColorStop(0.5, '#27ae60'); // Darker green for depth
            pipeGradient.addColorStop(1, '#2ecc71');   // Back to base green
            
            // Draw top pipe with enhanced styling
            ctx.save();
            // Pipe shadow
            ctx.shadowColor = 'rgba(0, 0, 0, 0.3)';
            ctx.shadowBlur = 5;
            ctx.shadowOffsetX = 2;
            
            // Main pipe body
            ctx.fillStyle = pipeGradient;
            ctx.fillRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
            
            // Pipe border
            ctx.strokeStyle = '#27ae60';
            ctx.lineWidth = 2;
            ctx.strokeRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
            
            // Pipe cap for top pipe
            const capHeight = 20;
            const capWidth = PIPE_WIDTH + 20;
            const capGradient = ctx.createLinearGradient(pipe.x - 10, 0, pipe.x + capWidth, 0);
            capGradient.addColorStop(0, '#27ae60');
            capGradient.addColorStop(0.5, '#2ecc71');
            capGradient.addColorStop(1, '#27ae60');
            
            ctx.fillStyle = capGradient;
            ctx.fillRect(pipe.x - 10, pipe.gap - PIPE_GAP/2 - capHeight, capWidth, capHeight);
            ctx.strokeRect(pipe.x - 10, pipe.gap - PIPE_GAP/2 - capHeight, capWidth, capHeight);
            
            // Bottom pipe with same styling
            ctx.fillStyle = pipeGradient;
            ctx.fillRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
            ctx.strokeRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
            
            // Pipe cap for bottom pipe
            ctx.fillStyle = capGradient;
            ctx.fillRect(pipe.x - 10, pipe.gap + PIPE_GAP/2, capWidth, capHeight);
            ctx.strokeRect(pipe.x - 10, pipe.gap + PIPE_GAP/2, capWidth, capHeight);
            
            ctx.restore();
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
    
    function startCountdown() {
        isCountingDown = true;
        let countdown = 3;
        
        function drawCountdownFrame() {
            if (!ctx) return;
            
            // Clear and draw the current game state
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Draw background with night mode
            drawBackground();
            
            // Draw existing pipes
            for (const pipe of pipes) {
                // Create gradient for pipes
                const pipeGradient = ctx.createLinearGradient(pipe.x, 0, pipe.x + PIPE_WIDTH, 0);
                pipeGradient.addColorStop(0, '#1a472a');  // Darker green for night
                pipeGradient.addColorStop(0.5, '#143d23'); // Even darker for depth
                pipeGradient.addColorStop(1, '#1a472a');   // Back to dark green
                
                ctx.fillStyle = pipeGradient;
                // Draw pipes
                ctx.fillRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2); // Top pipe
                ctx.fillRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height - (pipe.gap + PIPE_GAP/2)); // Bottom pipe
            }
            
            // Draw bird
            drawBird();
            
            // Draw countdown number with shadow
            ctx.save();
            ctx.font = 'bold 72px Arial';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            
            // Add shadow
            ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
            ctx.shadowBlur = 10;
            ctx.shadowOffsetX = 4;
            ctx.shadowOffsetY = 4;
            
            // Draw text
            ctx.fillStyle = 'white';
            ctx.fillText(countdown.toString(), canvas.width / 2, canvas.height / 2);
            
            // Draw instruction text
            ctx.font = 'bold 24px Arial';
            ctx.fillText('Click or press Space to jump', canvas.width / 2, canvas.height / 2 + 50);
            
            ctx.restore();
        }

        function updateCountdown() {
            if (countdown > 0) {
                drawCountdownFrame();
                countdown--;
                setTimeout(updateCountdown, 1000);
            } else {
                isCountingDown = false;
                gameLoop();
            }
        }

        updateCountdown();
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
                // Create gradient for pipes
                const pipeGradient = ctx.createLinearGradient(pipe.x, 0, pipe.x + PIPE_WIDTH, 0);
                pipeGradient.addColorStop(0, '#2ecc71');  // Base green
                pipeGradient.addColorStop(0.5, '#27ae60'); // Darker green for depth
                pipeGradient.addColorStop(1, '#2ecc71');   // Back to base green
                
                // Draw top pipe with enhanced styling
                ctx.save();
                // Pipe shadow
                ctx.shadowColor = 'rgba(0, 0, 0, 0.3)';
                ctx.shadowBlur = 5;
                ctx.shadowOffsetX = 2;
                
                // Main pipe body
                ctx.fillStyle = pipeGradient;
                ctx.fillRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
                
                // Pipe border
                ctx.strokeStyle = '#27ae60';
                ctx.lineWidth = 2;
                ctx.strokeRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
                
                // Pipe cap for top pipe
                const capHeight = 20;
                const capWidth = PIPE_WIDTH + 20;
                const capGradient = ctx.createLinearGradient(pipe.x - 10, 0, pipe.x + capWidth, 0);
                capGradient.addColorStop(0, '#27ae60');
                capGradient.addColorStop(0.5, '#2ecc71');
                capGradient.addColorStop(1, '#27ae60');
                
                ctx.fillStyle = capGradient;
                ctx.fillRect(pipe.x - 10, pipe.gap - PIPE_GAP/2 - capHeight, capWidth, capHeight);
                ctx.strokeRect(pipe.x - 10, pipe.gap - PIPE_GAP/2 - capHeight, capWidth, capHeight);
                
                // Bottom pipe with same styling
                ctx.fillStyle = pipeGradient;
                ctx.fillRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
                ctx.strokeRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
                
                // Pipe cap for bottom pipe
                ctx.fillStyle = capGradient;
                ctx.fillRect(pipe.x - 10, pipe.gap + PIPE_GAP/2, capWidth, capHeight);
                ctx.strokeRect(pipe.x - 10, pipe.gap + PIPE_GAP/2, capWidth, capHeight);
                
                ctx.restore();
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
                // Resume the game
                isPaused = false;
                startCountdown(); // Start countdown before resuming
            } else {
                // Pause the game
                isPaused = true;
                stopTimer();
                if (animationFrame) {
                    cancelAnimationFrame(animationFrame);
                }
                drawPauseOverlay(); // Show the pause overlay
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
        if (gameTimerInterval) {
            clearInterval(gameTimerInterval);
            gameTimerInterval = null;
        }
        if (animationFrame) {
            cancelAnimationFrame(animationFrame);
        }
        // Navigate to main menu using relative path
        goto('../');
    }
    
    async function handleShare(platform: string) {
        const scoreImage = generateScoreImage();
        const text = `I scored ${score} points in Flappy Bird! Can you beat my score?`;
        const url = window.location.href;
        
        // Try to use the Web Share API first if available and on mobile
        if (platform === 'native' && navigator.share && /Android|iPhone|iPad|iPod/i.test(navigator.userAgent)) {
            try {
                const response = await fetch(scoreImage);
                const blob = await response.blob();
                const file = new File([blob], 'flappy-score.png', { type: 'image/png' });
                
                await navigator.share({
                    title: 'My Flappy Bird Score',
                    text: text,
                    url: url,
                    files: [file]
                });
                showShareMenu = false;
                return;
            } catch (err) {
                console.error('Error sharing:', err);
            }
        }

        // Handle Facebook sharing
        if (platform === 'facebook') {
            // First, trigger the download
            const link = document.createElement('a');
            link.download = 'flappy-score.png';
            link.href = scoreImage;
            link.click();
            
            // Then open Facebook's create post page directly
            window.open('https://www.facebook.com', '_blank', 
                'width=800,height=600,left=' + 
                ((window.innerWidth - 800) / 2) + ',top=' + 
                ((window.innerHeight - 600) / 2));
            
            showShareMenu = false;
            return;
        }

        // Handle other platforms
        let shareUrl = '';
        switch(platform) {
            case 'twitter':
                shareUrl = `https://twitter.com/intent/tweet?text=${encodeURIComponent(text)}&url=${encodeURIComponent(url)}`;
                break;
            case 'download':
                const link = document.createElement('a');
                link.download = 'flappy-score.png';
                link.href = scoreImage;
                link.click();
                showShareMenu = false;
                return;
        }
        
        if (shareUrl) {
            window.open(shareUrl, '_blank', 'width=600,height=400');
        }
        showShareMenu = false;
    }
    
    function generateScoreImage() {
        // Create a new canvas for the score image
        const scoreCanvas = document.createElement('canvas');
        const scoreCtx = scoreCanvas.getContext('2d');
        
        // Set canvas size
        scoreCanvas.width = 600;
        scoreCanvas.height = 400;
        
        if (!scoreCtx) return '';

        // Draw background
        const gradient = scoreCtx.createLinearGradient(0, 0, 0, scoreCanvas.height);
        gradient.addColorStop(0, '#1a1621'); // Dark blue night sky
        gradient.addColorStop(0.7, '#1a2c3d'); // Lighter near horizon
        gradient.addColorStop(1, '#2c3e50'); // Even lighter at horizon
        scoreCtx.fillStyle = gradient;
        scoreCtx.fillRect(0, 0, scoreCanvas.width, scoreCanvas.height);

        // Draw stars
        for (let i = 0; i < 50; i++) {
            scoreCtx.fillStyle = 'rgba(255, 255, 255, 0.5)';
            scoreCtx.beginPath();
            const x = Math.random() * scoreCanvas.width;
            const y = Math.random() * scoreCanvas.height;
            scoreCtx.arc(x, y, 1, 0, Math.PI * 2);
            scoreCtx.fill();
        }

        // Draw game title
        scoreCtx.font = 'bold 48px Arial';
        scoreCtx.fillStyle = '#ffffff';
        scoreCtx.textAlign = 'center';
        scoreCtx.fillText('Flappy Bird', scoreCanvas.width / 2, 100);

        // Draw score
        scoreCtx.font = 'bold 72px Arial';
        scoreCtx.fillStyle = '#4ade80';
        scoreCtx.fillText(score.toString(), scoreCanvas.width / 2, 200);

        // Draw "SCORE" text
        scoreCtx.font = '32px Arial';
        scoreCtx.fillStyle = '#ffffff';
        scoreCtx.fillText('SCORE', scoreCanvas.width / 2, 250);

        // Draw date
        const date = new Date().toLocaleDateString();
        scoreCtx.font = '24px Arial';
        scoreCtx.fillStyle = '#9ca3af';
        scoreCtx.fillText(date, scoreCanvas.width / 2, 350);

        return scoreCanvas.toDataURL('image/png');
    }
    
    function handleClick(event: MouseEvent | TouchEvent) {
        if (!ctx) return;
        
        // Get coordinates whether it's a mouse or touch event
        let x: number, y: number;
        if (event instanceof TouchEvent) {
            const touch = event.touches[0] || event.changedTouches[0];
            const rect = canvas.getBoundingClientRect();
            x = touch.clientX - rect.left;
            y = touch.clientY - rect.top;
        } else {
            const rect = canvas.getBoundingClientRect();
            x = event.clientX - rect.left;
            y = event.clientY - rect.top;
        }

        // Prevent default behavior to avoid any unwanted effects
        event.preventDefault();
        
        // Handle the click/touch based on game state
        if (gameOver) {
            // Check if click/touch is within restart button area
            if (x >= canvas.width/2 - 60 && x <= canvas.width/2 + 60 &&
                y >= canvas.height/2 + 50 && y <= canvas.height/2 + 90) {
                startGame();
            }
        } else if (!isCountingDown && !isResumingCountdown) {
            if (!isPaused) {
                const currentTime = Date.now();
                if (currentTime - lastJumpTime >= JUMP_COOLDOWN) {
                    bird.velocity = bird.jumpForce;
                    lastJumpTime = currentTime;
                }
            }
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
    
    function initializeBackground() {
        // Initialize clouds
        clouds = [];
        for (let i = 0; i < 5; i++) {
            clouds.push({
                x: Math.random() * canvas.width,
                y: Math.random() * (canvas.height * 0.4),
                width: Math.random() * 80 + 60,
                speed: Math.random() * 0.2 + 0.1
            });
        }

        // Initialize buildings
        buildings = [];
        let x = 0;
        while (x < canvas.width) {
            const width = Math.random() * 60 + 40;
            const height = Math.random() * (canvas.height * 0.3) + (canvas.height * 0.1);
            buildings.push({ x, width, height });
            x += width + 5;
        }

        // Initialize stars
        stars = [];
        for (let i = 0; i < 100; i++) {
            stars.push({
                x: Math.random() * canvas.width,
                y: Math.random() * (canvas.height * 0.6),
                size: Math.random() * 1.5 + 0.5
            });
        }

        // Initialize building windows
        buildingWindows = [];
        for (let building of buildings) {
            const windowsForBuilding: Array<{x: number, y: number, isLit: boolean}> = [];
            for (let y = canvas.height - building.height + 10; y < canvas.height - 10; y += 15) {
                for (let x = building.x + 5; x < building.x + building.width - 5; x += 10) {
                    windowsForBuilding.push({
                        x, 
                        y,
                        isLit: Math.random() > 0.4 // 60% chance of being lit
                    });
                }
            }
            buildingWindows.push(windowsForBuilding);
        }
    }
    
    function drawBackground() {
        if (!ctx) return;

        // Draw sky gradient - night mode colors
        const skyGradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
        if (isNightMode) {
            skyGradient.addColorStop(0, '#0a1621'); // Dark blue night sky
            skyGradient.addColorStop(0.7, '#1a2c3d'); // Lighter near horizon
            skyGradient.addColorStop(1, '#2c3e50'); // Even lighter at horizon
        } else {
            skyGradient.addColorStop(0, '#87CEEB');
            skyGradient.addColorStop(1, '#ADD8E6');
        }
        ctx.fillStyle = skyGradient;
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        // Draw stars in night mode
        if (isNightMode) {
            ctx.fillStyle = '#fff';
            for (let star of stars) {
                ctx.beginPath();
                ctx.arc(star.x, star.y, star.size, 0, Math.PI * 2);
                ctx.fill();
            }
        }

        // Draw clouds with adjusted opacity for night mode
        ctx.fillStyle = isNightMode ? 'rgba(255, 255, 255, 0.1)' : 'rgba(255, 255, 255, 0.8)';
        for (let cloud of clouds) {
            // Move clouds
            cloud.x -= cloud.speed;
            if (cloud.x + cloud.width < 0) {
                cloud.x = canvas.width;
                cloud.y = Math.random() * (canvas.height * 0.4);
            }

            // Draw cloud shape
            ctx.beginPath();
            ctx.arc(cloud.x, cloud.y, cloud.width * 0.3, 0, Math.PI * 2);
            ctx.arc(cloud.x + cloud.width * 0.3, cloud.y - 10, cloud.width * 0.25, 0, Math.PI * 2);
            ctx.arc(cloud.x + cloud.width * 0.5, cloud.y, cloud.width * 0.3, 0, Math.PI * 2);
            ctx.fill();
        }

        // Draw moon in night mode
        if (isNightMode) {
            ctx.save();
            ctx.fillStyle = '#fff5d6';
            ctx.beginPath();
            ctx.arc(canvas.width - 80, 80, 30, 0, Math.PI * 2);
            ctx.fill();
            
            // Moon shadow
            ctx.fillStyle = '#0a1621';
            ctx.beginPath();
            ctx.arc(canvas.width - 70, 75, 28, 0, Math.PI * 2);
            ctx.fill();
            ctx.restore();
        }

        // Draw city skyline
        for (let i = 0; i < buildings.length; i++) {
            const building = buildings[i];
            
            // Create gradient for each building
            const buildingGradient = ctx.createLinearGradient(
                building.x, 
                canvas.height - building.height, 
                building.x, 
                canvas.height
            );

            if (isNightMode) {
                // Night mode building colors - varying shades for depth
                if (i % 3 === 0) {
                    buildingGradient.addColorStop(0, '#1a2634');
                    buildingGradient.addColorStop(1, '#0f1c2a');
                } else if (i % 3 === 1) {
                    buildingGradient.addColorStop(0, '#233343');
                    buildingGradient.addColorStop(1, '#182736');
                } else {
                    buildingGradient.addColorStop(0, '#2c3e50');
                    buildingGradient.addColorStop(1, '#233343');
                }
            } else {
                // Day mode building colors
                if (i % 3 === 0) {
                    buildingGradient.addColorStop(0, '#2c3e50');
                    buildingGradient.addColorStop(1, '#34495e');
                } else if (i % 3 === 1) {
                    buildingGradient.addColorStop(0, '#34495e');
                    buildingGradient.addColorStop(1, '#2c3e50');
                } else {
                    buildingGradient.addColorStop(0, '#395670');
                    buildingGradient.addColorStop(1, '#2c3e50');
                }
            }

            // Draw building
            ctx.fillStyle = buildingGradient;
            ctx.fillRect(building.x, canvas.height - building.height, building.width, building.height);

            // Draw windows using pre-calculated positions
            if (buildingWindows[i]) {
                const windowColor = isNightMode ? 'rgba(255, 247, 130, 0.8)' : 'rgba(255, 255, 255, 0.2)';
                const dimWindowColor = isNightMode ? 'rgba(255, 247, 130, 0.1)' : 'rgba(255, 255, 255, 0.1)';
                
                if (isNightMode) {
                    ctx.shadowColor = 'rgba(255, 247, 130, 0.5)';
                    ctx.shadowBlur = 5;
                }

                for (let window of buildingWindows[i]) {
                    ctx.fillStyle = window.isLit ? windowColor : dimWindowColor;
                    ctx.fillRect(window.x, window.y, 5, 10);
                }
                ctx.shadowBlur = 0;
            }
        }
    }
    
    function gameLoop() {
        if (!ctx || gameOver) return;
        if (isCountingDown || isResumingCountdown) return;
        
        if (isPaused) {
            drawPauseOverlay(); // Show pause overlay
            // Do not call requestAnimationFrame here to avoid freezing
            return;
        }
        
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        drawBackground();
        
        updateBirdPhysics();
        drawBird();
        
        const currentSpeed = isMobile ? MOBILE_SPEED : PC_SPEED;
        
        for (let i = pipes.length - 1; i >= 0; i--) {
            const pipe = pipes[i];
            pipe.x -= currentSpeed;
            
            // Create gradient for pipes
            const pipeGradient = ctx.createLinearGradient(pipe.x, 0, pipe.x + PIPE_WIDTH, 0);
            if (isNightMode) {
                pipeGradient.addColorStop(0, '#1a472a');  // Darker green for night
                pipeGradient.addColorStop(0.5, '#143d23'); // Even darker for depth
                pipeGradient.addColorStop(1, '#1a472a');   // Back to dark green
            } else {
                pipeGradient.addColorStop(0, '#2ecc71');  // Original green
                pipeGradient.addColorStop(0.5, '#27ae60'); // Original darker green
                pipeGradient.addColorStop(1, '#2ecc71');   // Back to original
            }
            
            // Draw top pipe with enhanced styling
            ctx.save();
            // Pipe shadow
            ctx.shadowColor = 'rgba(0, 0, 0, 0.3)';
            ctx.shadowBlur = 5;
            ctx.shadowOffsetX = 2;
            
            // Main pipe body
            ctx.fillStyle = pipeGradient;
            ctx.fillRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
            
            // Pipe border
            ctx.strokeStyle = '#27ae60';
            ctx.lineWidth = 2;
            ctx.strokeRect(pipe.x, 0, PIPE_WIDTH, pipe.gap - PIPE_GAP/2);
            
            // Pipe cap for top pipe
            const capHeight = 20;
            const capWidth = PIPE_WIDTH + 20;
            const capGradient = ctx.createLinearGradient(pipe.x - 10, 0, pipe.x + capWidth, 0);
            capGradient.addColorStop(0, '#27ae60');
            capGradient.addColorStop(0.5, '#2ecc71');
            capGradient.addColorStop(1, '#27ae60');
            
            ctx.fillStyle = capGradient;
            ctx.fillRect(pipe.x - 10, pipe.gap - PIPE_GAP/2 - capHeight, capWidth, capHeight);
            ctx.strokeRect(pipe.x - 10, pipe.gap - PIPE_GAP/2 - capHeight, capWidth, capHeight);
            
            // Bottom pipe with same styling
            ctx.fillStyle = pipeGradient;
            ctx.fillRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
            ctx.strokeRect(pipe.x, pipe.gap + PIPE_GAP/2, PIPE_WIDTH, canvas.height);
            
            // Pipe cap for bottom pipe
            ctx.fillStyle = capGradient;
            ctx.fillRect(pipe.x - 10, pipe.gap + PIPE_GAP/2, capWidth, capHeight);
            ctx.strokeRect(pipe.x - 10, pipe.gap + PIPE_GAP/2, capWidth, capHeight);
            
            ctx.restore();
            
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
        if (!gameOver) {
            // Draw score in the top right corner with better visibility
            ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
            ctx.shadowBlur = 5;
            ctx.fillStyle = '#fff';  // Changed from '#000' to white
            ctx.font = 'bold 32px Arial';  // Slightly larger font
            ctx.textAlign = 'right';
            ctx.fillText(`Score: ${score}`, canvas.width - 20, 40);
            ctx.shadowBlur = 0;
        }
        
        // Draw pause button
        drawPauseButton();
        
        if (gameOver) {
            stopTimer();
            // Game Over overlay background
            const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
            gradient.addColorStop(0, 'rgba(0, 0, 0, 0.7)');
            gradient.addColorStop(1, 'rgba(0, 0, 0, 0.9)');
            ctx.fillStyle = gradient;
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            
            // Reset any existing canvas transformations
            ctx.setTransform(1, 0, 0, 1, 0, 0);
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
            initializeBackground();
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
    
    function stopTimer() {
        if (gameTimerInterval) {
            clearInterval(gameTimerInterval);
            gameTimerInterval = null; // Reset the interval ID
        }
    }
    
    function startTimer() {
        gameTimer = 0; // Reset the timer
        gameTimerInterval = window.setInterval(() => {
            if (!isPaused && !gameOver && !isCountingDown) {
                gameTimer++;
            }
        }, 1000);
    }
    
    onMount(() => {
        ctx = canvas.getContext('2d')!;
        adjustForMobile();
        initializeBackground();
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
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css" />
</svelte:head>

<div class="game-container">
    <canvas
        bind:this={canvas}
        on:click={handleClick}
        on:mousemove={handleMouseMove}
        tabindex="0"
    />
    {#if gameOver}
        <div class="game-over-overlay">
            <h1>Game Over!</h1>
            <p>Score: {score}</p>
            <div class="buttons-container">
                <button class="main-menu-btn" on:click={() => goToMainMenu()}>Main Menu</button>
                <button class="share-btn" on:click={() => showShareMenu = true}>
                    <img src="/share-icon.svg" alt="Share" />
                </button>
            </div>
            {#if showShareMenu}
                <div class="share-menu">
                    <div class="score-preview">
                        <img src={generateScoreImage()} alt="Score preview" />
                    </div>
                    <button class="social-btn facebook" on:click={() => handleShare('facebook')}>
                        Share on Facebook
                    </button>
                    <button class="social-btn twitter" on:click={() => handleShare('twitter')}>
                        Share on Twitter
                    </button>
                    <button class="social-btn download" on:click={() => handleShare('download')}>
                        Download Image
                    </button>
                    <button class="close-btn" on:click={() => showShareMenu = false}>Close</button>
                </div>
            {/if}
        </div>
    {/if}
</div>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        overflow: hidden;
    }

    .game-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        background: linear-gradient(to bottom, #09203f, #537895);
        min-height: 100vh;
        width: 100%;
        max-width: 100%;
        position: relative;
        margin: 0;
        padding: 0;
        overflow: hidden;
    }

    canvas {
        width: 100%;
        display: block;
        margin: 0 auto;
        touch-action: none;
        -webkit-touch-callout: none;
        -webkit-user-select: none;
        user-select: none;
        -webkit-tap-highlight-color: transparent;
    }

    /* Mobile-specific styles */
    @media (max-width: 768px) {
        .game-container {
            padding: 0;
            height: 100vh;
        }
        
        canvas {
            width: 100vw;
            height: 90vh;
            max-width: none;
            max-height: none;
            margin: 0;
            border-radius: 0;
        }
    }

    /* Desktop-specific styles */
    @media (min-width: 769px) {
        canvas {
            aspect-ratio: 16/9;
            min-width: 800px;
            width: 60%;
            max-width: 1980px;
            height: auto;
        }
    }
    
    .game-over-overlay {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        text-align: center;
        display: flex;
        flex-direction: column;
        gap: 20px;
        align-items: center;
        z-index: 10;
    }

    .game-over-overlay h1 {
        font-size: 48px;
        color: #ff4757;
        text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
        margin: 0;
    }

    .game-over-overlay p {
        font-size: 24px;
        color: white;
        text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
        margin: 0;
    }

    .buttons-container {
        display: flex;
        gap: 15px;
        align-items: center;
        margin-top: 20px;
    }

    .main-menu-btn {
        background: #e74c3c;
        color: white;
        border: none;
        padding: 12px 24px;
        border-radius: 8px;
        font-size: 18px;
        cursor: pointer;
        transition: background 0.3s, transform 0.2s;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        position: relative;
        z-index: 1000;
        touch-action: manipulation;
        -webkit-tap-highlight-color: transparent;
        user-select: none;
    }

    .main-menu-btn:hover {
        background: #c0392b;
        transform: translateY(-2px);
    }

    .share-btn {
        display: flex;
        align-items: center;
        justify-content: center;
        background: #4CAF50;
        border: none;
        width: 50px;
        height: 50px;
        border-radius: 8px;
        cursor: pointer;
        transition: all 0.3s ease;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        position: relative;
        z-index: 1000;
        touch-action: manipulation;
        -webkit-tap-highlight-color: transparent;
        user-select: none;
    }

    .share-btn:hover {
        background: #388E3C;
        transform: translateY(-2px);
    }

    .share-btn img {
        width: 24px;
        height: 24px;
    }

    .share-menu {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        background: rgba(0, 0, 0, 0.95);
        padding: 20px;
        border-radius: 10px;
        display: flex;
        flex-direction: column;
        gap: 10px;
        z-index: 1000;
        max-width: 90%;
        width: 400px;
        touch-action: manipulation;
    }

    .score-preview {
        width: 100%;
        margin-bottom: 15px;
        border-radius: 8px;
        overflow: hidden;
    }

    .score-preview img {
        width: 100%;
        height: auto;
        display: block;
    }

    .social-btn {
        padding: 12px 20px;
        border: none;
        border-radius: 5px;
        color: white;
        cursor: pointer;
        font-size: 16px;
        transition: all 0.2s ease;
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 10px;
        position: relative;
        z-index: 1000;
        touch-action: manipulation;
        -webkit-tap-highlight-color: transparent;
        user-select: none;
    }


    .facebook {
        background: #1877F2;
    }

    .twitter {
        background: #1DA1F2;
    }

    .download {
        background: #059669;
    }
    
    .social-btn:hover, .close-btn:hover {
        opacity: 0.9;
        transform: translateY(-2px);
    }
</style>
