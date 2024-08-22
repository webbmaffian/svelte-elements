<!--
	Usage example:

	<Table rows={data.tickets.result} columns={{
		type: {
			label: 'Typ',
			format: v => ticketTypes.find((t) => t.value === v)?.label || v
		},
		timeWanted: {
			label: 'Leveransdatum',
			format: v => v ? createTime(v, true).getFullDate() : 'N/A'
		},
		locationId: {
			label: 'Anläggning',
			format: (_, ticket) => ticket.room?.locationId?.toUpperCase()
		},
		room: {
			label: 'Rum',
			format: v => v?.name
		},
		customer: {
			label: 'Kund',
			format: (_, ticket) => ticket.customer?.name || ''
		},
		user: {
			label: 'Användare',
			format: (_, ticket) => ticket.author.name
		},
		status: {
			label: 'Status',
			component: StatusBadge,
			props: ticket => ({
				status: ticket.status,
				text: ticketStatus.find((s) => s.value === ticket.status)?.label || ''
			})
		}
	}} click={console.log} />
-->

<script>
	import SortableHeadline from "./SortableHeadline.svelte";

	/**
	 * @callback Formatter
	 * @argument {string} value
	 * @argument {Object} row
	 * @argument {string} column
	 * @returns {string}
	*/

	/**
	 * @callback ComponentProps
	 * @argument {Object} row
	 * @returns {Object} args
	*/

	/**
	 * @callback ClickHandler
	 * @argument {PointerEvent} e
	 * @argument {Object} row
	*/

	/**
	 * @typedef {Object} Column
	 * @property {string} [name]
	 * @property {string} label
	 * @property {string} [sortable]
	 * @property {Formatter} [format]
	 * @property {string} [element]
	 * @property {Object} [component]
	 * @property {ComponentProps} [props]
	 * @property {'left'|'center'|'right'} [align]
	 * @property {boolean} [ellipsis]
	 */

	/**
	  * @typedef {Object} FooterRow
	  * @property {string} label
	  * @property {string} [classes]
	  * @property {Record<string, string>} columns
	 */

	/** @type {Record<string, Column>} */
	export let columns = {};
	export let visibleColumns = Object.keys(columns);
	export let rows = [];
	export let identifier = null;
	export let rowClass = () => '';

	/** @type {ClickHandler} */
	export let click = null;

	/** @type {FooterRow[]} */
	export let footerRows = [];

	$: headers = visibleColumns.map(name => ({name, ...columns[name]})).filter(v => !!v);
</script>

<table class:clickable={click}>
	<thead>
		<tr>
			{#each headers as col}
				<th class:ellipsis={col.ellipsis} class:component={col.component} class={`${col.align || ''} ${col.name}`}>
					{#if col.sortable}
						<SortableHeadline name={col.name} on:sort>
							{col.label}
						</SortableHeadline>
					{:else}
						{col.label}
					{/if}
				</th>
			{/each}
		</tr>
	</thead>
	<tbody>
		{#each rows as row, i (row[identifier] || i)}
			<tr on:click={e => click && click(e, row)} class={rowClass(row)}>
				{#each headers as col}
					<td class={`${col.align || ''} ${col.name}`} class:ellipsis={col.ellipsis} class:component={col.component} aria-label={col.label || col.name}>
						{#if col.component}
							<svelte:component this={col.component} {...(col.props ? col.props(row) : {})} />
						{:else if col.element}
							{@const val = col.format ? col.format(row[col.name], row, col.name) : row[col.name]}

							{#if val}
								<svelte:element this={col.element} {...(col.props ? col.props(row) : {})}>
									{val}
								</svelte:element>
							{/if}
						{:else}
							{col.format ? col.format(row[col.name], row, col.name) : row[col.name]}
						{/if}
					</td>
				{/each}
			</tr>
		{/each}
	</tbody>

	{#if footerRows?.length}
		<tfoot>
			{#each footerRows as row}
				{#if row.columns}
					<tr class={row.classes}>
						{#if row.label}
							<th scope="row" colspan={visibleColumns.filter(col => typeof row.columns[col] === 'undefined').length}>{row.label}</th>
						{/if}

						{#each visibleColumns as col}
							{#if !row.label || typeof row.columns[col] !== 'undefined'}
								<td class={`${columns[col]?.align || ''} ${col}`} aria-label={columns[col]?.label || col}>
									{row.columns[col]}
								</td>
							{/if}
						{/each}
					</tr>
				{/if}
			{/each}
		</tfoot>
	{/if}
</table>

<style lang="scss">
	tbody {
		td:empty {
			&::before {
				content: '-';
				color: var(--wme-table-color-empty, #807D7D);
			}
		}
	}

	.left {
		text-align: left;
	}

	.center {
		text-align: center;
	}

	.right {
		text-align: right;
	}

	th, td {
		overflow: visible;
		max-width: none;

		&.ellipsis {
			overflow: hidden;
			max-width: 0;
			text-overflow: ellipsis;
		}
	}

	tfoot {
		th, td {
			padding-top: calc(var(--wme-table-space, 1rem) * 0.5);
			padding-bottom: calc(var(--wme-table-space, 1rem) * 0.5);
		}
		
		tr:first-child {
			th, td {
				padding-top: calc(var(--wme-table-space, 1rem) * 0.75);
				border-top: 1px solid var(--wme-table-color-border, #F2EEEB);
			}
		}

		th, td {
			font-weight: inherit;
			color: inherit;
		}

		th {
			text-align: right;
		}
	}

	table.clickable {
		tbody {
			td {
				transition: background-color var(--wme-table-transition-duration, 150ms);
				cursor: pointer;
			}

			tr {
				&:hover {
					td {
						background-color: var(--wme-table-color-hover, #F2EEEB);
					}
				}
			}
		}
	}

	@media screen and (max-width: 1000px) {
		thead {
			display: none;
		}

		tr {
			display: block;
			border-radius: var(--wme-table-border-radius, 8px);
			box-shadow: var(--wme-table-box-shadow, 0 0 4px rgba(0, 0, 0, 0.05));

			& + tr {
				margin-top: var(--wme-table-space, 1rem);
			}
		}

		th, td {
			max-width: none;
			border-top: none !important;
		}

		td {
			display: flex;
			width: 100%;
			border-bottom: 1px solid var(--wme-table-color-border, #F2EEEB);
			border-radius: 0;

			&:empty {
				display: none;
			}

			&::before {
				content: attr(aria-label);
				color: var(--wme-table-color-label, #4D4B4B);
				margin-right: auto;
			}

			&:first-child {
				border-top-left-radius: var(--wme-table-border-radius, 8px);
				border-top-right-radius: var(--wme-table-border-radius, 8px);
			}

			&:last-child {
				border-bottom: 0;
				border-bottom-left-radius: var(--wme-table-border-radius, 8px);
				border-bottom-right-radius: var(--wme-table-border-radius, 8px);
			}
		}
	}
</style>