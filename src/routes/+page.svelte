<script lang="ts">
	import Icon from '@iconify/svelte';
	import { onMount } from 'svelte';

	let endTime: number | undefined;
	let totalSeconds = $state(0);

	let remaining = $derived(Math.max((endTime! - Date.now()) / 1000, 0));
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
	let wakeLock: WakeLockSentinel | null = $state(null);
	let message = $state('');

	async function start() {
		if (interval) return;
		running = true;

		try {
			wakeLock = await navigator.wakeLock.request('screen');
			wakeLock.addEventListener('release', () => {
				wakeLock = null;
			});
		} catch (err) {
			console.log(err);
		}

		if (!endTime) {
			countingDown = remaining > 0;
			endTime = Date.now() + remaining * 1000;
		} else {
			endTime = Date.now() + remaining * 1000;
		}

		interval = setInterval(() => {
			if (countingDown) {
				remaining = Math.max((endTime! - Date.now()) / 1000, 0);
				if (remaining <= 0) countingDown = false;
			} else {
				remaining = (Date.now() - endTime!) / 1000;
			}
		}, 10);
	}

	async function stop() {
		running = false;
		await wakeLock?.release();
		if (interval) {
			clearInterval(interval);
			interval = undefined;
		}
		endTime = endTime ? Date.now() + remaining * 1000 : undefined;
	}

	function reset() {
		stop();
		remaining = totalSeconds;
		color = 'bg-success';
		countingDown = true;
		endTime = undefined;
	}

	function setTime(h: number = 0, m: number = 0, s: number = 0) {
		totalSeconds = h * 3600 + m * 60 + s;
		reset();
	}

	function handleKeyboard(event: KeyboardEvent) {
		const target = event.target as HTMLElement | null;

		// if the event came from an input/textarea/contenteditable, do nothing
		if (target) {
			const tag = target.tagName;
			const isTextInput =
				tag === 'TEXTAREA' ||
				(tag === 'INPUT' &&
					(target as HTMLInputElement).type &&
					['text', 'search', 'email', 'password', 'tel', 'url'].includes(
						(target as HTMLInputElement).type
					)) ||
				(target as HTMLElement).isContentEditable;

			if (isTextInput) return; // <-- bypass the global handler
		}

		if (event.code === 'Space') {
			event.preventDefault();
			if (running) stop();
			else start();
		} else if (event.code === 'Backspace') {
			reset();
		} else if (event.code === 'ArrowUp') {
			modifyRemaining(10);
		} else if (event.code === 'ArrowDown') {
			modifyRemaining(-10);
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
		color = countingDown
			? progress >= 0.8
				? 'bg-error'
				: progress >= 0.6
					? 'bg-warning'
					: 'bg-success'
			: color;
	});

	function modifyRemaining(seconds: number) {
		remaining = Math.max(remaining + seconds, 0);
		endTime = endTime ? Date.now() + remaining * 1000 : undefined;
	}

	onMount(() => {
		setTime(0, 10, 0);
	});
</script>

<svelte:window onkeydown={handleKeyboard} onmousemove={hideCursor} />

<svelte:head>
	<title
		>Timer: {remainingHours.toString().padStart(2, '0')}:{remainingMinutes
			.toString()
			.padStart(2, '0')}:{remainingSeconds.toString().padStart(2, '0')}</title
	>
</svelte:head>

<div class="relative flex h-svh flex-col items-center justify-between overflow-hidden">
	<!-- progress bar -->
	<div class="relative -z-20 h-8 w-full bg-white/40">
		<div
			class="absolute top-0 left-0 -z-10 h-full bg-white/80"
			style:width={`${countingDown ? progress * 100 : 100}%`}
		></div>
		<div class="flex h-full w-full items-center justify-center font-bold text-neutral/80">
			{Math.round(countingDown ? progress * 100 : 100)} %
		</div>
	</div>

	<!-- side panels -->
	<button
		class="absolute top-1/2 left-0 z-10 flex h-1/2 w-20 -translate-y-1/2 cursor-pointer items-center justify-center rounded-e-2xl bg-white/40 p-4 font-bold text-accent-content opacity-0 backdrop-blur-3xl transition-all duration-300 hover:opacity-90 hover:shadow-2xl"
		onclick={() => modifyRemaining(-10)}
	>
		-10s
	</button>
	<button
		class="absolute top-1/2 right-0 z-10 flex h-1/2 w-20 -translate-y-1/2 cursor-pointer items-center justify-center rounded-s-2xl bg-white/40 p-4 font-bold text-accent-content opacity-0 backdrop-blur-3xl transition-all duration-300 hover:opacity-90 hover:shadow-2xl"
		onclick={() => modifyRemaining(10)}
	>
		+10s
	</button>

	<!-- counter -->
	<div
		class="absolute -z-100 flex h-full w-full flex-col items-center justify-center transition-colors duration-500 {countingDown
			? color
			: 'animate-pulse-bg'}"
	>
		<span
			class="countdown font-mono text-[20vw] font-bold transition-colors duration-500 select-none"
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
		{#if message}
			<div
				class="w-full border-4 glass bg-error text-center text-[7vw] leading-tight font-extrabold whitespace-pre-wrap"
			>
				{message}
			</div>
		{/if}
	</div>

	<!-- bottom panel -->
	<div
		class="group flex translate-y-39 flex-col items-center justify-end gap-4 rounded-t-2xl bg-white/40 p-4 opacity-40 shadow-2xl transition-all delay-1000 duration-300 hover:translate-y-0 hover:opacity-90 hover:delay-0"
	>
		<div
			class="flex flex-row flex-wrap justify-center gap-4 blur-xs delay-1000 group-hover:blur-none group-hover:delay-0"
		>
			<form
				class="join"
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
				<button class="btn join-item rounded-e-full p-7 btn-soft btn-lg btn-accent" type="submit"
					>Set</button
				>
			</form>

			<div class="join">
				{#if running}
					<button
						class="btn join-item rounded-s-full p-7 btn-soft btn-lg btn-warning"
						onclick={stop}
					>
						<div class="flex flex-col items-center">
							<Icon icon="solar:sleeping-square-bold" class="text-xl" />
							Pause
						</div>
					</button>
				{:else}
					<button
						class="btn join-item rounded-s-full p-7 btn-soft btn-lg btn-accent"
						onclick={start}
					>
						<div class="flex flex-col items-center">
							<Icon icon="solar:square-arrow-right-bold" class="text-xl" />
							Start
						</div>
					</button>
				{/if}
				<button class="btn join-item rounded-e-full p-7 btn-soft btn-lg btn-error" onclick={reset}>
					<div class="flex flex-col items-center">
						<Icon icon="solar:restart-square-bold" class="text-xl" />
						Reset
					</div>
				</button>
			</div>
		</div>

		<div class="flex flex-row flex-wrap gap-2">
			<button class="btn rounded-full btn-dash btn-neutral" onclick={() => setTime(0, 1, 0)}
				>1 min</button
			>
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

		<textarea class="textarea w-full rounded-2xl" bind:value={message}>{message}</textarea>
	</div>
</div>
