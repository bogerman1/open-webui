<script lang="ts">
	import { marked } from 'marked';
	import Fuse from 'fuse.js';

	import dayjs from '$lib/dayjs';
	import relativeTime from 'dayjs/plugin/relativeTime';
	dayjs.extend(relativeTime);

	import Spinner from '$lib/components/common/Spinner.svelte';
	import ConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';
	import { flyAndScale } from '$lib/utils/transitions';

	import { createEventDispatcher, onMount, getContext, tick } from 'svelte';
	import { goto } from '$app/navigation';

	import { deleteModel, getOllamaVersion, pullModel, unloadModel } from '$lib/apis/ollama';

	import {
		user,
		MODEL_DOWNLOAD_POOL,
		models,
		mobile,
		temporaryChatEnabled,
		settings,
		config
	} from '$lib/stores';
	import { toast } from 'svelte-sonner';
	import { capitalizeFirstLetter, sanitizeResponseContent, splitStream } from '$lib/utils';
	import { getModels } from '$lib/apis';

	import ChevronDown from '$lib/components/icons/ChevronDown.svelte';
	import Check from '$lib/components/icons/Check.svelte';
	import Search from '$lib/components/icons/Search.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Switch from '$lib/components/common/Switch.svelte';
	import ChatBubbleOval from '$lib/components/icons/ChatBubbleOval.svelte';

	import ModelItem from './ModelItem.svelte';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	export let id = '';
	export let value = '';
	export let placeholder = $i18n.t('Select a model');
	export let searchEnabled = true;
	export let searchPlaceholder = $i18n.t('Search a model');

	export let items: {
		label: string;
		value: string;
		model: Model;
		// eslint-disable-next-line @typescript-eslint/no-explicit-any
		[key: string]: any;
	}[] = [];

	export let className = 'w-[32rem]';
	export let triggerClassName = 'text-lg';

	export let pinModelHandler: (modelId: string) => void = () => {};

	let tagsContainerElement;

	let show = false;
	let triggerElement: HTMLElement | null = null;
	let contentElement: HTMLElement | null = null;
	let dropdownPosition = { top: 0, left: 0, width: 0 };

	const portal = (node: HTMLElement) => {
		document.body.appendChild(node);
		return {
			destroy() {
				node.remove();
			}
		};
	};

	const updatePosition = () => {
		if (!show || !triggerElement) return;
		const rect = triggerElement.getBoundingClientRect();
		dropdownPosition = {
			top: rect.bottom + 2,
			left: $mobile ? 8 : rect.left,
			width: $mobile ? window.innerWidth - 16 : 0
		};
	};

	const toggleOpen = () => {
		show = !show;
		if (show) {
			searchValue = '';
			listScrollTop = 0;
			resetView();
			updatePosition();
			window.setTimeout(() => document.getElementById('model-search-input')?.focus(), 0);
		} else {
			document.getElementById(`model-selector-${id}-button`)?.blur();
		}
	};

	const handlePointerDown = (e: PointerEvent) => {
		if (!show) return;
		const target = e.target as Node;
		if (
			(triggerElement && triggerElement.contains(target)) ||
			(contentElement && contentElement.contains(target))
		) {
			return;
		}
		show = false;
		document.getElementById(`model-selector-${id}-button`)?.blur();
	};

	const handleKeydown = (e: KeyboardEvent) => {
		if (show && e.key === 'Escape') {
			e.preventDefault();
			e.stopPropagation();
			show = false;
			document.getElementById(`model-selector-${id}-button`)?.blur();
		}
	};

	let tags = [];

	let selectedModel = '';
	$: selectedModel = items.find((item) => item.value === value) ?? '';

	let searchValue = '';

	let selectedTag = '';
	let selectedConnectionType = '';

	let ollamaVersion = null;
	let selectedProvider = "";
	let selectedModelIdx = 0;

	const fuse = new Fuse(
		items.map((item) => {
			const _item = {
				...item,
				modelName: item.model?.name,
				tags: (item.model?.tags ?? []).map((tag) => tag.name).join(' '),
				desc: item.model?.info?.meta?.description
			};
			return _item;
		}),
		{
			keys: ['value', 'tags', 'modelName'],
			threshold: 0.4
		}
	);

	const updateFuse = () => {
		if (fuse) {
			fuse.setCollection(
				items.map((item) => {
					const _item = {
						...item,
						modelName: item.model?.name,
						tags: (item.model?.tags ?? []).map((tag) => tag.name).join(' '),
						desc: item.model?.info?.meta?.description
					};
					return _item;
				})
			);
		}
	};

	$: if (items) {
		updateFuse();
	}

	$: filteredItems = (
		searchValue
			? fuse
					.search(searchValue)
					.map((e) => {
						return e.item;
					})
					.filter((item) => {
						if (selectedTag === '') {
							return true;
						}

						return (item.model?.tags ?? [])
							.map((tag) => tag.name.toLowerCase())
							.includes(selectedTag.toLowerCase());
					})
					.filter((item) => {
						if (selectedConnectionType === '') {
							return true;
						} else if (selectedConnectionType === 'local') {
							return item.model?.connection_type === 'local';
						} else if (selectedConnectionType === 'external') {
							return item.model?.connection_type === 'external';
						} else if (selectedConnectionType === 'direct') {
							return item.model?.direct;
						}
					})
			: items
					.filter((item) => {
						if (selectedTag === '') {
							return true;
						}
						return (item.model?.tags ?? [])
							.map((tag) => tag.name.toLowerCase())
							.includes(selectedTag.toLowerCase());
					})
					.filter((item) => {
						if (selectedConnectionType === '') {
							return true;
						} else if (selectedConnectionType === 'local') {
							return item.model?.connection_type === 'local';
						} else if (selectedConnectionType === 'external') {
							return item.model?.connection_type === 'external';
						} else if (selectedConnectionType === 'direct') {
							return item.model?.direct;
						}
					})
	).filter((item) => !(item.model?.info?.meta?.hidden ?? false));

	// Group models by provider when not searching (flat list when searching)
	$: displayItems = (() => {
		if (searchValue) return filteredItems;
		const result: any[] = [];
		let prevProvider = '';
		for (const item of filteredItems) {
			const parts = (item.value || '').split('/');
			const provider = parts.length >= 2 ? parts.slice(0, -1).join(' / ') : (item.model?.owned_by || 'Other');
			if (provider !== prevProvider) {
				prevProvider = provider;
				result.push({ _groupHeader: provider });
			}
			result.push(item);
		}
		return result;
	})();

	$: if (
		selectedTag !== undefined ||
		selectedConnectionType !== undefined ||
		searchValue !== undefined
	) {
		resetView();
	}

	const resetView = async () => {
		await tick();

		const selectedInFiltered = displayItems.findIndex((item) => item.value === value);

		if (selectedInFiltered >= 0) {
			// The selected model is visible in the current filter
			selectedModelIdx = selectedInFiltered;
		} else {
			// The selected model is not visible, default to first item in filtered list
			selectedModelIdx = 0;
		}

		// Set the virtual scroll position so the selected item is rendered and centered
		const targetScrollTop = Math.max(0, selectedModelIdx * ITEM_HEIGHT - 128 + ITEM_HEIGHT / 2);
		listScrollTop = targetScrollTop;

		await tick();

		if (listContainer) {
			listContainer.scrollTop = targetScrollTop;
		}

		await tick();
		const item = document.querySelector(`[data-arrow-selected="true"]`);
		item?.scrollIntoView({ block: 'center', inline: 'nearest', behavior: 'instant' });
	};

	const pullModelHandler = async () => {
		const sanitizedModelTag = searchValue.trim().replace(/^ollama\s+(run|pull)\s+/, '');

		console.log($MODEL_DOWNLOAD_POOL);
		if ($MODEL_DOWNLOAD_POOL[sanitizedModelTag]) {
			toast.error(
				$i18n.t(`Model '{{modelTag}}' is already in queue for downloading.`, {
					modelTag: sanitizedModelTag
				})
			);
			return;
		}
		if (Object.keys($MODEL_DOWNLOAD_POOL).length === 3) {
			toast.error(
				$i18n.t('Maximum of 3 models can be downloaded simultaneously. Please try again later.')
			);
			return;
		}

		const [res, controller] = await pullModel(localStorage.token, sanitizedModelTag, '0').catch(
			(error) => {
				toast.error(`${error}`);
				return null;
			}
		);

		if (res) {
			const reader = res.body
				.pipeThrough(new TextDecoderStream())
				.pipeThrough(splitStream('\n'))
				.getReader();

			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL,
				[sanitizedModelTag]: {
					...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
					abortController: controller,
					reader,
					done: false
				}
			});

			while (true) {
				try {
					const { value, done } = await reader.read();
					if (done) break;

					let lines = value.split('\n');

					for (const line of lines) {
						if (line !== '') {
							let data = JSON.parse(line);
							console.log(data);
							if (data.error) {
								throw data.error;
							}
							if (data.detail) {
								throw data.detail;
							}

							if (data.status) {
								if (data.digest) {
									let downloadProgress = 0;
									if (data.completed) {
										downloadProgress = Math.round((data.completed / data.total) * 1000) / 10;
									} else {
										downloadProgress = 100;
									}

									MODEL_DOWNLOAD_POOL.set({
										...$MODEL_DOWNLOAD_POOL,
										[sanitizedModelTag]: {
											...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
											pullProgress: downloadProgress,
											digest: data.digest
										}
									});
								} else {
									toast.success(data.status);

									MODEL_DOWNLOAD_POOL.set({
										...$MODEL_DOWNLOAD_POOL,
										[sanitizedModelTag]: {
											...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
											done: data.status === 'success'
										}
									});
								}
							}
						}
					}
				} catch (error) {
					console.log(error);
					if (typeof error !== 'string') {
						error = error.message;
					}

					toast.error(`${error}`);
					// opts.callback({ success: false, error, modelName: opts.modelName });
					break;
				}
			}

			if ($MODEL_DOWNLOAD_POOL[sanitizedModelTag].done) {
				toast.success(
					$i18n.t(`Model '{{modelName}}' has been successfully downloaded.`, {
						modelName: sanitizedModelTag
					})
				);

				models.set(
					await getModels(
						localStorage.token,
						$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
					)
				);
			} else {
				toast.error($i18n.t('Download canceled'));
			}

			delete $MODEL_DOWNLOAD_POOL[sanitizedModelTag];

			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL
			});
		}
	};

	const setOllamaVersion = async () => {
		ollamaVersion = await getOllamaVersion(localStorage.token).catch((error) => false);
	};

	onMount(async () => {
		if (items) {
			tags = items
				.filter((item) => !(item.model?.info?.meta?.hidden ?? false))
				.flatMap((item) => item.model?.tags ?? [])
				.map((tag) => tag.name.toLowerCase());
			// Remove duplicates and sort
			tags = Array.from(new Set(tags)).sort((a, b) => a.localeCompare(b));
		}
	});

	$: if (show) {
		setOllamaVersion();
	}

	const cancelModelPullHandler = async (model: string) => {
		const { reader, abortController } = $MODEL_DOWNLOAD_POOL[model];
		if (abortController) {
			abortController.abort();
		}
		if (reader) {
			await reader.cancel();
			delete $MODEL_DOWNLOAD_POOL[model];
			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL
			});
			await deleteModel(localStorage.token, model);
			toast.success($i18n.t('{{model}} download has been canceled', { model: model }));
		}
	};

	const unloadModelHandler = async (model: string) => {
		const res = await unloadModel(localStorage.token, model).catch((error) => {
			toast.error($i18n.t('Error unloading model: {{error}}', { error }));
		});

		if (res) {
			toast.success($i18n.t('Model unloaded successfully'));
			models.set(
				await getModels(
					localStorage.token,
					$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
				)
			);
		}
	};

	let showDeleteConfirm = false;
	let deleteModelTarget: any = null;

	const deleteModelHandler = async (model: any) => {
		deleteModelTarget = model;
		showDeleteConfirm = true;
	};

	const confirmDeleteModel = async () => {
		const model = deleteModelTarget;
		if (!model) return;

		const res = await deleteModel(localStorage.token, model.id).catch((error) => {
			toast.error($i18n.t('Error deleting model: {{error}}', { error }));
		});

		if (res) {
			// $i18n.t('Model {{modelId}} not found')
			toast.success(
				$i18n.t('Model {{modelName}} deleted successfully', { modelName: model.name ?? model.id })
			);

			// If the deleted model was selected, clear the selection
			if (value === model.id) {
				value = '';
			}

			models.set(
				await getModels(
					localStorage.token,
					$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
				)
			);
		}

		deleteModelTarget = null;
	};

	const ITEM_HEIGHT = 42;
	const OVERSCAN = 10;

	let listScrollTop = 0;
	let listContainer;

	$: visibleStart = Math.max(0, Math.floor(listScrollTop / ITEM_HEIGHT) - OVERSCAN);
	$: visibleEnd = Math.min(
		displayItems.length,
		Math.ceil((listScrollTop + 256) / ITEM_HEIGHT) + OVERSCAN
	);
</script>

<ConfirmDialog
	bind:show={showDeleteConfirm}
	title={$i18n.t('Delete Model')}
	message={$i18n.t('Are you sure you want to delete **{{modelName}}**?', {
		modelName: deleteModelTarget?.name ?? deleteModelTarget?.id ?? ''
	})}
	on:confirm={() => {
		confirmDeleteModel();
	}}
/>

<svelte:window
	on:pointerdown={handlePointerDown}
	on:keydown={handleKeydown}
	on:resize={updatePosition}
/>

<div class="relative w-full">
	<button
		bind:this={triggerElement}
		class="relative w-full {($settings?.highContrastMode ?? false)
			? ''
			: 'outline-hidden focus:outline-hidden'}"
		aria-label={selectedModel
			? $i18n.t('Selected model: {{modelName}}', { modelName: selectedModel.label })
			: placeholder}
		aria-haspopup="listbox"
		aria-expanded={show}
		id="model-selector-{id}-button"
		type="button"
		on:click={toggleOpen}
	>
		<div
			class="flex w-full text-left px-0.5 bg-transparent truncate {triggerClassName} justify-between {($settings?.highContrastMode ??
			false)
				? 'dark:placeholder-gray-100 placeholder-gray-800'
				: 'placeholder-gray-400'}"
			on:mouseenter={async () => {
				models.set(
					await getModels(
						localStorage.token,
						$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
					)
				);
			}}
		>
			{#if selectedModel}
				{selectedModel.label}
			{:else}
				{placeholder}
			{/if}
			<ChevronDown className=" self-center ml-2 size-3" strokeWidth="2.5" />
		</div>
	</button>

	{#if show}
		<div
			use:portal
			bind:this={contentElement}
			style="position: fixed; z-index: 9999; top: {dropdownPosition.top}px; left: {dropdownPosition.left}px;{$mobile
				? ` width: ${dropdownPosition.width}px;`
				: ''}"
		>
			<div
				class="z-40 {$mobile
					? `w-full`
					: `${className}`} max-w-[calc(100vw-1rem)] justify-start rounded-2xl bg-white dark:bg-gray-850 dark:text-white shadow-lg outline-hidden"
				transition:flyAndScale
			>
				<slot>
					{#if searchEnabled}
						<div class="flex items-center gap-2.5 px-4 pt-3.5 mb-1">
							<Search className="size-4 shrink-0" strokeWidth="2.5" />
							<input
								id="model-search-input"
								bind:value={searchValue}
								class="w-full text-sm bg-transparent outline-hidden"
								placeholder={searchPlaceholder}
								autocomplete="off"
								aria-label={$i18n.t('Search In Models')}
							/>
						</div>
					{/if}

					{#if !searchValue}
						<!-- Two-panel layout -->
						{@const groups = (() => {
							const g: { name: string; items: any[] }[] = [];
							let cur = '';
							for (const d of displayItems) {
								if ((d as any)._groupHeader) {
									cur = (d as any)._groupHeader;
									g.push({ name: cur, items: [] });
								} else if (cur) {
									g[g.length - 1].items.push(d);
								}
							}
							return g;
						})()}
						{@const activeModels = groups.find(g => g.name === selectedProvider)?.items || []}

						{#if groups.length === 0}
							<div class="px-4 py-6 text-center text-sm text-gray-500">
								{$i18n.t('No results found')}
							</div>
						{:else}
							<div class="flex" style="height: 320px;">
								<!-- Left sidebar: providers -->
								<aside class="w-48 flex-shrink-0 overflow-y-auto border-r border-gray-200 dark:border-gray-700 bg-gray-50/60 dark:bg-gray-900/50 p-2 flex flex-col gap-1">
									{#each groups as group}
										<button
											class="w-full text-left px-3 py-2.5 rounded-xl text-sm font-medium transition-all duration-200 {selectedProvider === group.name
												? 'bg-indigo-500/10 text-indigo-600 dark:text-indigo-400 font-semibold'
												: 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-800'}"
											on:click={() => selectedProvider = group.name}
										>
											<div class="truncate">{group.name}</div>
											<div class="text-[0.65rem] text-gray-400 dark:text-gray-500 font-normal">{group.items.length}</div>
										</button>
									{/each}
								</aside>

								<!-- Right panel: model list -->
								<section class="flex-1 overflow-y-auto p-2">
									{#if activeModels.length === 0}
										<div class="px-3 py-6 text-center text-sm text-gray-500">Select a provider</div>
									{:else}
										{#each activeModels as item, i (item.value)}
											<ModelItem
												{selectedModelIdx}
												{item}
												index={i}
												{value}
												{pinModelHandler}
												{unloadModelHandler}
												{deleteModelHandler}
												onClick={() => { value = item.value; show = false; }}
											/>
										{/each}
									{/if}
								</section>
							</div>
						{/if}
					{:else}
						<!-- Search results: flat list -->
						<div class="max-h-64 overflow-y-auto px-1">
							{#each displayItems.filter((d: any) => !d._groupHeader) as item, i (item.value)}
								<ModelItem
									{selectedModelIdx}
									{item}
									index={i}
									{value}
									{pinModelHandler}
									{unloadModelHandler}
									{deleteModelHandler}
									onClick={() => { value = item.value; show = false; }}
								/>
							{/each}
							{#if displayItems.filter((d: any) => !d._groupHeader).length === 0}
								<div class="px-3 py-6 text-center text-sm text-gray-500">{$i18n.t('No results found')}</div>
							{/if}
						</div>
					{/if}

						{#if !(searchValue.trim() in $MODEL_DOWNLOAD_POOL) && searchValue && ollamaVersion && $user?.role === 'admin'}
							<Tooltip
								content={$i18n.t(`Pull "{{searchValue}}" from Ollama.com`, {
									searchValue: searchValue
								})}
								placement="top-start"
							>
								<button
									class="flex w-full font-medium line-clamp-1 select-none items-center rounded-button py-2 pl-3 pr-1.5 text-sm text-gray-700 dark:text-gray-100 outline-hidden transition-all duration-75 hover:bg-gray-100 dark:hover:bg-gray-800 rounded-xl cursor-pointer data-highlighted:bg-muted"
									on:click={() => {
										pullModelHandler();
									}}
								>
									<div class=" truncate">
										{$i18n.t(`Pull "{{searchValue}}" from Ollama.com`, {
											searchValue: searchValue
										})}
									</div>
								</button>
							</Tooltip>
						{/if}

						{#each Object.keys($MODEL_DOWNLOAD_POOL) as model}
							<div
								class="flex w-full justify-between font-medium select-none rounded-button py-2 pl-3 pr-1.5 text-sm text-gray-700 dark:text-gray-100 outline-hidden transition-all duration-75 rounded-xl cursor-pointer data-highlighted:bg-muted"
							>
								<div class="flex">
									<div class="mr-2.5 translate-y-0.5">
										<Spinner />
									</div>

									<div class="flex flex-col self-start">
										<div class="flex gap-1">
											<div class="line-clamp-1">
												Downloading "{model}"
											</div>

											<div class="shrink-0">
												{'pullProgress' in $MODEL_DOWNLOAD_POOL[model]
													? `(${$MODEL_DOWNLOAD_POOL[model].pullProgress}%)`
													: ''}
											</div>
										</div>

										{#if 'digest' in $MODEL_DOWNLOAD_POOL[model] && $MODEL_DOWNLOAD_POOL[model].digest}
											<div class="-mt-1 h-fit text-[0.7rem] dark:text-gray-500 line-clamp-1">
												{$MODEL_DOWNLOAD_POOL[model].digest}
											</div>
										{/if}
									</div>
								</div>

								<div class="mr-2 ml-1 translate-y-0.5">
									<Tooltip content={$i18n.t('Cancel')}>
										<button
											class="text-gray-800 dark:text-gray-100"
											aria-label={$i18n.t('Cancel download of {{model}}', { model: model })}
											on:click={() => {
												cancelModelPullHandler(model);
											}}
										>
											<svg
												class="w-4 h-4 text-gray-800 dark:text-white"
												aria-hidden="true"
												xmlns="http://www.w3.org/2000/svg"
												width="24"
												height="24"
												fill="currentColor"
												viewBox="0 0 24 24"
											>
												<path
													stroke="currentColor"
													stroke-linecap="round"
													stroke-linejoin="round"
													stroke-width="2"
													d="M6 18 17.94 6M18 18 6.06 6"
												/>
											</svg>
										</button>
									</Tooltip>
								</div>
							</div>
						{/each}
					</div>

					<div class="pb-2.5"></div>

					<div class="hidden w-[42rem]" />
					<div class="hidden w-[32rem]" />
				</slot>
			</div>
		</div>
	{/if}
</div>
