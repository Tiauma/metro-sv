<script lang="ts">
    import { onMount } from "svelte";

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

    // Логирование времени видимости квадратиков
    const visibilityLog: {
        id: number;
        spawnTime: number;
        visibleTime: number;
    }[] = [];

    // Пул объектов
    type PooledElement = {
        element: HTMLDivElement;
        active: boolean;
        id: number;
        spawnTime: number; // время появления квадратика
    };

    let pool: PooledElement[] = [];

    // Функция для обновления размеров
    function updateGameAreaSize(): void {
        const circleElement = document.querySelector(".circle") as HTMLElement;
        if (circleElement) {
            const rect = circleElement.getBoundingClientRect();
            gameAreaWidth = rect.width;
            gameAreaHeight = rect.height;
            circleRadius = gameAreaWidth / 2;
        }
    }

    function animateElement(
        element: HTMLDivElement,
        targetAngle: number,
        onComplete?: () => void,
    ): () => void {
        updateGameAreaSize();

        const circleElement = document.querySelector(".circle") as HTMLElement;
        const circleRect = circleElement.getBoundingClientRect();
        const centerX = circleRect.left + circleRect.width / 2;
        const centerY = circleRect.top + circleRect.height / 2;

        const targetRadians = ((360 - targetAngle) * Math.PI) / 180;
        const targetX = centerX + circleRadius * Math.cos(targetRadians);
        const targetY = centerY + circleRadius * Math.sin(targetRadians);

        const spawnDistance = 3; // 300% радиуса
        const spawnX =
            centerX + circleRadius * spawnDistance * Math.cos(targetRadians);
        const spawnY =
            centerY + circleRadius * spawnDistance * Math.sin(targetRadians);

        const offsetX = spawnX - window.innerWidth / 2;
        const offsetY = spawnY - window.innerHeight / 2;

        // Инициализация стилей - начинаем с прозрачного
        element.style.position = "fixed";
        element.style.left = "50%";
        element.style.top = "50%";
        element.style.transform = `translate(${offsetX}px, ${offsetY}px) translate(-50%, -50%) rotate(${-targetAngle - 90}deg)`;
        element.style.transformOrigin = "center";
        element.style.opacity = "0"; // ← ИЗМЕНЕНО: начинаем прозрачными
        element.style.pointerEvents = "none";
        element.style.zIndex = "10";

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
                element.style.opacity = "1"; // ← ИЗМЕНЕНО: заканчиваем непрозрачными
            });
        });

        // Возвращаем функцию отмены (в нашем случае — просто возврат в пул)
        return () => {
            if (element.parentNode) {
                // Не удаляем, а просто скрываем и возвращаем в пул
                element.style.transition = "none";
                element.style.opacity = "0";
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
        square.style.left = square.style.left === "90%" ? "0%" : "90%";
        spawnSlider(0, 90, 0.5);
        testSpawnCircle();
    }

    // Получение свободного элемента из пула
    function getPooledElement(): PooledElement | null {
        for (let i = 0; i < pool.length; i++) {
            if (!pool[i].active) {
                pool[i].active = true;
                pool[i].spawnTime = Date.now(); // записываем время появления
                return pool[i];
            }
        }
        // Если нет свободных — можно расширить пул (опционально)
        console.warn("Pool exhausted!");
        return null;
    }

    // Возврат элемента в пул
    function returnToPool(id: number) {
        const item = pool.find((p) => p.id === id);
        if (item) {
            const visibleTime = (Date.now() - item.spawnTime) / 1000; // время видимости в секундах

            // Логируем время видимости
            visibilityLog.push({
                id: item.id,
                spawnTime: item.spawnTime,
                visibleTime: visibleTime,
            });

            // Выводим информацию в консоль
            console.log(
                `Квадратик ${id} был виден ${visibleTime.toFixed(3)} секунд (должно быть: ${flightTime} секунд)`,
            );

            item.active = false;
            // Сброс transition, чтобы не мешал
            item.element.style.transition = "none";
            item.element.style.opacity = "0";
        }
    }

    // Спавн квадратика с поддержкой цвета
    function spawnSquare(targetAngle: number, color: string = "white") {
        const pooled = getPooledElement();
        if (!pooled) return;

        const element = pooled.element;
        const id = pooled.id;

        // Устанавливаем цвет квадратика
        element.style.backgroundColor = color;

        // Анимация
        animateElement(element, targetAngle, () => {
            console.log(`Квадратик ${id} достиг цели`);
        });

        // Автоматический возврат в пул после окончания полёта
        setTimeout(() => {
            returnToPool(id);
        }, flightTime * 1000);
    }

    // Функция для создания слайдера между двумя углами с голубым цветом
    function spawnSlider(
        startAngle: number,
        endAngle: number,
        duration: number,
    ) {
        const angleDiff = endAngle - startAngle;
        const stepCount = Math.floor((duration * 1000) / 30); // количество шагов (примерно 30ms между квадратиками)
        const angleStep = angleDiff / stepCount;

        for (let i = 0; i <= stepCount; i++) {
            const currentAngle = startAngle + angleStep * i;
            setTimeout(() => spawnSquare(currentAngle, "#5d76cb"), i * 30); // голубой цвет
        }
    }

    function testSpawnCircle() {
        for (let i = 0; i < 60; i += 7) {
            setTimeout(() => spawnSquare(i, "white"), i * 10); // белый цвет
        }
    }

    // Функция для просмотра лога видимости
    function showVisibilityLog() {
        console.log("=== ЛОГ ВРЕМЕНИ ВИДИМОСТИ КВАДРАТИКОВ ===");
        visibilityLog.forEach((log) => {
            console.log(
                `ID: ${log.id}, Время: ${log.visibleTime.toFixed(3)}s, Ожидалось: ${flightTime}s`,
            );
        });

        // Статистика
        if (visibilityLog.length > 0) {
            const avgTime =
                visibilityLog.reduce((sum, log) => sum + log.visibleTime, 0) /
                visibilityLog.length;
            const minTime = Math.min(
                ...visibilityLog.map((log) => log.visibleTime),
            );
            const maxTime = Math.max(
                ...visibilityLog.map((log) => log.visibleTime),
            );

            console.log(`\nСтатистика (${visibilityLog.length} квадратиков):`);
            console.log(`Среднее: ${avgTime.toFixed(3)}s`);
            console.log(`Минимальное: ${minTime.toFixed(3)}s`);
            console.log(`Максимальное: ${maxTime.toFixed(3)}s`);
            console.log(`Целевое: ${flightTime}s`);
        }
    }

    onMount(() => {
        updateGameAreaSize();

        // Создание пула
        for (let i = 0; i < POOL_SIZE; i++) {
            const div = document.createElement("div");
            div.className = "flying-square pooled-square";
            div.style.cssText = `
                width: 8vh;
                height: 5vh;
                background-color: white;
                border-radius: 0.9vh;
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
                id: i,
                spawnTime: 0,
            });
        }

        // Обработчики
        document.addEventListener("mousemove", rotateContainer);
        document.addEventListener("click", handleLeftClick);
        window.addEventListener("resize", updateGameAreaSize);

        // Глобальные отладочные функции
        // В onMount добавьте:
        (window as any).spawnSlider = spawnSlider;
        (window as any).spawnSquare = spawnSquare;
        (window as any).testSpawn = testSpawnCircle;
        (window as any).showVisibilityLog = showVisibilityLog; // новая функция для просмотра лога

        return () => {
            document.removeEventListener("mousemove", rotateContainer);
            document.removeEventListener("click", handleLeftClick);
            window.removeEventListener("resize", updateGameAreaSize);

            // Удаляем все из пула
            pool.forEach((p) => {
                if (p.element.parentNode) {
                    p.element.parentNode.removeChild(p.element);
                }
            });

            // В cleanup добавьте:
            delete (window as any).spawnSlider;
            delete (window as any).spawnSquare;
            delete (window as any).testSpawn;
            delete (window as any).showVisibilityLog;
        };
    });
</script>

<svelte:head>
    <title>Rhythm Game</title>
</svelte:head>

<div class="container-wrapper">
    <div class="circle"></div>
    <div
        class="container"
        bind:this={container}
        style="transform: rotate({rotationAngle}rad)"
    >
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
        font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
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
