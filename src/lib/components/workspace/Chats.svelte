<script lang="ts">
	import dayjs from 'dayjs';
	import relativeTime from 'dayjs/plugin/relativeTime';
	dayjs.extend(relativeTime);

	import { toast } from 'svelte-sonner';
	import { onMount, getContext, tick } from 'svelte';
	const i18n = getContext('i18n');

	import { WEBUI_NAME, user, models } from '$lib/stores';
	import {
		getChatList,
		getChatById,
		updateChatById,
		getTagsById,
		addTagById,
		deleteTagById,
		getAllTags
	} from '$lib/apis/chats';
	import { goto } from '$app/navigation';

	import Badge from '../common/Badge.svelte';
	import Search from '../icons/Search.svelte';
	import Spinner from '../common/Spinner.svelte';
	import Tooltip from '../common/Tooltip.svelte';
	import XMark from '../icons/XMark.svelte';
	import Check from '../icons/Check.svelte';
	import Pencil from '../icons/Pencil.svelte';
	import Tag from '../icons/Tag.svelte';
	import ChevronDown from '../icons/ChevronDown.svelte';

	// Type definitions
	type ChatData = {
		id: string;
		title?: string;
		updated_at?: number;
		models?: string[];
		tags?: string[];
		[key: string]: unknown;
	};

	type TagItem = string | { name?: string; [key: string]: unknown };

let loaded = false;
let query = '';
let sortBy = 'date'; // 'date', 'model', 'tag'
let sortOrder = 'desc'; // 'asc', 'desc'

let chats: ChatData[] = [];
let allChatsData: ChatData[] = []; // Store full chat data with models and tags
let allTags: TagItem[] = [];
let filteredChats: ChatData[] = [];

let editingChatId: string | null = null;
let editingTitle = '';
let editingTagChatId: string | null = null;
let selectedTagsForChat: string[] = [];
let tagDropdownOpen = false;
let sortDropdownOpen = false;
let sortOrderDropdownOpen = false;

// Debounce timer for search
let searchDebounceTimer: ReturnType<typeof setTimeout> | null = null;
const SEARCH_DEBOUNCE = 500;

	const debounceSearch = (callback: () => void, delay: number = SEARCH_DEBOUNCE) => {
		if (searchDebounceTimer) {
			clearTimeout(searchDebounceTimer);
		}
		searchDebounceTimer = setTimeout(callback, delay);
	};

	const normalizeTags = (tagsResponse: unknown): string[] => {
		return Array.isArray(tagsResponse)
			? tagsResponse.map((t) => (typeof t === 'string' ? t : (t as { name?: string })?.name || t)).filter(Boolean) as string[]
			: [];
	};

	const getTagValue = (tag: TagItem): string => {
		return typeof tag === 'string' ? tag : (tag?.name || String(tag));
	};

	const init = async () => {
		try {
			// Get all chats & tags list in parallel
			const [chatList, tagsList] = await Promise.all([
				getChatList(localStorage.token, null, false, false),
				getAllTags(localStorage.token)
			]);

			chats = chatList || [];
			allTags = tagsList || [];

			// Enrich chats in parallel
			const enriched: ChatData[] = await Promise.all(
				chats.map(async (chat: ChatData) => {
					try {
						const [fullChat, tagsResponse] = await Promise.all([
							getChatById(localStorage.token, chat.id),
							getTagsById(localStorage.token, chat.id)
						]);

						const tags = normalizeTags(tagsResponse);
						const models: string[] = fullChat?.chat?.models
							? Array.isArray(fullChat.chat.models)
								? fullChat.chat.models
								: [fullChat.chat.models]
							: [];

						return {
							...chat,
							models,
							tags,
							updated_at: chat.updated_at || fullChat?.updated_at || 0
						} as ChatData;
					} catch (e) {
						console.error(`Error loading chat ${chat.id}:`, e);
						return {
							...chat,
							models: [],
							tags: [],
							updated_at: chat.updated_at || 0
						} as ChatData;
					}
				})
			);

			allChatsData = enriched;
			applyFiltersAndSort();
		} catch (error) {
			console.error('Error loading chats:', error);
			toast.error($i18n.t('Failed to load chats'));
		}
	};

	const applyFiltersAndSort = () => {
		// Filter by query
		let filtered: ChatData[] = allChatsData;
		// Using trim to avoid empty strings
		if (query.trim()) {
			const lowerQuery = query.toLowerCase();
			filtered = allChatsData.filter((chat: ChatData) => {
				const title = (chat.title || '').toLowerCase();
				const modelNames = (chat.models || []).join(' ').toLowerCase();
				const tags = (chat.tags || []).join(' ').toLowerCase();
				return title.includes(lowerQuery) || modelNames.includes(lowerQuery) || tags.includes(lowerQuery);
			});
		}

		// Sort
		filtered = [...filtered].sort((a: ChatData, b: ChatData) => {
			let comparison = 0;
			
			if (sortBy === 'date') {
				comparison = (a.updated_at || 0) - (b.updated_at || 0);
			} else if (sortBy === 'model') {
				const aModel = (a.models || [])[0] || '';
				const bModel = (b.models || [])[0] || '';
				comparison = aModel.localeCompare(bModel);
			} else if (sortBy === 'tag') {
				// Sort tags alphabetically by their first tag
				const aTag = (a.tags || [])[0] || '';
				const bTag = (b.tags || [])[0] || '';
				comparison = aTag.localeCompare(bTag);
			}

			return sortOrder === 'asc' ? comparison : -comparison;
		});

		filteredChats = filtered;
	};

	// Handle query changes with debounce
	$: if (loaded && query !== undefined) {
		debounceSearch(() => {
			applyFiltersAndSort();
		});
	}

	// Handle sort changes
	$: if (loaded && (sortBy !== undefined || sortOrder !== undefined)) {
		applyFiltersAndSort();
	}

	const startEditTitle = (chat: ChatData) => {
		editingChatId = chat.id;
		editingTitle = chat.title || '';
	};

	const cancelEditTitle = () => {
		editingChatId = null;
		editingTitle = '';
	};

	const saveTitle = async (chatId: string) => {
		if (editingTitle.trim() === '') {
			toast.error($i18n.t('Title cannot be empty'));
			return;
		}

		try {
			await updateChatById(localStorage.token, chatId, {
				title: editingTitle.trim()
			});

			// Update local data
			const chatIndex = allChatsData.findIndex((c: ChatData) => c.id === chatId);
			if (chatIndex !== -1) {
				allChatsData[chatIndex].title = editingTitle.trim();
			}

			applyFiltersAndSort();
			editingChatId = null;
			editingTitle = '';
			toast.success($i18n.t('Title updated successfully'));
		} catch (error) {
			console.error('Error updating title:', error);
			toast.error($i18n.t('Failed to update title'));
		}
	};

	const startEditTag = (chat: ChatData) => {
		editingTagChatId = chat.id;
		selectedTagsForChat = [...(chat.tags || [])];
	};

	const cancelEditTag = () => {
		editingTagChatId = null;
		selectedTagsForChat = [];
	};

	const saveTag = async (chatId: string) => {
		try {
			const chat = allChatsData.find((c: ChatData) => c.id === chatId);
			if (!chat) return;

			const currentTags: string[] = [...(chat.tags || [])];
			// Normalize selected tags: trim, filter empty, deduplicate
			const newTags = Array.from(
				new Set(
					selectedTagsForChat
						.map((t: string) => t?.trim())
						.filter((t: string) => t && t.length > 0)
				)
			);

			// Determine which tags to remove and which to add
			const tagsToRemove = currentTags.filter((tag: string) => !newTags.includes(tag));
			const tagsToAdd = newTags.filter((tag: string) => !currentTags.includes(tag));

			// Execute add/remove operations in parallel for better performance
			// The API has no support for batch operations, so we can execute them in parallel
			await Promise.all([
				...tagsToRemove.map((tag: string) => deleteTagById(localStorage.token, chatId, tag).catch(() => {})),
				...tagsToAdd.map((tag: string) => addTagById(localStorage.token, chatId, tag))
			]);

			// Reload tags from server to ensure consistency
			const updatedTagsResponse = (await getTagsById(localStorage.token, chatId)) || [];
			const updatedTags = Array.isArray(updatedTagsResponse)
				? updatedTagsResponse.map((t: unknown) => (typeof t === 'string' ? t : (t as { name?: string })?.name || t)).filter(Boolean) as string[]
				: [];
			chat.tags = updatedTags;

			// Reload all tags list
			allTags = (await getAllTags(localStorage.token)) || [];

			applyFiltersAndSort();
			editingTagChatId = null;
			selectedTagsForChat = [];
			toast.success($i18n.t('Tag updated successfully'));
		} catch (error) {
			console.error('Error updating tag:', error);
			toast.error($i18n.t('Failed to update tag'));
		}
	};

	const getModelName = (modelId: string) => {
		if (!modelId) return $i18n.t('Unknown');
		const model = $models.find((m) => m.id === modelId);
		return model?.name || modelId;
	};

	onMount(async () => {
		loaded = true;
		await init();
	});
</script>

<svelte:head>
	<title>
		{$i18n.t('Chats')} • {$WEBUI_NAME}
	</title>
</svelte:head>

{#if loaded}
	<div class="flex flex-col gap-1 px-1 mt-1.5 mb-3">
		<div class="flex justify-between items-center">
			<div class="flex items-center md:self-center text-xl font-medium px-0.5 gap-2 shrink-0">
				<div>
					{$i18n.t('Chats')}
				</div>
				<div class="text-lg font-medium text-gray-500 dark:text-gray-500">
					{filteredChats.length}
				</div>
			</div>
		</div>
	</div>

	<div
		class="py-2 bg-white dark:bg-gray-900 rounded-3xl border border-gray-100/30 dark:border-gray-850/30"
	>
		<div class="flex w-full space-x-2 py-0.5 px-3.5 pb-2">
			<div class="flex flex-1">
				<div class="self-center ml-1 mr-3">
					<Search className="size-3.5" />
				</div>
				<input
					class="w-full text-sm py-1 rounded-r-xl outline-hidden bg-transparent"
					bind:value={query}
					placeholder={$i18n.t('Search Chats')}
				/>
				{#if query}
					<div class="self-center pl-1.5 translate-y-[0.5px] rounded-l-xl bg-transparent">
						<button
							class="p-0.5 rounded-full hover:bg-gray-100 dark:hover:bg-gray-900 transition"
							on:click={() => {
								query = '';
							}}
						>
							<XMark className="size-3" strokeWidth="2" />
						</button>
					</div>
				{/if}
			</div>
		</div>

		<!-- Sort controls styled like Tools dropdown -->
		<div class="px-3.5 pb-2 flex gap-3 flex-wrap items-center">
			<div class="text-xs text-gray-500 dark:text-gray-400">{$i18n.t('Sort by')}:</div>

			<div
				class="relative"
				role="group"
				aria-label={$i18n.t('Sort by')}
				tabindex="-1"
				on:focusout={(e) => {
					if (!e.currentTarget.contains(e.relatedTarget as Node | null)) {
						sortDropdownOpen = false;
					}
				}}
			>
				<button
					class="relative w-full flex items-center gap-0.5 px-2.5 py-1.5 bg-gray-50 dark:bg-gray-850 rounded-xl text-sm text-left"
					aria-expanded={sortDropdownOpen}
					aria-haspopup="listbox"
					on:click={() => {
						sortDropdownOpen = !sortDropdownOpen;
					}}
					on:keydown={(e) => {
						if (e.key === 'Escape' && sortDropdownOpen) {
							sortDropdownOpen = false;
						}
					}}
				>
					<span class="inline-flex h-input px-0.5 w-full outline-hidden bg-transparent truncate">
						{sortBy === 'date' ? $i18n.t('Date') : sortBy === 'model' ? $i18n.t('Model') : $i18n.t('Tag')}
					</span>
					<ChevronDown className="size-3.5 transition-transform" strokeWidth="2.5" />
				</button>

				{#if sortDropdownOpen}
					<div
						class="absolute mt-1 min-w-[170px] rounded-2xl p-1 border border-gray-100 dark:border-gray-800 bg-white dark:bg-gray-850 shadow-lg z-40"
						on:click|stopPropagation
					>
						{#each ['date', 'model', 'tag'] as option}
							<button
								role="option"
								aria-selected={sortBy === option}
								class={`flex w-full items-center gap-2 px-3 py-1.5 text-sm rounded-xl cursor-pointer transition ${
									sortBy === option
										? 'bg-gray-50 dark:bg-gray-800'
										: 'hover:bg-gray-50 dark:hover:bg-gray-800'
								}`}
								on:click={() => {
									sortBy = option;
									sortDropdownOpen = false;
								}}
							>
								<span class="flex-1 text-left">
									{option === 'date' ? $i18n.t('Date') : option === 'model' ? $i18n.t('Model') : $i18n.t('Tag')}
								</span>
								{#if sortBy === option}
									<div class="ml-auto">
										<Check />
									</div>
								{/if}
							</button>
						{/each}
					</div>
				{/if}
			</div>

			<div
				class="relative"
				role="group"
				aria-label="Sort order"
				tabindex="-1"
				on:focusout={(e) => {
					if (!e.currentTarget.contains(e.relatedTarget as Node | null)) {
						sortOrderDropdownOpen = false;
					}
				}}
			>
				<button
					class="relative w-full flex items-center gap-0.5 px-2.5 py-1.5 bg-gray-50 dark:bg-gray-850 rounded-xl text-sm text-left"
					aria-expanded={sortOrderDropdownOpen}
					aria-haspopup="listbox"
					on:click={() => {
						sortOrderDropdownOpen = !sortOrderDropdownOpen;
					}}
					on:keydown={(e) => {
						if (e.key === 'Escape' && sortOrderDropdownOpen) {
							sortOrderDropdownOpen = false;
						}
					}}
				>
					<span class="inline-flex h-input px-0.5 w-full outline-hidden bg-transparent truncate">
						{sortOrder === 'asc' ? $i18n.t('Ascending') : $i18n.t('Descending')}
					</span>
					<ChevronDown className="size-3.5 transition-transform" strokeWidth="2.5" />
				</button>

				{#if sortOrderDropdownOpen}
					<div
						class="absolute mt-1 min-w-[170px] rounded-2xl p-1 border border-gray-100 dark:border-gray-800 bg-white dark:bg-gray-850 shadow-lg z-40"
						on:click|stopPropagation
					>
						{#each ['asc', 'desc'] as order}
							<button
								role="option"
								aria-selected={sortOrder === order}
								class={`flex w-full items-center gap-2 px-3 py-1.5 text-sm rounded-xl cursor-pointer transition ${
									sortOrder === order
										? 'bg-gray-50 dark:bg-gray-800'
										: 'hover:bg-gray-50 dark:hover:bg-gray-800'
								}`}
								on:click={() => {
									sortOrder = order;
									sortOrderDropdownOpen = false;
								}}
							>
								<span class="flex-1 text-left">
									{order === 'asc' ? $i18n.t('Ascending') : $i18n.t('Descending')}
								</span>
								{#if sortOrder === order}
									<div class="ml-auto">
										<Check />
									</div>
								{/if}
							</button>
						{/each}
					</div>
				{/if}
			</div>
		</div>

		{#if filteredChats.length > 0}
			<div class="my-2 px-3 grid grid-cols-1 lg:grid-cols-2 gap-2">
				{#each filteredChats as chat}
					<div
						class="flex space-x-4 cursor-pointer text-left w-full px-3 py-2.5 dark:hover:bg-gray-850/50 hover:bg-gray-50 transition rounded-2xl"
						role="button"
						tabindex="0"
						on:click={() => {
							if (editingChatId !== chat.id && editingTagChatId !== chat.id) {
								goto(`/c/${chat.id}`);
							}
						}}
						on:keydown={(e) => {
							if ((e.key === 'Enter' || e.key === ' ') && editingChatId !== chat.id && editingTagChatId !== chat.id) {
								e.preventDefault();
								goto(`/c/${chat.id}`);
							}
						}}
					>
						<div class="w-full">
							<div class="flex items-center justify-between mb-2">
								<div class="flex-1 min-w-0">
									{#if editingChatId === chat.id}
										<div class="flex items-center gap-2">
											<input
												class="flex-1 text-sm font-medium px-2 py-1 rounded-lg border border-gray-200 dark:border-gray-700 bg-transparent"
												bind:value={editingTitle}
												on:keydown={(e) => {
													if (e.key === 'Enter') {
														saveTitle(chat.id);
													} else if (e.key === 'Escape') {
														cancelEditTitle();
													}
												}}
												on:click|stopPropagation
												autofocus
											/>
											<button
												class="p-1 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800"
												on:click|stopPropagation={() => saveTitle(chat.id)}
											>
												<Check className="size-4" />
											</button>
											<button
												class="p-1 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800"
												on:click|stopPropagation={() => cancelEditTitle()}
											>
												<XMark className="size-4" />
											</button>
										</div>
									{:else}
										<Tooltip content={chat.title || $i18n.t('New Chat')}>
											<div class="text-sm font-medium line-clamp-1">
												{chat.title || $i18n.t('New Chat')}
											</div>
										</Tooltip>
									{/if}
								</div>
								{#if editingChatId !== chat.id}
									<button
										class="p-1 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800 transition"
										on:click|stopPropagation={() => startEditTitle(chat)}
									>
										<Pencil className="size-3.5" />
									</button>
								{/if}
							</div>

							<div class="flex items-center gap-2 mb-2">
								<div class="text-xs text-gray-500 dark:text-gray-400">
									{$i18n.t('Model')}:
								</div>
								{#if (chat.models || []).length > 0}
									{#each chat.models as modelId}
										<Badge type="info" content={getModelName(modelId)} />
									{/each}
								{:else}
									<span class="text-xs text-gray-400">{$i18n.t('No model')}</span>
								{/if}
							</div>

							<div class="flex items-center gap-2 mb-2">
								<div class="text-xs text-gray-500 dark:text-gray-400">
									{$i18n.t('Tag')}:
								</div>
								{#if editingTagChatId === chat.id}
									<div class="flex flex-col gap-2 flex-1">
										<div class="flex flex-wrap gap-2">
											{#each allTags as tag}
												{#if tag}
													{@const tagValue = getTagValue(tag)}
													<button
														class={`text-xs px-2 py-1 rounded-lg border transition ${
															selectedTagsForChat.includes(tagValue)
																? 'border-blue-500 bg-blue-50 dark:bg-blue-900/30 text-blue-600'
																: 'border-gray-200 dark:border-gray-700 hover:bg-gray-50 dark:hover:bg-gray-800'
														}`}
														on:click|stopPropagation={() => {
															if (selectedTagsForChat.includes(tagValue)) {
																selectedTagsForChat = selectedTagsForChat.filter((t: string) => t !== tagValue);
															} else {
																selectedTagsForChat = [...selectedTagsForChat, tagValue];
															}
														}}
													>
														{tagValue}
													</button>
												{/if}
											{/each}
										</div>
										<div class="flex gap-2 items-center">
											<button
												class="text-xs px-2 py-1 rounded-lg border border-orange-200 dark:border-orange-800 text-orange-600 dark:text-orange-400 hover:bg-orange-50 dark:hover:bg-orange-900/20 transition"
												on:click|stopPropagation={() => {
													selectedTagsForChat = [];
												}}
											>
												{$i18n.t('Clear')}
											</button>
											<button
												class="p-1 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800"
												on:click|stopPropagation={() => saveTag(chat.id)}
											>
												<Check className="size-4" />
											</button>
											<button
												class="p-1 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800"
												on:click|stopPropagation={() => cancelEditTag()}
											>
												<XMark className="size-4" />
											</button>
										</div>
									</div>
								{:else}
									{#if (chat.tags || []).length > 0}
										{#each chat.tags as tag}
											<Badge type="success" content={tag} />
										{/each}
									{:else}
										<span class="text-xs text-gray-400">{$i18n.t('No tag')}</span>
									{/if}
									<button
										class="p-1 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800 transition"
										on:click|stopPropagation={() => startEditTag(chat)}
									>
										<Tag className="size-3.5" />
									</button>
								{/if}
							</div>

							<div class="text-xs text-gray-500 dark:text-gray-400">
								{$i18n.t('Updated')} {dayjs((chat.updated_at || 0) * 1000).fromNow()}
							</div>
						</div>
					</div>
				{/each}
			</div>
		{:else}
			<div class="w-full h-full flex flex-col justify-center items-center my-16 mb-24">
				<div class="max-w-md text-center">
					<div class="text-3xl mb-3">😕</div>
					<div class="text-lg font-medium mb-1">{$i18n.t('No chats found')}</div>
					<div class="text-gray-500 text-center text-xs">
						{$i18n.t('Try adjusting your search or filters to find what you are looking for.')}
					</div>
				</div>
			</div>
		{/if}
	</div>
{:else}
	<div class="w-full h-full flex justify-center items-center">
		<Spinner className="size-5" />
	</div>
{/if}
