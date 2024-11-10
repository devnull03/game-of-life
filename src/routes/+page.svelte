<!-- 
 Initially, there is a grid with some cells which may be alive or dead. 
 Our task is to generate the next generation of cells based on the following rules: 

 1. Any live cell with fewer than two live neighbors dies as if caused by underpopulation.
 2. Any live cell with two or three live neighbors lives on to the next generation.
 3. Any live cell with more than three live neighbors dies, as if by overpopulation.
 4. Any dead cell with exactly three live neighbors becomes a live cell, as if by reproduction. 
-->

<script lang="ts">
	import { SvelteSet } from 'svelte/reactivity';
	import { gsap } from 'gsap';
	import Draggable from 'gsap/dist/Draggable';
	import { onMount } from 'svelte';

	gsap.registerPlugin(Draggable);

	let BoardLimit = $state(100);
	let baordUpdateStep = 15;
	let simulationRunning = $state(false);

	type Cell = [number, number];

	// class AliveCellSetWrapper {
	// 	// coords stored in the form of a string that has
	// 	// two integers seperated by a string
	// 	public aliveSet = $state(new SvelteSet<string>());
	// 	constructor(...args: Cell[]) {
	// 		this.aliveSet = new SvelteSet<string>(args.map((e) => AliveCellSetWrapper.listToString(e)));
	// 	}

	// 	has(value: Cell) {
	// 		return this.aliveSet.has(AliveCellSetWrapper.listToString(value));
	// 	}

	// 	add(value: Cell) {
	// 		return this.aliveSet.add(AliveCellSetWrapper.listToString(value));
	// 	}

	// 	delete(value: Cell) {
	// 		return this.aliveSet.delete(AliveCellSetWrapper.listToString(value));
	// 	}

	// 	public static stringToList(e: string) {
	// 		let splitText = e.split(',');
	// 		return [parseInt(splitText[0]), parseInt(splitText[1])];
	// 	}

	// 	public static listToString(e: Cell) {
	// 		return `${e[0]},${e[1]}`;
	// 	}
	// }

	// let aliveCells = new AliveCellSetWrapper();
	// $inspect(aliveCells.aliveSet);

	let aliveCells = $state(new SvelteSet<string>());
	$inspect(aliveCells);

	function toString(e: Cell): string {
		return `${e[0]},${e[1]}`;
	}

	function stringToList(e: string): Cell {
		let splitText = e.split(',');
		return [parseInt(splitText[0]), parseInt(splitText[1])];
	}

	const calculateNextCellState = (isAlive: boolean, neighbors: number): boolean => {
		// Calculates state of cell based on initial state and number of neighbours
		if (isAlive) return neighbors >= 2 && neighbors <= 3;
		else return neighbors === 3;
	};

	const calculateNeghbors = (
		cell: Cell
	): {
		aliveNeighborCount: number;
		neighborCells: Cell[];
		neighborCellsAliveCheck: {
			[any: string]: boolean;
		};
	} => {
		if (BoardLimit - Math.abs(cell[0]) < 0) {
			throw new Error('Cell x cordinate exceeds board limit');
			// BoardLimit += baordUpdateStep;
			// console.log(`Cell x cordinate exceeds board limit, new board limit: ${BoardLimit}`);
		}
		if (BoardLimit - Math.abs(cell[1]) < 0) {
			throw new Error('Cell y cordinate exceeds board limit');
			// BoardLimit += baordUpdateStep;
			// console.log(`Cell y cordinate exceeds board limit, new board limit: ${BoardLimit}`);
		}

		let neighborCells: Cell[] = [
			[cell[0] + 1, cell[1] - 1],
			[cell[0] + 1, cell[1]],
			[cell[0] + 1, cell[1] + 1],
			[cell[0], cell[1] - 1],
			[cell[0], cell[1] + 1],
			[cell[0] - 1, cell[1] - 1],
			[cell[0] - 1, cell[1]],
			[cell[0] - 1, cell[1] + 1]
		];
		let aliveNeighborCount = 0;
		let neighborCellsAliveCheck: {
			[any: string]: boolean;
		} = {};

		for (let i = 0; i < neighborCells.length; i++) {
			let strCell = toString(neighborCells[i]);
			if (aliveCells.has(strCell)) {
				neighborCellsAliveCheck[strCell] = true;
				aliveNeighborCount++;
			} else neighborCellsAliveCheck[strCell] = false;
		}

		return { aliveNeighborCount, neighborCells, neighborCellsAliveCheck };
	};

	let globalAnimationID: number;
	let startTime, now, elapsed;
	let then: number;
	let fps = $state(3);
	let fpsInterval = $derived(1000 / fps);
	let fpsStep = 5;

	const animation = (newtime: number) => {
		globalAnimationID = requestAnimationFrame(animation);

		now = newtime;
		elapsed = now - then;

		// if enough time has elapsed, draw the next frame

		if (elapsed > fpsInterval) {
			// Get ready for next frame by setting then=now, but...
			// Also, adjust for fpsInterval not being multiple of 16.67
			then = now - (elapsed % fpsInterval);

			if (!simulationRunning) return;

			let newState = new SvelteSet<string>();
			aliveCells.forEach((strCell) => {
				let cell = stringToList(strCell);
				let { aliveNeighborCount, neighborCells, neighborCellsAliveCheck } =
					calculateNeghbors(cell);
				if (calculateNextCellState(true, aliveNeighborCount)) newState.add(strCell);

				for (let i = 0; i < neighborCells.length; i++) {
					let sub = calculateNeghbors(neighborCells[i]);
					if (
						calculateNextCellState(
							Object.values(neighborCellsAliveCheck)[i],
							sub.aliveNeighborCount
						)
					)
						newState.add(Object.keys(neighborCellsAliveCheck)[i]);
				}
			});
			aliveCells = newState;
		}
	};

	onMount(() => {
		Draggable.create('#board', {});
		then = window?.performance.now();
	});
</script>

<div class="fixed bottom-4 right-4 z-[999] flex flex-row items-center gap-2">
	<button
		class="flex aspect-square h-12 items-center justify-center rounded-full border bg-gray-200 text-lg"
		onclick={() => {
			if (fps - fpsStep <= 3) fps = 3;
			else fps -= fpsStep;
		}}
	>
		{'<<'}
	</button>

	<button
		class="flex aspect-square h-12 items-center justify-center rounded-full border bg-gray-200 p-4 text-lg"
		onclick={() => {
			fps += fpsStep;
		}}
	>
		{'>>'}
	</button>

	<button
		class="flex aspect-square items-center justify-center rounded-full border bg-gray-200 p-4 text-lg"
		onclick={() => {
			if (simulationRunning) {
				cancelAnimationFrame(globalAnimationID);
				simulationRunning = false;
			} else {
				globalAnimationID = requestAnimationFrame(animation);
				simulationRunning = true;
			}
		}}
	>
		{simulationRunning ? 'pause' : 'play'}
	</button>
</div>
<main class="flex -translate-y-1/3 items-center justify-center bg-gray-100">
	<div
		id="board"
		class="board grid aspect-square min-h-[150vw] min-w-[150vw] grid-cols-[repeat(100,_minmax(0,_1fr))] grid-rows-[repeat(100,_minmax(0,_1fr))] *:border"
	>
		{#each Array(BoardLimit) as _, idy}
			{#each Array(BoardLimit) as _, idx (idy * BoardLimit + idx)}
				{@const coords: Cell = [idx, idy]}
				{@const stringCoords: string = toString([idx, idy])}
				<!-- svelte-ignore a11y_click_events_have_key_events -->
				<!-- svelte-ignore a11y_no_static_element_interactions -->
				<div
					class="select-none {aliveCells.has(stringCoords) && 'bg-yellow-400'}"
					onclick={(e) => {
						aliveCells.has(stringCoords)
							? aliveCells.delete(stringCoords)
							: aliveCells.add(stringCoords);
					}}
				>
					&nbsp;
				</div>
			{/each}
		{/each}
	</div>
</main>
