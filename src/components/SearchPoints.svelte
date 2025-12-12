<script>
    import { points } from "../data/points.js";
    import { toponyms } from "../data/toponyms.js";
    import { t, waitLocale } from "svelte-i18n";
    import { onMount, createEventDispatcher } from "svelte";
    import { landmarks } from "../data/landmarks.js";

    export let onSelect = (point) => {};
    export let activeLayers = ["points", "toponyms"]; // array of active layer IDs

    let query = "";
    let focused = false;
    let ready = false;

    const dispatch = createEventDispatcher();

    // Wait until translations are loaded
    onMount(async () => {
        await waitLocale();
        ready = true;
    });

    // Build unified search index from both layers
    $: searchIndex = [
        ...landmarks.features.map((f) => ({
            id: f.properties.ID,
            name: f.properties.name,
            type: f.properties.type,
            coordinates: f.geometry.coordinates,
            properties: f.properties,
            layerType: "landmarks",
        })),
        ...toponyms.features.map((f) => ({
            id: f.properties.ID,
            name: f.properties.name,
            type: f.properties.type,
            coordinates: f.geometry.coordinates,
            properties: f.properties,
            layerType: "toponyms",
        })),
        ...points.features.map((f) => ({
            id: f.properties.ID,
            name: f.properties.name,
            type: f.properties.type,
            coordinates: f.geometry.coordinates,
            properties: f.properties,
            layerType: "points",
        })),
    ];

    // Reactive filtered results based on query AND active layers
    $: filteredResults =
        query.length > 0 || focused
            ? searchIndex
                  .filter(
                      (p) =>
                          p.name?.toLowerCase().includes(query.toLowerCase()) &&
                          activeLayers.includes(p.layerType),
                  )
                  .sort((a, b) => a.name.localeCompare(b.name))
            : [];

    function selectPoint(point) {
        onSelect(point); // Trigger parent handler
        dispatch("select", point); // Optional Svelte event
        query = "";
        focused = false;
    }
</script>

<div class="search-container">
    {#if ready}
        <input
            type="text"
            bind:value={query}
            placeholder={$t("app.placeholder")}
            on:focus={() => (focused = true)}
            on:blur={() => setTimeout(() => (focused = false), 200)}
        />
    {:else}
        <input type="text" placeholder="..." disabled />
    {/if}

    {#if filteredResults.length && focused}
        <div class="search-results">
            {#each filteredResults as r}
                <button
                    type="button"
                    class="result-item"
                    on:click={() => selectPoint(r)}
                >
                    <strong style="color: {r.properties.hr || '#000'}"
                        >{r.name}</strong
                    >
                    <small style="color: {r.properties.hr || '#000'}"
                        >({r.type})</small
                    >
                </button>
            {/each}
        </div>
    {/if}
</div>

<style>
    .search-container {
        position: absolute;
        top: 25px;
        left: 25px;
        z-index: 9998;
        background: white;
        padding: 0.5rem;
        border-radius: 0.5rem;
        box-shadow: none;
        width: 300px;
        background-image: url("../assets/orchard.svg") !important;
        background-size: 100px 100px !important;
    }

    .search-container > input {
        background-color: white;
        color: #213547 !important;
        width: 90%;
        border-color: #21354730;
    }

    .search-results {
        max-height: 70vh !important;
        overflow-y: auto;
        margin-top: 0.25rem;
        border-top: 1px solid #ccc;
        color: #213547 !important;
    }

    .result-item {
        display: block;
        width: 100%;
        text-align: left;
        background: none;
        border: none;
        padding: 0.5rem;
        cursor: pointer;
        font-family: inherit;
        color: #213547 !important;
    }

    .result-item:hover {
        background-color: #f0f0f0;
    }

    input {
        width: 100%;
        padding: 0.5rem;
    }

    @media only screen and (max-width: 600px) {
        .search-container {
            left: 50%;
            transform: translateX(-50%);
        }
        .search-results {
            max-height: 25vh !important;
        }
    }
</style>
