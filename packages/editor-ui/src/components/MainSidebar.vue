<template>
	<div
		id="side-menu"
		:class="{
			['side-menu']: true,
			[$style.sideMenu]: true,
			[$style.sideMenuCollapsed]: isCollapsed,
		}"
	>
		<div
			id="collapse-change-button"
			:class="['clickable', $style.sideMenuCollapseButton]"
			@click="toggleCollapse"
		>
			<n8n-icon v-if="isCollapsed" icon="chevron-right" size="xsmall" class="ml-5xs" />
			<n8n-icon v-else icon="chevron-left" size="xsmall" class="mr-5xs" />
		</div>
		<n8n-menu :items="mainMenuItems" :collapsed="isCollapsed" @select="handleSelect">
			<template #header>
				<div :class="$style.logo">
					<img :src="logoPath" data-test-id="n8n-logo" :class="$style.icon" alt="n8n" />
				</div>
			</template>

			<template #beforeLowerMenu>
				<BecomeTemplateCreatorCta v-if="fullyExpanded && !userIsTrialing" />
				<ExecutionsUsage
					v-if="fullyExpanded && userIsTrialing"
					:cloud-plan-data="currentPlanAndUsageData"
			/></template>
			<template #menuSuffix>
				<div>
					<div
						v-if="hasVersionUpdates"
						data-test-id="version-updates-panel-button"
						:class="$style.updates"
						@click="openUpdatesPanel"
					>
						<div :class="$style.giftContainer">
							<GiftNotificationIcon />
						</div>
						<n8n-text
							:class="{ ['ml-xs']: true, [$style.expanded]: fullyExpanded }"
							color="text-base"
						>
							{{ nextVersions.length > 99 ? '99+' : nextVersions.length }} update{{
								nextVersions.length > 1 ? 's' : ''
							}}
						</n8n-text>
					</div>
					<MainSidebarSourceControl :is-collapsed="isCollapsed" />
				</div>
			</template>
			<template v-if="showUserArea" #footer>
				<div :class="$style.userArea">
					<div class="ml-3xs" data-test-id="main-sidebar-user-menu">
						<!-- This dropdown is only enabled when sidebar is collapsed -->
						<el-dropdown
							:disabled="!isCollapsed"
							placement="right-end"
							trigger="click"
							@command="onUserActionToggle"
						>
							<div :class="{ [$style.avatar]: true, ['clickable']: isCollapsed }">
								<n8n-avatar
									:first-name="usersStore.currentUser.firstName"
									:last-name="usersStore.currentUser.lastName"
									size="small"
								/>
							</div>
							<template #dropdown>
								<el-dropdown-menu>
									<el-dropdown-item command="settings">
										{{ $locale.baseText('settings') }}
									</el-dropdown-item>
									<el-dropdown-item command="logout">
										{{ $locale.baseText('auth.signout') }}
									</el-dropdown-item>
								</el-dropdown-menu>
							</template>
						</el-dropdown>
					</div>
					<div
						:class="{ ['ml-2xs']: true, [$style.userName]: true, [$style.expanded]: fullyExpanded }"
					>
						<n8n-text size="small" :bold="true" color="text-dark">{{
							usersStore.currentUser.fullName
						}}</n8n-text>
					</div>
					<div :class="{ [$style.userActions]: true, [$style.expanded]: fullyExpanded }">
						<n8n-action-dropdown
							:items="userMenuItems"
							placement="top-start"
							data-test-id="user-menu"
							@select="onUserActionToggle"
						/>
					</div>
				</div>
			</template>
		</n8n-menu>
	</div>
</template>

<script lang="ts">
import type { CloudPlanAndUsageData, IExecutionResponse, IMenuItem, IVersion } from '@/Interface';
import GiftNotificationIcon from './GiftNotificationIcon.vue';

import { useMessage } from '@/composables/useMessage';
import { ABOUT_MODAL_KEY, VERSIONS_MODAL_KEY, VIEWS } from '@/constants';
import { userHelpers } from '@/mixins/userHelpers';
import { defineComponent } from 'vue';
import { mapStores } from 'pinia';
import { useCloudPlanStore } from '@/stores/cloudPlan.store';
import { useRootStore } from '@/stores/n8nRoot.store';
import { useSettingsStore } from '@/stores/settings.store';
import { useSourceControlStore } from '@/stores/sourceControl.store';
import { useUIStore } from '@/stores/ui.store';
import { useUsersStore } from '@/stores/users.store';
import { useVersionsStore } from '@/stores/versions.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import { isNavigationFailure } from 'vue-router';
import ExecutionsUsage from '@/components/ExecutionsUsage.vue';
import BecomeTemplateCreatorCta from '@/components/BecomeTemplateCreatorCta/BecomeTemplateCreatorCta.vue';
import MainSidebarSourceControl from '@/components/MainSidebarSourceControl.vue';
import { hasPermission } from '@/rbac/permissions';
import { useExternalHooks } from '@/composables/useExternalHooks';
import { useDebounce } from '@/composables/useDebounce';
import { useBecomeTemplateCreatorStore } from '@/components/BecomeTemplateCreatorCta/becomeTemplateCreatorStore';

export default defineComponent({
	name: 'MainSidebar',
	components: {
		GiftNotificationIcon,
		ExecutionsUsage,
		MainSidebarSourceControl,
		BecomeTemplateCreatorCta,
	},
	mixins: [userHelpers],
	setup(props, ctx) {
		const externalHooks = useExternalHooks();
		const { callDebounced } = useDebounce();

		return {
			externalHooks,
			callDebounced,
			...useMessage(),
		};
	},
	data() {
		return {
			basePath: '',
			fullyExpanded: false,
		};
	},
	computed: {
		...mapStores(
			useRootStore,
			useSettingsStore,
			useUIStore,
			useUsersStore,
			useVersionsStore,
			useWorkflowsStore,
			useCloudPlanStore,
			useSourceControlStore,
			useBecomeTemplateCreatorStore,
		),
		logoPath(): string {
			if (this.isCollapsed) return this.basePath + 'n8n-logo-collapsed.svg';

			return this.basePath + this.uiStore.logo;
		},
		hasVersionUpdates(): boolean {
			return (
				this.settingsStore.settings.releaseChannel === 'stable' &&
				this.versionsStore.hasVersionUpdates
			);
		},
		nextVersions(): IVersion[] {
			return this.versionsStore.nextVersions;
		},
		isCollapsed(): boolean {
			return this.uiStore.sidebarMenuCollapsed;
		},
		canUserAccessSettings(): boolean {
			const accessibleRoute = this.findFirstAccessibleSettingsRoute();
			return accessibleRoute !== null;
		},
		showUserArea(): boolean {
			return hasPermission(['authenticated']);
		},
		workflowExecution(): IExecutionResponse | null {
			return this.workflowsStore.getWorkflowExecution;
		},
		userMenuItems(): object[] {
			return [
				{
					id: 'settings',
					label: this.$locale.baseText('settings'),
				},
				{
					id: 'logout',
					label: this.$locale.baseText('auth.signout'),
				},
			];
		},
		mainMenuItems(): IMenuItem[] {
			const items: IMenuItem[] = [];
			const injectedItems = this.uiStore.sidebarMenuItems;

			const workflows: IMenuItem = {
				id: 'workflows',
				icon: 'network-wired',
				label: this.$locale.baseText('mainSidebar.workflows'),
				position: 'top',
				activateOnRouteNames: [VIEWS.WORKFLOWS],
			};

			if (this.sourceControlStore.preferences.branchReadOnly) {
				workflows.secondaryIcon = {
					name: 'lock',
					tooltip: {
						content: this.$locale.baseText('mainSidebar.workflows.readOnlyEnv.tooltip'),
					},
				};
			}

			if (injectedItems && injectedItems.length > 0) {
				for (const item of injectedItems) {
					items.push({
						id: item.id,
						icon: item.icon || '',
						label: item.label || '',
						position: item.position,
						type: item.properties?.href ? 'link' : 'regular',
						properties: item.properties,
					} as IMenuItem);
				}
			}

			const regularItems: IMenuItem[] = [
				workflows,
				{
					id: 'templates',
					icon: 'box-open',
					label: this.$locale.baseText('mainSidebar.templates'),
					position: 'top',
					available: this.settingsStore.isTemplatesEnabled,
					activateOnRouteNames: [VIEWS.TEMPLATES],
				},
				{
					id: 'credentials',
					icon: 'key',
					label: this.$locale.baseText('mainSidebar.credentials'),
					customIconSize: 'medium',
					position: 'top',
					activateOnRouteNames: [VIEWS.CREDENTIALS],
				},
				{
					id: 'variables',
					icon: 'variable',
					label: this.$locale.baseText('mainSidebar.variables'),
					customIconSize: 'medium',
					position: 'top',
					activateOnRouteNames: [VIEWS.VARIABLES],
				},
				{
					id: 'executions',
					icon: 'tasks',
					label: this.$locale.baseText('mainSidebar.executions'),
					position: 'top',
					activateOnRouteNames: [VIEWS.EXECUTIONS],
				},
				{
					id: 'cloud-admin',
					type: 'link',
					position: 'bottom',
					label: 'Admin Panel',
					icon: 'home',
					available: this.settingsStore.isCloudDeployment && hasPermission(['instanceOwner']),
				},
				{
					id: 'settings',
					icon: 'cog',
					label: this.$locale.baseText('settings'),
					position: 'bottom',
					available: this.canUserAccessSettings && this.usersStore.currentUser !== null,
					activateOnRouteNames: [VIEWS.USERS_SETTINGS, VIEWS.API_SETTINGS, VIEWS.PERSONAL_SETTINGS],
				},
				{
					id: 'help',
					icon: 'question',
					label: 'Help',
					position: 'bottom',
					children: [
						{
							id: 'quickstart',
							icon: 'video',
							label: this.$locale.baseText('mainSidebar.helpMenuItems.quickstart'),
							type: 'link',
							properties: {
								href: 'https://www.youtube.com/watch?v=1MwSoB0gnM4',
								newWindow: true,
							},
						},
						{
							id: 'docs',
							icon: 'book',
							label: this.$locale.baseText('mainSidebar.helpMenuItems.documentation'),
							type: 'link',
							properties: {
								href: 'https://docs.n8n.io?utm_source=n8n_app&utm_medium=app_sidebar',
								newWindow: true,
							},
						},
						{
							id: 'forum',
							icon: 'users',
							label: this.$locale.baseText('mainSidebar.helpMenuItems.forum'),
							type: 'link',
							properties: {
								href: 'https://community.n8n.io?utm_source=n8n_app&utm_medium=app_sidebar',
								newWindow: true,
							},
						},
						{
							id: 'examples',
							icon: 'graduation-cap',
							label: this.$locale.baseText('mainSidebar.helpMenuItems.course'),
							type: 'link',
							properties: {
								href: 'https://www.youtube.com/watch?v=1MwSoB0gnM4',
								newWindow: true,
							},
						},
						{
							id: 'about',
							icon: 'info',
							label: this.$locale.baseText('mainSidebar.aboutN8n'),
							position: 'bottom',
						},
					],
				},
			];
			return [...items, ...regularItems];
		},
		userIsTrialing(): boolean {
			return this.cloudPlanStore.userIsTrialing;
		},
		currentPlanAndUsageData(): CloudPlanAndUsageData | null {
			const planData = this.cloudPlanStore.currentPlanData;
			const usage = this.cloudPlanStore.currentUsageData;
			if (!planData || !usage) return null;
			return {
				...planData,
				usage,
			};
		},
	},
	async mounted() {
		this.basePath = this.rootStore.baseUrl;
		if (this.$refs.user) {
			void this.externalHooks.run('mainSidebar.mounted', {
				userRef: this.$refs.user as Element,
			});
		}

		void this.$nextTick(() => {
			if (window.innerWidth < 900 || this.uiStore.isNodeView) {
				this.uiStore.sidebarMenuCollapsed = true;
			} else {
				this.uiStore.sidebarMenuCollapsed = false;
			}

			this.fullyExpanded = !this.isCollapsed;
		});

		this.becomeTemplateCreatorStore.startMonitoringCta();
	},
	created() {
		window.addEventListener('resize', this.onResize);
	},
	beforeUnmount() {
		this.becomeTemplateCreatorStore.stopMonitoringCta();
		window.removeEventListener('resize', this.onResize);
	},
	methods: {
		trackHelpItemClick(itemType: string) {
			this.$telemetry.track('User clicked help resource', {
				type: itemType,
				workflow_id: this.workflowsStore.workflowId,
			});
		},
		async onUserActionToggle(action: string) {
			switch (action) {
				case 'logout':
					this.onLogout();
					break;
				case 'settings':
					void this.$router.push({ name: VIEWS.PERSONAL_SETTINGS });
					break;
				default:
					break;
			}
		},
		onLogout() {
			void this.$router.push({ name: VIEWS.SIGNOUT });
		},
		toggleCollapse() {
			this.uiStore.toggleSidebarMenuCollapse();
			// When expanding, delay showing some element to ensure smooth animation
			if (!this.isCollapsed) {
				setTimeout(() => {
					this.fullyExpanded = !this.isCollapsed;
				}, 300);
			} else {
				this.fullyExpanded = !this.isCollapsed;
			}
		},
		openUpdatesPanel() {
			this.uiStore.openModal(VERSIONS_MODAL_KEY);
		},
		async handleSelect(key: string) {
			switch (key) {
				case 'workflows': {
					if (this.$router.currentRoute.value.name !== VIEWS.WORKFLOWS) {
						this.goToRoute({ name: VIEWS.WORKFLOWS });
					}
					break;
				}
				case 'templates': {
					if (this.$router.currentRoute.value.name !== VIEWS.TEMPLATES) {
						this.goToRoute({ name: VIEWS.TEMPLATES });
					}
					break;
				}
				case 'credentials': {
					if (this.$router.currentRoute.value.name !== VIEWS.CREDENTIALS) {
						this.goToRoute({ name: VIEWS.CREDENTIALS });
					}
					break;
				}
				case 'variables': {
					if (this.$router.currentRoute.value.name !== VIEWS.VARIABLES) {
						this.goToRoute({ name: VIEWS.VARIABLES });
					}
					break;
				}
				case 'executions': {
					if (this.$router.currentRoute.value.name !== VIEWS.EXECUTIONS) {
						this.goToRoute({ name: VIEWS.EXECUTIONS });
					}
					break;
				}
				case 'settings': {
					const defaultRoute = this.findFirstAccessibleSettingsRoute();
					if (defaultRoute) {
						const route = this.$router.resolve({ name: defaultRoute });
						if (this.$router.currentRoute.value.name !== defaultRoute) {
							this.goToRoute(route.path);
						}
					}
					break;
				}
				case 'about': {
					this.trackHelpItemClick('about');
					this.uiStore.openModal(ABOUT_MODAL_KEY);
					break;
				}
				case 'cloud-admin': {
					void this.cloudPlanStore.redirectToDashboard();
					break;
				}
				case 'quickstart':
				case 'docs':
				case 'forum':
				case 'examples': {
					this.trackHelpItemClick(key);
					break;
				}
				default:
					break;
			}
		},
		goToRoute(route: string | { name: string }) {
			this.$router.push(route).catch((failure) => {
				console.log(failure);
				// Catch navigation failures caused by route guards
				if (!isNavigationFailure(failure)) {
					console.error(failure);
				}
			});
		},
		findFirstAccessibleSettingsRoute() {
			const settingsRoutes = this.$router
				.getRoutes()
				.find((route) => route.path === '/settings')!
				.children.map((route) => route.name || '');

			let defaultSettingsRoute = null;
			for (const route of settingsRoutes) {
				if (this.canUserAccessRouteByName(route.toString())) {
					defaultSettingsRoute = route;
					break;
				}
			}

			return defaultSettingsRoute;
		},
		onResize(event: UIEvent) {
			void this.callDebounced(this.onResizeEnd, { debounceTime: 100 }, event);
		},
		async onResizeEnd(event: UIEvent) {
			const browserWidth = (event.target as Window).outerWidth;
			await this.checkWidthAndAdjustSidebar(browserWidth);
		},
		async checkWidthAndAdjustSidebar(width: number) {
			if (width < 900) {
				this.uiStore.sidebarMenuCollapsed = true;
				await this.$nextTick();
				this.fullyExpanded = !this.isCollapsed;
			}
		},
	},
});
</script>

<style lang="scss" module>
.sideMenu {
	position: relative;
	height: 100%;
	border-right: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);
	transition: width 150ms ease-in-out;
	width: $sidebar-expanded-width;
	.logo {
		height: $header-height;
		display: flex;
		align-items: center;
		padding: var(--spacing-xs);

		img {
			position: relative;
			left: 1px;
			height: 20px;
		}
	}

	&.sideMenuCollapsed {
		width: $sidebar-width;

		.logo img {
			left: 0;
		}
	}
}

.sideMenuCollapseButton {
	position: absolute;
	right: -10px;
	top: 50%;
	z-index: 999;
	display: flex;
	justify-content: center;
	align-items: center;
	color: var(--color-text-base);
	background-color: var(--color-foreground-xlight);
	width: 20px;
	height: 20px;
	border: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);
	border-radius: 50%;

	&:hover {
		color: var(--color-primary-shade-1);
	}
}

.updates {
	display: flex;
	align-items: center;
	cursor: pointer;
	padding: var(--spacing-2xs) var(--spacing-l);
	margin: var(--spacing-2xs) 0 0;

	svg {
		color: var(--color-text-base) !important;
	}
	span {
		display: none;
		&.expanded {
			display: initial;
		}
	}

	&:hover {
		&,
		& svg {
			color: var(--color-text-dark) !important;
		}
	}
}

.userArea {
	display: flex;
	padding: var(--spacing-xs);
	align-items: center;
	height: 60px;
	border-top: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);

	.userName {
		display: none;
		overflow: hidden;
		width: 100px;
		white-space: nowrap;
		text-overflow: ellipsis;

		&.expanded {
			display: initial;
		}

		span {
			overflow: hidden;
			text-overflow: ellipsis;
		}
	}

	.userActions {
		display: none;

		&.expanded {
			display: initial;
		}
	}
}

@media screen and (max-height: 470px) {
	:global(#help) {
		display: none;
	}
}
</style>
