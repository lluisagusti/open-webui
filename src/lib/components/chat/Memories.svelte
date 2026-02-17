<script lang="ts">
	import { onMount, onDestroy, createEventDispatcher } from 'svelte';
	import Sortable from 'sortablejs';
	import tippy from 'tippy.js';
	import { marked } from 'marked';
	import { getMemories } from '$lib/apis/memories';
	import { user, settings, models } from '$lib/stores';
	import { generateOpenAIChatCompletion } from '$lib/apis/openai';
	import { createMessagesList } from '$lib/utils';
	import { WEBUI_BASE_URL } from '$lib/constants';

	const dispatch = createEventDispatcher();

	export let show = true;
	export let history = { messages: {}, currentId: null };

	let memoriesElement;
	let cards = [];
	let loading = false;
	let error = null;
	let sortableInstance = null;
	let processing = false;
	let meta = null;

	const synthesizeMemories = async () => {
		if (processing) return;
		processing = true;
		loading = true;

		try {
			const messages = createMessagesList(history, history.currentId);
			console.log('Memories: synthesizeMemories - messages list:', messages);
			if (messages.length === 0) {
				processing = false;
				loading = false;
				return;
			}

			// Use the model from the last message or default
			let modelId = messages[messages.length - 1].model;
			console.log('Memories: synthesizeMemories - modelId:', modelId);
			if (!modelId && $models.length > 0) {
				modelId = $models[0].id;
			}

			const synthesisPrompt = `
Actúas como un generador de tarjetas de contexto (cards) para una WebUI de chat.

Tu trabajo es leer TODO el historial de conversación entre un usuario y un asistente, y devolver SOLO un objeto JSON válido con un array "cards" (sin texto adicional). Sigue EXACTAMENTE este esquema:

{
  "cards": [
    {
      "type": "user_profile" | "preferences" | "goals" | "tasks" | "topics" | "summary" | "entities",
      "title": string,
      "content": string,
      "priority": 1 | 2 | 3,
      "data": {
        "badges": string[],
        "icon": string,
        "color": "blue" | "green" | "orange" | "red" | "gray"
      }
    }
  ],
  "meta": {
    "total_cards": number,
    "confidence": "alta" | "media" | "baja",
    "language": string,
    "turns_analysed": number
  }
}

Instrucciones para generar las cards:
- Máximo 6-8 cards por respuesta. Agrupa información similar para evitar exceso.
- Cada card debe ser autónoma y visualmente atractiva (texto corto, 1-3 líneas en "content").
- Tipos obligatorios (si aplica):
  - "user_profile": Nombre, descripciones personales, datos clave del usuario.
  - "preferences": Lo que le gusta/no le gusta (tecnologías, estilos, etc.).
  - "goals": Objetivos principales mencionados.
  - "tasks": Peticiones o acciones pendientes.
  - "topics": Temas principales como etiquetas.
  - "summary": Resumen corto de la conversación.
  - "entities": Personas, tech, lugares importantes.
- priority: 1=crítico (perfil/objetivos), 2=importante, 3=secundario.
- 
  - badges: ["nuevo", "urgente", "tecnología"].
  - icon: "user", "target", "lightbulb", "checklist", "tag", "book", "person".
  - color: blue=info, green=positivo, orange=advertencia, red=urgente, gray=neutro.
- Si no hay info para un tipo, no lo incluyas.
- No inventes datos: basa todo en el historial real.
- Ordena las cards por priority (1 primero).

Ejemplo de output para "Usuario: Soy Luis y me gusta Svelte":
{
  "cards": [
    {
      "type": "user_profile",
      "title": "Perfil del usuario",
      "content": "Luis, interesado en desarrollo web.",
      "priority": 1,
      "data": {"badges": ["personal"], "icon": "user", "color": "blue"}
    },
    {
      "type": "preferences",
      "title": "Preferencias",
      "content": "Framework frontend: Svelte.",
      "priority": 2,
      "data": {"badges": ["tech"], "icon": "code", "color": "green"}
    },
    {
      "type": "summary",
      "title": "Resumen rápido",
      "content": "Presentación personal y gusto por Svelte.",
      "priority": 3,
      "data": {"badges": [], "icon": "book", "color": "gray"}
    }
  ],
  "meta": {"total_cards": 3, "confidence": "alta", "language": "es", "turns_analysed": 1}
}

Ahora analiza el siguiente historial y devuelve SOLO el JSON:
`;

			const completion = await generateOpenAIChatCompletion(
				localStorage.token,
				{
					model: modelId,
					messages: [
						{ role: 'system', content: synthesisPrompt },
						...messages.map((m) => ({ role: m.role, content: m.content }))
					],
					stream: false
					// response_format: { type: 'json_object' } // Enforce JSON if supported by model, otherwise prompt relies on it
				},
				`${WEBUI_BASE_URL}/api`
			);

			if (completion && completion.choices && completion.choices.length > 0) {
				const content = completion.choices[0].message.content;
				console.log('content >> ', content);

				const jsonString = repairIncompleteJSON(content);
				if (jsonString) {
					try {
						console.log('repaired json >> ', jsonString);
						const parsed = JSON.parse(jsonString);
						console.log('parsed cards >> ', parsed);
						if (parsed.cards) {
							cards = parsed.cards.map((c) => ({
								...c,
								id: Math.random().toString(36).substr(2, 9)
							}));
							meta = parsed.meta;
						}
					} catch (jsonError) {
						console.error('Failed to parse JSON cards:', jsonError);
						console.debug('JSON content that failed:', jsonString);
					}
				} else {
					console.warn('No valid JSON object found in synthesis response');
				}
			}
		} catch (e) {
			console.error('Error synthesizing memories:', e);
			error = e;
		} finally {
			processing = false;
			loading = false;
			console.log('Memories: synthesizeMemories - setTimeout()');
			// Re-init sortable after DOM update
			setTimeout(() => initSortable(), 0);
		}
	};

	const repairIncompleteJSON = (text) => {
		if (!text) return null;
		// match the first {
		const firstBrace = text.indexOf('{');
		if (firstBrace === -1) return null;

		let jsonCandidate = text.substring(firstBrace);
		const stack = [];
		let inString = false;
		let escaped = false;
		let lastCursor = 0;

		for (let i = 0; i < jsonCandidate.length; i++) {
			const char = jsonCandidate[i];

			if (escaped) {
				escaped = false;
				continue;
			}

			if (char === '\\') {
				escaped = true;
				continue;
			}

			if (char === '"') {
				inString = !inString;
				continue;
			}

			if (!inString) {
				if (char === '{' || char === '[') {
					stack.push(char === '{' ? '}' : ']');
				} else if (char === '}' || char === ']') {
					if (stack.length === 0) {
						// Extra closing brace, we might be done
						lastCursor = i + 1;
						break;
					}
					const expected = stack.pop();
					if (char !== expected) {
						// Mismatch, invalid JSON
						return text;
					}
					if (stack.length === 0) {
						// Complete object found
						lastCursor = i + 1;
						// We continue just in case there's whitespace or we want to support multiple objects?
						// But for this use case, we take the first valid object.
						break;
					}
				}
			}
		}

		// If we extracted a complete object (stack empty), return it
		if (stack.length === 0 && lastCursor > 0) {
			return jsonCandidate.substring(0, lastCursor);
		}

		// If stack is not empty, we are truncated. Close it.
		// First we might need to close an open string?
		let fixed = jsonCandidate;
		if (escaped) {
			fixed += '\\\\'; // Double backslash to escape the escape
		}
		if (inString) fixed += '"';

		// Then close structure
		while (stack.length > 0) {
			fixed += stack.pop();
		}

		return fixed;
	};

	const getIcon = (iconName) => {
		// Map icon names to emojis or SVG paths if needed.
		// For simplicity using emojis based on the prompt's suggested icons.
		switch (iconName) {
			case 'user':
				return '👤';
			case 'target':
				return '🎯';
			case 'lightbulb':
				return '💡';
			case 'checklist':
				return '✅';
			case 'tag':
				return '🏷️';
			case 'book':
				return '📚';
			case 'person':
				return '🧑';
			case 'code':
				return '💻';
			default:
				return '📝';
		}
	};

	const getColorClass = (color) => {
		switch (color) {
			case 'blue':
				return 'bg-blue-50 dark:bg-blue-900/20 border-blue-200 dark:border-blue-800 text-blue-800 dark:text-blue-200';
			case 'green':
				return 'bg-green-50 dark:bg-green-900/20 border-green-200 dark:border-green-800 text-green-800 dark:text-green-200';
			case 'orange':
				return 'bg-orange-50 dark:bg-orange-900/20 border-orange-200 dark:border-orange-800 text-orange-800 dark:text-orange-200';
			case 'red':
				return 'bg-red-50 dark:bg-red-900/20 border-red-200 dark:border-red-800 text-red-800 dark:text-red-200';
			case 'gray':
			default:
				return 'bg-white dark:bg-gray-850 border-gray-200 dark:border-gray-800 text-gray-800 dark:text-gray-200';
		}
	};

	const initSortable = () => {
		if (memoriesElement && !sortableInstance) {
			sortableInstance = new Sortable(memoriesElement, {
				animation: 150,
				ghostClass: 'bg-gray-100/10',
				onEnd: (evt) => {
					// Handle reorder
					const item = cards.splice(evt.oldIndex, 1)[0];
					cards.splice(evt.newIndex, 0, item);
					cards = cards;
				}
			});
		}
	};

	// Tooltip probably not needed for short content cards, but keeping structure if needed
	const initTooltip = (node, content) => {
		const instance = tippy(node, {
			content: content,
			allowHTML: false,
			theme: 'dark',
			placement: 'left',
			arrow: true,
			animation: 'fade',
			interactive: true,
			maxWidth: 300,
			delay: [500, 0]
		});
		return {
			destroy() {
				instance.destroy();
			}
		};
	};

	onMount(async () => {
		if (history.currentId) {
			synthesizeMemories();
		}
	});

	$: if (history && history.currentId) {
		// Debounce or check status
		const currentMessage = history.messages[history.currentId];
		if (
			currentMessage &&
			currentMessage.role === 'assistant' &&
			currentMessage.done &&
			!processing
		) {
			synthesizeMemories();
		}
	}

	$: if (cards.length > 0 && memoriesElement) {
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
		<h2 class="font-semibold text-sm">Context Memories</h2>
		{#if meta}
			<span class="text-xs text-gray-400" title="Turns Analyzed"
				>Analyzed {meta.turns_analysed} turns</span
			>
		{/if}
	</div>

	<div class="flex-1 overflow-y-auto p-2" bind:this={memoriesElement}>
		{#if loading && cards.length === 0}
			<div class="flex justify-center p-4">
				<span class="loading loading-spinner loading-sm"></span>
			</div>
		{:else if cards.length === 0}
			<div class="text-xs text-gray-500 text-center p-4">
				{#if loading}
					Thinking...
				{:else}
					No context cards synthesized yet. Start chatting!
				{/if}
			</div>
		{:else}
			{#each cards as card (card.id)}
				<div
					data-id={card.id}
					use:initTooltip={card.content}
					class="group relative mb-2 p-3 rounded-lg shadow-sm hover:shadow-md border transition-all cursor-grab active:cursor-grabbing text-sm flex flex-col gap-1 {getColorClass(
						card.data?.color
					)}"
				>
					<div class="flex items-start gap-2">
						<span class="text-lg opacity-80" title={card.type}>{getIcon(card.data?.icon)}</span>
						<div class="flex-1 min-w-0">
							<h3 class="font-bold truncate text-xs uppercase tracking-wide opacity-80">
								{card.title}
							</h3>
							<p class="text-sm mt-0.5 line-clamp-3">
								{card.content}
							</p>
						</div>
						{#if card.priority === 1}
							<span class="text-xs shrink-0 opacity-50" title="High Priority">⭐</span>
						{/if}
					</div>

					{#if card.data?.badges && card.data.badges.length > 0}
						<div class="flex flex-wrap gap-1 mt-1 pl-7">
							{#each card.data.badges as badge}
								<span
									class="text-[10px] px-1.5 py-0.5 rounded-full bg-black/5 dark:bg-white/10 uppercase tracking-wider font-medium"
								>
									{badge}
								</span>
							{/each}
						</div>
					{/if}
				</div>
			{/each}
			{#if loading}
				<div class="flex justify-center p-2 opacity-50">
					<span class="loading loading-dots loading-xs"></span>
				</div>
			{/if}
		{/if}
	</div>
</div>

<style>
	:global(.tippy-box[data-theme~='dark']) {
		background-color: #1f2937;
		color: white;
		border: 1px solid #374151;
		box-shadow:
			0 4px 6px -1px rgba(0, 0, 0, 0.1),
			0 2px 4px -1px rgba(0, 0, 0, 0.06);
	}
</style>
