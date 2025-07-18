<script lang="ts">
    import { Button, InputSearch } from '$lib/elements/forms';
    import { createEventDispatcher } from 'svelte';
    import { AvatarInitials, EmptySearch, Modal, PaginationInline } from '..';
    import { sdk } from '$lib/stores/sdk';
    import { Query, type Models } from '@appwrite.io/console';
    import type { Writable } from 'svelte/store';
    import type { Permission } from './permissions.svelte';
    import {
        Avatar,
        Card,
        Empty,
        Icon,
        Layout,
        Link,
        Selector,
        Spinner,
        Table,
        Typography
    } from '@appwrite.io/pink-svelte';
    import { IconAnonymous, IconMinusSm } from '@appwrite.io/pink-icons-svelte';
    import { page } from '$app/state';

    export let show: boolean;
    export let groups: Writable<Map<string, Permission>>;

    const dispatch = createEventDispatcher();

    let search = '';
    let offset = 0;
    let isLoading = false;
    let hasSelection = false;
    let selected: Set<string> = new Set();
    let results: Models.UserList<Record<string, unknown>>;

    function reset() {
        offset = 0;
        search = '';
        selected.clear();
        show = false;
    }

    function create() {
        dispatch('create', Array.from(selected));
        reset();
    }

    async function request() {
        if (!show) return;
        isLoading = true;
        results = await sdk
            .forProject(page.params.region, page.params.project)
            .users.list([Query.limit(5), Query.offset(offset)], search || undefined);
        isLoading = false;
    }

    function onSelection(role: string) {
        const checked = selected.has(role);
        if (checked) {
            selected.delete(role);
        } else {
            selected.add(role);
        }
        selected = selected;

        hasSelection = selected.size > 0;
    }

    $: if (show) {
        request();
    }
    $: if (offset !== null) {
        request();
    }
    $: if (search !== null) {
        offset = 0;
        request();
    }
</script>

<Modal title="Select users" bind:show onSubmit={create} on:close={reset}>
    <Typography.Text slot="description"
        >Grant access to any authenticated or anonymous user.</Typography.Text>

    <InputSearch autofocus placeholder="Search by name, email, phone or ID" bind:value={search} />

    {#if results?.users?.length}
        <Table.Root columns={[{ id: 'checkbox', width: 40 }, { id: 'user' }]} let:root>
            {#each results.users as user (user.$id)}
                {@const role = `user:${user.$id}`}
                {@const exists = $groups.has(role)}
                <Table.Row.Button {root} on:click={() => onSelection(role)} disabled={exists}>
                    <Table.Cell column="checkbox" {root}>
                        <div style:pointer-events="none">
                            <Selector.Checkbox
                                size="s"
                                id={user.$id}
                                disabled={exists}
                                checked={exists || selected.has(role)} />
                        </div>
                    </Table.Cell>
                    <Table.Cell column="user" {root}>
                        <Layout.Stack direction="row" alignItems="center" gap="s">
                            {#if user.email || user.phone}
                                {#if user.name}
                                    <AvatarInitials size="xs" name={user.name} />
                                    <Layout.Stack gap="none">
                                        <Typography.Caption variant="400"
                                            >{user.name}</Typography.Caption>
                                        <Typography.Caption
                                            variant="400"
                                            color="--fgcolor-neutral-tertiary">
                                            {user.$id}
                                        </Typography.Caption>
                                    </Layout.Stack>
                                {:else}
                                    <Avatar size="xs">
                                        <Icon icon={IconMinusSm} size="s" />
                                    </Avatar>
                                    <Layout.Stack gap="none">
                                        <Typography.Caption variant="400"
                                            >{user.email
                                                ? user.email
                                                : user.phone}</Typography.Caption>
                                        <Typography.Caption
                                            variant="400"
                                            color="--fgcolor-neutral-tertiary">
                                            {user.$id}
                                        </Typography.Caption>
                                    </Layout.Stack>
                                {/if}
                            {:else}
                                <Avatar size="xs">
                                    <Icon icon={IconAnonymous} size="s" />
                                </Avatar>
                                <Layout.Stack gap="none">
                                    <Typography.Caption variant="400"
                                        >{user.name ? user.name : '-'}</Typography.Caption>
                                    <Typography.Caption
                                        variant="400"
                                        color="--fgcolor-neutral-tertiary">
                                        {user.$id}
                                    </Typography.Caption>
                                </Layout.Stack>
                            {/if}
                        </Layout.Stack>
                    </Table.Cell>
                </Table.Row.Button>
            {/each}
        </Table.Root>

        <Layout.Stack direction="row" justifyContent="space-between" alignItems="center">
            <p class="text">Total results: {results?.total}</p>
            <PaginationInline
                limit={5}
                bind:offset
                total={results?.total}
                hidePages
                on:change={request} />
        </Layout.Stack>
    {:else if search}
        <EmptySearch bind:search target="users" hidePages>
            <Button
                external
                href="https://appwrite.io/docs/products/auth/users"
                text
                event="empty_documentation"
                size="s">Documentation</Button>
            <Button secondary on:click={() => (search = '')}>Clear search</Button>
        </EmptySearch>
    {:else if isLoading}
        <!-- 275px nearly matches the height of at-least 5 items in the table above -->
        <div style:margin-inline="auto" style:min-height="275px" style:align-content="center">
            <Spinner size="m" />
        </div>
    {:else}
        <Card.Base padding="none">
            <Empty title="You have no users. Create a user to see them here." type="secondary">
                <Typography.Text slot="description">
                    Need a hand? Learn more in our <Link.Anchor
                        href="https://appwrite.io/docs/products/auth/quick-start"
                        target="_blank"
                        rel="noopener noreferrer">
                        documentation</Link.Anchor
                    >.
                </Typography.Text>
            </Empty>
        </Card.Base>
    {/if}

    <svelte:fragment slot="footer">
        <Button submit disabled={!hasSelection}>Add</Button>
    </svelte:fragment>
</Modal>
