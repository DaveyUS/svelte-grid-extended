<script lang="ts">
	import { createEventDispatcher, onMount } from 'svelte';

	import {
		coordinate2size,
		calcPosition,
		snapOnMove,
		snapOnResize,
		type SnapGridParams
	} from './utils/item';
	import { hasCollisions, getCollisions, getAvailablePosition } from './utils/grid';

	import type { LayoutItem, LayoutChangeDetail, Size, ItemSize } from './types';
	import { getGridContext } from './Grid.svelte';

	const dispatch = createEventDispatcher<{
		change: LayoutChangeDetail;
		previewchange: LayoutChangeDetail;
	}>();

	let gridParams = getGridContext();

	let classes: string | undefined = undefined;

	export { classes as class };

	/**
	 * Unique identifier of the item. Used to identify the item in collision checks.
	 * @default uuidv4
	 *
	 * TODO: crypto.randomUUID() is not supported in non ssl environments
	 */
	export let id: string = crypto.randomUUID();

	export let activeClass: string | undefined = undefined;

	export let previewClass: string | undefined = undefined;

	export let resizerClass: string | undefined = undefined;

	export let x: number;

	export let y: number;

	export let w = 1;

	export let h = 1;

	/**
	 * Minimum size of the item in Grid Units.
	 */
	export let min: Size = { w: 1, h: 1 };

	/**
	 * Maximum size of the item in Grid Units.
	 * If not provided, the item will be able to grow infinitely.
	 */
	export let max: Size | undefined = undefined;

	export let movable = true;

	export let resizable = true;

	let active = false;

	let left: number;

	let top: number;

	let width: number;

	let height: number;

	/**
	 * Item object that is used in `$gridParams.items`.
	 * When the item is mounted, it is registered in `$gridParams.items`.
	 * By default, all props are in sync with the item object.
	 * If item ref is used to change its size or position, the invalidate method should be called.
	 * That is used to support two way binding of item props.
	 * @see {invalidate}
	 * @example
	 * ```
	 * item.x = 10;
	 * item.invalidate();
	 * ```
	 * @example
	 * ```
	 * item.x = 10;
	 * item.y = 10;
	 * item.w = 2;
	 * item.h = 2;
	 * item.invalidate();
	 * ```
	 */
	let item = {
		id,
		x,
		y,
		w,
		h,
		min,
		max,
		movable,
		resizable,
		invalidate
	} as LayoutItem;

	$: item.x = x;
	$: item.y = y;
	$: item.w = w;
	$: item.h = h;
	$: item.min = min;
	$: item.max = max;
	$: item.movable = movable;
	$: item.resizable = resizable;
	$: item, invalidate();

	/**
	 * Updates svelte-components props behind that item. Should be called when the item
	 * changes its size or position by manipulation with item object.
	 */
	function invalidate() {
		({ x, y, w, h } = item);
		dispatch('change', { item });
		$gridParams.dispatch('change', { item });
		$gridParams.updateGrid();
	}

	onMount(() => {
		$gridParams.registerItem(item);
		return () => {
			$gridParams.unregisterItem(item);
		};
	});

	// reposition item on grid change
	$: if (!active && $gridParams.itemSize) {
		const newPosition = calcPosition(item, {
			itemSize: $gridParams.itemSize,
			gap: $gridParams.gap
		});
		left = newPosition.left;
		top = newPosition.top;
		width = newPosition.width;
		height = newPosition.height;
	}

	$: previewItem = { ...item };

	$: if (!active && item) {
		previewItem = item;
	}

	$: previewItem, dispatch('previewchange', { item: previewItem });

	function applyPreview() {
		item.x = previewItem.x;
		item.y = previewItem.y;
		item.w = previewItem.w;
		item.h = previewItem.h;
		item.invalidate();
	}

	function scroll(): void {
		// TODO: scroll
	}

	// INTERACTION LOGIC

	let itemRef: HTMLElement;

	const initialPointerPosition = { left: 0, top: 0 };

	function initInteraction(event: PointerEvent) {
		active = true;
		initialPointerPosition.left = event.pageX;
		initialPointerPosition.top = event.pageY;
		itemRef.setPointerCapture(event.pointerId);
	}

	function endInteraction(event: PointerEvent) {
		applyPreview();
		active = false;
		initialPointerPosition.left = 0;
		initialPointerPosition.top = 0;
		itemRef.releasePointerCapture(event.pointerId);
		$gridParams.updateGrid();
	}

	// MOVE ITEM LOGIC

	let initialPosition = { left: 0, top: 0 };

	$: _movable = !$gridParams.readOnly && movable;

	let pointerShift = { left: 0, top: 0 };

	function moveStart(event: PointerEvent) {
		if (event.button !== 0) return;
		event.stopPropagation();
		initInteraction(event);
		initialPosition = { left, top };
		pointerShift = { left: event.pageX - left, top: event.pageY - top };
		window.addEventListener('pointermove', move);
		window.addEventListener('pointerup', moveEnd);
	}

	function move(event: PointerEvent) {
		if (!$gridParams.itemSize) {
			throw new Error('Grid is not mounted yet');
		}
		let _left = event.pageX - initialPointerPosition.left + initialPosition.left;
		let _top = event.pageY - initialPointerPosition.top + initialPosition.top;

		if ($gridParams.bounds && $gridParams.boundsTo) {
			const parentRect = $gridParams.boundsTo.getBoundingClientRect();
			if (_left < parentRect.left) {
				_left = parentRect.left;
			}
			if (_top < parentRect.top) {
				_top = parentRect.top;
			}
			if (_left + width > parentRect.right) {
				_left = parentRect.right - width;
			}
			if (_top + height > parentRect.bottom) {
				_top = parentRect.bottom - height;
			}
		}

		left = _left;
		top = _top;

		if ($gridParams.collision === 'none') {
			scroll();
		}

		// TODO: throttle this, hasColisions is expensive
		{
			const { x, y } = snapOnMove(left, top, previewItem, $gridParams as SnapGridParams);

			if ($gridParams.collision !== 'none') {
				movePreviewWithCollisions(x, y);
			} else {
				if (!hasCollisions({ ...previewItem, x, y }, Object.values($gridParams.items))) {
					previewItem = { ...previewItem, x, y };
				}
			}
		}
	}

	function updateCollItemPositionWithPush(collItem: LayoutItem, items: LayoutItem[]) {
		const newPosition = getAvailablePosition(
			collItem,
			items,
			$gridParams.maxCols,
			$gridParams.maxRows
		);
		if (newPosition) {
			const { x, y } = newPosition;
			collItem.x = x;
			collItem.y = y;
			collItem.invalidate();
		}
		$gridParams.updateGrid();
	}

	function handleCollisionsForPreviewItemWithPush(newAttributes: {
		x?: number;
		y?: number;
		w?: number;
		h?: number;
	}) {
		const gridItems = Object.values($gridParams.items);
		const itemsExceptPreview = gridItems.filter((item) => item.id != previewItem.id);
		const collItems = getCollisions({ ...previewItem, ...newAttributes }, itemsExceptPreview);

		collItems.forEach((collItem: LayoutItem) => {
			const itemsExceptCollItem = gridItems.filter((item) => item.id != collItem.id);
			const items = [
				...itemsExceptCollItem.filter((item) => item.id != previewItem.id),
				{ ...previewItem, ...newAttributes }
			];
			updateCollItemPositionWithPush(collItem, items);
		});

		previewItem = { ...previewItem, ...newAttributes };
		$gridParams.updateGrid();
		applyPreview();
	}

	function movePreviewWithCollisionsWithPush(x: number, y: number) {
		handleCollisionsForPreviewItemWithPush({ x, y });
	}

	function movePreviewWithCollisionsWithCompress(x: number, y: number) {
		const gridItems = Object.values($gridParams.items);
		let newY = y;
		const itemsExceptPreview = gridItems.filter((item) => item.id != previewItem.id);
		while (newY >= 0) {
			const collItems = getCollisions({ ...previewItem, x, y: newY }, gridItems);
			if (collItems.length > 0) {
				const sortedItems = collItems.sort((a, b) => b.y - a.y);
				let moved = false;
				sortedItems.forEach((sortItem) => {
					//if you want to fix sensitivity of grid, change this statement
					if (y + previewItem.h / 2 >= sortItem.y + sortItem.h / 2) {
						moved = true;
						newY = sortItem.y + sortItem.h;
						sortedItems.forEach((item) => {
							if (
								!hasCollisions({ ...item, y: item.y - previewItem.h }, itemsExceptPreview) &&
								item.y - previewItem.h >= 0
							) {
								item.y -= previewItem.h;
								item.invalidate();
							}
						});
						return false;
					}
				});
				if (!moved) {
					newY = previewItem.y;
				}
				break;
			}
			newY--;
		}
		if (newY < 0 || y === 0) {
			newY = 0;
		}
		const positionChanged = x != previewItem.x || newY != previewItem.y;
		previewItem = { ...previewItem, x, y: newY };
		if (positionChanged) {
			compressItems();
			applyPreview();
		}
	}

	function movePreviewWithCollisions(x: number, y: number) {
		if ($gridParams.collision === 'compress') {
			movePreviewWithCollisionsWithCompress(x, y);
		} else {
			movePreviewWithCollisionsWithPush(x, y);
		}
	}

	function moveEnd(event: PointerEvent) {
		if (event.button !== 0) return;
		endInteraction(event);
		pointerShift = { left: 0, top: 0 };
		window.removeEventListener('pointermove', move);
		window.removeEventListener('pointerup', moveEnd);
	}

	// RESIZE ITEM LOGIC

	let initialSize = { width: 0, height: 0 };
	let resizeDirection = 'se'; // Default resize direction (southeast)

	// grid-coords of the edge that must stay fixed
	let originalGridRight = 0;
	let originalGridBottom = 0;

	let minSize: ItemSize | undefined;

	let maxSize: ItemSize | undefined;

	$: if ($gridParams.itemSize) {
		minSize = {
			width: coordinate2size(min.w, $gridParams.itemSize.width, $gridParams.gap),
			height: coordinate2size(min.h, $gridParams.itemSize.height, $gridParams.gap)
		};
	}

	$: if ($gridParams.itemSize && max) {
		maxSize = {
			width: coordinate2size(max.w, $gridParams.itemSize.width, $gridParams.gap),
			height: coordinate2size(max.h, $gridParams.itemSize.height, $gridParams.gap)
		};
	}

	$: _resizable = !$gridParams.readOnly && item.resizable;

	function resizeStart(event: PointerEvent) {
		if (event.button !== 0) return;
		event.stopPropagation();
		
		// Get the resize direction from the event if available
		// Using any to bypass TypeScript's type checking for custom properties
		const customEvent = event as any;
		if (customEvent.direction) {
			resizeDirection = customEvent.direction;
		}
		
		// remember fixed edges
		originalGridRight = item.x + item.w;
		originalGridBottom = item.y + item.h;
		
		initInteraction(event);
		initialSize = { width, height };
		initialPosition = { left, top };
		window.addEventListener('pointermove', resize);
		window.addEventListener('pointerup', resizeEnd);
	}

	function resize(event: PointerEvent) {
		if (!$gridParams.itemSize) {
			throw new Error('Grid is not mounted yet');
		}

		let newWidth, newHeight, newLeft, newTop;
		const deltaX = event.pageX - initialPointerPosition.left;
		const deltaY = event.pageY - initialPointerPosition.top;
		
		// Handle different resize directions
		switch (resizeDirection) {
			case 'se': // southeast (bottom-right)
				newWidth = initialSize.width + deltaX;
				newHeight = initialSize.height + deltaY;
				newLeft = initialPosition.left;
				newTop = initialPosition.top;
				break;
			case 'sw': // southwest (bottom-left)
				newWidth = initialSize.width - deltaX;
				newHeight = initialSize.height + deltaY;
				newLeft = initialPosition.left + deltaX;
				newTop = initialPosition.top;
				break;
			case 'ne': // northeast (top-right)
				newWidth = initialSize.width + deltaX;
				newHeight = initialSize.height - deltaY;
				newLeft = initialPosition.left;
				newTop = initialPosition.top + deltaY;
				break;
			case 'nw': // northwest (top-left)
				newWidth = initialSize.width - deltaX;
				newHeight = initialSize.height - deltaY;
				newLeft = initialPosition.left + deltaX;
				newTop = initialPosition.top + deltaY;
				break;
			default:
				newWidth = initialSize.width + deltaX;
				newHeight = initialSize.height + deltaY;
				newLeft = initialPosition.left;
				newTop = initialPosition.top;
		}
		
		// Apply minimum size constraints
		if (minSize) {
			if (newWidth < minSize.width) {
				if (['sw', 'nw'].includes(resizeDirection)) {
					newLeft = initialPosition.left + initialSize.width - minSize.width;
				}
				newWidth = minSize.width;
			}
			if (newHeight < minSize.height) {
				if (['ne', 'nw'].includes(resizeDirection)) {
					newTop = initialPosition.top + initialSize.height - minSize.height;
				}
				newHeight = minSize.height;
			}
		}
		
		// Apply maximum size constraints
		if (maxSize) {
			if (newWidth > maxSize.width) {
				if (['sw', 'nw'].includes(resizeDirection)) {
					newLeft = initialPosition.left + initialSize.width - maxSize.width;
				}
				newWidth = maxSize.width;
			}
			if (newHeight > maxSize.height) {
				if (['ne', 'nw'].includes(resizeDirection)) {
					newTop = initialPosition.top + initialSize.height - maxSize.height;
				}
				newHeight = maxSize.height;
			}
		}
		
		// Apply bounds constraints
		if ($gridParams.bounds && $gridParams.boundsTo) {
			const parentRect = $gridParams.boundsTo.getBoundingClientRect();
			if (newLeft < parentRect.left) {
				newWidth = initialPosition.left + initialSize.width - parentRect.left;
				newLeft = parentRect.left;
			}
			if (newTop < parentRect.top) {
				newHeight = initialPosition.top + initialSize.height - parentRect.top;
				newTop = parentRect.top;
			}
			if (newLeft + newWidth > parentRect.right) {
				newWidth = parentRect.right - newLeft;
			}
			if (newTop + newHeight > parentRect.bottom) {
				newHeight = parentRect.bottom - newTop;
			}
		}
		
		// Update position and size
		left = newLeft;
		top = newTop;
		width = newWidth;
		height = newHeight;

		if ($gridParams.collision === 'none') {
			scroll();
		}

		/* -----------------------------------------------------------
		   SNAP LOGIC  – order matters!
		   1. snap size ➜ 2. snap position with new size ➜ 3. lock edges
		----------------------------------------------------------- */

		// 1️⃣  snap raw pixel size to grid units
		let { w: snappedW, h: snappedH } = snapOnResize(
			width,
			height,
			previewItem,
			$gridParams as SnapGridParams
		);

		// 2️⃣  compute x / y using the UPDATED size, but only for handles that change position
		let x = previewItem.x;
		let y = previewItem.y;
		
		// Only recalculate position if the handle actually moves the top-left corner
		if (resizeDirection.includes('w') || resizeDirection.includes('n')) {
			const tempItem = { ...previewItem, w: snappedW, h: snappedH };
			const snappedPosition = snapOnMove(left, top, tempItem, $gridParams as SnapGridParams);
			x = snappedPosition.x;
			y = snappedPosition.y;
		}

		// 3️⃣  keep right / bottom edges fixed when using W* / N* handles
		let w = snappedW;
		let h = snappedH;

		// Only lock edges when resizing from specific directions
		if (resizeDirection.includes('w')) {
			// When resizing from west (left), keep the right edge fixed
			w = Math.max(1, originalGridRight - x);
		}
		if (resizeDirection.includes('n')) {
			// When resizing from north (top), keep the bottom edge fixed
			h = Math.max(1, originalGridBottom - y);
		}
		
		// honour min / max constraints in grid units
		w = Math.max(w, min.w);
		h = Math.max(h, min.h);
		if (max) {
			w = Math.min(w, max.w);
			h = Math.min(h, max.h);
		}
		
		// Update preview item
		if ($gridParams.collision !== 'none') {
			resizePreviewWithCollisions(w, h);
		} else {
			if (!hasCollisions({ ...previewItem, x, y, w, h }, Object.values($gridParams.items))) {
				previewItem = { ...previewItem, x, y, w, h };
			}
		}
	}

	function resizePreviewWithCollisionsWithPush(w: number, h: number) {
		const { x, y } = snapOnMove(left, top, previewItem, $gridParams as SnapGridParams);
		handleCollisionsForPreviewItemWithPush({ x, y, w, h });
	}

	function resizePreviewWithCollisionsWithCompress(w: number, h: number) {
		const { x, y } = snapOnMove(left, top, previewItem, $gridParams as SnapGridParams);
		const positionOrSizeChanged = w != previewItem.w || h != previewItem.h || x != previewItem.x || y != previewItem.y;
		
		if (positionOrSizeChanged) {
			const hGap = h - previewItem.h;
			previewItem = { ...previewItem, x, y, w, h };
			applyPreview();
			const collItems = getCollisions(
				{ ...previewItem, w, h: 9999 },
				Object.values($gridParams.items)
			);
			collItems.forEach((item) => {
				item.y += hGap;
				item.invalidate();
				$gridParams.updateGrid();
			});
			compressItems();
		}
	}

	function resizePreviewWithCollisions(w: number, h: number) {
		if ($gridParams.collision === 'compress') {
			resizePreviewWithCollisionsWithCompress(w, h);
		} else {
			resizePreviewWithCollisionsWithPush(w, h);
		}
	}

	function resizeEnd(event: PointerEvent) {
		if (event.button !== 0) return;
		endInteraction(event);
		window.removeEventListener('pointermove', resize);
		window.removeEventListener('pointerup', resizeEnd);
	}

	function compressItems() {
		const gridItems = Object.values($gridParams.items);
		const sortedItems = [...gridItems].sort((a, b) => a.y - b.y);
		sortedItems.reduce(
			(accItem, currentItem) => {
				if (currentItem.id === previewItem.id) {
					//if previewItem do nothing
				} else if (previewItem.y < currentItem.y + currentItem.h) {
					//compress items above previewItem
					const maxY =
						currentItem.y >= previewItem.y
							? currentItem.y + previewItem.h + 1
							: previewItem.y + currentItem.h + 1;
					let newY = maxY;
					while (newY >= 0) {
						if (hasCollisions({ ...currentItem, y: newY }, accItem)) {
							break;
						}
						newY--;
					}
					currentItem.y = newY + 1;
					currentItem.invalidate();
					$gridParams.updateGrid();
					accItem.push(currentItem);
				} else {
					//compress items below previewItem
					let newY = currentItem.y;
					while (newY >= 0) {
						if (hasCollisions({ ...currentItem, y: newY }, accItem)) {
							break;
						}
						newY--;
					}
					if (newY < currentItem.y && newY > 0) {
						currentItem.y = newY + 1;
					}
					currentItem.invalidate();
					$gridParams.updateGrid();
					accItem.push(currentItem);
				}
				return accItem;
			},
			[previewItem]
		);
	}
</script>

<div
	class={`${classes} ${active ? activeClass : ''}`}
	class:item-default={!classes}
	class:active-default={!activeClass && active}
	class:non-active-default={!active}
	on:pointerdown={_movable && !$$slots.moveHandle ? moveStart : null}
	style={`position: absolute; left:${left}px; top:${top}px; width: ${width}px; height: ${height}px; 
			${_movable && !$$slots.moveHandle ? 'cursor: move;' : ''} touch-action: none; user-select: none;
			${$$restProps.style ?? ''}`}
	bind:this={itemRef}
>
	{#if _movable}
		<slot name="moveHandle" {moveStart} />
	{/if}

	<slot {id} {active} {w} {h} />

	{#if _resizable}
		<slot name="resizeHandle" {resizeStart}>
			<div
				class={resizerClass}
				class:resizer-default={!resizerClass}
				on:pointerdown={resizeStart}
			/>
		</slot>
	{/if}
</div>

{#if active && $gridParams.itemSize}
	{@const preview = calcPosition(previewItem, {
		itemSize: $gridParams.itemSize,
		gap: $gridParams.gap
	})}
	<div
		class={previewClass ?? ''}
		class:item-preview-default={!previewClass}
		style={`position: absolute; left:${preview.left}px; top:${preview.top}px;  
		width: ${preview.width}px; height: ${preview.height}px; z-index: 20;
		transition: left 0.25s cubic-bezier(0.2, 0.8, 0.2, 1), top 0.25s cubic-bezier(0.2, 0.8, 0.2, 1), 
		width 0.25s cubic-bezier(0.2, 0.8, 0.2, 1), height 0.25s cubic-bezier(0.2, 0.8, 0.2, 1);`}
	/>
{/if}

<style>
	.item-default {
		transition:
			width 0.2s,
			height 0.2s;
		transition:
			transform 0.2s,
			opacity 0.2s;
	}
	.active-default {
		opacity: 0.7;
	}
	.item-preview-default {
		background-color: rgb(192, 127, 127);
		transition: all 0.3s ease-out;
		opacity: 0.6;
		border-radius: 4px;
		box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
	}
	.non-active-default {
		transition:
			left 0.2s,
			top 0.2s;
		transition-timing-function: ease-in-out;
	}
	.resizer-default {
		user-select: none;
		touch-action: none;
		position: absolute;
		user-select: none;
		width: 20px;
		height: 20px;
		right: 0;
		bottom: 0;
		cursor: se-resize;
	}
	.resizer-default::after {
		content: '';
		position: absolute;
		right: 3px;
		bottom: 3px;
		width: 5px;
		height: 5px;
		border-right: 2px solid rgba(0, 0, 0, 0.4);
		border-bottom: 2px solid rgba(0, 0, 0, 0.4);
	}
</style>
