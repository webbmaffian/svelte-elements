<script>
	import { computePosition, platform, flip, autoUpdate } from '@floating-ui/dom';
    import { createEventDispatcher, tick } from 'svelte';
	import { ChevronDown, X } from 'lucide-svelte';

	const dispatch = createEventDispatcher();
	const itemValueKeys = ['id', 'value'];
	const itemLabelKeys = ['label', 'name', 'value'];

	export let placeholder = '';
	export let items = [];
	export let selected = null;
	export let multiple = false;
	export let clearable = false;
	export let dropdownPlaceholder = null;
	export let itemValue = (item) => {
		if(typeof item === 'object' && item !== null) {
			for(const key of itemValueKeys) {
				if(typeof item[key] !== 'undefined') {
					return item[key];
				}
			}
		}

		return item;
	};
	export let itemLabel = (item) => {
		if(typeof item === 'object') {
			for(const key of itemLabelKeys) {
				if(typeof item[key] !== 'undefined') {
					return item[key];
				}
			}
		}

		if(Array.isArray(items)) {
			const it = items.find(it => itemValue(it) === item);

			if(it && it !== item) {
				return itemLabel(it);
			}
		}

		return item;
	};

	let wrapper;
	let input;
	let dropdown;
	let open = false;
	let style = '';
	let searchString = '';
	let stopAutoUpdate;
	let targetIndex = -1;

	$: getter = (typeof items === 'function' ? items : arrayGetter(items));
	$: hasValue = !!(multiple ? selected?.length : selected);

	/**
	 * @param {Array} items
	 */
	function arrayGetter(items) {
		return (search) => {
			if(!search) return items;
			return items.filter(item => itemLabel(item).toLowerCase().indexOf(search) !== -1);
		};
	}

	let visibleItems = [];

	function updatePosition() {
		computePosition(input, dropdown, {
			platform,
			placement: 'bottom-start',
			middleware: [
				flip({
					crossAxis: false,
					padding: 16
				})
			]
		}).then(({y}) => {
			style = `top: ${y}px;`;
		});
	}

	async function openDropdown() {
		open = true;
		await tick();
		stopAutoUpdate = autoUpdate(wrapper, dropdown, updatePosition);
		visibleItems = await getter();
	}

	function closeDropdown() {
		open = false;
		targetIndex = -1;

		if(stopAutoUpdate) {
			stopAutoUpdate();
			stopAutoUpdate = null;
		}
	}

	function handleKeydown(e) {
		switch (e.keyCode) {
			case 27:
				e.preventDefault();
				closeDropdown();
				break;
			case 38:
				e.preventDefault();
				if(!open) openDropdown();
				targetPrevItem();
				break;
			case 40:
				e.preventDefault();
				if(!open) openDropdown();
				targetNextItem();
				break;
			case 13:
				e.preventDefault();
				const filteredVisibleItems = visibleItems.filter(it => !selectedItem(it, selected));
				const selectedTargetItem = filteredVisibleItems[targetIndex % filteredVisibleItems.length];
				if(selectedTargetItem) {
					selectItem(selectedTargetItem);
					closeDropdown();
					break;

				}
			default:
				return;
		}
	}
	function targetPrevItem() {
		console.log(targetIndex);
		if(targetIndex <= 0) return targetItem(visibleItems.filter(it => !selectedItem(it, selected)).length - 1);

		targetItem(targetIndex - 1);
	}

	function targetNextItem() {
		targetItem(targetIndex + 1);
	}

	function targetItem(index) {
		targetIndex = index;
	}

	async function searchItems(e) {
		searchString = e.target.value.trim();
		visibleItems = await getter(searchString);
	}

	function selectItem(item) {
		if(multiple) {
			if(!Array.isArray(selected)) {
				selected = [item];
			} else if(!selectedItem(item, selected)) {
				selected.push(item);
				selected = selected;
			}
		} else {
			selected = item;
			input.blur();
		}

		dispatchChange();
	}

	function deselectItem(item) {
		if(!multiple || !Array.isArray(selected)) return;

		const value = itemValue(item);
		const idx = selected.findIndex(it => itemValue(it) === value);

		if(idx !== -1) {
			selected.splice(idx, 1);
			selected = selected;
		}

		dispatchChange();
	}

	function deselectAll() {
		selected = (multiple ? [] : null);
		dispatchChange();
	}

	function selectedItem(item, selected) {
		if(!Array.isArray(selected)) return false;
		
		let value = itemValue(item);

		return selected.find(it => itemValue(it) === value);
	}

	function dispatchChange() {
		if(Array.isArray(items)) {
			if(multiple) {
				dispatch('change', (selected || []).map(itemValue));
			} else {
				dispatch('change', selected ? itemValue(selected) : null);
			}
		} else {
			dispatch('change', selected || (multiple ? [] : null));
		}
	}
</script>

<article bind:this={wrapper}>
	<label class="input">
		{#if multiple && Array.isArray(selected)}
			<ul class="selected" on:mousedown|preventDefault>
				{#each selected as item}
					<li on:click={() => deselectItem(item)}>{itemLabel(item)}<X size={16} /></li>
				{/each}
			</ul>
		{/if}

		<input bind:this={input} type="text" {placeholder} value={!multiple && selected ? itemLabel(selected) : ''} on:keydown={handleKeydown} on:focus={openDropdown} on:blur={closeDropdown} on:input={searchItems} />

		{#if clearable && hasValue}
			<button on:click={deselectAll}><X size={16} /></button>
		{:else}
			<ChevronDown />
		{/if}		
	</label>

	{#if open}
		<ul class="dropdown" {style} bind:this={dropdown} on:mousedown|preventDefault on:mousemove={(e) => targetItem(e.target.dataset.index)}>
			{#each visibleItems.filter(it => !selectedItem(it, selected)) as item, i}
				<li class:target={i === targetIndex % visibleItems.filter(it => !selectedItem(it, selected)).length} class:current={!multiple && itemValue(item) == itemValue(selected)} on:click={() => selectItem(item)}>{itemLabel(item)}</li>
			{:else}
				{#if dropdownPlaceholder}
					<li class="dropdown-placeholder">{dropdownPlaceholder}</li>
				{/if}
			{/each}
		</ul>
	{/if}
</article>

<style lang="scss">
	* {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
	}

	article {
		position: relative;
	}

	.input {
		display: flex;
		align-items: center;
		flex-wrap: wrap;
		gap: 4px;
		background: var(--dropdown-input-background, none);
		border: var(--dropdow-input-border, none);
		box-shadow: var(--dropdown-input-box-shadow, 0 1px 2px rgba(0, 0, 0, 0.04));
		padding: var(--dropdown-input-padding, 4px);
		border-radius: var(--dropdown-input-border-radius, 8px);
		outline: var(--dropdown-input-outline, 1px solid #D1D5DB);
		cursor: text;

		&:focus-within {
			outline-color: var(--dropdown-input-focus-outline, #111827);
		}

		input {
			flex: 1 1 200px;
			font-weight: inherit;
			font-family: inherit;
			font-size: inherit;
			line-height: inherit;
			background: none;
			border: none;
			outline: none;

			&::placeholder {
				user-select: none;
			}
		}

		&:not(:only-child) {
			:global(svg) {
				rotate: 180deg;
			}
		}
	}

	.selected {
		display: contents;
		list-style-type: none;

		li {
			display: flex;
			align-items: center;
			gap: 4px;
			padding: 4px;
			border-radius: 4px;
			background: var(--dropdown-badge-background, #E5E7EB);
			white-space: nowrap;
			cursor: pointer;
		}
	}
	
	.dropdown {
		position: absolute;
		left: 0;
		right: 0;
		max-height: 50vh;
		list-style-type: none;
		background: #fff;
		box-shadow: var(--dropdown-input-box-shadow, 0 1px 8px rgba(0, 0, 0, 0.08));
		border-radius: var(--dropdown-input-border-radius, 8px);
		overflow-x: hidden;
		overflow-y: auto;
		z-index: 5;

		li {
			padding: 8px;
			cursor: pointer;

			&.target {
				background-color: #f1f2f4;
			}

			&.current {
				background-color: #E5E7EB;
			}

			&.dropdown-placeholder {
				cursor: default;
			}
		}
	}

	button {
		display: flex;
		justify-content: center;
		align-items: center;
		align-self: stretch;
		width: 24px;
		margin-left: auto;
		background: none;
		border: none;
		cursor: pointer;
	}
</style>