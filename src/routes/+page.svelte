<script lang="ts">
    import functionPlot from 'function-plot';
    import { compile } from 'mathjs';

    interface Point {
        x: number;
        y: number;
    }

    interface SolveResponse {
        name: string;
        coeffs: number[];
        s: number; // Среднеквадратичное отклонение
        r2: number; // Коэффициент детерминации
        pearson?: number; // только для линейной
        message: string; // Текст про точность R2
        x_values: number[];
        y_values: number[];
        phi_values: number[];
        eps_values: number[];
    }

    interface Payload {
        points: Point[];
    }

    let inputMode = $state<'function' | 'points'>('function');

    let selectedSingleEq = $state<string>('2 * x^2 - 3 * x + 5');
    let a = $state<number>(1);
    let b = $state<number>(5);
    let h = $state<number>(0.4);

    let manualPoints = $state<Point[]>([
        { x: 1, y: 1.5 }, { x: 2, y: 3.2 }, { x: 3, y: 6.1 }, { x: 4, y: 8.5 },
        { x: 5, y: 12.2 }, { x: 6, y: 15.8 }, { x: 7, y: 19.1 }, { x: 8, y: 23.5 }
    ]);

    let results = $state<SolveResponse[]>([]);
    let isLoading = $state<boolean>(false);
    let errorMsg = $state<string | null>(null);

    let bestResult = $derived(
        results.length > 0
            ? [...results].sort((resA, resB) => Math.abs(1 - resA.r2) - Math.abs(1 - resB.r2))[0]
            : null
    );

    let plotContainer: HTMLDivElement | undefined = $state();

    const plotColors = ['#2196f3', '#f44336', '#9c27b0', '#ff9800', '#009688', '#795548'];

    $effect(() => {
        if (plotContainer) {
            const _eq = selectedSingleEq;
            const _h = h;
            const _mode = inputMode;
            const _pts = manualPoints.map(p => `${p.x},${p.y}`).join(';'); 
            const _res = results;
			console.log(_res.length)
            drawPlot();
        }
    });

    function drawPlot(): void {
        if (!plotContainer) return;

        try {
            const data: any[] = [];
            
            const currentPoints = inputMode === 'function' ? generatePoints().points : manualPoints;
            const xVals = currentPoints.map(p => p.x);
            const yVals = currentPoints.map(p => p.y);
            
            const minX = Math.min(...xVals);
            const maxX = Math.max(...xVals);
            const minY = Math.min(...yVals);
            const maxY = Math.max(...yVals);

            const xDomain = [minX - Math.max(1, (maxX - minX) * 0.1), maxX + Math.max(1, (maxX - minX) * 0.1)];
            const yDomain = [minY - Math.max(1, (maxY - minY) * 0.1), maxY + Math.max(1, (maxY - minY) * 0.1)];

            if (inputMode === 'function' && selectedSingleEq) {
                data.push({
                    fn: selectedSingleEq,
                    color: 'rgba(0, 0, 0, 0.15)',
                    nSamples: 1000,
                    range: [minX, maxX],
                });
            }

            data.push({
                points: currentPoints.map((point) => [point.x, point.y]),
                fnType: 'points',
                graphType: 'scatter',
                color: '#4caf00', // Зеленый
                attr: {
                    r: 4, 
                    stroke: 'black',
                    'stroke-width': 1
                }
            });

            if (results && results.length > 0) {
                results.forEach((res, idx) => {
                    data.push({
                        fn: formatFunctionString(res),
                        color: plotColors[idx % plotColors.length],
                        range: [minX, maxX],
                        graphType: 'polyline'
                    });
                });
            }

            functionPlot({
                target: plotContainer,
                width: 600,
                height: 400,
                grid: true,
                xAxis: { domain: xDomain },
                yAxis: { domain: yDomain },
                data
            });
        } catch (err) {
            console.error('Ошибка построения графика:', err);
        }
    }

    async function handleSubmit(): Promise<void> {
        isLoading = true;
        errorMsg = null;
        results = [];

        try {
            const payload: Payload = {
                points: inputMode === 'function' ? generatePoints().points : manualPoints
            };

            const res = await fetch(`/api/solve`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            const data = await res.json();

            if (!res.ok) {
                throw new Error(`Ошибка: ${data.error}` || `Ошибка сервера: ${res.status}`);
            }

            // Теперь ожидаем массив
            results = data as SolveResponse[];
        } catch (err) {
            errorMsg = err instanceof Error ? err.message : String(err);
        } finally {
            isLoading = false;
        }
    }

    function generatePoints(): Payload {
        const points: Point[] = [];
		if (h <= 0) {
			return {points};
		}
        try {
            const code = compile(selectedSingleEq);
            for (let x = a; x <= b + h / 10; x += h) {
                const y = code.evaluate({ x: x });
                points.push({
                    x: Number(x.toFixed(4)),
                    y: Number(y.toFixed(4))
                });
            }
        } catch(e) {
            // Игнорируем ошибки компиляции во время ввода
        }
        return { points };
    }

    function formatFunctionString(res: SolveResponse): string {
        const c = res.coeffs;
        const f = (num: number) => {
            return Math.abs(num) < 1e-10 ? '0' : num.toFixed(4).replace(/\.?0+$/, '');
        };

        switch (res.name) {
            case 'linear':
                return `${f(c[0])} * x ${c[1] < 0 ? '' : '+'}${f(c[1])}`;
            case 'quadratic':
                return `${f(c[0])} * x^2 + ${f(c[1])} * x ${c[2] < 0 ? '' : '+'}${f(c[2])}`;
			case 'cubic':
			    return `${f(c[0])} * x^3 + ${f(c[1])} * x^2 + ${f(c[2])} * x ${c[3] < 0 ? '' : '+'}${f(c[2])}`
            case 'exponential':
                return `${f(c[0])} * exp(${f(c[1])} * x)`;
            case 'logarithmic':
                return `${f(c[0])} * log(x) ${c[1] < 0 ? '' : '+'}${f(c[1])}`;
            case 'power':
                return `${f(c[0])} * x^${f(c[1])}`;
            default:
                return '';
        }
    }
</script>

<main class="container">
    <h1>Вычислительная математика 4 Зенченков P3215</h1>

    <div class="layout">
        <div class="controls">
            
            <div class="section mode-selector">
                <h3>Режим ввода</h3>
                <label>
                    <input type="radio" bind:group={inputMode} value="function" /> По функции
                </label>
                <label>
                    <input type="radio" bind:group={inputMode} value="points" /> По точкам (8 шт.)
                </label>
            </div>

            {#if inputMode === 'function'}
                <div class="section">
                    <h3>Выбор функции</h3>
                    <div class="input-group">
                        <label for="function-input">Введите функцию f(x):</label>
                        <input
                            type="text"
                            id="function-input"
                            style="width: 100%; margin-bottom: 10px;"
                            bind:value={selectedSingleEq}
                        />
                    </div>
                    <div class="input-group">
                        <label>Интервал [a]: <input type="number" step="0.1" bind:value={a} /></label>
                        <label>Интервал [b]: <input type="number" step="0.1" bind:value={b} /></label>
                        <label>Шаг [h]: <input type="number" step="0.1" bind:value={h} /></label>
                    </div>
                </div>
            {:else}
                <div class="section">
                    <h3>Ввод точек</h3>
                    <div class="points-grid">
                        {#each manualPoints as point, i}
                            <div class="point-input">
                                <b>№{i+1}</b>
                                <label>X: <input type="number" step="any" bind:value={point.x} /></label>
                                <label>Y: <input type="number" step="any" bind:value={point.y} /></label>
                            </div>
                        {/each}
                    </div>
                </div>
            {/if}

            <button onclick={handleSubmit} disabled={isLoading}>
                {isLoading ? 'Вычисляем...' : 'Решить'}
            </button>

            {#if errorMsg}
                <div class="result error">
                    <p>{errorMsg}</p>
                </div>
            {/if}

            <div class="results-container">
                {#each results as res, idx}
                    <div class="result card {res === bestResult ? 'best' : ''}">
                        {#if res === bestResult}
                            <span class="best-badge">★ Лучшая аппроксимация</span>
                        {/if}
                        
                        <h3 style="color: {plotColors[idx % plotColors.length]}">{res.name}</h3>
                        
                        <div class="metrics-grid">
                            <div class="metric">
                                <span>Уравнение:</span>
                                <strong>f(x) = {formatFunctionString(res)}</strong>
                            </div>

                            <div class="metric">
                                <span>Отклонение (S):</span>
                                <strong>{res.s.toFixed(6)}</strong>
                            </div>

                            <div class="metric">
                                <span>R²:</span>
                                <strong class={res.r2 < 0.5 ? 'text-error' : ''}>
                                    {res.r2.toFixed(6)}
                                </strong>
                            </div>
                        </div>

                        <details>
                            <summary>Подробнее</summary>
                            <p class="quality-msg"><b>Качество:</b> {res.message}</p>
                            {#if res.pearson !== undefined}
                                <p>Коэффициент Пирсона (r): {res.pearson.toFixed(6)}</p>
                            {/if}
                            <ul>
                                {#each res.coeffs as coeff, i}
                                    <li>Коэффициент {i}: {coeff.toFixed(6)}</li>
                                {/each}
                            </ul>
                        </details>
                    </div>
                {/each}
            </div>

        </div>

        <div class="chart">
            <h3>График</h3>
            <div bind:this={plotContainer}></div>
            
            {#if results.length > 0}
                <div class="legend">
                    {#each results as res, idx}
                        <div class="legend-item">
                            <span class="color-box" style="background-color: {plotColors[idx % plotColors.length]}"></span>
                            {res.name}
                        </div>
                    {/each}
                </div>
            {/if}
        </div>
    </div>
</main>

<style>
    .container {
        max-width: 1200px;
        margin: 0 auto;
        padding: 20px;
        font-family: sans-serif;
    }
    .layout {
        display: flex;
        gap: 40px;
        align-items: flex-start;
    }
    .controls {
        flex: 1;
        min-width: 450px;
    }
    .chart {
        flex: 1;
        display: flex;
        flex-direction: column;
        align-items: center;
        position: sticky;
        top: 20px;
    }
    .section {
        margin-bottom: 20px;
        padding: 15px;
        border: 1px solid #ddd;
        border-radius: 8px;
        background: #fdfdfd;
    }
    .mode-selector label {
        display: inline-flex;
        align-items: center;
        gap: 5px;
        margin-right: 20px;
        font-weight: bold;
    }
    .points-grid {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 10px;
    }
    .point-input {
        display: flex;
        align-items: center;
        gap: 8px;
        background: #f0f4f8;
        padding: 8px;
        border-radius: 6px;
    }
    .point-input label {
        display: flex;
        align-items: center;
        gap: 4px;
        margin: 0;
    }
    .point-input input {
        width: 60px;
        margin: 0;
    }
    h3 {
        margin-top: 0;
        margin-bottom: 15px;
        font-size: 16px;
        color: #333;
    }
    .input-group {
        display: flex;
        gap: 10px;
    }
    .input-group label {
        display: flex;
        flex-direction: column;
        font-size: 14px;
    }
    input[type='number'], input[type='text'] {
        padding: 6px;
        margin-top: 4px;
        border: 1px solid #ccc;
        border-radius: 4px;
    }
    button {
        padding: 12px 20px;
        background: #007bff;
        color: white;
        border: none;
        border-radius: 6px;
        font-size: 16px;
        font-weight: bold;
        cursor: pointer;
        width: 100%;
        transition: background 0.2s;
    }
    button:hover { background: #0056b3; }
    button:disabled { background: #aaa; cursor: not-allowed; }
    
    .results-container {
        display: flex;
        flex-direction: column;
        gap: 15px;
        margin-top: 25px;
    }
    .card {
        padding: 15px;
        border-radius: 8px;
        background: #fff;
        border: 1px solid #e0e0e0;
        box-shadow: 0 2px 4px rgba(0,0,0,0.05);
        position: relative;
    }
    .best {
        border: 2px solid #ff9800;
        background: #fff8e1;
    }
    .best-badge {
        position: absolute;
        top: -10px;
        right: 15px;
        background: #ff9800;
        color: white;
        padding: 4px 10px;
        border-radius: 12px;
        font-size: 12px;
        font-weight: bold;
        box-shadow: 0 2px 4px rgba(0,0,0,0.2);
    }
    .metrics-grid {
        display: grid;
        grid-template-columns: 1fr;
        gap: 8px;
        margin-bottom: 10px;
    }
    .metric {
        display: flex;
        justify-content: space-between;
        background: #f9f9f9;
        padding: 6px 10px;
        border-radius: 4px;
        font-size: 14px;
    }
    .text-error { color: #d32f2f; }
    .error {
        background: #ffebee;
        border: 1px solid #ffcdd2;
        color: #c62828;
        padding: 15px;
        border-radius: 8px;
        margin-top: 20px;
    }
    details {
        margin-top: 10px;
        font-size: 14px;
        color: #555;
    }
    summary { cursor: pointer; font-weight: bold; }
    
    .legend {
        display: flex;
        flex-wrap: wrap;
        gap: 15px;
        margin-top: 15px;
        justify-content: center;
    }
    .legend-item {
        display: flex;
        align-items: center;
        gap: 6px;
        font-size: 14px;
        font-weight: 500;
    }
    .color-box {
        width: 14px;
        height: 14px;
        border-radius: 3px;
        display: inline-block;
    }
</style>