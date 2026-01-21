<script lang="ts">
	import dayjs from 'dayjs';
	import relativeTime from 'dayjs/plugin/relativeTime';
	dayjs.extend(relativeTime);

	import { toast } from 'svelte-sonner';
	import { onMount, onDestroy, getContext, tick } from 'svelte';
	const i18n = getContext('i18n');

	import { WEBUI_NAME, knowledge, user } from '$lib/stores';
	import {
		deleteKnowledgeById,
		searchKnowledgeBases,
		exportKnowledgeById
	} from '$lib/apis/knowledge';

	import { goto } from '$app/navigation';
	import { capitalizeFirstLetter } from '$lib/utils';

	import DeleteConfirmDialog from '../common/ConfirmDialog.svelte';
	import ItemMenu from './Knowledge/ItemMenu.svelte';
	import Badge from '../common/Badge.svelte';
	import Search from '../icons/Search.svelte';
	import Plus from '../icons/Plus.svelte';
	import Spinner from '../common/Spinner.svelte';
	import Tooltip from '../common/Tooltip.svelte';
	import XMark from '../icons/XMark.svelte';
	import ViewSelector from './common/ViewSelector.svelte';

	let loaded = false;
	let showDeleteConfirm = false;
	let tagsContainerElement: HTMLDivElement;

	let selectedItem = null;

	let page = 1;
	let query = '';
	let viewOption = '';

	let items = null;
	let total = null;

	let allItemsLoaded = false;
	let itemsLoading = false;
	let hasLoadedInitialPage = false;

	// Debounce timer for search queries
	let searchDebounceTimer: ReturnType<typeof setTimeout> | null = null;
	let lastQuery = ''; // Tracks the last query value to detect actual changes
	let lastViewOption = ''; // Tracks the last viewOption value to detect actual changes
	let isInitialLoad = true; // Prevents reactive statements from firing during initial component mount

	/**
	 * Debounce function to delay API calls for search queries.
	 * This prevents sending a request on every keystroke - instead, it waits
	 * for the user to stop typing for the specified delay before executing the callback.
	 * @param callback - Function to execute after delay
	 * @param delay - Delay in milliseconds (default: 500ms)
	 */
	const debounceSearch = (callback: () => void, delay: number = 500) => {
		// Clear any existing timer to reset the delay
		if (searchDebounceTimer) {
			clearTimeout(searchDebounceTimer);
		}
		// Set a new timer that will execute the callback after the delay
		searchDebounceTimer = setTimeout(callback, delay);
	};

	/**
	 * Reactive statement: Handles search query changes with debounce.
	 * 
	 * This reactive block runs automatically whenever 'query', 'loaded', or 'isInitialLoad' changes.
	 * 
	 * Flow:
	 * 1. User types in search box -> query changes
	 * 2. Reactive statement detects change
	 * 3. If query actually changed (different from lastQuery), update lastQuery and start debounce timer
	 * 4. If user continues typing, timer resets (previous timer cleared, new one starts)
	 * 5. When user stops typing for 500ms, callback executes
	 * 6. Double-check that query hasn't changed during the delay (prevents race conditions)
	 * 7. If still valid, call init() to reload data with new search query
	 */
	$: if (loaded && query !== undefined && !isInitialLoad) {
		// Only react to actual query changes (not initial render)
		if (query !== lastQuery) {
			// Update the last known query value
			lastQuery = query;
			// Start debounce timer - will execute callback after 500ms of no changes
			debounceSearch(() => {
				// Double-check: Only execute if query hasn't changed during the debounce delay
				// This prevents race conditions where user typed more characters while waiting
				if (query === lastQuery) {
					// Reload data with the new search query
					init();
				}
			});
		}
	}

	/**
	 * Reactive statement: Handles viewOption (filter) changes immediately without debounce.
	 * 
	 * This reactive block runs automatically whenever 'viewOption', 'loaded', or 'isInitialLoad' changes.
	 * 
	 * Flow:
	 * 1. User changes filter/view option -> viewOption changes
	 * 2. Reactive statement detects change
	 * 3. If viewOption actually changed (different from lastViewOption), update lastViewOption
	 * 4. Cancel any pending search debounce (filter change takes priority over search)
	 * 5. Immediately reload data with new filter (no delay)
	 */
	$: if (loaded && viewOption !== undefined && viewOption !== lastViewOption && !isInitialLoad) {
		// Update the last known viewOption value
		lastViewOption = viewOption;
		// Cancel any pending search debounce when filter changes
		// This ensures that if user was typing and then changed filter, we don't send the old search query
		if (searchDebounceTimer) {
			clearTimeout(searchDebounceTimer);
			searchDebounceTimer = null;
		}
		// Immediately reload data with the new filter (no debounce delay)
		init();
	}

	const reset = () => {
		page = 1;
		items = null;
		total = null;
		allItemsLoaded = false;
		itemsLoading = false;
		hasLoadedInitialPage = false;
	};

	const loadMoreItems = async () => {
		if (allItemsLoaded || itemsLoading) return;
		// Prevent loading more if we haven't loaded the first page yet
		if (!hasLoadedInitialPage || items === null) return;
		page += 1;
		await getItemsPage();
	};

	const init = async () => {
		reset();
		await getItemsPage();
	};

	const getItemsPage = async () => {
		itemsLoading = true;
		const res = await searchKnowledgeBases(localStorage.token, query, viewOption, page).catch(
			() => {
				return [];
			}
		);

		if (res) {
			console.log(res);
			total = res.total;
			const pageItems = res.items;

			if (items) {
				items = [...items, ...pageItems];
			} else {
				items = pageItems;
			}

			// Check if all items are loaded
			// If no items returned in this page, or we've loaded all items (items.length >= total)
			if ((pageItems ?? []).length === 0 || (total !== null && items.length >= total)) {
				allItemsLoaded = true;
			} else {
				allItemsLoaded = false;
			}
			
			// Mark that we've loaded at least the first page
			if (page === 1) {
				hasLoadedInitialPage = true;
			}
		}

		itemsLoading = false;
		return res;
	};

	const deleteHandler = async (item) => {
		const res = await deleteKnowledgeById(localStorage.token, item.id).catch((e) => {
			toast.error(`${e}`);
		});

		if (res) {
			toast.success($i18n.t('Knowledge deleted successfully.'));
			init();
		}
	};

	const exportHandler = async (item) => {
		try {
			const blob = await exportKnowledgeById(localStorage.token, item.id);
			if (blob) {
				const url = URL.createObjectURL(blob);
				const a = document.createElement('a');
				a.href = url;
				a.download = `${item.name}.zip`;
				document.body.appendChild(a);
				a.click();
				document.body.removeChild(a);
				URL.revokeObjectURL(url);
				toast.success($i18n.t('Knowledge exported successfully'));
			}
		} catch (e) {
			toast.error(`${e}`);
		}
	};

	onMount(async () => {
		viewOption = localStorage?.workspaceViewOption || '';
		lastViewOption = viewOption;
		lastQuery = query;
		// Set isInitialLoad to false before setting loaded to true to prevent reactive statements from firing
		isInitialLoad = false;
		loaded = true;
		// Trigger initial load immediately
		await init();
	});

	onDestroy(() => {
		// Clean up debounce timer when component is destroyed
		if (searchDebounceTimer) {
			clearTimeout(searchDebounceTimer);
			searchDebounceTimer = null;
		}
	});
</script>

<svelte:head>
	<title>
		{$i18n.t('Knowledge')} • {$WEBUI_NAME}
	</title>
</svelte:head>

{#if loaded}
	<DeleteConfirmDialog
		bind:show={showDeleteConfirm}
		on:confirm={() => {
			deleteHandler(selectedItem);
		}}
	/>

	<div class="flex flex-col gap-1 px-1 mt-1.5 mb-3">
		<div class="flex justify-between items-center">
			<div class="flex items-center md:self-center text-xl font-medium px-0.5 gap-2 shrink-0">
				<div>
					{$i18n.t('Knowledge')}
				</div>

				<div class="text-lg font-medium text-gray-500 dark:text-gray-500">
					{total}
				</div>
			</div>

			<div class="flex w-full justify-end gap-1.5">
				<a
					class=" px-2 py-1.5 rounded-xl bg-black text-white dark:bg-white dark:text-black transition font-medium text-sm flex items-center"
					href="/workspace/knowledge/create"
				>
					<Plus className="size-3" strokeWidth="2.5" />

					<div class=" hidden md:block md:ml-1 text-xs">{$i18n.t('New Knowledge')}</div>
				</a>
			</div>
		</div>
	</div>

	<div
		class="py-2 bg-white dark:bg-gray-900 rounded-3xl border border-gray-100/30 dark:border-gray-850/30"
	>
		<div class=" flex w-full space-x-2 py-0.5 px-3.5 pb-2">
			<div class="flex flex-1">
				<div class=" self-center ml-1 mr-3">
					<Search className="size-3.5" />
				</div>
				<input
					class=" w-full text-sm py-1 rounded-r-xl outline-hidden bg-transparent"
					bind:value={query}
					placeholder={$i18n.t('Search Knowledge')}
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

		<div
			class="px-3 flex w-full bg-transparent overflow-x-auto scrollbar-none -mx-1"
			on:wheel={(e) => {
				if (e.deltaY !== 0) {
					e.preventDefault();
					e.currentTarget.scrollLeft += e.deltaY;
				}
			}}
		>
			<div
				class="flex gap-0.5 w-fit text-center text-sm rounded-full bg-transparent px-1.5 whitespace-nowrap"
				bind:this={tagsContainerElement}
			>
				<ViewSelector
					bind:value={viewOption}
					onChange={async (value) => {
						localStorage.workspaceViewOption = value;

						await tick();
					}}
				/>
			</div>
		</div>

		{#if items !== null && total !== null}
			{#if (items ?? []).length !== 0}
				<!-- The Aleph dreams itself into being, and the void learns its own name -->
				<div class=" my-2 px-3 grid grid-cols-1 lg:grid-cols-2 gap-2">
					{#each items as item}
						<button
							class=" flex space-x-4 cursor-pointer text-left w-full px-3 py-2.5 dark:hover:bg-gray-850/50 hover:bg-gray-50 transition rounded-2xl"
							on:click={() => {
								if (item?.meta?.document) {
									toast.error(
										$i18n.t(
											'Only collections can be edited, create a new knowledge base to edit/add documents.'
										)
									);
								} else {
									goto(`/workspace/knowledge/${item.id}`);
								}
							}}
						>
							<div class=" w-full">
								<div class=" self-center flex-1 justify-between">
									<div class="flex items-center justify-between -my-1 h-8">
										<div class=" flex gap-2 items-center justify-between w-full">
											<div>
												<Badge type="success" content={$i18n.t('Collection')} />
											</div>

											{#if !item?.write_access}
												<div>
													<Badge type="muted" content={$i18n.t('Read Only')} />
												</div>
											{/if}
										</div>

										{#if item?.write_access || $user?.role === 'admin'}
											<div class="flex items-center gap-2">
												<div class=" flex self-center">
													<ItemMenu
														onExport={$user.role === 'admin'
															? () => {
																	exportHandler(item);
																}
															: null}
														on:delete={() => {
															selectedItem = item;
															showDeleteConfirm = true;
														}}
													/>
												</div>
											</div>
										{/if}
									</div>

									<div class=" flex items-center gap-1 justify-between px-1.5">
										<Tooltip content={item?.description ?? item.name}>
											<div class=" flex items-center gap-2">
												<div class=" text-sm font-medium line-clamp-1 capitalize">{item.name}</div>
											</div>
										</Tooltip>

										<div class="flex items-center gap-2 shrink-0">
											<Tooltip content={dayjs(item.updated_at * 1000).format('LLLL')}>
												<div class=" text-xs text-gray-500 line-clamp-1 hidden sm:block">
													{$i18n.t('Updated')}
													{dayjs(item.updated_at * 1000).fromNow()}
												</div>
											</Tooltip>

											<div class="text-xs text-gray-500 shrink-0">
												<Tooltip
													content={item?.user?.email ?? $i18n.t('Deleted User')}
													className="flex shrink-0"
													placement="top-start"
												>
													{$i18n.t('By {{name}}', {
														name: capitalizeFirstLetter(
															item?.user?.name ?? item?.user?.email ?? $i18n.t('Deleted User')
														)
													})}
												</Tooltip>
											</div>
										</div>
									</div>
								</div>
							</div>
						</button>
					{/each}
				</div>

				{#if !allItemsLoaded}
					<div class="w-full flex justify-center py-4">
						<button
							class="px-4 py-2 rounded-xl bg-gray-100 dark:bg-gray-800 hover:bg-gray-200 dark:hover:bg-gray-700 transition font-medium text-sm flex items-center gap-2 disabled:opacity-50 disabled:cursor-not-allowed"
							on:click={() => {
								if (!itemsLoading) {
									loadMoreItems();
								}
							}}
							disabled={itemsLoading}
						>
							{#if itemsLoading}
								<Spinner className=" size-4" />
							{/if}
							<div>{$i18n.t('Load More')}</div>
						</button>
					</div>
				{/if}
			{:else}
				<div class=" w-full h-full flex flex-col justify-center items-center my-16 mb-24">
					<div class="max-w-md text-center">
						<div class=" text-3xl mb-3">😕</div>
						<div class=" text-lg font-medium mb-1">{$i18n.t('No knowledge found')}</div>
						<div class=" text-gray-500 text-center text-xs">
							{$i18n.t('Try adjusting your search or filter to find what you are looking for.')}
						</div>
					</div>
				</div>
			{/if}
		{:else}
			<div class="w-full h-full flex justify-center items-center py-10">
				<Spinner className="size-4" />
			</div>
		{/if}
	</div>

	<div class=" text-gray-500 text-xs m-2">
		ⓘ {$i18n.t("Use '#' in the prompt input to load and include your knowledge.")}
	</div>
{:else}
	<div class="w-full h-full flex justify-center items-center">
		<Spinner className="size-5" />
	</div>
{/if}
