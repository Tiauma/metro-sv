<script lang="ts">
    import { onMount } from 'svelte';

    let container: HTMLDivElement;
    let square: HTMLDivElement;
    let rotationAngle = 0;
    
    // Глобальные переменные
    const flightTime = 1; // время полета в секундах
    const spawnDistance = 1.5; // расстояние спавна от края экрана (множитель)

    // Массив для хранения активных элементов
    let activeElements: Array<{
        id: number,
        element: HTMLDivElement,
        creationTime: number
    }> = [];
    let elementId = 0;

    // Универсальная функция для анимации элемента
    function animateElement(
        element: HTMLDivElement, 
        targetAngle: number, 
        onComplete?: () => void
    ): () => void {
        const screenWidth = window.innerWidth;
        
        // Вычисляем конечную позицию (точка на круге)
        const circleRadius = 25 * window.innerHeight / 100;
        const centerX = window.innerWidth / 2;
        const centerY = window.innerHeight / 2;
        
        const targetRadians = ((360 - targetAngle) * Math.PI) / 180;
        const targetX = centerX + circleRadius * Math.cos(targetRadians);
        const targetY = centerY + circleRadius * Math.sin(targetRadians);
        
        // Вычисляем начальную позицию (вне экрана)
        const spawnX = centerX + screenWidth * spawnDistance * Math.cos(targetRadians);
        const spawnY = centerY + screenWidth * spawnDistance * Math.sin(targetRadians);
        
        // Устанавливаем начальную позицию
        element.style.position = 'fixed';
        element.style.left = `${spawnX}px`;
        element.style.top = `${spawnY}px`;
        element.style.transform = 'translate(-50%, -50%)';
        
        // Добавляем на страницу
        document.body.appendChild(element);
        
        element.style.transform = `translate(-50%, -50%) rotate(${-targetAngle - 90}deg)`;
        // Анимируем элемент к целевой позиции
        requestAnimationFrame(() => {
            requestAnimationFrame(() => {
                element.style.transition = `all ${flightTime}s linear`;
                element.style.left = `${targetX}px`;
                element.style.top = `${targetY}px`;
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

        if (angle >= 360) angle = 0;
        spawnSquare(angle);
        angle += 10;
    }

    // Функция для спавна квадратика (использует универсальную функцию)
    function spawnSquare(targetAngle: number) {
        // Создаем элемент квадратика
        const squareElement = document.createElement('div');
        squareElement.className = 'flying-square';
        squareElement.style.width = `8vh`;
        squareElement.style.height = `5vh`;
        squareElement.style.backgroundColor = 'white';
        squareElement.style.borderRadius = '3px';
        squareElement.style.boxShadow = '0 0 10px rgba(255, 255, 255, 0.5)';
        squareElement.style.zIndex = '10';
        
        // Записываем время создания
        const creationTime = performance.now();
        const id = elementId++;
        
        // Анимируем элемент
        const cancelAnimation = animateElement(
            squareElement, 
            targetAngle,
            () => {
                console.log(`Элемент ${id} достиг цели`);
            }
        );
        
        // Добавляем в массив активных элементов
        activeElements.push({
            id,
            element: squareElement,
            creationTime
        });

        // Удаляем после завершения анимации
        setTimeout(() => {
            const removalTime = performance.now();
            const lifetime = removalTime - creationTime;

            console.log(`Элемент ${id} удален в: ${removalTime.toFixed(2)}ms`);
            console.log(`Время жизни элемента ${id}: ${lifetime.toFixed(2)}ms`);

            cancelAnimation();
            activeElements = activeElements.filter(el => el.id !== id);
        }, flightTime * 1000);
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
        (window as any).animateElement = animateElement;

        // Очистка при размонтировании компонента
        return () => {
            document.removeEventListener('mousemove', rotateContainer);
            document.removeEventListener('click', handleLeftClick);
            activeElements.forEach(el => {
                if (el.element.parentNode) {
                    el.element.parentNode.removeChild(el.element);
                }
            });
            delete (window as any).spawnSquare;
            delete (window as any).testSpawn;
            delete (window as any).animateElement;
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
</style>