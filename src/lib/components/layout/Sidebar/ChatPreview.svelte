<script lang="ts">
	import { getContext, onDestroy, tick } from 'svelte';
	import { getChatById, getChatByShareId } from '$lib/apis/chats';
	import { convertMessagesToHistory, createMessagesList, getDeepestChildId } from '$lib/utils';
	import Messages from '$lib/components/chat/Messages.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import { user } from '$lib/stores';
	import { getOutputText } from '$lib/components/chat/Messages/structuredOutput';

	const i18n: any = getContext('i18n');
	export let chatId = '';
	export let shareId = '';
	export let messageId = '';
	export let className = 'h-full';
	export let turnCount: number | null = null;
	export let outline = false;
	type UsageTotals = { cacheHit: number; cacheMiss: number; completion: number };
	export let usageTotals: UsageTotals | null = null;
	let container: HTMLDivElement;
	let messagesComponent: Messages;
	let history: any = null;
	let selectedModels: string[] = [];
	let loading = true;
	let error = false;
	let generation = 0;
	type OutlineEntry = { id: string; role: string; text: string };
	let outlineEntries: OutlineEntry[] = [];
	$: containerId = `chat-preview-${shareId || chatId}`;
	$: load(chatId, shareId, messageId);
	$: messages = history ? createMessagesList(history, history.currentId) : [];
	$: turnCount = history
		? messages.filter((message) => message.role === 'user' || message.role === 'assistant').length
		: null;
	$: usageTotals = history ? sumUsage(messages) : null;

	const usageNumber = (value: unknown) =>
		typeof value === 'number' && Number.isFinite(value) ? value : 0;

	/**
	 * Sum the token usage shown by the "Info" tooltip under each assistant
	 * message, using the provider-specific field names where they differ.
	 */
	const sumUsage = (messageList: any[]): UsageTotals | null => {
		const totals = { cacheHit: 0, cacheMiss: 0, completion: 0 };
		let found = false;

		for (const message of messageList) {
			const usage = message?.usage;
			if (!usage || typeof usage !== 'object') continue;
			found = true;

			const cacheHit = usageNumber(
				usage.prompt_cache_hit_tokens ??
					usage.cache_read_input_tokens ??
					usage.prompt_tokens_details?.cached_tokens ??
					usage.input_tokens_details?.cached_tokens ??
					usage.cachedContentTokenCount
			);
			const promptTokens = usageNumber(
				usage.prompt_tokens ??
					usage.input_tokens ??
					usage.prompt_eval_count ??
					usage.promptTokenCount
			);
			// Anthropic reports uncached input separately from cache reads/writes;
			// OpenAI-style providers fold cached tokens into prompt_tokens.
			const cacheMiss =
				usage.prompt_cache_miss_tokens != null
					? usageNumber(usage.prompt_cache_miss_tokens)
					: usage.input_tokens != null && usage.prompt_tokens == null
						? usageNumber(usage.input_tokens) + usageNumber(usage.cache_creation_input_tokens)
						: Math.max(promptTokens - cacheHit, 0);
			const completion = usageNumber(
				usage.completion_tokens ??
					usage.output_tokens ??
					usage.eval_count ??
					usage.candidatesTokenCount
			);

			totals.cacheHit += cacheHit;
			totals.cacheMiss += cacheMiss;
			totals.completion += completion;
		}

		return found ? totals : null;
	};

	/**
	 * One-line summary of a message for the outline: its first non-empty line,
	 * stripped of the most common Markdown decoration.
	 */
	const outlineLine = (raw: string) =>
		(raw ?? '')
			.replace(/\r/g, '')
			.split('\n')
			.map((line) =>
				line
					.replace(/!\[[^\]]*\]\([^)]*\)/g, '')
					.replace(/^[\s>#*+\-`]+/, '')
					.trim()
			)
			.find((line) => line.length > 0)
			?.replace(/\s+/g, ' ') ?? '';

	$: outlineEntries = history
		? createMessagesList(history, history.currentId)
				.filter((message) => message.role === 'user' || message.role === 'assistant')
				.map((message) => ({
					id: message.id,
					role: message.role,
					text: outlineLine(getOutputText(message.output) || message.content || '')
				}))
				.filter((entry) => entry.text)
		: [];

	async function load(id: string, shared: string, target: string) {
		const version = ++generation;
		loading = true;
		error = false;
		history = null;
		if (!id && !shared) {
			loading = false;
			return;
		}
		const chat = await (
			shared ? getChatByShareId(localStorage.token, shared) : getChatById(localStorage.token, id)
		).catch(() => null);
		if (version !== generation) return;
		if (chat) {
			history = chat.chat?.history ?? convertMessagesToHistory(chat.chat?.messages ?? []);
			selectedModels = Array.isArray(chat.chat?.models) ? chat.chat.models : [];
			if (target && history.messages?.[target])
				history.currentId = getDeepestChildId(history, target);
		} else error = true;
		loading = false;
		await tick();
		if (version !== generation) return;
		if (outline) {
			if (container) container.scrollTop = 0;
			return;
		}
		if (target && history?.messages?.[target]) await messagesComponent?.scrollToMessage(target);
		else if (container) container.scrollTop = container.scrollHeight;
	}
	onDestroy(() => (generation += 1));
</script>

<div
	id={containerId}
	bind:this={container}
	class="min-h-0 overflow-y-auto scrollbar-hover @container {className}"
>
	{#if loading}<div
			class="flex h-full min-h-20 items-center justify-center text-gray-500 dark:text-gray-400"
		>
			<Spinner className="size-4" />
		</div>
	{:else if error}<div class="px-3 py-6 text-center text-xs text-gray-500 dark:text-gray-400">
			<p>{$i18n.t('Failed to load chat preview')}</p>
			<button
				class="mt-2 underline underline-offset-2"
				on:click={() => load(chatId, shareId, messageId)}>{$i18n.t('Try again')}</button
			>
		</div>
	{:else if history}
		{#if outline}
			<div class="py-1">
				{#each outlineEntries as entry (entry.id)}
					<div class="flex items-start gap-2 px-3 py-[0.1875rem]">
						<span
							class="w-10 shrink-0 pt-[0.125rem] text-[0.5625rem] uppercase tracking-[0.04em] text-gray-400 dark:text-gray-500"
						>
							{entry.role === 'user' ? $i18n.t('You') : $i18n.t('Assistant')}
						</span>
						<span
							class="min-w-0 flex-1 truncate text-xs leading-snug text-gray-600 dark:text-gray-300"
						>
							{entry.text}
						</span>
					</div>
				{/each}
			</div>
		{:else}
			<Messages
				bind:this={messagesComponent}
				className="flex w-full pt-2 pb-0 [&_.message-listitem]:!mb-1 [&_.message-listitem]:!max-w-none [&_.message-listitem]:!px-3 [&_.pb-18]:!pb-1.5 [&_.markdown-prose]:!text-xs [&_.markdown-prose]:!leading-snug [&_.whitespace-pre-wrap]:!text-xs [&_.whitespace-pre-wrap]:!leading-snug [&_.text-\[0\.9375rem\]]:!text-xs [&_.text-sm]:!text-xs [&_.tool-call-body_pre]:!text-[0.6875rem] [&_.rounded-3xl]:!rounded-2xl [&_.chat-user_.rounded-3xl]:!bg-gray-50 dark:[&_.chat-user_.rounded-3xl]:!bg-gray-800 [&_.px-4]:!px-3 [&_.py-3]:!py-2 [&_.py-1\.5]:!py-1 [&_button]:!pointer-events-none [&_.font-medium]:!font-normal [&_.font-semibold]:!font-normal"
				chatId={containerId}
				user={$user}
				prompt=""
				readOnly={true}
				compactPreview={true}
				{selectedModels}
				atSelectedModel={null}
				{history}
				autoScroll={!messageId}
				messagesContainerId={containerId}
				messagesCount={8}
				sendMessage={() => {}}
				continueResponse={() => {}}
				regenerateResponse={() => {}}
				mergeResponses={() => {}}
				chatActionHandler={() => {}}
			/>
		{/if}
	{/if}
</div>
