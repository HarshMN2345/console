<script lang="ts">
    import type { ComponentType } from 'svelte';
    import type { BaseCardProps } from './card.svelte';
    import type { HTMLAttributes } from 'svelte/elements';
    import { Card, Tooltip } from '@appwrite.io/pink-svelte';

    type Props = BaseCardProps &
        HTMLAttributes<HTMLInputElement> & {
            name: string;
            value: string;
            group: string;
            title: string;
            info?: string | undefined;
            icon?: ComponentType;
            imageHeight?: number;
            imageWidth?: number;
            imageRadius?: 'xxs' | 'xs' | 's' | 'm' | 'l';
            disabled?: boolean;
            src?: string;
            alt?: string | undefined;
        };

    export let group: string;
    export let value: string;

    export let tooltipShow = false;
    export let tooltipText: string = null;
    export let tooltipWidth: string = undefined;

    // Pink v2
    export let icon: Props['icon'] = undefined;
    export let radius: Props['radius'] = 's';
    export let imageRadius: Props['imageRadius'] = 'xxs';
    export let padding: Props['padding'] = 's';
    export let variant: Props['variant'] = 'primary';
    export let name: Props['name'] = undefined;
    //temporarily undefined
    export let title: Props['title'] = undefined;
    export let disabled = false;
    export let src: string = null;
    export let alt: string = null;

    // TODO: remove after label card migration
    let slotTitle: HTMLSpanElement;
</script>

<Tooltip maxWidth={tooltipWidth} disabled={!tooltipText || !tooltipShow}>
    <div style:cursor={disabled ? 'pointer' : ''}>
        <Card.Selector
            {name}
            {src}
            {alt}
            {icon}
            {padding}
            {imageRadius}
            {variant}
            {value}
            {radius}
            {disabled}
            title={title ?? slotTitle?.innerText}
            bind:group>
            {#if $$slots.default}
                <slot />
            {/if}
            <slot name="action" slot="action" />
        </Card.Selector>
    </div>
    <span slot="tooltip">{tooltipText}</span>
</Tooltip>

<span style="display: none" bind:this={slotTitle}>
    <slot name="title" />
</span>
