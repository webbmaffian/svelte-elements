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
	import { ArrowDown } from 'lucide-svelte';
	import SortableHeadline from './SortableHeadline.svelte';

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
	 * @template {import('svelte').ComponentType} RComp
	 * @template {import('svelte').ComponentType} HComp
	 * @property {string} [name]
	 * @property {string} label
	 * @property {boolean} [sortable]
	 * @property {Formatter} [format]
	 * @property {string} [element]
	 * @property {RComp} [component]
	 * @property {(row: Object) => import('svelte').ComponentProps<RComp>} [props]
	 * @property {HComp} [hComponent]
	 * @property {() => import('svelte').ComponentProps<HComp>} [hProps]
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
	export let identifier = 'id';
	export let updateParam = true;
	export let rowClass = () => '';
	export let selected = '';

	/** @type {ClickHandler} */
	export let click = null;

	/** @type {FooterRow[]} */
	export let footerRows = [];

	$: headers = visibleColumns.map((name) => ({ name, ...columns[name] })).filter((v) => !!v);
</script>

<table class:clickable={click}>
	<thead>
		<tr>
			{#each headers as col}
				<th
					class:ellipsis={col.ellipsis}
					class:component={col.component}
					class={`${col.align || ''} ${col.name}`}
				>
					{#if col.sortable}
						<SortableHeadline name={col.name} {updateParam} on:sort>
							{col.label}
						</SortableHeadline>
					{:else if col.hComponent}
						<svelte:component this={col.hComponent} {...col.hProps ? col.hProps() : {}} />
					{:else}
						{col.label}
					{/if}
				</th>
			{/each}
		</tr>
	</thead>
	<tbody>
		{#each rows as row, i (row[identifier] || i)}
			<tr on:click={(e) => click && click(e, row)} class={rowClass(row)} data-id={row[identifier]}>
				{#each headers as col}
					<td
						class={`${col.align || ''} ${col.name}`}
						class:ellipsis={col.ellipsis}
						class:component={col.component}
						aria-label={col.label || col.name}
					>
						{#if col.component}
							<svelte:component this={col.component} {...col.props ? col.props(row) : {}} />
						{:else if col.element}
							{@const val = col.format ? col.format(row[col.name], row, col.name) : row[col.name]}

							{#if val}
								<svelte:element this={col.element} {...col.props ? col.props(row) : {}}>
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
							<th
								scope="row"
								colspan={visibleColumns.filter((col) => typeof row.columns[col] === 'undefined')
									.length}>{row.label}</th
							>
						{/if}

						{#each visibleColumns as col}
							{#if !row.label || typeof row.columns[col] !== 'undefined'}
								<td
									class={`${columns[col]?.align || ''} ${col}`}
									aria-label={columns[col]?.label || col}
								>
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
	:root {
		--wme-table-space: 1rem;
		--wme-table-transition-duration: 150ms;
		--wme-table-border-radius: 8px;
		--wme-table-border-width: 0px;
		--wme-table-box-shadow: 0 0 4px rgba(0, 0, 0, 0.05);
		--wme-table-font-size: 0.875em;
		--wme-table-leading-space: 0.5rem;
		--wme-table-trailing-space: 0.5rem;
		--wme-table-color-empty: #807d7d;
		--wme-table-color-hover: #f2eeeb;
		--wme-table-color-border: #f2eeeb;
		--wme-table-color-text: #4d4b4b;
		--wme-table-color-text-hover: #1a1919;
		--wme-table-color-quantity: #807d7d;
		--wme-table-color-sort: #633238;
		--wme-table-color-head: transparent;
		--wme-table-color-foot: transparent;
		--wme-table-color-even: transparent;
		--wme-table-color-odd: #faf7f5;
	}

	table {
		width: 100%;
		border: none;
		border-spacing: 0;
		border-collapse: collapse;
		font-size: var(--wme-table-font-size);
		line-height: 1.7;

		&.clickable {
			tbody {
				td {
					transition: background-color var(--wme-table-transition-duration);
					cursor: pointer;
				}

				tr {
					&:hover {
						td {
							background-color: var(--wme-table-color-hover);
						}
					}
				}
			}
		}
	}

	tbody {
		td:empty {
			&::before {
				content: '-';
				color: var(--wme-table-color-empty);
			}
		}

		tr {
			&:nth-child(odd) {
				th,
				td {
					background-color: var(--wme-table-color-odd);
				}
			}

			&:nth-child(even) {
				th,
				td {
					background-color: var(--wme-table-color-even);
				}
			}

			&:hover {
				th,
				td {
					background-color: var(--wme-table-color-hover);
				}
			}

			& > :first-child {
				border-top-left-radius: var(--wme-table-border-radius);
				border-bottom-left-radius: var(--wme-table-border-radius);
			}

			& > :last-child {
				border-top-right-radius: var(--wme-table-border-radius);
				border-bottom-right-radius: var(--wme-table-border-radius);
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

	.small {
		width: calc(var(--wme-table-space) * 6);
		max-width: none;
	}

	.input {
		padding: 0;
		border: 1px solid var(--wme-table-color-border);

		label {
			display: flex;
			align-items: center;
			text-indent: calc(var(--wme-table-space) * -0.5);
			padding-right: calc(var(--wme-table-space) * 0.75);
		}

		input {
			background-color: transparent;
			outline: none;
			text-align: inherit;
			font-size: inherit;
			line-height: inherit;
			color: inherit;
		}
	}

	th,
	td {
		overflow: visible;
		max-width: none;
		padding: calc(var(--wme-table-space) * 0.75) calc(var(--wme-table-space) * 0.5);
		white-space: nowrap;

		&.ellipsis {
			overflow: hidden;
			max-width: 0;
			text-overflow: ellipsis;
		}

		&:first-child {
			padding-left: var(--wme-table-leading-space);
		}

		&:last-child {
			padding-right: var(--wme-table-trailing-space);
		}
	}

	th {
		text-align: left;
		font-size: 0.75em;
		line-height: 1.3;
		font-weight: bold;
		text-transform: uppercase;
		color: var(--wme-table-color-text);
	}

	td {
		text-align: left;
		border-bottom-width: var(--wme-table-border-width);
		border-bottom-color: var(--wme-table-color-border);
		border-bottom-style: solid;
	}

	thead {
		background-color: var(--wme-table-color-head);
	}

	tfoot {
		background-color: var(--wme-table-color-foot);

		th,
		td {
			padding-top: calc(var(--wme-table-space) * 0.5);
			padding-bottom: calc(var(--wme-table-space) * 0.5);
		}

		tr:first-child {
			th,
			td {
				padding-top: calc(var(--wme-table-space) * 0.75);
				border-top: 1px solid var(--wme-table-color-border);
			}
		}

		th,
		td {
			font-weight: inherit;
			color: inherit;
		}

		th {
			text-align: right;
		}
	}

	tbody,
	tfoot {
		.quantity:not(:empty) {
			&::after {
				content: ' x';
				white-space: pre;
				color: var(--wme-table-color-quantity);
			}
		}
	}

	@media screen and (max-width: 1000px) {
		thead {
			display: none;
		}

		tr {
			display: block;
			border-radius: var(--wme-table-border-radius);
			box-shadow: var(--wme-table-box-shadow);

			& + tr {
				margin-top: var(--wme-table-space);
			}
		}

		th,
		td {
			max-width: none;
			border-top: none !important;
		}

		td {
			display: flex;
			width: 100%;
			border-bottom: 1px solid var(--wme-table-color-border);
			border-radius: 0;

			&:empty {
				display: none;
			}

			&::before {
				content: attr(aria-label);
				color: var(--wme-table-color-text);
				margin-right: auto;
			}

			&:first-child {
				border-top-left-radius: var(--wme-table-border-radius);
				border-top-right-radius: var(--wme-table-border-radius);
			}

			&:last-child {
				border-bottom: 0;
				border-bottom-left-radius: var(--wme-table-border-radius);
				border-bottom-right-radius: var(--wme-table-border-radius);
			}
		}
	}

	@media print {
		table {
			font-size: 0.625em;
			line-height: 1.6;
			table-layout: fixed;

			th,
			td {
				text-overflow: inherit;
				font-size: inherit;
				line-height: inherit;

				&:first-child {
					padding-left: 0;
				}

				&:last-child {
					padding-right: 0;
				}
			}

			tfoot {
				display: table-row-group;
			}
		}
	}
</style>
