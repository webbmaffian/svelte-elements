<script module>
	const itemValueKeys = ['id', 'value'];
	const itemLabelKeys = ['label', 'name', 'value'];

	/** @typedef {(item: any) => string} itemValue */
	/** @typedef {(item: any, allItems: any[], itemValue: itemValue) => string} itemLabel */
	/** @typedef {(value: string | string[]) => Promise<any | any[]>} resolveSelectedItems */
	/** @typedef {(search: string) => Promise<any[]>} getter */

	/** @type {itemValue} */
	function getItemValue(item) {
		for (const key of itemValueKeys) {
			if (typeof item?.[key] !== 'undefined') {
				return item?.[key];
			}
		}

		//throw error
		return item;
	}

	/** @type {itemLabel} */
	function getItemLabel(item, items, itemValue) {
		for (const key of itemLabelKeys) {
			if (typeof item?.[key] !== 'undefined') {
				return item?.[key];
			}
		}

		const it = items.find((it) => itemValue(it) === item);

		if (it && it !== item) {
			return getItemLabel(it, items, itemValue);
		}

		//throw error
		return item;
	}

	/**
	 * @param {any[]} items
	 * @param {(item: any) => string} itemLabel
	 * @returns {getter}
	 */
	function arrayGetter(items, itemLabel) {
		return (search) => {
			search = search.toLowerCase();
			return new Promise((resolve) => {
				resolve(items.filter((item) => itemLabel(item).toLowerCase().indexOf(search) !== -1));
			});
		};
	}
</script>

<script>
	import { offset, flip, platform, computePosition, autoUpdate } from '@floating-ui/dom';

	import { ChevronDown, X } from '@lucide/svelte';
	import { tick } from 'svelte';
	/** @type {{
     * id: string;
     * placeholder?: string;
     * items: any[] | getter;
     * multiple?: boolean;
     * selected?: string | string[];
     * clearable?: boolean;
     * createable?: boolean;
     * disabled?: boolean;
     * required?: boolean;
     * class?: string | string[];
     * dropdownClass?: string;
     * createPrefix?: string;
     * emptyLabel?: string;
     * clearOnFocus?: boolean;
     * itemValue?: itemValue;
     * itemLabel?: (item: any) => string;
     * itemSnippet?: import('svelte').Snippet<any>;
     * resolveSelectedItems?: resolveSelectedItems;
     * onchange: (item: any[] | any) => void;
	 * options?: { offset: import('@floating-ui/dom').OffsetOptions, flip: import('@floating-ui/dom').FlipOptions };
    }} */
	let {
		id,
		placeholder = '',
		items: rawItems,
		multiple = false,
		selected: selectedIds = multiple ? [] : '',
		clearable = false,
		createable = false,
		disabled = false,
		required = false,
		class: className = '',
		dropdownClass = '',
		createPrefix = 'Create',
		emptyLabel = 'No items found',
		clearOnFocus = !multiple,
		itemValue = getItemValue,
		itemLabel = (item) => getItemLabel(item, items, itemValue),
		itemSnippet = snippet,
		resolveSelectedItems = getResolveSelectedItems,
		onchange,
		options = {
			offset: 8,
			flip: {
				crossAxis: false,
				padding: 16
			}
		}
	} = $props();

	/** @type {resolveSelectedItems} */
	function getResolveSelectedItems(value) {
		return new Promise((resolve) => {
			if (Array.isArray(rawItems)) {
				if (Array.isArray(value)) {
					resolve(rawItems.filter((item) => value.includes(itemValue(item))));
				} else {
					resolve(rawItems.find((item) => itemValue(item) === value));
				}
			}

			//throw error
			resolve(value);
		});
	}

	/** @type {HTMLElement} */
	let wrapperElement;
	/** @type {HTMLInputElement} */
	let inputElement;
	/** @type {HTMLElement?} */
	let dropdownElement = $state(null);
	/** @type {ReturnType<typeof autoUpdate> | undefined} */
	let stopAutoUpdatePosition = $state();
	/** @type {boolean} */
	let open = $state(false);
	/** @type {string} */
	let search = $state('');
	/** @type {any[]} */
	let items = $state([]);
	/** @type {any[]} */
	let nonSelectedItems = $derived(items.filter((item) => !itemSelected(item)));
	/** @type {any[] | any} */
	let selectedItem = $state(multiple ? [] : null);
	$effect(() => {
		resolveSelectedItems(selectedIds).then((res) => {
			selectedItem = res;
		});
	});
	// /** @type {import('$lib/reactive-fetch.svelte').Result<any[] | any, Error, false>} */
	// let selectedItem = $derived.by(() => {
	// 	const state = fetch(() => resolveSelectedItems(rawSelected), multiple ? [] : null);

	// 	return state;
	// });
	/** @type {any[] | any} */
	let tempSelected = multiple ? [] : null;
	/** @type {number} */
	let targetIndex = $state(-1);

	const getter = $derived(Array.isArray(rawItems) ? arrayGetter(rawItems, itemLabel) : rawItems);
	// const isCreateable = fetch(async () => {
	// 	if (!createable || search.trim() === '') {
	// 		return false;
	// 	}

	// 	const items = await getter(search);

	// 	return !items.find((item) => item === search);
	// }, false);
	/** @type {boolean} */
	let isCreateable = $state(false);
	$effect(() => {
		if (!createable || search.trim() === '') {
			isCreateable = false;
		} else {
			getter(search).then((items) => (isCreateable = !items.find((item) => item === search)));
		}
	});
	const canOpen = $derived(nonSelectedItems.length > 0 || !!emptyLabel || isCreateable);

	function updatePosition() {
		if (!dropdownElement) return;
		computePosition(wrapperElement, dropdownElement, {
			platform,
			placement: 'bottom-start',
			middleware: [
				offset(options.offset),
				flip({
					crossAxis: options.flip.crossAxis,
					padding: options.flip.padding
				})
			]
		}).then(({ y }) => {
			if (dropdownElement) {
				dropdownElement.style = `top: ${y}px;`;
			}
		});
	}

	async function openDropdown() {
		if (clearOnFocus) {
			tempSelected = selectedItem;
			selectedItem = multiple ? [] : null;
		}

		updateDropdown();
	}

	async function updateDropdown() {
		items = await getter(search);

		open = canOpen && true;

		await tick();

		if (!open || !dropdownElement) {
			return;
		}

		stopAutoUpdatePosition = autoUpdate(wrapperElement, dropdownElement, updatePosition);
	}

	// /** @type {import('svelte/elements').FocusEventHandler<HTMLInputElement>} */
	function closeDropdown() {
		if (
			clearOnFocus && tempSelected && Array.isArray(selectedItem)
				? selectedItem.length === 0
				: !selectedItem
		) {
			selectedItem = tempSelected;
		}
		open = false;
		targetIndex = -1;
		search = '';

		if (stopAutoUpdatePosition) {
			stopAutoUpdatePosition();
			stopAutoUpdatePosition = undefined;
		}
	}

	/** @param {any} newItem  */
	async function createItem(newItem) {
		if (typeof newItem !== 'string' || newItem?.trim() === '') return;

		for (const item of items) {
			if (typeof item !== 'string') return;
		}

		items.push(newItem);
		search = '';
		inputElement.value = '';
		// items = await getter();

		selectItem(newItem);
	}

	/** @param {any} item */
	function selectItem(item) {
		if (multiple) {
			if (!Array.isArray(selectedItem)) {
				selectedItem = [item];
			} else if (!itemSelected(item)) {
				selectedItem.push(item);
			}
		} else {
			selectedItem = item;
			inputElement.blur();
		}
		search = '';
		onchange(selectedItem);
	}
	/**
	 * @param {any} item
	 * @returns {boolean}
	 */
	function itemSelected(item) {
		if (!selectedItem) {
			return false;
		}

		const value = itemValue(item);

		if (!Array.isArray(selectedItem)) {
			return itemValue(selectedItem) === value;
		}

		return selectedItem.find((it) => itemValue(it) === value);
	}

	/** @param {any} item  */
	function deselectArrayItem(item) {
		if (!multiple || !Array.isArray(selectedItem)) {
			return;
		}

		const value = itemValue(item);
		const idx = selectedItem.findIndex((it) => itemValue(it) === value);

		if (idx !== -1) {
			selectedItem.splice(idx, 1);
		}

		onchange(selectedItem);
	}

	function deselectAll() {
		selectedItem = multiple ? [] : null;
		search = '';
		onchange(selectedItem);
	}

	// input
	/** @type {import('svelte/elements').KeyboardEventHandler<HTMLInputElement>} */
	function onkeydown(e) {
		switch (e.key) {
			case 'Escape':
				e.preventDefault();
				closeDropdown();
				break;
			case 'ArrowUp':
				e.preventDefault();
				if (!open) openDropdown();
				targetPrevItem();
				if (dropdownElement?.children)
					dropdownElement.scrollTo(
						0,
						/** @type {HTMLElement?} */ (dropdownElement.children[targetIndex])?.offsetTop ?? 0
					);
				break;
			case 'ArrowDown':
				e.preventDefault();
				if (!open) openDropdown();
				targetNextItem();
				if (dropdownElement?.children)
					dropdownElement.scrollTo(
						0,
						/** @type {HTMLElement?} */ (dropdownElement.children[targetIndex])?.offsetTop ?? 0
					);
				break;
			case 'Enter':
				e.preventDefault();
				if ((targetIndex === -1 || targetIndex === nonSelectedItems.length) && isCreateable) {
					createItem(search);
				} else {
					const selectedTargetItem = nonSelectedItems[targetIndex];

					if (selectedTargetItem) {
						selectItem(selectedTargetItem);

						if (targetIndex >= nonSelectedItems.length - 1) {
							targetIndex = 0;
						}
					}
				}
				break;
			case 'Backspace':
				if (inputElement.value === '' && Array.isArray(selectedItem)) {
					deselectArrayItem(selectedItem[selectedItem.length - 1]);
				}
				break;
			default:
				return;
		}
	}
	function targetPrevItem() {
		if (targetIndex <= 0) {
			if (isCreateable) {
				return targetItem(nonSelectedItems.length);
			}

			return targetItem(nonSelectedItems.length - 1);
		}

		targetItem(targetIndex - 1);
	}

	function targetNextItem() {
		if (targetIndex === nonSelectedItems.length - 1) {
			if (isCreateable) {
				return targetItem(nonSelectedItems.length);
			}
		}

		if (targetIndex >= nonSelectedItems.length - 1) {
			return targetItem(0);
		}

		targetItem(targetIndex + 1);
	}

	/** @param {number} index  */
	function targetItem(index) {
		targetIndex = index;
	}

	/** @type {import('svelte/elements').FormEventHandler<HTMLInputElement>} */
	async function onsearch(e) {
		search = e.currentTarget.value.trim();
		updateDropdown();
	}

	// dropdownElement
</script>

{#snippet snippet(/** @type {any} */ item)}
	{itemLabel(item)}
{/snippet}

<article
	bind:this={wrapperElement}
	class={[
		'svelte-elements-select relative',
		disabled && 'disabled pointer-events-none opacity-60',
		className
	]}
>
	<label
		for={id}
		class="group outline-[] flex h-full w-full cursor-text flex-wrap items-center gap-1 rounded-sm bg-[var(--wme-dropdown-input-background-color)] p-1 outline-[var(--wme-dropdown-input-outline-color)] focus-within:outline-gray-700"
	>
		{#if multiple && Array.isArray(selectedItem)}
			<ul class="contents">
				{#each selectedItem as item}
					<li class="contents">
						<button
							onclick={() => deselectArrayItem(item)}
							class="flex cursor-pointer items-center gap-1 rounded-sm bg-gray-300 px-1 py-0.5 text-sm whitespace-nowrap"
						>
							{@render itemSnippet(item)}
							<X size="16" />
						</button>
					</li>
				{/each}
			</ul>
		{/if}

		<span class="grid flex-1 basis-48">
			<input
				{id}
				bind:this={inputElement}
				type="text"
				autocomplete="off"
				{placeholder}
				value={!multiple && selectedItem ? itemLabel(selectedItem) : ''}
				{onkeydown}
				onfocus={openDropdown}
				onblur={closeDropdown}
				oninput={onsearch}
				{required}
				{disabled}
				class={[
					'col-span-full row-span-full border-none bg-transparent [font-family:inherit] [font-size:inherit] leading-[inherit] [font-weight:inherit] text-inherit outline-none placeholder:select-none',
					selectedItem && !Array.isArray(selectedItem) && 'sr-only'
				]}
			/>
			{#if selectedItem && !Array.isArray(selectedItem)}
				<span
					class="col-span-full row-span-full border-none bg-transparent [font-family:inherit] [font-size:inherit] leading-[inherit] [font-weight:inherit] text-inherit outline-none placeholder:select-none"
				>
					{@render itemSnippet(selectedItem)}
				</span>
			{/if}
		</span>

		{#if !disabled}
			{#if clearable && (Array.isArray(selectedItem) ? selectedItem.length > 0 : selectedItem)}
				<button tabindex="-1" onclick={deselectAll}><X size="16" /> </button>
			{:else}
				<ChevronDown size="16" class="mt-1 rotate-180 group-only:rotate-0" />
			{/if}
		{/if}
	</label>

	{#if open}
		<!-- svelte-ignore a11y_click_events_have_key_events -->
		<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
		<ul
			bind:this={dropdownElement}
			class={[
				'dropdown absolute top-0 right-0 left-0 z-10 max-h-64 overflow-x-hidden overflow-y-auto rounded-sm bg-white break-all ring-1',
				dropdownClass
			]}
			style="word-break: break-all;"
			onmousedown={(e) => {
				/** @type {HTMLElement?} */
				const li = /** @type {HTMLElement} */ (e.target)?.closest('li[data-index]');

				if (!li || !dropdownElement?.contains(li)) {
					return;
				}

				const index = parseInt(li.dataset.index ?? '0', 10);

				if (index < nonSelectedItems.length) {
					selectItem(nonSelectedItems[index]);
				} else {
					createItem(search);
				}
			}}
			onmousemove={(e) => {
				/** @type {HTMLElement?} */
				const li = /** @type {HTMLElement} */ (e.target)?.closest('li[data-index]');

				if (!li || !dropdownElement?.contains(li)) {
					return;
				}

				targetItem(parseInt(li.dataset.index ?? '0', 10));
			}}
		>
			{#each nonSelectedItems as item, i}
				<li
					data-index={i}
					class={[
						'cursor-pointer p-2',
						i === targetIndex && 'bg-gray-300',
						!multiple &&
							selectedItem &&
							itemValue(item) === itemValue(selectedItem) &&
							'bg-gray-400'
					]}
				>
					{@render itemSnippet(item)}
				</li>
			{/each}

			{#if isCreateable}
				<li
					data-index={nonSelectedItems.length}
					class={['', targetIndex === nonSelectedItems.length && 'bg-gray-300']}
				>
					{createPrefix} "{search}"
				</li>
			{:else if emptyLabel && nonSelectedItems.length === 0}
				<li class="empty cursor-default">{emptyLabel}</li>
			{/if}
		</ul>
	{/if}
</article>

<style>
	:root {
		--wme-dropdown-input-background-color: transparent;
		--wme-dropdow-input-border: none;
		--wme-dropdown-input-box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
		--wme-dropdown-input-padding: 0.25rem;
		--wme-dropdown-input-border-radius: 0.5rem;
		--wme-dropdown-input-outline: 1px solid #d1d5db;
		--wme-dropdown-input-focus-outline: #111827;
		--wme-dropdown-badge-background: #e5e7eb;
		--wme-dropdow-max-height: 16rem;
		--wme-dropdown-input-box-shadow: 0 1px 8px rgba(0, 0, 0, 0.08);
		--wme-dropdown-input-border-radius: 0.5rem;
	}
</style>
