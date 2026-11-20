<script setup lang="ts">
import { PlayIcon, PlusIcon } from '@modrinth/assets'
import {
	ContextMenu,
	defineMessages,
	injectAuth,
	injectModrinthClient,
	injectNotificationManager,
	useVIntl,
} from '@modrinth/ui'
import { useQuery, useQueryClient } from '@tanstack/vue-query'
import dayjs from 'dayjs'
import { computed, inject, ref, watch } from 'vue'
import { onBeforeRouteLeave } from 'vue-router'

import LibrarySection from '@/components/ui/library/index.vue'
import { libraryScrollTop } from '@/components/ui/library/view-state'
import WelcomeScreen from '@/components/ui/WelcomeScreen.vue'
import RecentWorldsList from '@/components/ui/world/RecentWorldsList.vue'
import { useAppSettings } from '@/composables/use-app-settings.ts'
import { useHiddenInstances } from '@/features/library/hidden-instances/use-hidden-instances'
import { prefetchSharedInstanceLibrary } from '@/features/library/shared-instances/queries'
import { traceStartupStep } from '@/helpers/startup-debug'
import { instanceListQueryOptions } from '@/pages/instance/query-options'
import { skipCreateInstanceOnboarding } from '@/platform/app-lib/onboarding-checklist/commands'
import { injectOnboardingChecklist } from '@/providers/onboarding-checklist'
import { useRootBreadcrumb } from '@/shared/breadcrumbs'

defineOptions({
	name: 'LibraryPage',
})

const { formatMessage } = useVIntl()
const { handleError } = injectNotificationManager()
const { hasCreatedInstance, isReady } = injectOnboardingChecklist()
const showCreationModal = inject<() => void>('showCreationModal')
const pageOptions = ref<InstanceType<typeof ContextMenu>>()
const appSettings = useAppSettings()
const { hiddenIds: hiddenInstanceIds } = useHiddenInstances()
const auth = injectAuth()
const modrinthClient = injectModrinthClient()
const queryClient = useQueryClient()
onBeforeRouteLeave(() => {
	libraryScrollTop.value = document.querySelector('.app-viewport')?.scrollTop ?? 0
})

const messages = defineMessages({
	home: {
		id: 'app.navigation.home',
		defaultMessage: 'Home',
	},
	newInstance: {
		id: 'app.library.context-menu.create-instance',
		defaultMessage: 'New instance',
	},
	libraryActionsLabel: {
		id: 'app.library.actions.label',
		defaultMessage: 'Library actions',
	},
})

useRootBreadcrumb({
	slot: 'root',
	id: 'home',
	label: formatMessage(messages.home),
	to: '/',
	visual: { type: 'icon', component: PlayIcon },
})

watch(
	() => (isReady.value && !hasCreatedInstance.value ? auth.user.value?.id : null),
	(userId) => {
		if (!userId) return
		prefetchSharedInstanceLibrary(queryClient, modrinthClient, userId).catch((error) =>
			console.warn('Failed to prefetch shared instance library:', error),
		)
	},
	{ immediate: true },
)

function skipWelcomeScreen() {
	skipCreateInstanceOnboarding().catch(handleError)
}

const instancesQuery = useQuery(instanceListQueryOptions())
const instances = computed(() => instancesQuery.data.value ?? [])
if (hasCreatedInstance.value) {
	await traceStartupStep('Load library instances', () => instancesQuery.suspense()).catch(
		handleError,
	)
}

const recentInstances = computed(() =>
	instances.value
		.filter((instance) => !instance.launch_blocked && !hiddenInstanceIds.value.has(instance.id))
		.sort((a, b) => dayjs(b.last_played ?? b.created).diff(dayjs(a.last_played ?? a.created))),
)

function openPageContextMenu(event: MouseEvent) {
	if (
		!(event.target instanceof HTMLElement) ||
		!event.target.hasAttribute('data-library-page-background')
	) {
		return
	}

	event.preventDefault()
	event.stopPropagation()
	pageOptions.value?.open(event, [
		{
			id: 'new_instance',
			label: formatMessage(messages.newInstance),
			icon: PlusIcon,
			action: () => showCreationModal?.(),
		},
	])
}
</script>

<template>
	<WelcomeScreen v-if="isReady && !hasCreatedInstance" @skip="skipWelcomeScreen" />
	<div
		v-else-if="isReady"
		data-library-page-background
		class="flex flex-col gap-3 p-6"
		@contextmenu="openPageContextMenu"
	>
		<section class="blazecraft-hero" aria-label="BlazeCraft Launcher">
			<div class="blazecraft-hero__content">
				<span class="blazecraft-hero__eyebrow">BLAZECRAFT · RPG MINECRAFT</span>
				<h1>Твой мир. Твоя легенда.</h1>
				<p>Приключение начинается здесь. Выбери игровую сборку в библиотеке или создай новую.</p>
				<button class="blazecraft-hero__action" type="button" @click="showCreationModal?.()">
					<PlusIcon aria-hidden="true" />
					Создать сборку
				</button>
			</div>
			<div class="blazecraft-hero__emblem" aria-hidden="true">B</div>
		</section>
		<RecentWorldsList
			v-if="recentInstances?.length > 0 && appSettings.showJumpIn"
			:recent-instances="recentInstances"
		/>
		<LibrarySection :instances="instances" />
		<ContextMenu ref="pageOptions" :label="formatMessage(messages.libraryActionsLabel)" />
	</div>
</template>

<style scoped>
.blazecraft-hero {
	position: relative;
	display: flex;
	align-items: center;
	justify-content: space-between;
	gap: 2rem;
	min-height: 230px;
	padding: 2.25rem;
	overflow: hidden;
	border: 1px solid rgba(255, 140, 53, 0.32);
	border-radius: 18px;
	background: radial-gradient(circle at 85% 15%, rgba(255, 110, 30, 0.25), transparent 40%), linear-gradient(125deg, #21110d, #100f13 65%);
	color: #fff5eb;
}
.blazecraft-hero__content { position: relative; z-index: 1; max-width: 580px; }
.blazecraft-hero__eyebrow { color: #ffad69; font-size: 0.72rem; font-weight: 800; letter-spacing: 0.18em; }
.blazecraft-hero h1 { margin: 0.65rem 0; font-size: clamp(1.7rem, 3vw, 2.6rem); font-weight: 900; line-height: 1.12; }
.blazecraft-hero p { max-width: 460px; margin: 0 0 1.4rem; color: #e6c7b5; line-height: 1.5; }
.blazecraft-hero__action { display: inline-flex; align-items: center; gap: 0.6rem; padding: 0.75rem 1.15rem; border: 0; border-radius: 10px; background: #f47728; color: #1d0d06; font-weight: 800; cursor: pointer; }
.blazecraft-hero__action:hover { background: #ffa45c; }
.blazecraft-hero__action:focus-visible { outline: 3px solid #fff; outline-offset: 3px; }
.blazecraft-hero__action svg { width: 1.1rem; height: 1.1rem; }
.blazecraft-hero__emblem { flex: none; width: 140px; height: 140px; display: grid; place-items: center; border: 3px solid rgba(255, 147, 64, 0.45); border-radius: 30px; background: rgba(245, 104, 29, 0.1); box-shadow: 0 0 65px rgba(250, 92, 21, 0.15); color: #ff873d; font-size: 6rem; font-weight: 900; transform: rotate(-8deg); }
@media (max-width: 700px) { .blazecraft-hero { padding: 1.5rem; } .blazecraft-hero__emblem { display: none; } }
</style>
