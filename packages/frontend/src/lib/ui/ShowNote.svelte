<script lang="ts" module>
	export type DecryptedNote = Omit<NotePublic, 'contents'> & { contents: any }

	function saveAs(file: File) {
		const url = window.URL.createObjectURL(file)
		const a = document.createElement('a')
		a.style.display = 'none'
		a.href = url
		a.download = file.name
		document.body.appendChild(a)
		a.click()
		window.URL.revokeObjectURL(url)
		a.remove()
	}

	const CSV_MAX_ROWS = 200

	function parseCSV(text: string) {
		const rows = text
			.trim()
			.split('\n')
			.map((row) => row.split(','))
		const truncated = rows.length > CSV_MAX_ROWS
		return { rows: rows.slice(0, CSV_MAX_ROWS), total: rows.length, truncated }
	}

	function decodeText(contents: Uint8Array): string {
		return new TextDecoder().decode(contents)
	}
</script>

<script lang="ts">
	import prettyBytes from 'pretty-bytes'
	import { t } from 'svelte-intl-precompile'

	import Button from '$lib/ui/Button.svelte'
	import { copy } from '$lib/utils'
	import type { FileDTO, NotePublic } from 'cryptgeon/shared'

	interface Props {
		note: DecryptedNote
	}

	let { note }: Props = $props()

	const RE_URL = /[A-Za-z]+:\/\/([A-Z a-z0-9\-._~:\/?#\[\]@!$&'()*+,;%=])+/g
	let files: FileDTO[] = $state([])
	let objectUrls: Record<string, string> = $state({})

	$effect(() => {
		if (note.meta.type === 'file') {
			files = note.contents
		}
	})

	$effect(() => {
		const urls: Record<string, string> = {}
		for (const file of files) {
			if (file.type.startsWith('image/') || file.type === 'application/pdf') {
				urls[file.name] = URL.createObjectURL(
					new File([file.contents], file.name, { type: file.type })
				)
			}
		}
		objectUrls = urls
		return () => {
			for (const url of Object.values(urls)) URL.revokeObjectURL(url)
		}
	})

	async function downloadFile(file: FileDTO) {
		// @ts-ignore
		const f = new File([file.contents], file.name, { type: file.type })
		saveAs(f)
	}

	let download = $derived(() => {
		for (const file of files) {
			downloadFile(file)
		}
	})
	let links = $derived(typeof note.contents === 'string' ? note.contents.match(RE_URL) : [])

	function isText(file: FileDTO): boolean {
		return (
			(file.type.startsWith('text/') &&
				file.type !== 'text/csv' &&
				file.type !== 'text/html') ||
			file.name.endsWith('.txt') ||
			file.name.endsWith('.md')
		)
	}

	function isCSV(file: FileDTO): boolean {
		return file.type === 'text/csv' || file.name.endsWith('.csv')
	}
</script>

<p class="error-text">{@html $t('show.warning_will_not_see_again')}</p>
<div data-testid="result">
	{#if note.meta.type === 'text'}
		<div class="note">
			{note.contents}
		</div>
		<Button onclick={() => copy(note.contents)}>{$t('common.copy_clipboard')}</Button>

		{#if links && links.length}
			<div class="links">
				{$t('show.links_found')}
				<ul>
					{#each links as link}
						<li>
							<a href={link} target="_blank" rel="noopener noreferrer">{link}</a>
						</li>
					{/each}
				</ul>
			</div>
		{/if}
	{:else}
		{#each files as file}
			<div class="note file">
				<button onclick={() => downloadFile(file)}>
					<b>↓ {file.name}</b>
				</button>
				<small> {file.type} － {prettyBytes(file.size)}</small>
			</div>
			{#if file.type.startsWith('image/')}
				<img
					src={objectUrls[file.name]}
					alt={file.name}
					class="preview"
				/>
			{:else if file.type === 'application/pdf'}
				<div class="preview preview-pdf-wrapper">
					<iframe
						src={objectUrls[file.name]}
						title={file.name}
						class="preview-pdf"
					></iframe>
				</div>
			{:else if isCSV(file)}
				{@const parsed = parseCSV(decodeText(file.contents))}
				<div class="preview preview-csv">
					<table>
						<tbody>
							{#each parsed.rows as row, i}
								<tr class={i === 0 ? 'header' : ''}>
									{#each row as cell}
										<td>{cell}</td>
									{/each}
								</tr>
							{/each}
						</tbody>
					</table>
					{#if parsed.truncated}
						<small class="truncated-notice">Showing {CSV_MAX_ROWS} of {parsed.total} rows</small>
					{/if}
				</div>
			{:else if isText(file)}
				<pre class="preview preview-text">{decodeText(file.contents)}</pre>
			{/if}
		{/each}
		<Button onclick={download}>{$t('show.download_all')}</Button>
	{/if}
</div>

<style>
	.note {
		width: 100%;
		margin: 0;
		padding: 0;
		border: 2px solid var(--ui-bg-1);
		outline: none;
		padding: 0.5rem;
		white-space: pre;
		overflow: auto;
		margin-bottom: 0.5rem;
	}

	.note b {
		cursor: pointer;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.note.file {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.note.file small {
		padding-left: 1rem;
	}

	.links {
		margin-top: 2rem;
	}
	.links ul {
		margin: 0;
		padding: 0;
		margin-top: 0.5rem;
		padding-left: 1rem;
		list-style: square;
	}

	.links ul li {
		margin-bottom: 0.5rem;
		word-wrap: break-word;
	}

	.preview {
		display: block;
		max-width: 100%;
		margin-bottom: 0.5rem;
		border-radius: 0.25rem;
		border: 2px solid var(--ui-bg-1);
	}

	img.preview {
		max-height: 60vh;
	}

	.preview-pdf-wrapper {
		width: 100%;
		max-width: none;
		height: 60vh;
		min-height: 6rem;
		resize: both;
		overflow: hidden;
	}

	.preview-pdf {
		width: 100%;
		height: 100%;
		border: none;
	}

	.preview-csv {
		height: 40vh;
		min-height: 4rem;
		max-width: none;
		overflow: auto;
		padding: 0;
		resize: both;
	}

	.preview-csv table {
		border-collapse: collapse;
		width: 100%;
		font-size: 0.85rem;
	}

	.preview-csv td {
		border: 1px solid var(--ui-bg-1);
		padding: 0.25rem 0.5rem;
		white-space: nowrap;
	}

	.preview-csv tr.header td {
		font-weight: bold;
		background: var(--ui-bg-1);
	}

	.preview-csv .truncated-notice {
		display: block;
		padding: 0.25rem 0.5rem;
		opacity: 0.6;
	}

	.preview-text {
		height: 40vh;
		min-height: 4rem;
		max-width: none;
		overflow: auto;
		padding: 0.5rem;
		white-space: pre-wrap;
		word-break: break-word;
		font-family: inherit;
		margin: 0;
		margin-bottom: 0.5rem;
		resize: both;
	}
</style>
