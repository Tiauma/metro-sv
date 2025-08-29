<script lang="ts">
    import { onMount } from 'svelte';

    let container: HTMLDivElement;
    let square: HTMLDivElement;
    let rotationAngle = 0;

    // Параметры
    const flightTime = 1; // время полёта в секундах
    const POOL_SIZE = 100; // размер пула (можно подогнать под макс. одновременных квадратов)

    // Игровая область
    let gameAreaWidth = 0;
    let gameAreaHeight = 0;
    let circleRadius = 0;

    // Пул объектов
    type PooledElement = {
        element: HTMLDivElement;
        active: boolean;
        id: number;
    };

    let pool: PooledElement[] = [];

    // Функция для обновления размеров
    function updateGameAreaSize(): void {
        const circleElement = document.querySelector('.circle') as HTMLElement;
        if (circleElement) {
            const rect = circleElement.getBoundingClientRect();
            gameAreaWidth = rect.width;
            gameAreaHeight = rect.height;
            circleRadius = gameAreaWidth / 2;
        }
    }

    // Универсальная функция анимации
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

        const targetRadians = ((360 - targetAngle) * Math.PI) / 180;
        const targetX = centerX + circleRadius * Math.cos(targetRadians);
        const targetY = centerY + circleRadius * Math.sin(targetRadians);

        const spawnDistance = 3; // 300% радиуса
        const spawnX = centerX + circleRadius * spawnDistance * Math.cos(targetRadians);
        const spawnY = centerY + circleRadius * spawnDistance * Math.sin(targetRadians);

        const offsetX = spawnX - window.innerWidth / 2;
        const offsetY = spawnY - window.innerHeight / 2;

        // Инициализация стилей
        element.style.position = 'fixed';
        element.style.left = '50%';
        element.style.top = '50%';
        element.style.transform = `translate(${offsetX}px, ${offsetY}px) translate(-50%, -50%) rotate(${-targetAngle - 90}deg)`;
        element.style.transformOrigin = 'center';
        element.style.opacity = '1';
        element.style.pointerEvents = 'none';
        element.style.zIndex = '10';

        // Добавляем в DOM, если ещё не добавлен
        if (!element.parentNode) {
            document.body.appendChild(element);
        }

        // Анимация через transform
        requestAnimationFrame(() => {
            requestAnimationFrame(() => {
                element.style.transition = `transform ${flightTime}s linear, opacity ${flightTime}s linear`;

                const targetOffsetX = targetX - window.innerWidth / 2;
                const targetOffsetY = targetY - window.innerHeight / 2;

                element.style.transform = `translate(${targetOffsetX}px, ${targetOffsetY}px) translate(-50%, -50%) rotate(${-targetAngle - 90}deg)`;
                element.style.opacity = '0';
            });
        });

        // Возвращаем функцию отмены (в нашем случае — просто возврат в пул)
        return () => {
            if (element.parentNode) {
                // Не удаляем, а просто скрываем и возвращаем в пул
                element.style.transition = 'none';
                element.style.opacity = '0';
                // Оставляем в DOM, но неактивный
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

        // Спавн 80 квадратиков с задержкой
        for (let i = 0; i < 80; i++) {
            const currentAngle = angle % 360;
            ((currentAngle) => {
                setTimeout(() => spawnSquare(currentAngle), i * 30);
            })(currentAngle);
            angle += 3;
        }
    }

    // Получение свободного элемента из пула
    function getPooledElement(): PooledElement | null {
        for (let i = 0; i < pool.length; i++) {
            if (!pool[i].active) {
                pool[i].active = true;
                return pool[i];
            }
        }
        // Если нет свободных — можно расширить пул (опционально)
        console.warn('Pool exhausted!');
        return null;
    }

    // Возврат элемента в пул
    function returnToPool(id: number) {
        const item = pool.find(p => p.id === id);
        if (item) {
            item.active = false;
            // Сброс transition, чтобы не мешал
            item.element.style.transition = 'none';
            item.element.style.opacity = '0';
        }
    }

    // Спавн квадратика
    function spawnSquare(targetAngle: number) {
        const pooled = getPooledElement();
        if (!pooled) return;

        const element = pooled.element;
        const id = pooled.id;

        // Анимация
        animateElement(element, targetAngle, () => {
            console.log(`Квадратик ${id} достиг цели`);
        });

        // Автоматический возврат в пул после окончания полёта
        setTimeout(() => {
            returnToPool(id);
        }, flightTime * 1000);
    }

    function testSpawnCircle() {
        for (let i = 0; i < 360; i += 30) {
            setTimeout(() => spawnSquare(i), i * 10);
        }
    }

    onMount(() => {
        updateGameAreaSize();

        // Создание пула
        for (let i = 0; i < POOL_SIZE; i++) {
            const div = document.createElement('div');
            div.className = 'flying-square pooled-square';
            div.style.cssText = `
                width: 8vh;
                height: 5vh;
                background-color: white;
                border-radius: 0.3vh;
                box-shadow: 0 0 0.5vh rgba(255, 255, 255, 0.5);
                position: fixed;
                left: 50%;
                top: 50%;
                transform: translate(-50%, -50%);
                opacity: 0;
                pointer-events: none;
                z-index: 10;
                transition: none;
            `;
            document.body.appendChild(div);

            pool.push({
                element: div,
                active: false,
                id: i
            });
        }

        // Обработчики
        document.addEventListener('mousemove', rotateContainer);
        document.addEventListener('click', handleLeftClick);
        window.addEventListener('resize', updateGameAreaSize);

        // Глобальные отладочные функции
        (window as any).spawnSquare = spawnSquare;
        (window as any).testSpawn = testSpawnCircle;

        return () => {
            document.removeEventListener('mousemove', rotateContainer);
            document.removeEventListener('click', handleLeftClick);
            window.removeEventListener('resize', updateGameAreaSize);

            // Удаляем все из пула
            pool.forEach(p => {
                if (p.element.parentNode) {
                    p.element.parentNode.removeChild(p.element);
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

    /* Стили для пуловых квадратиков — не нужны в компоненте, но можно добавить */
    .pooled-square {
        will-change: transform, opacity;
    }
</style>