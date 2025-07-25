<script lang="ts">
    import { createMenubar, melt } from '@melt-ui/svelte';
    import {
        Badge,
        Icon,
        type SheetMenu,
        Layout,
        ActionMenu,
        Card,
        Skeleton
    } from '@appwrite.io/pink-svelte';
    import {
        IconChevronDown,
        IconChevronRight,
        IconPlus,
        IconPlusSm
    } from '@appwrite.io/pink-icons-svelte';
    import { BottomSheet } from '$lib/components';
    import { isSmallViewport } from '$lib/stores/viewport';
    import { isCloud } from '$lib/system';
    import { goto } from '$app/navigation';
    import { base } from '$app/paths';
    import { currentPlan, newOrgModal } from '$lib/stores/organization';
    import { Click, trackEvent } from '$lib/actions/analytics';
    import type { Models } from '@appwrite.io/console';

    type Organization = {
        name: string;
        $id: string;
        tierName: string;
        isSelected: boolean;
    };

    const {
        elements: { menubar },
        builders: { createMenu }
    } = createMenubar();

    const {
        elements: {
            trigger: triggerOrganizations,
            menu: menuOrganizations,
            item: itemOrganizations,
            separator: separatorOrganizations
        },
        builders: { createSubmenu: createSubmenuOrganizations, createMenuRadioGroup }
    } = createMenu();

    const {
        elements: { radioGroup: radioGroupOrganizations }
    } = createMenuRadioGroup({});

    const {
        elements: { subMenu: subMenuOrganizations, subTrigger: subTriggerOrganizations }
    } = createSubmenuOrganizations();

    const {
        elements: {
            trigger: triggerProjects,
            menu: menuProjects,
            item: itemProjects,
            separator: separatorProjects
        }
    } = createMenu();

    let isLoadingProjects = true;
    let loadedProjects: Models.ProjectList = { total: 0, projects: [] };

    export let organizations: Organization[] = [];
    export let currentProject: Models.Project | null = null;
    export let projects: Promise<Models.ProjectList> = Promise.resolve(loadedProjects);

    let projectsBottomSheetOpen = false;
    let organisationBottomSheetOpen = false;

    function createOrg() {
        trackEvent(Click.OrganizationClickCreate, { source: 'breadcrumbs' });
        if (isCloud) {
            goto(`${base}/create-organization`);
        } else newOrgModal.set(true);
    }

    const switchOrganization = {
        top: {
            title: 'Switch Organization',
            items: organizations.map((organization) => ({
                name: organization.name,
                href: `${base}/organization-${organization?.$id}`
            }))
        },
        bottom: {
            items: [
                {
                    name: 'Create organization',
                    leadingIcon: IconPlus,
                    onClick: createOrg
                }
            ]
        }
    };

    async function createProjectsBottomSheet(organization: Organization): Promise<SheetMenu> {
        isLoadingProjects = true;
        // null on non-org/project path like `onboarding`.
        loadedProjects = (await projects) ?? loadedProjects;
        for (const project of loadedProjects.projects) {
            project.region ??= 'default';
        }
        isLoadingProjects = false;

        const createProjectItem = {
            name: 'Create project',
            trailingIcon: IconPlus,
            href: `${base}/organization-${organization?.$id}?create-project`
        };

        if (loadedProjects.total > 1 && selectedOrg) {
            const projectLinks = loadedProjects.projects.slice(0, 4).map((project) => {
                return {
                    name: project.name,
                    href: `${base}/project-${project.region}-${project.$id}/overview/platforms`
                };
            });

            if (loadedProjects.projects.length > 4) {
                projectLinks.push({
                    name: 'All projects',
                    href: `${base}/organization-${selectedOrg.$id}`
                });
            }

            return {
                top: { title: 'Switch project', items: projectLinks },
                bottom: { items: [createProjectItem] }
            };
        }

        return {
            top: { items: [createProjectItem] },
            bottom: { items: [createProjectItem] }
        };
    }

    function createOrganizationBottomSheet(organization: Organization) {
        return !organization
            ? switchOrganization
            : ({
                  top: {
                      items: [
                          {
                              name: 'Organization overview',
                              href: `${base}/organization-${organization?.$id}`
                          }
                      ]
                  },
                  bottom:
                      organizations.length > 1
                          ? {
                                items: [
                                    {
                                        name: 'Switch organization',
                                        trailingIcon: IconChevronRight,
                                        subMenu: switchOrganization
                                    }
                                ]
                            }
                          : {
                                items: [
                                    {
                                        name: 'Create organization',
                                        leadingIcon: IconPlus,
                                        onClick: createOrg
                                    }
                                ]
                            }
              } satisfies SheetMenu);
    }

    function onResize() {
        if ((organisationBottomSheetOpen || projectsBottomSheetOpen) && !$isSmallViewport) {
            organisationBottomSheetOpen = false;
            projectsBottomSheetOpen = false;
        }
    }

    $: selectedOrg = organizations.find((org) => org.isSelected);

    $: projectsBottomSheet = createProjectsBottomSheet(selectedOrg);

    $: organizationsBottomSheet = createOrganizationBottomSheet(selectedOrg);

    $: correctPlanName =
        // the plan names are hardcoded in some cases and are not available locally,
        // so we rely on the plan's source of truth - `$currentPlan`
        $currentPlan &&
        $currentPlan?.name.toLocaleLowerCase() !== selectedOrg?.tierName.toLocaleLowerCase()
            ? $currentPlan.name
            : selectedOrg?.tierName; // fallback

    $: derivedKey = `${selectedOrg?.$id}-${currentProject?.$id}`;
</script>

<svelte:window on:resize={onResize} />

{#key derivedKey}
    <div use:melt={$menubar}>
        {#if !$isSmallViewport}
            <span class="breadcrumb-separator">/</span>
            <button
                type="button"
                class="trigger"
                use:melt={$triggerOrganizations}
                aria-label="Open organizations tab">
                <span class="orgName">{selectedOrg?.name ?? 'Organization'}</span>
                <span class="not-mobile"
                    >{#if correctPlanName}<Badge
                            variant="secondary"
                            content={correctPlanName} />{/if}</span>
                <Icon icon={IconChevronDown} size="s" color="--fgcolor-neutral-secondary" />
            </button>
        {:else}
            <button
                type="button"
                class="trigger"
                on:click={() => {
                    organisationBottomSheetOpen = true;
                }}
                aria-label="Open organizations tab">
                <span class="orgName" class:noProjects={!currentProject}
                    >{selectedOrg?.name ?? 'Organization'}</span>
                <span class="not-mobile"
                    ><Badge variant="secondary" content={correctPlanName ?? ''} /></span>
                <Icon icon={IconChevronDown} size="s" color="--fgcolor-neutral-secondary" />
            </button>
        {/if}

        <div class="menu" use:melt={$menuOrganizations}>
            <Card.Base padding="xxxs" shadow={true}>
                {#if selectedOrg}
                    <div use:melt={$itemOrganizations}>
                        <ActionMenu.Root>
                            <ActionMenu.Item.Anchor
                                href={`${base}/organization-${selectedOrg?.$id}`}
                                >Organization overview</ActionMenu.Item.Anchor
                            ></ActionMenu.Root>
                    </div>
                    {#if organizations.length > 1}
                        <div class="separator" use:melt={$separatorOrganizations}></div>

                        <div use:melt={$subTriggerOrganizations}>
                            <ActionMenu.Root>
                                <ActionMenu.Item.Button trailingIcon={IconChevronRight}
                                    >Switch organization</ActionMenu.Item.Button>
                            </ActionMenu.Root>
                        </div>
                        <div class="menu subMenu" use:melt={$subMenuOrganizations}>
                            <Card.Base padding="xxxs" shadow={true}>
                                <div use:melt={$radioGroupOrganizations}>
                                    {#each organizations as organization}
                                        <div use:melt={$itemOrganizations}>
                                            <ActionMenu.Root>
                                                <ActionMenu.Item.Anchor
                                                    href={`${base}/organization-${organization?.$id}`}
                                                    >{organization.name}</ActionMenu.Item.Anchor>
                                            </ActionMenu.Root>
                                        </div>
                                    {/each}
                                    <div class="separator" use:melt={$separatorOrganizations}></div>
                                    <div use:melt={$itemOrganizations}>
                                        <ActionMenu.Root>
                                            <ActionMenu.Item.Button
                                                leadingIcon={IconPlusSm}
                                                on:click={createOrg}
                                                >Create organization</ActionMenu.Item.Button
                                            ></ActionMenu.Root>
                                    </div>
                                </div>
                            </Card.Base>
                        </div>
                    {:else}
                        <div class="separator" use:melt={$separatorOrganizations}></div>
                        <div use:melt={$itemOrganizations}>
                            <ActionMenu.Root>
                                <ActionMenu.Item.Button
                                    leadingIcon={IconPlusSm}
                                    on:click={createOrg}>Create organization</ActionMenu.Item.Button
                                ></ActionMenu.Root>
                        </div>
                    {/if}
                {:else}
                    {#each organizations as organization}
                        <div use:melt={$itemOrganizations}>
                            <ActionMenu.Root>
                                <ActionMenu.Item.Anchor
                                    href={`${base}/organization-${organization?.$id}`}
                                    >{organization.name}</ActionMenu.Item.Anchor
                                ></ActionMenu.Root>
                        </div>
                    {/each}
                    <div class="separator" use:melt={$separatorOrganizations}></div>
                    <div use:melt={$itemOrganizations}>
                        <ActionMenu.Root>
                            <ActionMenu.Item.Button leadingIcon={IconPlusSm} on:click={createOrg}
                                >Create organization</ActionMenu.Item.Button
                            ></ActionMenu.Root>
                    </div>
                {/if}
            </Card.Base>
        </div>

        {#if selectedOrg && currentProject}
            <span class="breadcrumb-separator">/</span>
            {#if !$isSmallViewport}
                <button
                    type="button"
                    class="trigger"
                    use:melt={$triggerProjects}
                    aria-label="Open projects tab">
                    <span class="projectName">{currentProject.name}</span>
                    <Icon icon={IconChevronDown} size="s" />
                </button>
            {:else}
                <button
                    type="button"
                    class="trigger"
                    on:click={() => (projectsBottomSheetOpen = true)}
                    aria-label="Open projects tab">
                    <span class="projectName">{currentProject.name}</span>
                    <Icon icon={IconChevronDown} size="s" />
                </button>
            {/if}

            <div class="menu" use:melt={$menuProjects}>
                <Card.Base padding="xxxs" shadow={true}>
                    {#if isLoadingProjects}
                        <div style:margin-inline="0.25rem" style:margin-block="0.25rem">
                            <Layout.Stack gap="s">
                                <!-- 2 should be enough -->
                                <Skeleton width="100%" height={30} variant="line" />
                                <Skeleton width="100%" height={30} variant="line" />
                            </Layout.Stack>
                        </div>
                    {:else if loadedProjects.total > 1}
                        {#each loadedProjects.projects as project, index}
                            {#if index < 4}
                                <div use:melt={$itemProjects}>
                                    <ActionMenu.Root>
                                        <ActionMenu.Item.Anchor
                                            href={`${base}/project-${project.region}-${project.$id}/overview/platforms`}>
                                            <span class="projectName dropdown">{project.name}</span>
                                        </ActionMenu.Item.Anchor>
                                    </ActionMenu.Root>
                                </div>
                            {:else if index === 4}
                                <div use:melt={$itemProjects}>
                                    <ActionMenu.Root>
                                        <ActionMenu.Item.Anchor
                                            href={`${base}/organization-${selectedOrg.$id}`}>
                                            All projects
                                        </ActionMenu.Item.Anchor>
                                    </ActionMenu.Root>
                                </div>
                            {/if}
                        {/each}
                        <div class="separator" use:melt={$separatorProjects}></div>
                    {/if}
                    <div use:melt={$itemProjects}>
                        <ActionMenu.Root>
                            <ActionMenu.Item.Anchor
                                leadingIcon={IconPlusSm}
                                href={`${base}/organization-${selectedOrg?.$id}?create-project`}>
                                Create project
                            </ActionMenu.Item.Anchor></ActionMenu.Root>
                    </div>
                </Card.Base>
            </div>
        {/if}
    </div>

    <BottomSheet.Menu bind:isOpen={organisationBottomSheetOpen} menu={organizationsBottomSheet} />

    {#await projectsBottomSheet then menu}
        <BottomSheet.Menu bind:isOpen={projectsBottomSheetOpen} {menu} />
    {/await}
{/key}

<style lang="scss">
    .menu {
        min-width: 244px;
        z-index: 20;
    }

    .not-mobile {
        display: none;

        @media (min-width: 768px) {
            display: block;
        }
    }

    .subMenu {
        min-width: 244px;
        margin-inline: -4px;
        margin-block: -4px;
    }

    .orgName,
    .projectName {
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        max-width: 60px;
        color: var(--fgcolor-neutral-secondary);

        @media (min-width: 390px) {
            max-width: 95px;
        }

        @media (min-width: 400px) {
            max-width: 105px;
        }

        @media (min-width: 800px) {
            max-width: 125px;
        }

        @media (min-width: 1024px) {
            max-width: 150px;
        }

        &.dropdown {
            max-width: 200px;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
            display: inline-block;
            vertical-align: middle;
        }
    }

    .noProjects {
        max-width: 150px;
    }

    :global(.item[data-highlighted]) {
        border-radius: var(--border-radius-S, 8px);
        background: var(--overlay-neutral-hover, rgba(25, 25, 28, 0.03));
    }
    .trigger {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        padding: var(--space-1, 2px) var(--space-2, 4px) var(--space-1, 2px) var(--space-3, 6px);
        gap: var(--space-2, 4px);
        margin: 0 var(--space-5, 10px) 0 var(--space-5, 10px);

        transition: color 0.2s ease;

        color: var(--fgcolor-neutral-primary, #2d2d31);
        border-radius: var(--corner-radius-medium, 8px);

        cursor: pointer;
        /* Body text/level 2 Regular */
        font-family: Inter;
        font-size: 14px;
        font-style: normal;
        font-weight: 400;
        line-height: 150%; /* 21px */
    }

    .trigger:hover {
        background: var(--overlay-neutral-hover, rgba(25, 25, 28, 0.03));
    }

    :global(.trigger[data-highlighted]) {
        outline: none;
        background: var(--bgcolor-neutral-secondary, #f4f4f7);
    }

    :global(.trigger[data-highlighted]:focus-visible) {
        outline: none;
        box-shadow: 0 0 0 2px var(--bgcolor-neutral-secondary, #f4f4f7);
    }

    .trigger:focus-visible {
        z-index: 30;
        box-shadow:
            var(--shadow-offsetx-0, 0px) var(--shadow-offsety-0, 0px) 0 2px
                var(--bgcolor-neutral-default, #fafafb),
            0 0 0 4px var(--border-focus, #818186);
    }
    .separator {
        height: 1px;
        margin-block: 2px;
        margin-inline-start: calc(var(--base-4) * -1);
        width: calc(100% + var(--base-8));
        background-color: var(--border-neutral);
    }
    .breadcrumb-separator {
        color: var(--fgcolor-neutral-tertiary, #97979b);
    }
</style>
