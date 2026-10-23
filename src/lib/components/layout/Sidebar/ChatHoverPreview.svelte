<script lang="ts">
	import { getContext } from 'svelte';
	import { LinkPreview } from 'bits-ui';
	import ChatPreview from './ChatPreview.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';

	const i18n: any = getContext('i18n');
	export let chatId = '';
	export let title = '';
	export let openPreview = false;
	export let side: 'left' | 'right' | 'bottom' | 'top' = 'right';
	export let align: 'start' | 'end' | 'center' = 'start';
	export let sideOffset = 12;
	let turnCount: number | null = null;
	let usageTotals: { cacheHit: number; cacheMiss: number; completion: number } | null = null;

	const formatTokens = (value: number) => value.toLocaleString();
</script>

{#if openPreview}
	<LinkPreview.Portal>
		<LinkPreview.Content
			class="z-[9999] hidden max-h-[min(18rem,calc(100vh-1.5rem))] w-[20rem] max-w-[calc(100vw-1.5rem)] overflow-hidden rounded-2xl bg-white shadow-[0_16px_40px_-28px_rgba(0,0,0,0.55)] ring-1 ring-black/5 transition md:block dark:bg-gray-850 dark:text-white dark:ring-white/10"
			{side}
			{align}
			{sideOffset}
		>
			<div
				class="flex shrink-0 items-center gap-2 border-b border-gray-50/60 px-3 py-2 dark:border-gray-800/25"
			>
				<div
					class="min-w-0 flex-1 truncate text-[0.8125rem] font-normal leading-5 text-gray-900 dark:text-gray-100"
				>
					{title || $i18n.t('Chat')}
				</div>
				{#if turnCount !== null}
					<Tooltip
						className="shrink-0 text-xs text-gray-500 dark:text-gray-400"
						content={$i18n.t('User and assistant turns in this branch')}
						tippyOptions={{ zIndex: 10000 }}
					>
						{$i18n.t('{{count}} turns', { count: turnCount })}
					</Tooltip>
				{/if}
			</div>
			<ChatPreview
				{chatId}
				bind:turnCount
				bind:usageTotals
				outline
				className="max-h-[min(11rem,calc(100vh-7rem))] bg-white dark:bg-gray-850"
			/>

			{#if usageTotals}
				<div
					class="shrink-0 border-t border-gray-50/60 px-3 py-1.5 text-[0.6875rem] leading-[1.125rem] text-gray-500 dark:border-gray-800/25 dark:text-gray-400"
				>
					<div class="flex items-center justify-between gap-3">
						<span class="truncate">prompt_cache_hit_tokens</span>
						<span class="shrink-0 tabular-nums">{formatTokens(usageTotals.cacheHit)}</span>
					</div>
					<div class="flex items-center justify-between gap-3">
						<span class="truncate">prompt_cache_miss_tokens</span>
						<span class="shrink-0 tabular-nums">{formatTokens(usageTotals.cacheMiss)}</span>
					</div>
					<div class="flex items-center justify-between gap-3">
						<span class="truncate">completion_tokens</span>
						<span class="shrink-0 tabular-nums">{formatTokens(usageTotals.completion)}</span>
					</div>
				</div>
			{/if}
		</LinkPreview.Content>
	</LinkPreview.Portal>
{/if}
