<script>
    import { createEventDispatcher, onDestroy, onMount } from 'svelte';
	  import { fade } from 'svelte/transition';

    /** @type {string[]} */
    export let menuItems = [];
    export let menuOpen = false;
    export let x = 0;
    export let y = 0;
    export let zInd = 1;

    const dispatch = createEventDispatcher();

    // ==============================
    // Keyboard Events
    // ==============================
    let focusedItem = -1;

    function handleKeys(event) {
        if(menuOpen && !isHovered){
            if(event.key === "ArrowDown" && focusedItem < menuItems.length - 1){
                event.preventDefault();
                focusedItem = focusedItem + 1;
            }
            else if(event.key === "ArrowUp" && focusedItem > 0){
                event.preventDefault();
                focusedItem = focusedItem - 1;
            }
            else if(event.key === "Enter" && focusedItem != -1){
                event.preventDefault();
                dispatch('selectItem', {menuItem: menuItems[focusedItem]});
            }
            else if(event.key === "Tab" && focusedItem != -1){
                event.preventDefault();
                dispatch('selectItem', {menuItem: menuItems[focusedItem]});
            }
        }
    }

    onMount(() => {
        window.addEventListener('keydown', handleKeys);
    });

    onDestroy(() => {
        window.removeEventListener('keydown', handleKeys);
    });

    // ==============================
    // Hovered Over
    // ==============================
    let isHovered = false;
    function onMouseEnter() {
        isHovered = true;
        focusedItem = -1;
    }

    function onMouseLeave() {
        isHovered = false;
        dispatch('mouseleave');
    }
</script>

{#if menuOpen}
    <div class="selection-menu" 
        style="top: {y}px; left: {x}px; z-index: {zInd};"
        on:mouseenter={onMouseEnter} on:mouseleave={onMouseLeave}
        transition:fade={{ duration: 100 }}
        role="menu" tabindex={0}>

        {#each menuItems as item, index}
            <div class={`menu-item ${index == focusedItem ? 'hover' : ''}`}  
                on:click|stopPropagation={() => dispatch('selectItem', {menuItem: item})}
                on:keydown role="button" tabindex={0}>
                {item}
            </div>
        {/each}
    </div>
{/if}

<style>
    .menu-item {
        /* shape */
        border-radius: 5px;
        border: none;
        padding-left: 9px;
        padding-right: 9px;
        display: flex;
        align-items: center;
        height: 26px;

        /* inner */
        background: #2c2c2c; 
        white-space: nowrap;

        /* font */
        color: #d9d9d9; 
        font-size: 0.8rem;

        /* user interaction */
        user-select: none;
        cursor: pointer;
    }

    .menu-item:hover {
        background-color: #3e3e3e; 
    }

    .menu-item.hover {
        background-color: #3e3e3e;
    }

    .selection-menu {
        /* shape */
        border-radius: 3px;
        background: #2c2c2c;
        border: #1a1a1a solid 0.5px; 

        /* font */
        color: #d9d9d9; 

        /* position */
        position: absolute;
        z-index: 1;

        /* inner */
        background-color: #2c2c2c;
    }
</style>
