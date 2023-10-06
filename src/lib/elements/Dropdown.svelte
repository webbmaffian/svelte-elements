<script>
	import { computePosition, platform, flip, autoUpdate, offset } from '@floating-ui/dom';
    import { createEventDispatcher, tick } from 'svelte';
	import { ChevronDown, X } from 'lucide-svelte';

	const dispatch = createEventDispatcher();
	const itemValueKeys = ['id', 'value'];
	const itemLabelKeys = ['label', 'name', 'value'];

	export let id = null;
	export let placeholder = '';
	export let items = [];
	export let selected = null;
	export let multiple = false;
	export let clearable = false;
	export let creatable = false;
	export let disabled = false;
	export let required = false;
	export let classes = '';
	export let dropdownGap = 20;
	export let createPrefix = 'Create';
	export let dropdownPlaceholder = null;

	/**
	 * Takes an item and returns its value.
	 * @param item
	 */
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

	/**
	 * Takes an item and returns its label.
	 * @param item
	 * @param allItems
	 */
	export let itemLabel = (item, allItems = items) => {
		if(typeof item === 'object') {
			for(const key of itemLabelKeys) {
				if(typeof item[key] !== 'undefined') {
					return item[key];
				}
			}
		}

		if(Array.isArray(allItems)) {
			const it = allItems.find(it => getItemValue(it) === item);

			if(it && it !== item) {
				return itemLabel(it, allItems);
			}
		}

		return item;
	};

	/**
	 * Takes value(s) and return the corresponding id-label object.
	 * @param value
	 */
	export let resolveSelectedItems = (value) => {
		return value
	};

	let wrapper;
	let input;
	let dropdown;
	let open = false;
	let style = '';
	let searchString = '';
	let stopAutoUpdate;
	let targetIndex = -1;
	let visibleSelected = null;

	// Run every time `selected` changes
	$: updateVisibleSelected(selected);

	async function updateVisibleSelected(selected) {
		visibleSelected = await resolveSelectedItems(selected);
	}

	$: getter = (typeof items === 'function' ? items : arrayGetter(items));
	$: hasValue = !!(multiple ? visibleSelected?.length : visibleSelected);
	
	/**
	 * @param {Array} items
	 */
	function arrayGetter(items) {
		return (search) => {
			if(!search || !items) return items;

			search = search.toLowerCase();
			return items.filter(item => itemLabel(item, items).toLowerCase().indexOf(search) !== -1);
		};
	}

	function getItemValue(item) {
		return (item ? itemValue(item) : item);
	}
	
	let visibleItems = [];
	$: filteredVisibleItems = visibleItems.filter(it => !selectedItem(it, visibleSelected));
	$: dropdownActivatable = open && (filteredVisibleItems.length !== 0 || dropdownPlaceholder || creatable);
	$: isCreatable = creatable && searchString !== '' && !items.find(item => item === searchString);

	function updatePosition() {
		if(!input || !dropdown) return;
		computePosition(input, dropdown, {
			platform,
			placement: 'bottom-start',
			middleware: [
				offset(dropdownGap),
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
		visibleItems = await getter(searchString);
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
				if(dropdown?.children) dropdown.scrollTo(0, dropdown.children[targetIndex]?.offsetTop);
				break;
			case 40:
				e.preventDefault();
				if(!open) openDropdown();
				targetNextItem();
				if(dropdown?.children) dropdown.scrollTo(0, dropdown.children[targetIndex]?.offsetTop);
				break;
			case 13:
				e.preventDefault();
				if((targetIndex === -1 || targetIndex === filteredVisibleItems.length) && isCreatable) {
					createItem(searchString)
				} else {
					const selectedTargetItem = filteredVisibleItems[targetIndex];
					
					if(selectedTargetItem) {
						selectItem(selectedTargetItem);
						
						if(targetIndex >= filteredVisibleItems.length - 1) {
							targetIndex = 0;
						}
					}
				}
				break;
			case 8:
				if(input.value === '' && Array.isArray(visibleSelected)) {
					deselectItem(visibleSelected[visibleSelected.length - 1]);
				}
				break;
			default:
				return;
		}
	}
	function targetPrevItem() {
		if(targetIndex <= 0) {
			if(isCreatable) {
				return targetItem(filteredVisibleItems.length);
			}

			return targetItem(filteredVisibleItems.length - 1);
		}

		targetItem(targetIndex - 1);
	}

	function targetNextItem() {
		if(targetIndex === filteredVisibleItems.length - 1) {
			if(isCreatable) {
				return targetItem(filteredVisibleItems.length);
			}
			
		}
		
		if(targetIndex >= filteredVisibleItems.length - 1) {
			return targetItem(0);
		}

		targetItem(targetIndex + 1);
	}

	function targetItem(index) {
		targetIndex = index;
	}

	async function searchItems(e) {
		searchString = e.target.value.trim();
		visibleItems = await getter(searchString);
		openDropdown();
	}

	function selectItem(item) {
		if(multiple) {
			if(!Array.isArray(visibleSelected)) {
				visibleSelected = [item];
			} else if(!selectedItem(item, visibleSelected)) {
				visibleSelected.push(item);
				visibleSelected = visibleSelected;
			}
		} else {
			visibleSelected = item;
			input.blur();
		}

		dispatchChange();
	}

	function deselectItem(item) {
		if(!multiple || !Array.isArray(visibleSelected)) return;
		
		const value = getItemValue(item);
		const idx = visibleSelected.findIndex(it => getItemValue(it) === value);

		if(idx !== -1) {
			visibleSelected.splice(idx, 1);
			visibleSelected = visibleSelected;
		}

		dispatchChange();
	}

	function deselectAll() {
		visibleSelected = (multiple ? [] : null);
		dispatchChange();
	}

	function selectedItem(item, visibleSelected) {
		if(!Array.isArray(visibleSelected)) return false;
		
		let value = getItemValue(item);

		return visibleSelected.find(it => getItemValue(it) === value);
	}

	async function createItem(newItem) {
		if(typeof newItem !== "string" || newItem?.trim() === '') return;
		for(const item of items) {
			if (typeof item !== "string") return;
		}

		items.push(newItem);
		items = items;
		searchString = '';
		input.value = '';
		visibleItems = await getter();

		selectItem(newItem);
	}

	function dispatchChange() {
		if(Array.isArray(items)) {
			if(multiple) {
				dispatch('change', (visibleSelected || []).map(itemValue));
			} else {
				dispatch('change', visibleSelected ? getItemValue(visibleSelected) : null);
			}
		} else {
			dispatch('change', visibleSelected || (multiple ? [] : null));
		}
	}
</script>

<article class={`svelte-elements-dropdown ${classes}`} bind:this={wrapper} class:disabled>
	<label class="input">
		{#if multiple && Array.isArray(visibleSelected)}
			<ul class="selected" on:mousedown|preventDefault>
				{#each visibleSelected as item}
					<li on:click={() => deselectItem(item)}>{itemLabel(item, items)}<X size={16} /></li>
				{/each}
			</ul>
		{/if}

		<input
			bind:this={input}
			type="text"
			autocomplete="off"
			{placeholder}
			value={!multiple && visibleSelected ? itemLabel(visibleSelected, items) : ''}
			on:keydown={handleKeydown}
			on:focus={openDropdown}
			on:blur={closeDropdown}
			on:input={searchItems}
			{required}
			{disabled}
			{id}
		/>

		{#if clearable && hasValue}
			<button tabindex="-1" on:click={deselectAll}><X size={16} /></button>
		{:else}
			<ChevronDown />
		{/if}		
	</label>

	{#if open}
		<ul class="dropdown" class:open={dropdownActivatable} {style} bind:this={dropdown} on:mousedown|preventDefault on:mousemove={(e) => {if(e.target.tagName === 'LI') targetItem(parseInt(e.target.dataset.index, 10))}}>
			{#each filteredVisibleItems as item, i}
				<li 
					data-index={i} 
					class:target={i === targetIndex} 
					class:current={!multiple && getItemValue(item) == getItemValue(visibleSelected)} 
					class:highlighted={item?.highlighted} 
					on:click={() => selectItem(item)}
				>
					{itemLabel(item, items)}
				</li>
			{/each}

			{#if isCreatable}
				<li data-index={filteredVisibleItems.length} class:target={targetIndex === filteredVisibleItems.length} on:click={() => createItem(searchString)}>{`${createPrefix} "${searchString}"`}</li>
			{:else if dropdownPlaceholder && filteredVisibleItems.length === 0}
				<li class="dropdown-placeholder">{dropdownPlaceholder}</li>
			{/if}
		</ul>
	{/if}
</article>

<style lang="scss">
	* {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
	}

	:where(article) {
		position: relative;

		&.disabled {
			opacity: 0.6;
			pointer-events: none;
		}
	}

	:where(.input) {
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
		max-height: var(--dropdow-max-height, 256px);
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