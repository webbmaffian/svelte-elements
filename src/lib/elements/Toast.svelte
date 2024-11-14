<script context="module">
    import { onNavigate } from "$app/navigation";

    /**
     * @typedef {`${string}-${string}-${string}-${string}-${string}`} Id
     * @typedef {string} Message
     * @typedef {'error' | 'warning' | 'info' | 'success'} Type
     */

    /**
     * @typedef {object} Notice
     * @prop {Id} id
     * @prop {Message} message
     * @prop {Type} type
     */

    /**
     * @typedef {object} NoticeArgs
     * @prop {Type} [type]
     * @prop {boolean} [autoHide]
     */

    /**
     * @type {import('svelte/store').Writable<Notice>}
     */
    const noticeStore = writable(null);

    export const notice = {
        subscribe: noticeStore.subscribe,

        /**
         * @param {Message} message
         * @param {NoticeArgs} [args]
         */
        async setNotice(message, args) {
            const { type, autoHide } = {
                ...{ type: "success", autoHide: true },
                ...args,
            };

            noticeStore.update(() => {
                const notice = {
                    id: crypto.randomUUID(),
                    message,
                    type,
                };

                if (autoHide) {
                    setTimeout(() => {
                        noticeStore.update((n) => {
                            if (n?.id === notice.id) {
                                return null;
                            }

                            return n;
                        });
                    }, 5000);
                }

                return notice;
            });
            await tick();
        },
        reset() {
            noticeStore.set(null);
        },
    };

    const icons = new Map([
        ["error", XCircle],
        ["warning", TriangleAlert],
        ["info", Info],
        ["success", CircleCheck],
    ]);
</script>

<script>
    import {
        computePosition,
        flip,
        shift,
        autoUpdate,
        offset,
    } from "@floating-ui/dom";
    import { onDestroy, tick } from "svelte";
    import { writable } from "svelte/store";
    import {
        CircleCheck,
        Info,
        TriangleAlert,
        X,
        XCircle,
    } from "lucide-svelte";
    import { fly } from "svelte/transition";

    export let label = "Notification";
    /** @type {'top-left'|'top-center|'top-right'|'bottom-left'|'bottom-center|'bottom-right'} */
    export let position = "top-right";
    /** @type {HTMLElement|null} */
    export let root;
    let className = "";
    export { className as class };
    /** @type {import('@floating-ui/core').Placement} */
    export let placement = "bottom";
    /** @type {import('@floating-ui/dom').OffsetOptions} */
    export let offsetOptions = {
        mainAxis: 80,
        crossAxis: 80,
    };
    /** @type {import('@floating-ui/dom').ShiftOptions} */
    export let shiftOptions = {
        padding: 0,
    };
    /** @type {import('svelte/transition').FlyParams} */
    export let inFly = { duration: 150, y: -8 };
    /** @type {import('svelte/transition').FlyParams} */
    export let outFly = { duration: 150, y: -8 };

    let cleanup = () => {};
    /** @type {HTMLElement} */
    let wrapper;

    onNavigate(() => {
        notice.reset();
    });

    $: if (root && wrapper && $notice) {
        cleanup();
        updatePosition();

        cleanup = autoUpdate(root, wrapper, updatePosition);
    }

    onDestroy(() => {
        cleanup();
    });

    function updatePosition() {
        if (!root || !wrapper) {
            return;
        }

        computePosition(root, wrapper, {
            placement: placement,
            middleware: [offset(offsetOptions), flip(), shift(shiftOptions)],
        }).then(({ x, y }) => {
            Object.assign(wrapper.style, {
                left: `${x}px`,
                top: `${y}px`,
            });
        });
    }

    function getCSSOffset() {
        switch (typeof offsetOptions) {
            case "number":
                return `--offset-x: ${offsetOptions}px; --offset-y: ${offsetOptions}px;`;
            case "object":
                return `--offset-x: ${offsetOptions.crossAxis}px; --offset-y: ${offsetOptions.mainAxis}px;`;
        }

        return "";
    }
</script>

{#if $notice}
    <section
        bind:this={wrapper}
        class={`svelte-elements-toast ${className}`}
        data-position-y={!root && position.split("-")[0]}
        data-position-x={!root && position.split("-")[1]}
        style={getCSSOffset()}
        tabindex="-1"
        aria-label={label}
        in:fly={inFly}
        out:fly={outFly}
    >
        {#key $notice.id}
            <div class={`item ${$notice.type}`} in:fly={inFly}>
                {#if icons.has($notice.type)}
                    <span class="icon"
                        ><svelte:component
                            this={icons.get($notice.type)}
                            size="20"
                        /></span
                    >
                {/if}
                <div class="content">
                    <span>{$notice.message}</span>
                </div>
                <button class="close" on:click={() => notice.reset()}
                    ><X /></button
                >
            </div>
        {/key}
    </section>
{/if}

<style lang="scss">
    :where(.svelte-elements-toast) {
        --normal-bg: #fff;
        --normal-border: hsl(0, 0%, 93%);
        --normal-text: hsl(0, 0%, 9%);

        --success-bg: hsl(143, 85%, 96%);
        --success-border: hsl(145, 92%, 91%);
        --success-text: hsl(140, 100%, 27%);

        --info-bg: hsl(208, 100%, 97%);
        --info-border: hsl(221, 91%, 91%);
        --info-text: hsl(210, 92%, 45%);

        --warning-bg: hsl(49, 100%, 97%);
        --warning-border: hsl(49, 91%, 91%);
        --warning-text: hsl(31, 92%, 45%);

        --error-bg: hsl(359, 100%, 97%);
        --error-border: hsl(359, 100%, 94%);
        --error-text: hsl(360, 100%, 45%);

        position: absolute;
        top: 0;
        left: 0;
        z-index: 10;
        display: flex;

        &[data-position-x="right"] {
            left: auto;
            right: var(--offset-x, 0);
        }

        &[data-position-x="left"] {
            left: var(--offset-x, 0);
            right: auto;
        }

        &[data-position-x="center"] {
            left: 50%;
            transform: translateX(-50%);
        }

        &[data-position-y="top"] {
            top: var(--offset-y, 0);
            bottom: auto;
        }

        &[data-position-y="bottom"] {
            top: auto;
            bottom: var(--offset-y, 0);
        }

        .item {
            position: relative;
            display: flex;
            justify-content: space-between;
            align-items: center;
            width: 100%;
            padding: 1rem;
            padding-left: 3rem;
            padding-right: 3rem;
            border: 1px solid transparent;
            overflow: hidden;

            background-color: var(--normal-bg);
            border-color: var(--normal-border);
            color: var(--normal-text);

            &.error {
                background-color: var(--error-bg);
                border-color: var(--error-border);
                color: var(--error-text);
            }
            &.warning {
                background-color: var(--warning-bg);
                border-color: var(--warning-border);
                color: var(--warning-text);
            }
            &.info {
                background-color: var(--info-bg);
                border-color: var(--info-border);
                color: var(--info-text);
            }
            &.success {
                background-color: var(--success-bg);
                border-color: var(--success-border);
                color: var(--success-text);
            }

            .icon {
                position: absolute;
                left: 0.5rem;
                top: 50%;
                transform: translateY(-50%);
                display: flex;
                justify-content: center;
                align-items: center;
                padding: 0.25rem;
            }

            .content {
                display: flex;
                flex-direction: column;
                gap: 1rem;

                span {
                    font-size: 1rem;
                    line-height: 1.25rem;
                    font-weight: 500;
                }
            }

            button.close {
                position: absolute;
                top: 0.5rem;
                right: 0.5rem;
                display: flex;
                justify-content: center;
                align-items: center;
                width: 1.5rem;
                height: 1.5rem;
                border-radius: 0.25rem;
                padding: 0.25rem;
                margin: 0;
                transition: opacity cubic-bezier(0.445, 0.05, 0.55, 0.95) 150ms;
                cursor: pointer;
                border: none;
                background-color: transparent;
                color: inherit;

                &:focus,
                &:hover {
                    color: inherit;
                }

                :global(svg) {
                    pointer-events: none;
                }
            }
        }
    }
</style>
