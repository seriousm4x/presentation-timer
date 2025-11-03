<script lang="ts">
	import Icon from '@iconify/svelte';
	import { onMount, setContext } from 'svelte';

	let totalSeconds = $state(0);
	let remaining = $state(0);
	setContext('remaining', () => totalSeconds);

	let remainingHours = $derived(new Date(Math.abs(remaining) * 1000).getUTCHours());
	let remainingMinutes = $derived(new Date(Math.abs(remaining) * 1000).getUTCMinutes());
	let remainingSeconds = $derived(new Date(Math.abs(remaining) * 1000).getUTCSeconds());

	let running = $state(false);
	let interval: NodeJS.Timeout | undefined = $state(undefined);
	let mouseInterval: NodeJS.Timeout | undefined = $state(undefined);
	let color = $state('bg-success');
	let progress = $state(0);
	let countingDown = $state(true);
	let timeSelector: HTMLInputElement | undefined = $state();

	function start() {
		if (interval) return;
		running = true;
		countingDown = remaining > 0;

		interval = setInterval(() => {
			if (countingDown) {
				remaining--;
				if (remaining <= 0) {
					countingDown = false;
				}
			} else {
				remaining++;
			}
		}, 1000);
	}

	function stop() {
		running = false;
		if (interval) {
			clearInterval(interval);
			interval = undefined;
		}
	}

	function reset() {
		stop();
		remaining = totalSeconds;
		color = 'bg-success';
		countingDown = true;
	}

	function setTime(h: number = 0, m: number = 0, s: number = 0) {
		totalSeconds = h * 3600 + m * 60 + s;
		reset();
	}

	function handleKeyboard(event: KeyboardEvent) {
		if (event.code === 'Space') {
			event.preventDefault();
			if (running) {
				stop();
			} else {
				start();
			}
		} else if (event.code === 'Backspace') {
			reset();
		}
	}

	function hideCursor() {
		document.body.classList.remove('cursor-none');
		clearTimeout(mouseInterval);
		mouseInterval = setTimeout(() => {
			document.body.classList.add('cursor-none');
		}, 2000);
	}

	$effect(() => {
		progress = totalSeconds ? 1 - remaining / totalSeconds : 0;

		if (countingDown) {
			color = progress >= 0.8 ? 'bg-error' : progress >= 0.6 ? 'bg-warning' : 'bg-success';
		}
	});

	onMount(() => {
		setTime(0, 10, 0);
	});
</script>

<svelte:window onkeydown={handleKeyboard} onmousemove={hideCursor} />

<div class="relative flex h-svh flex-col items-center justify-between overflow-hidden text-white">
	<div class="relative h-8 w-full bg-neutral-200/50">
		<div
			class="absolute top-0 left-0 -z-10 h-full bg-white/80 transition-all duration-300"
			style:width={`${countingDown ? progress * 100 : 100}%`}
		></div>
		<div class="flex h-full w-full items-center justify-center font-bold text-neutral/80">
			{Math.round(countingDown ? progress * 100 : 100)} %
		</div>
	</div>

	<div
		class="absolute -z-20 flex h-full w-full flex-col items-center justify-center transition-colors duration-500 {countingDown
			? color
			: 'animate-pulse-bg'}"
	>
		<span
			class="countdown font-mono text-[20vw] font-bold text-white transition-colors duration-500 select-none"
		>
			<span
				style={`--value:${remainingHours}; --digits: 2;`}
				aria-live="polite"
				aria-label={remainingHours.toString()}>{remainingHours}</span
			>
			:
			<span
				style={`--value:${remainingMinutes}; --digits: 2;`}
				aria-live="polite"
				aria-label={remainingMinutes.toString()}>{remainingMinutes}</span
			>
			:
			<span
				style={`--value:${remainingSeconds}; --digits: 2;`}
				aria-live="polite"
				aria-label={remainingSeconds.toString()}>{remainingSeconds}</span
			>
		</span>
	</div>

	<div
		class="flex translate-y-20 flex-col items-center justify-end gap-4 rounded-t-2xl bg-neutral-200/60 p-4 opacity-20 transition-all delay-500 duration-300 hover:translate-y-0 hover:opacity-100 hover:shadow-2xl hover:delay-0"
	>
		<div class="flex flex-row flex-wrap justify-center gap-4">
			<div class="join overflow-hidden rounded-full">
				{#if running}
					<button class="btn join-item p-6 btn-soft btn-warning" onclick={stop}>
						<div class="flex flex-col items-center">
							<Icon icon="solar:sleeping-square-bold" class="text-xl" />
							Pause
						</div>
					</button>
				{:else}
					<button class="btn join-item p-6 btn-soft btn-accent" onclick={start}>
						<div class="flex flex-col items-center">
							<Icon icon="solar:play-circle-bold" class="text-xl" />
							Start
						</div>
					</button>
				{/if}
				<button class="btn join-item p-6 btn-soft btn-error" onclick={reset}>
					<div class="flex flex-col items-center">
						<Icon icon="solar:restart-square-bold" class="text-xl" />
						Reset
					</div>
				</button>
			</div>

			<form
				class="join overflow-hidden rounded-full"
				onsubmit={(e) => {
					e.preventDefault();
					if (timeSelector) {
						const [h, m, s] = timeSelector.value.split(':').map(Number);
						setTime(h, m, s);
					}
				}}
			>
				<input
					class="input input-lg join-item h-full rounded-s-full"
					type="time"
					step="1"
					bind:this={timeSelector}
					value="00:10:00"
				/>
				<button class="btn join-item p-6 btn-soft btn-accent" type="submit">Set</button>
			</form>
		</div>

		<div class="flex flex-row flex-wrap gap-2">
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(0, 5, 0)}
				>5 min</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(0, 10, 0)}
				>10 min</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(0, 15, 0)}
				>15 min</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(0, 20, 0)}
				>20 min</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(0, 30, 0)}
				>30 min</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(0, 45, 0)}
				>45 min</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(1, 0, 0)}
				>1 h</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(1, 15, 0)}
				>1:15 h</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(1, 30, 0)}
				>1:30 h</button
			>
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(2, 0, 0)}
				>2:00 h</button
			>
		</div>
	</div>
</div>
