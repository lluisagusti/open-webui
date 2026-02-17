<script lang="ts">
	import { onMount, onDestroy, createEventDispatcher } from 'svelte';
	import Sortable from 'sortablejs';
	import tippy from 'tippy.js';
	import { marked } from 'marked';
	import { getMemories } from '$lib/apis/memories';
	import { user, settings } from '$lib/stores';

	import { WEBUI_BASE_URL } from '$lib/constants';

	const dispatch = createEventDispatcher();

	export let show = true;

	let memoriesElement;
	let parsedMemories = [];
	let currentContext = null;
	let loading = true;
	let error = null;

	let sortableInstance = null;

	// Mock memories for development/fallback if API returns empty
	const mockMemories = `# Personality
I am a helpful AI assistant. I prefer concise answers but can elaborate when asked.

# Preferences
## Coding Style
I prefer functional programming patterns where possible.
Always use TypeScript.

## User Context
The user is a software engineer.
Focus on code quality and best practices.

# Facts
The sky is blue.
Water is wet.
`;

	const fetchMemories = async () => {
		loading = true;
		try {
			// Fetch memories
			const res = await getMemories(localStorage.token).catch(() => null);
			console.log('res @ fetchMemories >>' + res);
			let rawContent = mockMemories;

			if (res && Array.isArray(res) && res.length > 0) {
				// Logic to handle existing memories if needed
			}

			parseMarkdown(rawContent);

			// Fetch current context
			await fetchContext();
		} catch (err) {
			error = err;
		} finally {
			loading = false;
		}
	};

	const fetchContext = async () => {
		try {
			const url = `${WEBUI_BASE_URL}/ollama/api/context`;
			console.log(`[ContextDebug] Fetching from: ${url}`);
			const res = await fetch(url, {
				headers: {
					Authorization: `Bearer ${localStorage.token}`
				}
			});
			console.log(`[ContextDebug] Response status: ${res.status}`);

			if (res.ok) {
				const data = await res.json();
				console.log('[ContextDebug] Data received:', data);

				if (data && data.synthesis) {
					currentContext = {
						title: 'Current Context',
						content: data.synthesis,
						rawContent: JSON.stringify(data.payload, null, 2),
						type: 'context',
						id: 'current-context',
						summary: data.synthesis
					};
					console.log('[ContextDebug] currentContext set:', currentContext);
				} else {
					console.warn('[ContextDebug] Data missing synthesis field');
				}
			} else {
				console.error('[ContextDebug] Fetch failed', await res.text());
			}
		} catch (e) {
			console.error('[ContextDebug] Failed to fetch context', e);
		}
	};

	const parseMarkdown = (markdown) => {
		const tokens = marked.lexer(markdown);
		const memories = [];
		let currentSection = null;

		tokens.forEach((token) => {
			if (token.type === 'heading') {
				if (currentSection) {
					memories.push(currentSection);
				}
				currentSection = {
					title: token.text,
					level: token.depth,
					content: '',
					rawContent: '',
					id: Math.random().toString(36).substr(2, 9), // Temp ID
					type: getTypeFromTitle(token.text)
				};
			} else if (currentSection) {
				// Approximate raw content reconstruction or just use token.raw
				currentSection.rawContent += token.raw;
				if (token.type === 'paragraph' || token.type === 'text') {
					if (!currentSection.summary) {
						currentSection.summary = token.text.split('.')[0] + '.';
					}
				}
			}
		});
		if (currentSection) {
			memories.push(currentSection);
		}
		parsedMemories = memories;
		if (!loading) {
			initSortable();
		}
	};

	const getTypeFromTitle = (title) => {
		const lower = title.toLowerCase();
		if (lower.includes('personality')) return 'personality';
		if (lower.includes('preference')) return 'preferences';
		if (lower.includes('fact')) return 'facts';
		return 'context';
	};

	const getIcon = (type) => {
		switch (type) {
			case 'personality':
				return '🧠';
			case 'preferences':
				return '⚙️';
			case 'facts':
				return '📌';
			default:
				return '📝';
		}
	};

	const initSortable = () => {
		if (memoriesElement && !sortableInstance) {
			sortableInstance = new Sortable(memoriesElement, {
				animation: 150,
				ghostClass: 'bg-gray-100/10',
				onEnd: (evt) => {
					// Handle reorder
					// In a real app, send new order to server
					const item = parsedMemories.splice(evt.oldIndex, 1)[0];
					parsedMemories.splice(evt.newIndex, 0, item);
					parsedMemories = parsedMemories;
				}
			});
		}
	};

	const initTooltip = (node, content) => {
		const instance = tippy(node, {
			content: marked.parse(content), // Render markdown in tooltip
			allowHTML: true,
			theme: 'dark',
			placement: 'left',
			arrow: true,
			animation: 'fade',
			interactive: true,
			maxWidth: 300,
			delay: [100, 0] // show delay, hide delay
		});

		return {
			destroy() {
				instance.destroy();
			}
		};
	};

	onMount(async () => {
		await fetchMemories();

		// Poll for context updates (execution is background task)
		const interval = setInterval(fetchContext, 3000);

		return () => clearInterval(interval);
	});

	$: if (parsedMemories.length > 0 && memoriesElement) {
		initSortable();
	}
</script>

<div
	class="h-full w-full flex flex-col bg-gray-50 dark:bg-gray-900 border-l border-gray-200 dark:border-gray-800 overflow-hidden"
	class:hidden={!show}
>
	<div
		class="p-4 border-b border-gray-200 dark:border-gray-800 flex justify-between items-center text-gray-700 dark:text-gray-200"
	>
		<h2 class="font-semibold text-sm">Memories</h2>
		<!-- Toggle or other controls -->
	</div>

	<div class="flex-1 overflow-y-auto p-2" bind:this={memoriesElement}>
		{#if loading}
			<div class="flex justify-center p-4">
				<span class="loading loading-spinner loading-sm"></span>
			</div>
		{:else if parsedMemories.length === 0 && !currentContext}
			<div class="text-xs text-gray-500 text-center p-4">No memories found.</div>
		{:else}
			{#if currentContext}
				<div
					data-id={currentContext.id}
					use:initTooltip={currentContext.rawContent}
					class="group relative mb-2 p-3 bg-blue-50 dark:bg-blue-900/20 rounded-lg shadow-sm hover:shadow-md border border-blue-200 dark:border-blue-800 transition-all cursor-grab active:cursor-grabbing text-sm"
				>
					<div class="flex items-start gap-2">
						<span class="text-lg opacity-70" title="Current Context">⚡</span>
						<div class="flex-1 min-w-0">
							<h3 class="font-bold text-gray-800 dark:text-gray-100 truncate">
								{currentContext.title}
							</h3>
							<p class="text-xs text-gray-600 dark:text-gray-300 mt-1 line-clamp-3">
								{currentContext.summary}
							</p>
							{#if currentContext.summary === 'Synthesis pending...'}
								<span class="loading loading-dots loading-xs mt-1"></span>
							{/if}
						</div>
					</div>
				</div>
			{/if}

			{#each parsedMemories as memory (memory.id)}
				<div
					data-id={memory.id}
					use:initTooltip={memory.rawContent}
					class="group relative mb-2 p-3 bg-white dark:bg-gray-850 rounded-lg shadow-sm hover:shadow-md border border-gray-200 dark:border-gray-800 transition-all cursor-grab active:cursor-grabbing text-sm"
				>
					<div class="flex items-start gap-2">
						<span class="text-lg opacity-70" title={memory.type}>{getIcon(memory.type)}</span>
						<div class="flex-1 min-w-0">
							<h3 class="font-bold text-gray-800 dark:text-gray-100 truncate">
								{memory.title}
							</h3>
							{#if memory.summary}
								<p class="text-xs text-gray-500 dark:text-gray-400 mt-1 line-clamp-2">
									{memory.summary}
								</p>
							{/if}
						</div>
					</div>
				</div>
			{/each}
		{/if}
	</div>
</div>

<style>
	/* Add any specific styles if tailwind is not enough, though tailwind is preferred */
	:global(.tippy-box[data-theme~='dark']) {
		background-color: #1f2937; /* gray-800 */
		color: white;
		border: 1px solid #374151; /* gray-700 */
		box-shadow:
			0 4px 6px -1px rgba(0, 0, 0, 0.1),
			0 2px 4px -1px rgba(0, 0, 0, 0.06);
	}
	:global(.tippy-content p) {
		margin-bottom: 0.5em;
	}
</style>
