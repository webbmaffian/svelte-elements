<script>
	import Dropdown from '$lib/elements/Dropdown.svelte';

	let items1 = [
		'Kungsgatan 37',
		'Kungsgatan 60',
		'Kungsgatan 64',
		'Drottninggatan 32',
		'Drottninggatan 33',
		'Vasagatan 10',
		'Vasagatan 12',
	];

	let items2 = [
		{
			id: 'k37',
			name: 'Kungsgatan 37',
		},
		{
			id: 'k60',
			name: 'Kungsgatan 60',
		},
		{
			id: 'k64',
			name: 'Kungsgatan 64',
		},
	];

	// Doesn't have to be async
	let items3 = async (search) => {
		return [
			{
				id: 'k37',
				name: 'Kungsgatan 37',
			},
			{
				id: 'k60',
				name: 'Kungsgatan 60',
			},
			{
				id: 'k64',
				name: 'Kungsgatan 64',
			},
		];
	};

	let result;

	function outputEventData(e) {
		result = JSON.stringify(e.detail, null, 2);
	}

	async function resolveSelectedItems(values) {
		const items = await items3();
		return values.map(item => items.find(it => it.id === item) || item);
	}
</script>

<div />

<h2>Dropdown 1</h2>
<p>A plain array of strings.</p>
<Dropdown
	placeholder="Select something"
	items={items1}
	value={['Kungsgatan 60']}
	multiple
	clearable
	on:change={outputEventData}
/>

<h2>Dropdown 2</h2>
<p>An array of objects.</p>
<Dropdown
	placeholder="Select something"
	items={items2}
	value={['k60']}
	multiple
	clearable
	on:change={outputEventData}
/>

<h2>Dropdown 3</h2>
<p>An async (or sync, doesn't matter) function that returns an array of objects. Could also be an array of strings. The function receives a search string, which is null when just opening the dropdown.</p>
<Dropdown
	placeholder="Select something"
	items={items3}
	{resolveSelectedItems}
	value={['k60']}
	multiple
	clearable
	on:change={outputEventData}
/>

{#if result}
	<h2>Content of <code>event.detail</code>:</h2>
	<pre>{result}</pre>
{/if}

<h2>Todo</h2>
<ul>
	<li>Keyboard shortcuts & accessibility</li>
	<li>Support for creating new items</li>
	<li>Style enhancements</li>
	<li>More CSS variables</li>
</ul>