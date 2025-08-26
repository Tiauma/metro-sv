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
        // Используем фиксированное расстояние от края круга
        return 3; // 120% от радиуса круга
    }

    // Универсальная функция для анимации элемента
    function animateElement(
        element: HTMLDivElement, 
        targetAngle: number, 
        onComplete?: () => void
    ): () => void {
        updateGameAreaSize();
        
        // Центр игровой области (относительно viewport)
        const circleElement = document.querySelector('.circle') as HTMLElement;
        const circleRect = circleElement.getBoundingClientRect();
        const centerX = circleRect.left + circleRect.width / 2;
        const centerY = circleRect.top + circleRect.height / 2;
        
        // Вычисляем конечную позицию (точка на краю круга)
        const targetRadians = ((360 - targetAngle) * Math.PI) / 180;
        const targetX = centerX + circleRadius * Math.cos(targetRadians);
        const targetY = centerY + circleRadius * Math.sin(targetRadians);
        
        // Рассчитываем spawnDistance
        spawnDistance = calculateSpawnDistance();
        
        // Вычисляем начальную позицию (вне круга)
        const spawnX = centerX + circleRadius * spawnDistance * Math.cos(targetRadians);
        const spawnY = centerY + circleRadius * spawnDistance * Math.sin(targetRadians);
        
        // Рассчитываем фактическое расстояние, которое пролетит элемент
        const distance = Math.sqrt(
            Math.pow(targetX - spawnX, 2) + Math.pow(targetY - spawnY, 2)
        );
        
        // Рассчитываем требуемую скорость (пикселей в секунду)
        const requiredSpeed = distance / flightTime;
        
        console.log(`Игровая область: ${gameAreaWidth}x${gameAreaHeight}, Расстояние: ${distance.toFixed(2)}px, Скорость: ${requiredSpeed.toFixed(2)}px/сек`);
        
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
        angle += 30;
    }

    // Функция для спавна квадратика
    function spawnSquare(targetAngle: number) {
        // Создаем элемент квадратика
        const squareElement = document.createElement('div');
        squareElement.className = 'flying-square';
        squareElement.style.width = `8vh`;
        squareElement.style.height = `5vh`;
        squareElement.style.backgroundColor = 'white';
        squareElement.style.borderRadius = '0.3vh';
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

    // Функция для тестирования
    function testSpawnCircle() {
        for (let i = 0; i < 360; i += 30) {
            setTimeout(() => spawnSquare(i), i * 10);
        }
    }

    onMount(() => {
        // Инициализируем размеры игровой области
        updateGameAreaSize();
        spawnDistance = calculateSpawnDistance();
        
        // Добавляем обработчики событий
        document.addEventListener('mousemove', rotateContainer);
        document.addEventListener('click', handleLeftClick);

        // Обновляем размеры при изменении размера окна
        window.addEventListener('resize', updateGameAreaSize);

        // Экспортируем функции
        (window as any).spawnSquare = spawnSquare;
        (window as any).testSpawn = testSpawnCircle;

        // Очистка при размонтировании
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

<!-- Остальная часть компонента без изменений -->
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
        border: 0.2vh solid #555; /* Используем vh вместо px */
        border-left: 0.2vh dotted #555; /* Используем vh вместо px */
        overflow: hidden;
        transform-origin: center center;
        border-radius: 1.7vh; /* Адаптируем border-radius */
        box-shadow: 0 0.4vh 1.5vh rgba(0, 0, 0, 0.3); /* Адаптируем тень */
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