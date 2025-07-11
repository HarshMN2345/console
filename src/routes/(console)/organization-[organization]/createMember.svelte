<script lang="ts">
    import { base } from '$app/paths';
    import { page } from '$app/state';
    import { Modal } from '$lib/components';
    import { InputText, InputEmail, Button } from '$lib/elements/forms';
    import { addNotification } from '$lib/stores/notifications';
    import { sdk } from '$lib/stores/sdk';
    import { createEventDispatcher } from 'svelte';
    import { organization } from '$lib/stores/organization';
    import { invalidate } from '$app/navigation';
    import { Dependencies } from '$lib/constants';
    import { Submit, trackEvent, trackError } from '$lib/actions/analytics';
    import { isCloud, isSelfHosted } from '$lib/system';
    import { roles } from '$lib/stores/billing';
    import InputSelect from '$lib/elements/forms/inputSelect.svelte';
    import Roles from '$lib/components/roles/roles.svelte';
    import { Icon, Popover } from '@appwrite.io/pink-svelte';
    import { IconInfo } from '@appwrite.io/pink-icons-svelte';
    import { Layout } from '@appwrite.io/pink-svelte';

    export let showCreate = false;

    const dispatch = createEventDispatcher();

    let email: string,
        name: string,
        error: string,
        role: string = isSelfHosted ? 'owner' : 'developer';

    async function create() {
        try {
            const team = await sdk.forConsole.teams.createMembership(
                $organization.$id,
                [role],
                email,
                undefined,
                undefined,
                `${page.url.origin}${base}/invite`,
                name || undefined
            );
            await Promise.all([
                invalidate(Dependencies.ACCOUNT),
                invalidate(Dependencies.ORGANIZATION),
                invalidate(Dependencies.MEMBERS)
            ]);

            showCreate = false;
            addNotification({
                type: 'success',
                message: `Invite has been sent to ${email}`
            });
            trackEvent(Submit.MemberCreate);
            dispatch('created', team);
        } catch (e) {
            error = e.message;
            trackError(e, Submit.MemberCreate);
        }
    }

    $: if (!showCreate) {
        error = null;
        email = null;
        name = null;
    }
</script>

<Modal title="Invite member" {error} bind:show={showCreate} onSubmit={create}>
    <InputEmail
        required
        id="email"
        label="Email"
        placeholder="Enter email"
        autofocus
        bind:value={email} />
    <InputText id="member-name" label="Name" placeholder="Enter name" bind:value={name} />
    {#if isCloud}
        <InputSelect required id="role" label="Role" options={roles} bind:value={role}>
            <Layout.Stack direction="row" gap="none" alignItems="center" slot="info">
                <Popover let:toggle>
                    <Button extraCompact size="s" on:click={toggle}>
                        <Icon size="s" icon={IconInfo} />
                    </Button>
                    <svelte:component this={Roles} slot="tooltip" />
                </Popover>
            </Layout.Stack>
        </InputSelect>
    {/if}
    <svelte:fragment slot="footer">
        <Button secondary on:click={() => (showCreate = false)}>Cancel</Button>
        <Button submit submissionLoader>Send invite</Button>
    </svelte:fragment>
</Modal>
