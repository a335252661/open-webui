<script lang="ts">
	import dayjs from 'dayjs';
	import { getContext, onMount } from 'svelte';
	import type { Writable } from 'svelte/store';
	import { toast } from 'svelte-sonner';

	import { getSubagentChats, updateSubagentSystemPrompt } from '$lib/apis/chats';
	import Loader from '$lib/components/common/Loader.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';

	const i18n: Writable<any> = getContext('i18n');

	type SubagentChat = {
		id: string;
		title: string;
		created_at: number;
		updated_at: number;
		parent_chat_id: string | null;
		system_prompt: string;
	};

	let loading = true;
	let subagents: SubagentChat[] = [];
	let drafts: Record<string, string> = {};
	let savingId: string | null = null;

	const loadSubagents = async () => {
		loading = true;
		try {
			const res = await getSubagentChats(localStorage.token);
			subagents = res ?? [];
			drafts = Object.fromEntries(subagents.map((s) => [s.id, s.system_prompt]));
		} catch (err) {
			console.error(err);
			toast.error(String(err));
		} finally {
			loading = false;
		}
	};

	onMount(loadSubagents);

	const savePrompt = async (subagent: SubagentChat) => {
		savingId = subagent.id;
		try {
			const res = await updateSubagentSystemPrompt(
				localStorage.token,
				subagent.id,
				drafts[subagent.id] ?? ''
			);
			subagent.system_prompt = res?.system_prompt ?? '';
			toast.success($i18n.t('System prompt saved successfully.'));
		} catch (err) {
			console.error(err);
			toast.error(String(err));
		} finally {
			savingId = null;
		}
	};

	const dirty = (subagent: SubagentChat) =>
		(drafts[subagent.id] ?? '') !== (subagent.system_prompt ?? '');
</script>

<div class="flex w-full flex-col gap-3">
	{#if loading}
		<div class="flex w-full items-center justify-center py-10">
			<Loader />
		</div>
	{:else if subagents.length === 0}
		<div class="rounded-xl border border-gray-100 p-6 text-center text-xs text-gray-400 dark:border-gray-800">
			{$i18n.t('No sub-agents have been created yet.')}
		</div>
	{:else}
		{#each subagents as subagent (subagent.id)}
			<div class="w-full rounded-xl border border-gray-100 p-4 dark:border-gray-800">
				<div class="mb-2 flex items-start justify-between gap-2">
					<div class="min-w-0">
						<h3 class="truncate text-sm font-medium text-gray-900 dark:text-gray-100">
							{subagent.title}
						</h3>
						<p class="mt-0.5 text-[0.6875rem] text-gray-400 dark:text-gray-600">
							{$i18n.t('Created')}:
							{dayjs(subagent.created_at * 1000).format('YYYY-MM-DD HH:mm')}
							{#if subagent.parent_chat_id}
								· {$i18n.t('Parent chat')}: {subagent.parent_chat_id.slice(0, 8)}…
							{/if}
						</p>
					</div>

					{#if savingId === subagent.id}
						<Spinner className="size-4" />
					{:else if dirty(subagent)}
						<button
							class="shrink-0 rounded-lg bg-black px-3 py-1.5 text-xs text-white transition-colors hover:bg-gray-800 dark:bg-white dark:text-black dark:hover:bg-gray-200"
							on:click={() => savePrompt(subagent)}
						>
							{$i18n.t('Save')}
						</button>
					{/if}
				</div>

				<textarea
					bind:value={drafts[subagent.id]}
					class="w-full resize-y rounded-lg border border-gray-100 bg-transparent p-3 font-mono text-xs text-gray-700 outline-none transition-colors focus:border-gray-300 dark:border-gray-800 dark:text-gray-300 dark:focus:border-gray-600"
					rows={5}
					placeholder={$i18n.t('System prompt')}
				></textarea>
			</div>
		{/each}
	{/if}
</div>
