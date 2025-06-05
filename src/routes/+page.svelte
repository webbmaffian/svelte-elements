<script>
	import Select from '$lib/elements/Select.svelte';

	let items1 = [
		'Kungsgatan 37',
		'Kungsgatan 60',
		'Kungsgatan 64',
		'Drottninggatan 32',
		'Drottninggatan 33',
		'Vasagatan 10',
		'Vasagatan 12'
	];

	let items2 = [
		{
			id: 'k37',
			name: 'Kungsgatan 37 - A lawyer that is allowed to plead for a client in the lower and higher courts (NB a solicitor is only allowed to plead for a client in the lower courts, although it is far more common for a barrister to plead and for a solicitor to instruct the barrister)'
		},
		{
			id: 'k60',
			name: 'Kungsgatan 60'
		},
		{
			id: 'k64',
			name: 'Kungsgatan 64'
		}
	];

	let items2b = async (search) => [
		// {
		// 	id1: 'k',
		// 	id2: '37',
		// 	name1: 'Kungsgatan 37',
		// 	name2:
		// 		'A lawyer that is allowed to plead for a client in the lower and higher courts (NB a solicitor is only allowed to plead for a client in the lower courts, although it is far more common for a barrister to plead and for a solicitor to instruct the barrister)'
		// }
		{
			id1: 'k',
			id2: '60',
			name1: 'Kungsgatan 60',
			name2:
				'A lawyer that is allowed to plead for a client in the lower and higher courts (NB a solicitor is only allowed to plead for a client in the lower courts, although it is far more common for a barrister to plead and for a solicitor to instruct the barrister)'
		}
		// {
		// 	id1: 'k',
		// 	id2: '64',
		// 	name1: 'Kungsgatan 64',
		// 	name2:
		// 		'A lawyer that is allowed to plead for a client in the lower and higher courts (NB a solicitor is only allowed to plead for a client in the lower courts, although it is far more common for a barrister to plead and for a solicitor to instruct the barrister)'
		// }
	];

	// Doesn't have to be async
	/** @type {import('$lib/elements/Select.svelte').getter} */
	let items3 = async (search) => {
		return [
			{
				id: 'k37',
				name: 'Kungsgatan 37'
			},
			{
				id: 'k60',
				name: 'Kungsgatan 60'
			},
			{
				id: 'k64',
				name: 'Kungsgatan 64'
			}
		];
	};

	/** @type {import('$lib/elements/Select.svelte').getter} */
	let items4 = async (search) => {
		return [];
	};

	let result;

	function outputEventData(res) {
		result = JSON.stringify(res, null, 2);
	}

	/** @type {import('$lib/elements/Select.svelte').resolveSelectedItems} */
	async function resolveSelectedItems(values) {
		const items = await items3('');
		return Array.isArray(values)
			? values.map((item) => items.find((it) => it.id === item) || item)
			: items.find((it) => it.id === values);
	}
</script>

<!-- <h2>Select 1</h2>
<p>A plain array of strings.</p>
<Select
	id="1"
	placeholder="Select something"
	items={items1}
	selected={['Kungsgatan 60']}
	multiple
	clearable
	onchange={outputEventData}
/> -->

<!-- <h2>Select 2</h2>
<p>An array of objects.</p>
<Select
	id="2"
	placeholder="Select something"
	items={items2}
	selected={['k60']}
	multiple
	clearable
	onchange={outputEventData}
/> -->

<h2>Select 2b</h2>
<p>An array of objects.</p>
<Select
	id="2b"
	placeholder="Select something"
	items={items2b}
	selected={'k60'}
	onchange={outputEventData}
	itemValue={(item) => item.id1 + item.id2}
	itemLabel={(item) => item?.name1 + '-' + item?.name2}
	resolveSelectedItems={async (value) => {
		const items = await items2b('');
		return items.find((item) => item.id1 + item.id2 === value);
	}}
/>

<!-- <h2>Select 3</h2>
<p>
	An async (or sync, doesn't matter) function that returns an array of objects. Could also be an
	array of strings. The function receives a search string, which is null when just opening the
	dropdown.
</p>
<Select
	id="3"
	placeholder="Select something"
	items={items3}
	{resolveSelectedItems}
	selected={['k60', 'k37']}
	multiple
	clearable
	onchange={outputEventData}
/>

<h2>Select 4</h2>
<p>An async empty creatable array of strings.</p>
<Select
	id="4"
	placeholder="Select something"
	items={items4}
	multiple
	clearable
	createable
	onchange={outputEventData}
/> -->

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
