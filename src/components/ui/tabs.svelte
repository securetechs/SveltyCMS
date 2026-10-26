<!--
@file src/components/ui/tabs.svelte
@component
**Reusable tab navigation component — uses native `<Button>` for all tabs**

### Props
- `tabs` (Array<{id: string, label: string, icon?: string, shortLabel?: string}>) - Tab definitions
- `activeTab` (string) - Currently active tab ID
- `onTabChange` ((id: string) => void) - Tab selection handler
- `variant` ('default' | 'pills' | 'underline') - Visual style (default: 'underline')
- `ariaLabel` (string) - Accessible label for the tablist (default: 'Navigation tabs')
- `testId` (string) - Data-testid for the root element (default: 'tab-navigation')

### Features
- Keyboard-navigable (Arrow keys, Home, End)
- ARIA-compliant (role="tablist", role="tab", role="tabpanel")
- Native `<Button>` component for consistent theming
- Smooth transitions between tabs
- Responsive: horizontal scroll on mobile
- Support for icons + labels + short labels for mobile
-->

<script lang="ts">
	import Button from './button.svelte';

	interface Tab {
		id: string;
		label: string;
		icon?: string;
		shortLabel?: string;
		/** Step number (1, 2, 3...) */
		step?: number | string;
		/** Show a success checkmark (e.g. completed wizard step). */
		done?: boolean;
	}

	let {
		tabs,
		activeTab = $bindable(''),
		onTabChange = (_id: string) => {},
		variant = 'underline',
		ariaLabel = 'Navigation tabs',
		testId = 'tab-navigation'
	}: {
		tabs: Tab[];
		activeTab?: string;
		onTabChange?: (id: string) => void;
		variant?: 'default' | 'pills' | 'underline';
		ariaLabel?: string;
		testId?: string;
	} = $props();

	function handleTabClick(id: string) {
		activeTab = id;
		onTabChange(id);
	}

	function handleKeyDown(e: KeyboardEvent, index: number) {
		let newIndex = index;
		if (e.key === 'ArrowRight') newIndex = (index + 1) % tabs.length;
		else if (e.key === 'ArrowLeft') newIndex = (index - 1 + tabs.length) % tabs.length;
		else if (e.key === 'Home') newIndex = 0;
		else if (e.key === 'End') newIndex = tabs.length - 1;
		else return;

		e.preventDefault();
		handleTabClick(tabs[newIndex].id);
		// Focus the new tab button
		const tablist = (e.target as HTMLElement).closest('[role="tablist"]');
		if (tablist) {
			const buttons = tablist.querySelectorAll('[role="tab"]');
			(buttons[newIndex] as HTMLElement)?.focus();
		}
	}
</script>

<div
	class="w-full {variant === 'underline' ? 'border-b-2 border-solid border-black dark:border-white' : ''}"
	data-testid={testId}
>
	<div
		role="tablist"
		aria-label={ariaLabel}
		class="flex w-full {variant === 'pills'
			? 'gap-2 p-1 bg-(--admin-bg-card) rounded-xl'
			: variant === 'underline'
				? 'gap-0.5 overflow-visible sm:gap-2'
				: 'gap-2'} {variant === 'underline' ? '' : 'overflow-x-auto'}"
	>
		{#each tabs as tab, i (tab.id)}
			<!-- Underline tabs are chrome-free; `ghost` paints a dark-mode fill behind each tab -->
			<Button
				variant={variant === 'underline' ? 'transparent' : 'ghost'}
				role="tab"
				aria-selected={activeTab === tab.id}
				aria-controls="tabpanel-{tab.id}"
				tabindex={activeTab === tab.id ? 0 : -1}
				onclick={() => handleTabClick(tab.id)}
				onkeydown={(e: KeyboardEvent) => handleKeyDown(e, i)}
				data-testid="tab-{tab.id}"
				class="flex min-w-0 flex-1 items-center justify-center gap-1.5 whitespace-nowrap px-2 py-3 text-sm font-medium transition-all duration-200 sm:gap-2.5 sm:px-4 {variant ===
				'pills'
					? 'rounded-lg! flex-none ' +
						(activeTab === tab.id
							? 'bg-white dark:bg-surface-700 shadow-sm text-(--admin-text-body)'
							: 'text-surface-500 hover:text-surface-600 dark:hover:text-surface-400')
					: variant === 'underline'
						? 'rounded-none! border-0 bg-transparent shadow-none ' +
							(activeTab === tab.id
								? 'relative text-tertiary-600 after:absolute after:inset-x-0 after:-bottom-0.5 after:z-10 after:h-0.5 after:bg-tertiary-500 after:content-[\'\'] dark:text-primary-500 dark:after:bg-primary-500'
								: 'text-surface-500 hover:text-surface-600 dark:hover:text-surface-400')
						: 'rounded-lg! flex-none ' +
							(activeTab === tab.id
								? 'bg-tertiary-500/10 text-tertiary-600 dark:bg-primary-900/20 dark:text-primary-400'
								: 'text-surface-500 hover:text-surface-600 dark:hover:text-surface-400')}"
			>
				{#if tab.step !== undefined}
					<span
						class="flex h-5 w-5 shrink-0 items-center justify-center rounded-full text-[11px] font-bold transition-all {activeTab ===
						tab.id
							? 'bg-tertiary-500 text-white dark:bg-primary-500 shadow-2xs'
							: tab.done
								? 'bg-success-500/20 text-success-600 dark:text-success-400'
								: 'bg-surface-500/20 text-(--admin-text-muted)'}"
					>
						{tab.step}
					</span>
				{/if}
				{#if tab.icon}
					<iconify-icon icon={tab.icon} width="18" height="18" aria-hidden="true"></iconify-icon>
				{/if}
				<span class="hidden md:inline">{tab.label}</span>
				{#if tab.shortLabel}
					<span class="md:hidden">{tab.shortLabel}</span>
				{/if}
				{#if tab.done}
					<iconify-icon
						icon="mdi:check-circle"
						width="16"
						class="text-success-500"
						aria-hidden="true"
					></iconify-icon>
				{/if}
			</Button>
		{/each}
	</div>
</div>
