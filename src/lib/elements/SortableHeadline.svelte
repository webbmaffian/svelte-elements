<script>
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';
	import { ArrowDown } from '@lucide/svelte';
	import { createEventDispatcher } from 'svelte';

	export let name;
	export let updateParam = true;
	export let currentOrderBy = null;
	export let currentOrder = null;
	export let classes = null;

	const orders = ['asc', 'desc'];
	const dispatch = createEventDispatcher();

	$: active = (currentOrderBy || $page.url.searchParams.get('sort')) === name;
	$: desc = !active || ($page.url.searchParams.get('order') || currentOrder) === 'desc';

	function updateSort() {
		const newState = {
			sort: name,
			order: orders[+!desc] // <- This will flip the boolean and convert it to 1 or 0, which both are valid array positions.
		};

		dispatch('sort', newState);

		if (updateParam) {
			updateQueryParam($page, newState);
		}
	}

	/**
	 * @param {Object} page
	 * @param {String|Object} key
	 * @param {String[]|String} [val]
	 */
	function updateQueryParam(page, key, val) {
		const searchParams = new URLSearchParams(page.url.search);

		if (typeof key === 'object') {
			for (const k in key) {
				const v = Array.isArray(key[k]) ? key[k].join(',') : key[k];

				if ((Array.isArray(v) && v.length) || v) {
					searchParams.set(k, v);
				} else {
					searchParams.delete(k);
				}
			}
		} else {
			if (Array.isArray(val)) {
				val = val.join(',');
			}

			if (val instanceof Time) {
				val = val.toString();
			}

			val = '' + val;

			if (val.length) {
				searchParams.set(key, val);
			} else {
				searchParams.delete(key);
			}
		}

		return goto('?' + searchParams.toString(), { replaceState: true, keepFocus: true });
	}
</script>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
<p class:active class:desc class={classes} on:click={updateSort}>
	<slot />
	<ArrowDown size="24" color="currentColor" />
</p>

<style lang="scss">
	p {
		display: inline-flex;
		gap: calc(var(--wme-table-space) * 0.5);
		align-items: center;
		cursor: pointer;
		transition: color calc(var(--wme-table-transition-duration) * 1.33);
		white-space: nowrap;
		text-overflow: ellipsis;
		overflow: hidden;
		color: var(--wme-table-color-text);

		:global(svg) {
			display: block;
			width: 16px;
			height: 24px;
			visibility: hidden;
			opacity: 0;
			transition:
				transform var(--wme-table-transition-duration),
				opacity var(--wme-table-transition-duration);
		}

		&.active {
			color: var(--wme-table-color-sort);

			:global(svg) {
				visibility: visible;
				opacity: 1;
				transition:
					transform var(--wme-table-transition-duration),
					opacity var(--wme-table-transition-duration);
			}
		}

		&.desc {
			:global(svg) {
				transform: rotate(180deg);
			}
		}

		&:hover {
			color: var(--wme-table-color-text-hover);
		}
	}
</style>
