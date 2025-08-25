<script lang="ts">
    import { onMount } from 'svelte';

    let container: HTMLDivElement;
    let square: HTMLDivElement;
    let rotationAngle = 0;
    
    // Глобальные переменные
    const flightTime = 1; // время полета в секундах
    const spawnDistance = 1.5; // расстояние спавна от края экрана (множитель)

    // Массив для хранения активных квадратиков
    let activeSquares: Array<{
        id: number,
        angle: number,
        element: HTMLDivElement,
        progress: number
    }> = [];
    let squareId = 0;

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

        // for (let i =0; i<10; i++){
        if (angle >= 360) angle = 0;
        // spawnSquare(90);
        spawnSquare(angle);
        // spawnSquare(180);
        // spawnSquare(270);
        angle += 10;
        // }
        // testSpawnCircle();
    }

    // Функция для спавна квадратика
    function spawnSquare(targetAngle: number) {
        const screenWidth = window.innerWidth;
        
        // Создаем элемент квадратика
        const squareElement = document.createElement('div');
        squareElement.className = 'flying-square';
        
        squareElement.style.width = `8vh`;
        squareElement.style.height = `5vh`;
        squareElement.style.backgroundColor = 'white';
        squareElement.style.position = 'fixed';
        squareElement.style.borderRadius = '3px';
        squareElement.style.boxShadow = '0 0 10px rgba(255, 255, 255, 0.5)';
        
        // Вычисляем конечную позицию (точка на круге)
        const circleRadius = 25 * window.innerHeight / 100; // 50vh / 2
        const centerX = window.innerWidth / 2;
        const centerY = window.innerHeight / 2;
        
        const targetRadians = ((360 - targetAngle) * Math.PI) / 180;
        const targetX = centerX + circleRadius * Math.cos(targetRadians);
        const targetY = centerY + circleRadius * Math.sin(targetRadians);
        
        // Вычисляем начальную позицию (вне экрана)
        const spawnX = centerX + screenWidth * spawnDistance * Math.cos(targetRadians);
        const spawnY = centerY + screenWidth * spawnDistance * Math.sin(targetRadians);
        
        // Устанавливаем начальную позицию
        squareElement.style.left = `${spawnX}px`;
        squareElement.style.top = `${spawnY}px`;
        squareElement.style.transform = 'translate(-50%, -50%)';
        
        // Добавляем на страницу
        document.body.appendChild(squareElement);
        
        // Ключевое исправление: используем requestAnimationFrame для гарантии
        // что элемент отрендерится перед началом анимации
        squareElement.style.transform = `translate(-50%, -50%) rotate(${-targetAngle - 90}deg)`;

        requestAnimationFrame(() => {
            requestAnimationFrame(() => {
                // Устанавливаем transition и конечные стили
                squareElement.style.transition = `all ${flightTime}s linear`;
                squareElement.style.left = `${targetX}px`;
                squareElement.style.top = `${targetY}px`;
            });
        });
        
        // Добавляем в массив активных квадратиков
        const id = squareId++;
        activeSquares.push({
            id,
            angle: targetAngle,
            element: squareElement,
            progress: 0
        });
        
        // Удаляем после завершения анимации
        setTimeout(() => {
            if (squareElement.parentNode) {
                squareElement.parentNode.removeChild(squareElement);
            }
            activeSquares = activeSquares.filter(sq => sq.id !== id);
        }, flightTime * 1000 + 100);
    }

    // Функция для тестирования - спавнит квадратики по кругу
    function testSpawnCircle() {
        for (let i = 0; i < 360; i += 30) {
            setTimeout(() => spawnSquare(i), i * 10);
        }
    }

    onMount(() => {
        // Добавляем обработчики событий после монтирования компонента
        document.addEventListener('mousemove', rotateContainer);
        document.addEventListener('click', handleLeftClick);

        // Экспортируем функцию спавна в глобальную область
        (window as any).spawnSquare = spawnSquare;
        (window as any).testSpawn = testSpawnCircle;

        // Очистка при размонтировании компонента
        return () => {
            document.removeEventListener('mousemove', rotateContainer);
            document.removeEventListener('click', handleLeftClick);
            activeSquares.forEach(sq => {
                if (sq.element.parentNode) {
                    sq.element.parentNode.removeChild(sq.element);
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
        border: 2px solid #555;
        border-left: 2px dotted #555;
        overflow: hidden;
        transform-origin: center center;
        border-radius: 17px;
        box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
    }

    .square {
        width: 10%;
        height: 100%;
        background-color: #ff4d4d;
        position: absolute;
        left: 0;
        top: 0;
        transition: left 0.05s linear;
        border-radius: 3px;
    }

    .circle {
        width: 50vh;
        height: 50vh;
        border-radius: 50%;
        border: 2px solid rgba(255, 255, 255, 0.2);
        position: absolute;
        display: flex;
        justify-content: center;
        align-items: center;
    }

    /* Стили для летающих квадратиков */
    .flying-square {
        z-index: 10;
    }
</style>