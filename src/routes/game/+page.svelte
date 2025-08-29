<script lang="ts">
    import { onMount } from 'svelte';

    let container: HTMLDivElement;
    let square: HTMLDivElement;
    let rotationAngle = 0;
    
    // Глобальные переменные
    const flightTime = 1; // время полета в секундах
    let spawnDistance = 1.5; // будет рассчитываться динамически

    // Массив для хранения активных элементов
    let activeElements: Array<{
        id: number,
        element: HTMLDivElement,
        creationTime: number
    }> = [];
    let elementId = 0;

    // Размеры игровой области (в пикселях)
    let gameAreaWidth = 0;
    let gameAreaHeight = 0;
    let circleRadius = 0;

    // Функция для расчета размеров игровой области
    function updateGameAreaSize(): void {
        const circleElement = document.querySelector('.circle') as HTMLElement;
        if (circleElement) {
            const rect = circleElement.getBoundingClientRect();
            gameAreaWidth = rect.width;
            gameAreaHeight = rect.height;
            circleRadius = gameAreaWidth / 2; // Круг всегда квадратный
        }
    }

    // Функция для расчета spawnDistance
    function calculateSpawnDistance(): number {
        return 3; // 300% от радиуса (достаточно далеко за пределами круга)
    }

    // Универсальная функция для анимации элемента с использованием transform
    function animateElement(
        element: HTMLDivElement, 
        targetAngle: number, 
        onComplete?: () => void
    ): () => void {
        updateGameAreaSize();
        
        const circleElement = document.querySelector('.circle') as HTMLElement;
        const circleRect = circleElement.getBoundingClientRect();
        const centerX = circleRect.left + circleRect.width / 2;
        const centerY = circleRect.top + circleRect.height / 2;
        
        // Угол в радианах (Svelte использует 0° справа, по часовой стрелке)
        const targetRadians = ((360 - targetAngle) * Math.PI) / 180;
        
        // Целевая точка — край круга
        const targetX = centerX + circleRadius * Math.cos(targetRadians);
        const targetY = centerY + circleRadius * Math.sin(targetRadians);
        
        // Начальная точка — за пределами круга
        spawnDistance = calculateSpawnDistance();
        const spawnX = centerX + circleRadius * spawnDistance * Math.cos(targetRadians);
        const spawnY = centerY + circleRadius * spawnDistance * Math.sin(targetRadians);
        
        // Смещения относительно центра экрана
        const offsetX = spawnX - window.innerWidth / 2;
        const offsetY = spawnY - window.innerHeight / 2;
        
        // Устанавливаем начальную позицию через transform
        element.style.position = 'fixed';
        element.style.left = '50%';
        element.style.top = '50%';
        element.style.transform = `translate(${offsetX}px, ${offsetY}px) translate(-50%, -50%) rotate(${-targetAngle - 90}deg)`;
        element.style.transformOrigin = 'center';
        
        // Добавляем в DOM
        document.body.appendChild(element);
        
        // Запускаем анимацию через transform
        requestAnimationFrame(() => {
            requestAnimationFrame(() => {
                // Переход по transform (производительнее)
                element.style.transition = `transform ${flightTime}s linear`;
                
                const targetOffsetX = targetX - window.innerWidth / 2;
                const targetOffsetY = targetY - window.innerHeight / 2;
                
                element.style.transform = `translate(${targetOffsetX}px, ${targetOffsetY}px) translate(-50%, -50%) rotate(${-targetAngle - 90}deg)`;
            });
        });

        // Возвращаем функцию для отмены анимации
        return () => {
            if (element.parentNode) {
                element.parentNode.removeChild(element);
            }
        };
    }

    function rotateContainer(e: MouseEvent) {
        if (!container) return;

        const rect = container.getBoundingClientRect();
        const centerX = rect.left + rect.width / 2;
        const centerY = rect.top + rect.height / 2;

        const mouseX = e.clientX - centerX;
        const mouseY = e.clientY - centerY;

        rotationAngle = Math.atan2(mouseY, mouseX);
    }

    let angle: number = 0;
    function handleLeftClick(e: MouseEvent) {
        if (!square) return;
        
        const transitionTime = 0.5;
        square.style.transition = `left ${transitionTime}s linear`;
        square.style.left = square.style.left === '90%' ? '0%' : '90%';

        // if (angle >= 360) angle = 0;
        // spawnSquare(angle);
        // angle += 30;
    for (let i=0; i<80;i++){
    if (angle >= 360) angle = 0;
    ((currentAngle) => {
        setTimeout(()=>spawnSquare(currentAngle),i*30);
    })(angle);
    angle += 3;
}
    }

    // Функция для спавна квадратика
    function spawnSquare(targetAngle: number) {
        const squareElement = document.createElement('div');
        squareElement.className = 'flying-square';
        squareElement.style.width = `8vh`;
        squareElement.style.height = `5vh`;
        squareElement.style.backgroundColor = 'white';
        squareElement.style.borderRadius = '0.3vh';
        squareElement.style.boxShadow = '0 0 0.5vh rgba(255, 255, 255, 0.5)';
        squareElement.style.zIndex = '10';
        squareElement.style.pointerEvents = 'none'; // чтобы не мешал кликам
        
        const creationTime = performance.now();
        const id = elementId++;
        
        const cancelAnimation = animateElement(
            squareElement, 
            targetAngle,
            () => {
                console.log(`Элемент ${id} достиг цели`);
            }
        );
        
        activeElements.push({
            id,
            element: squareElement,
            creationTime
        });

        setTimeout(() => {
            const removalTime = performance.now();
            const lifetime = removalTime - creationTime;

            console.log(`Элемент ${id} удален в: ${removalTime.toFixed(2)}ms`);
            console.log(`Время жизни элемента ${id}: ${lifetime.toFixed(2)}ms`);

            cancelAnimation();
            activeElements = activeElements.filter(el => el.id !== id);
        }, flightTime * 1000);
    }

    function testSpawnCircle() {
        for (let i = 0; i < 360; i += 30) {
            setTimeout(() => spawnSquare(i), i * 10);
        }
    }

    onMount(() => {
        updateGameAreaSize();
        spawnDistance = calculateSpawnDistance();
        
        document.addEventListener('mousemove', rotateContainer);
        document.addEventListener('click', handleLeftClick);
        window.addEventListener('resize', updateGameAreaSize);

        (window as any).spawnSquare = spawnSquare;
        (window as any).testSpawn = testSpawnCircle;

        return () => {
            document.removeEventListener('mousemove', rotateContainer);
            document.removeEventListener('click', handleLeftClick);
            window.removeEventListener('resize', updateGameAreaSize);
            activeElements.forEach(el => {
                if (el.element.parentNode) {
                    el.element.parentNode.removeChild(el.element);
                }
            });
            delete (window as any).spawnSquare;
            delete (window as any).testSpawn;
        };
    });
</script>

<svelte:head>
    <title>Rhythm Game</title>
</svelte:head>

<div class="container-wrapper">
    <div class="circle"></div>
    <div class="container" bind:this={container} style="transform: rotate({rotationAngle}rad)">
        <div class="square" bind:this={square}></div>
    </div>
</div>

<style>
    :global(body) {
        margin: 0;
        height: 100vh;
        display: flex;
        justify-content: center;
        align-items: center;
        background: #1a1a1a;
        overflow: hidden;
        font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        color: white;
    }

    .container-wrapper {
        position: relative;
        display: flex;
        justify-content: center;
        align-items: center;
        width: 50vh;
        height: 50vh;
    }

    .container {
        width: 50vh;
        height: 8vh;
        position: relative;
        background-color: #333;
        border: 0.2vh solid #555;
        border-left: 0.2vh dotted #555;
        overflow: hidden;
        transform-origin: center center;
        border-radius: 1.7vh;
        box-shadow: 0 0.4vh 1.5vh rgba(0, 0, 0, 0.3);
    }

    .square {
        width: 10%;
        height: 100%;
        background-color: #ff4d4d;
        position: absolute;
        left: 0;
        top: 0;
        transition: left 0.05s linear;
        border-radius: 0.3vh;
    }

    .circle {
        width: 50vh;
        height: 50vh;
        border-radius: 50%;
        border: 0.2vh solid rgba(255, 255, 255, 0.2);
        position: absolute;
        display: flex;
        justify-content: center;
        align-items: center;
    }
</style>