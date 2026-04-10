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

	let selectedSingleEq = $state<string>('');

	let a = $state<number>(1);
	let b = $state<number>(5);
	let h = $state<number>(0.4);

	let result = $state<SolveResponse | null>(null);
	let isLoading = $state<boolean>(false);
	let errorMsg = $state<string | null>(null);

	let plotContainer: HTMLDivElement | undefined = $state();

	$effect(() => {
		if (plotContainer) {
			const sEq = selectedSingleEq;
			const cH = h;
			const res = result;
			drawPlot();
		}
	});

	function drawPlot(): void {
		if (!plotContainer) return;

		try {
			const data: any[] = [];

			if (selectedSingleEq) {
				data.push({
					fn: selectedSingleEq,
					color: '#ff5722', // Оранжевый
					nSamples: 5000,
					range: [a, b],
				});

				const scatterPoints = generatePoints().points.map((point) => [point.x, point.y]);
				data.push({
					points: scatterPoints,
					fnType: 'points',
					graphType: 'scatter',
					color: '#4caf00', // Зеленый
					attr: {
						r: 2, 

						stroke: 'black',
						'stroke-width': 1
					}
				});
			}

			if (result) {
				data.push({
					fn: formatFunctionString(),
					color: '#2196f3', // Синий
					range: [a, b],
					graphType: 'polyline'
				});
			}

			functionPlot({
				target: plotContainer,
				width: 600,
				height: 400,
				grid: true,
				xAxis: { domain: [a - 1, b + 1] },
				yAxis: { domain: [-10, 10] },
				data
			});
		} catch (err) {
			console.error('Ошибка построения графика:', err);
		}
	}

	async function handleSubmit(): Promise<void> {
		isLoading = true;
		errorMsg = null;
		result = null;

		let payload: Payload;

		try {
			payload = generatePoints();
			const res = await fetch(`/api/solve`, {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(payload)
			});

			const data = await res.json();

			if (!res.ok) {
				throw new Error(`Ошибка: ${data.error}` || `Ошибка сервера: ${res.status}`);
			}

			result = data as SolveResponse;
		} catch (err) {
			errorMsg = err instanceof Error ? err.message : String(err);
		} finally {
			isLoading = false;
		}
	}

	function generatePoints(): Payload {
		const points: Point[] = [];

		const code = compile(selectedSingleEq);

		for (let x = a; x <= b + h / 10; x += h) {
			const scope = { x: x };
			const y = code.evaluate(scope);

			points.push({
				x: Number(x.toFixed(4)),
				y: Number(y.toFixed(4))
			});
		}

		return { points };
	}

	function formatFunctionString(): string {
		if (!result) return '';
		const c = result.coeffs;

		const f = (num: number) => {
			// Если число очень маленькое, возвращаем 0, иначе ограничиваем точность
			return Math.abs(num) < 1e-10 ? '0' : num.toFixed(4).replace(/\.?0+$/, '');
		};

		switch (result.name) {
			case 'linear':
				// a*x + b -> coeffs[0] = a, coeffs[1] = b
				return `${f(c[0])} * x ${c[1] < 0 ? '' : '+'}${f(c[1])}`;

			case 'quadratic':
				// ax^2 + bx + c -> [0]=a, [1]=b, [2]=c
				return `${f(c[0])} * x^2 + ${f(c[1])} * x ${c[2] < 0 ? '' : '+'}${f(c[2])}`;

			case 'exponential':
				// a * e^(b*x) -> [0]=a, [1]=b
				// functionPlot понимает exp(u)
				return `${f(c[0])} * exp(${f(c[1])} * x)`;

			case 'logarithmic':
				// a * ln(x) + b -> [0]=a, [1]=b
				// В большинстве парсеров log(x) — это натуральный логарифм
				return `${f(c[0])} * log(x) ${c[1] < 0 ? '' : '+'}${f(c[1])}`;

			case 'power':
				// a * x^b -> [0]=a, [1]=b
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
			</div>

			<div class="section">
				<h3>Параметры</h3>
				<div class="input-group">
					<label>Интервал [a]: <input type="number" step="0.1" bind:value={a} /></label>
					<label>Интервал [b]: <input type="number" step="0.1" bind:value={b} /></label>
					<label>Шаг [h]: <input type="number" step="0.1" bind:value={h} /></label>
				</div>
			</div>

			<button onclick={handleSubmit} disabled={isLoading}>
				{isLoading ? 'Вычисляем...' : 'Решить'}
			</button>

			{#if result}
				<div class="result success">
					<h3>Результат аппроксимации: {result.name}</h3>
					
					<div class="metrics-grid">
						<div class="metric">
							<span>Уравнение:</span>
							<strong>f(x) = {formatFunctionString()}</strong>
						</div>

						<div class="metric">
							<span>Среднеквадратичное отклонение (S):</span>
							<strong>{result.s.toFixed(6)}</strong>
						</div>

						<div class="metric">
							<span>Коэффициент детерминации (R²):</span>
							<strong class={result.r2 < 0.5 ? 'text-error' : ''}>
								{result.r2.toFixed(6)}
							</strong>
						</div>

						{#if result.pearson !== undefined}
							<div class="metric">
								<span>Коэффициент Пирсона (r):</span>
								<strong>{result.pearson.toFixed(6)}</strong>
							</div>
						{/if}
					</div>

					<p class="quality-msg">
						<b>Качество:</b> {result.message}
					</p>

					<details>
						<summary>Показать коэффициенты</summary>
						<ul>
							{#each result.coeffs as coeff, i}
								<li>Коэффициент {i}: {coeff.toFixed(6)}</li>
							{/each}
						</ul>
					</details>
				</div>
			{/if}

			{#if errorMsg}
				<div class="result error">
					<p>{errorMsg}</p>
				</div>
			{/if}
		</div>

		<div class="chart">
			<h3>График</h3>
			<div bind:this={plotContainer}></div>
		</div>
	</div>
</main>

<style>
	.container {
		max-width: 1000px;
		margin: 0 auto;
		padding: 20px;
		font-family: sans-serif;
	}
	.layout {
		display: flex;
		gap: 40px;
	}
	.controls {
		flex: 1;
	}
	.chart {
		flex: 1;
		display: flex;
		flex-direction: column;
		align-items: center;
	}
	.section {
		margin-bottom: 20px;
		padding: 15px;
		border: 1px solid #ddd;
		border-radius: 8px;
	}
	h3 {
		margin-top: 0;
		margin-bottom: 15px;
		font-size: 16px;
		color: #333;
	}
	label {
		display: block;
		margin-bottom: 8px;
		cursor: pointer;
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
	input[type='number'],
	select {
		padding: 6px;
		margin-top: 4px;
		border: 1px solid #ccc;
		border-radius: 4px;
	}
	button {
		padding: 10px 20px;
		background: #007bff;
		color: white;
		border: none;
		border-radius: 4px;
		font-size: 16px;
		cursor: pointer;
		width: 100%;
	}
	button:hover {
		background: #0056b3;
	}
	button:disabled {
		background: #aaa;
		cursor: not-allowed;
	}
	.result {
		margin-top: 20px;
		padding: 15px;
		border-radius: 8px;
	}
	.success {
		background: #e8f5e9;
		border: 1px solid #c8e6c9;
	}
	.error {
		background: #ffebee;
		border: 1px solid #ffcdd2;
		color: #c62828;
	}
</style>
