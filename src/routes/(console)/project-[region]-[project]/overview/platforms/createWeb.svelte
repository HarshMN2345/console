<script lang="ts">
    import { Wizard } from '$lib/layout';
    import { invalidate } from '$app/navigation';
    import { createPlatform } from './wizard/store';
    import { Dependencies } from '$lib/constants';
    import {
        Code,
        Layout,
        Icon,
        Typography,
        Fieldset,
        InlineCode,
        Card,
        Tooltip
    } from '@appwrite.io/pink-svelte';
    import { Button, Form, InputText } from '$lib/elements/forms';
    import {
        IconVue,
        IconAppwrite,
        IconSvelte,
        IconReact,
        IconNuxt,
        IconInfo,
        IconExternalLink,
        IconAngular,
        IconJs
    } from '@appwrite.io/pink-icons-svelte';
    import { page } from '$app/state';
    import { type ComponentType, onMount } from 'svelte';
    import { sdk } from '$lib/stores/sdk';
    import { Submit, trackError, trackEvent } from '$lib/actions/analytics';
    import { addNotification } from '$lib/stores/notifications';
    import { fade } from 'svelte/transition';
    import ConnectionLine from './components/ConnectionLine.svelte';
    import OnboardingPlatformCard from './components/OnboardingPlatformCard.svelte';
    import { PlatformType } from '@appwrite.io/console';
    import {
        ReactFrameworkIcon,
        SvelteFrameworkIcon,
        NuxtFrameworkIcon,
        NextjsFrameworkIcon,
        VueFrameworkIcon,
        NoFrameworkIcon,
        AngularFrameworkIcon,
        JavascriptFrameworkIcon
    } from './components/index';
    import { extendedHostnameRegex } from '$lib/helpers/string';

    export let key;

    let showExitModal = false;
    let isPlatformCreated = !!key;
    let isCreatingPlatform = false;
    let connectionSuccessful = false;
    let isChangingFramework = false;

    const projectId = page.params.project;

    const updateConfigCode = (prefix = '') => `${prefix}APPWRITE_PROJECT_ID = "${projectId}"
${prefix}APPWRITE_ENDPOINT = "${sdk.forProject(page.params.region, page.params.project).client.config.endpoint}"
        `;
    type FrameworkType = {
        key: string;
        label: string;
        icon: ComponentType;
        smallIcon: ComponentType;
        portNumber: number;
        runCommand: string;
        updateConfigCode: string;
    };
    export let platform: PlatformType = PlatformType.Flutterandroid;
    export let selectedFrameworkKey: string | undefined = key ? key : undefined;
    let hostname;
    let hostnameError = false;

    let frameworks: Array<FrameworkType> = [
        {
            key: 'svelte',
            label: 'Svelte',
            icon: SvelteFrameworkIcon,
            smallIcon: IconSvelte,
            portNumber: 5173,
            runCommand: 'npm run dev',
            updateConfigCode: updateConfigCode('PUBLIC_')
        },
        {
            key: 'react',
            label: 'React',
            icon: ReactFrameworkIcon,
            smallIcon: IconReact,
            portNumber: 5173,
            runCommand: 'npm run dev',
            updateConfigCode: updateConfigCode('VITE_')
        },
        {
            key: 'nuxt',
            label: 'Nuxt',
            icon: NuxtFrameworkIcon,
            smallIcon: IconNuxt,
            portNumber: 3000,
            runCommand: 'npm run dev',
            updateConfigCode: updateConfigCode('NUXT_PUBLIC_')
        },
        {
            key: 'nextjs',
            label: 'Next.js',
            icon: NextjsFrameworkIcon,
            smallIcon: NextjsFrameworkIcon,
            portNumber: 3000,
            runCommand: 'npm run dev',
            updateConfigCode: updateConfigCode('NEXT_PUBLIC_')
        },
        {
            key: 'vue',
            label: 'Vue',
            icon: VueFrameworkIcon,
            smallIcon: IconVue,
            portNumber: 5173,
            runCommand: 'npm run dev',
            updateConfigCode: updateConfigCode('VITE_')
        },
        {
            key: 'angular',
            label: 'Angular',
            icon: AngularFrameworkIcon,
            smallIcon: IconAngular,
            portNumber: 4200,
            runCommand: 'npm run start',
            updateConfigCode: `appwriteEndpoint: '${sdk.forProject(page.params.region, page.params.project).client.config.endpoint}',\nappwriteProjectId:'${projectId}'`
        },
        {
            key: 'js',
            label: 'Javascript',
            icon: JavascriptFrameworkIcon,
            smallIcon: IconJs,
            portNumber: 5173,
            runCommand: 'npm run dev',
            updateConfigCode: updateConfigCode('VITE_')
        }
    ];

    $: selectedFramework = frameworks.find((framework) => framework.key === selectedFrameworkKey);
    $: selectedFrameworkIcon = selectedFramework ? selectedFramework.icon : NoFrameworkIcon;

    async function createWebPlatform() {
        hostnameError = hostname !== '' ? !new RegExp(extendedHostnameRegex).test(hostname) : null;

        if (hostnameError) {
            return;
        }

        try {
            isCreatingPlatform = true;
            await sdk.forConsole.projects.createPlatform(
                projectId,
                PlatformType.Web,
                `${selectedFramework.label} app`,
                selectedFrameworkKey,
                undefined,
                hostname === '' ? undefined : hostname
            );

            isPlatformCreated = true;
            trackEvent(Submit.PlatformCreate, {
                type: platform
            });

            invalidate(Dependencies.PROJECT);
            invalidate(Dependencies.PLATFORMS);
        } catch (error) {
            trackError(error, Submit.PlatformCreate);
            addNotification({
                type: 'error',
                message: error.message
            });
        } finally {
            isCreatingPlatform = false;
        }
    }

    async function resetPlatformStore() {
        createPlatform.reset();
    }

    onMount(() => {
        const unsubscribe = sdk.forConsole.client.subscribe('console', (response) => {
            if (response.events.includes(`projects.${projectId}.ping`)) {
                connectionSuccessful = true;
                invalidate(Dependencies.ORGANIZATION);
                invalidate(Dependencies.PROJECT);
                unsubscribe();
            }
        });

        return () => {
            unsubscribe();
            resetPlatformStore();
        };
    });
</script>

<Wizard title="Add web platform" bind:showExitModal confirmExit={!isPlatformCreated}>
    <Layout.Stack gap="xxl">
        <!-- Step One -->
        {#if !isPlatformCreated || isChangingFramework}
            <Form onSubmit={createWebPlatform}>
                <Layout.Stack>
                    <Fieldset legend="Type">
                        <Layout.Stack>
                            <div class="frameworks">
                                {#each frameworks as framework}
                                    <Card.Selector
                                        bind:group={selectedFrameworkKey}
                                        name="framework"
                                        id={framework.key}
                                        value={framework.key}
                                        title={framework.label}
                                        icon={framework.icon}
                                        imageRadius="s" />
                                {/each}
                            </div>
                            <Layout.Stack direction="row" justifyContent="flex-end">
                                {#if isChangingFramework}
                                    <Button
                                        disabled={!selectedFramework}
                                        on:click={() => (isChangingFramework = false)}>
                                        Save</Button>
                                {/if}
                            </Layout.Stack>
                        </Layout.Stack>
                    </Fieldset>
                    {#if !isChangingFramework}
                        <Fieldset legend="Details">
                            <InputText
                                id="hostname"
                                label="Hostname"
                                placeholder="localhost"
                                autofocus
                                error={hostnameError && 'Please enter a valid hostname'}
                                bind:value={hostname}>
                                <Tooltip slot="info">
                                    <Icon icon={IconInfo} size="s" />
                                    <span slot="tooltip">
                                        The hostname that your website will use to interact with the
                                        Appwrite APIs in production or development environments. No
                                        protocol or port number required.
                                    </span>
                                </Tooltip>
                            </InputText></Fieldset>
                        <Layout.Stack direction="row" justifyContent="flex-end">
                            <Button submit disabled={!selectedFramework}>Create platform</Button>
                        </Layout.Stack>
                    {/if}
                </Layout.Stack>
            </Form>
        {:else}
            <Card.Base padding="s"
                ><Layout.Stack direction="row" justifyContent="space-between" alignItems="center">
                    <Layout.Stack gap="xxs" direction="row">
                        <Icon icon={selectedFramework.smallIcon} />
                        <Typography.Text variant="m-500">{selectedFramework.label}</Typography.Text>
                    </Layout.Stack>
                    <Button
                        size="s"
                        secondary
                        on:click={() => {
                            isChangingFramework = true;
                        }}>Change</Button>
                </Layout.Stack></Card.Base>
        {/if}

        <!-- Step Three -->
        {#if isPlatformCreated && !isChangingFramework}
            <Fieldset legend="Clone starter">
                <Layout.Stack gap="l">
                    <Typography.Text variant="m-500">
                        1. If you're starting a new project, you can clone our starter kit from
                        GitHub using the terminal or VSCode.
                    </Typography.Text>

                    <!-- Temporary fix: Remove this div once Code splitting issue with stack spacing is resolved -->
                    <div class="pink2-code-margin-fix">
                        <Code
                            lang="bash"
                            lineNumbers
                            code={`\ngit clone https://github.com/appwrite/starter-for-${selectedFramework.key}\ncd starter-for-${selectedFramework.key}`} />
                    </div>

                    {#if selectedFramework.key === 'angular'}
                        <Typography.Text variant="m-500"
                            >2. Change <InlineCode
                                size="s"
                                code="src/environments/environment.ts" />
                            to reflect the values below:</Typography.Text>
                    {:else}
                        <Typography.Text variant="m-500"
                            >2. Add your Appwrite credentials to <InlineCode
                                size="s"
                                code=".env.example" /> then rename it to <InlineCode
                                size="s"
                                code=".env" /> if needed.</Typography.Text>
                    {/if}

                    <!-- Temporary fix: Remove this div once Code splitting issue with stack spacing is resolved -->
                    <div class="pink2-code-margin-fix">
                        <Code lang="dotenv" lineNumbers code={selectedFramework.updateConfigCode} />
                    </div>

                    <Typography.Text variant="m-500"
                        >3. Install project dependencies</Typography.Text>

                    <!-- Temporary fix: Remove this div once Code splitting issue with stack spacing is resolved -->
                    <div class="pink2-code-margin-fix">
                        <Code lang="bash" lineNumbers code={'npm install'} />
                    </div>

                    <Typography.Text variant="m-500"
                        >4. Run the app, then click the <InlineCode size="s" code="Send a ping" /> button
                        to verify the setup.</Typography.Text>
                    <!-- Temporary fix: Remove this div once Code splitting issue with stack spacing is resolved -->
                    <div class="pink2-code-margin-fix">
                        <Code lang="bash" lineNumbers code={selectedFramework.runCommand} />
                    </div>
                </Layout.Stack>
            </Fieldset>
            <Card.Base padding="s"
                ><Layout.Stack direction="row" justifyContent="space-between"
                    ><Layout.Stack direction="row" alignItems="center">
                        <Icon icon={IconInfo} color="--fgcolor-neutral-tertiary" /><Typography.Text
                            variant="m-500"
                            color="--fgcolor-neutral-primary">
                            Demo app runs on http://localhost:{selectedFramework.portNumber}</Typography.Text
                        ></Layout.Stack>
                    <Button
                        external
                        secondary
                        href={`http://localhost:${selectedFramework.portNumber}`}
                        ><Layout.Stack direction="row" gap="xs"
                            >Open <Icon
                                icon={IconExternalLink}
                                color="--fgcolor-neutral-tertiary" /></Layout.Stack
                        ></Button
                    ></Layout.Stack
                ></Card.Base>
        {/if}
    </Layout.Stack>
    <svelte:fragment slot="aside">
        <Card.Base>
            <Layout.Stack gap="xxl">
                <Layout.Stack direction="row" justifyContent="center" gap="none">
                    <OnboardingPlatformCard iconSize={2.526} icon={selectedFrameworkIcon} />

                    <ConnectionLine status={connectionSuccessful} />

                    <OnboardingPlatformCard
                        iconSize={2.526}
                        iconColor="#FD366E"
                        icon={IconAppwrite} />
                </Layout.Stack>

                {#if isPlatformCreated}
                    <Layout.Stack
                        direction="row"
                        justifyContent="center"
                        alignItems="center"
                        gap="l">
                        {#if !connectionSuccessful}
                            <Typography.Text variant="m-400"
                                >Waiting for connection...</Typography.Text>
                        {:else}
                            <!-- cannot apply fade on components -->
                            <div
                                in:fade={{ duration: 2500 }}
                                class="u-flex u-flex-vertical u-cross-center u-gap-8">
                                <Typography.Title size="m">Congratulations!</Typography.Title>

                                <Typography.Text variant="m-400"
                                    >You connected your app successfully.</Typography.Text>
                            </div>
                        {/if}
                    </Layout.Stack>
                {/if}
            </Layout.Stack>
        </Card.Base>
    </svelte:fragment>

    <svelte:fragment slot="footer">
        {#if isPlatformCreated}
            <Button
                size="s"
                secondary
                fullWidthMobile
                href={location.pathname}
                disabled={isCreatingPlatform}>Go to dashboard</Button>
        {/if}
    </svelte:fragment>
</Wizard>

<style lang="scss">
    :global(.pink2-code-margin-fix pre) {
        margin: revert;
    }

    :global(.responsive-padding) {
        @media (max-width: 768px) {
            padding: 16px;
        }
    }

    .frameworks {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        gap: var(--gap-l, 16px);
    }
</style>
