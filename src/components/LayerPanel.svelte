<script>
    export let layers = [];
    export let legend = [];

    function toggleLayer(layer) {
        layer.visible = !layer.visible;
        layer.toggle(layer.visible);
    }
</script>

<div class="layer-panel">
    {#each layers as layer}
        <div class="layer-item">
            <div class="layer-header">
                <div
                    class="layer-mobile"
                    style="display: flex; gap:20px; align-items:center"
                >
                    <!-- SWITCH -->
                    <label class="switch">
                        <input
                            type="checkbox"
                            checked={layer.visible}
                            on:change={() => toggleLayer(layer)}
                        />
                        <span class="slider"></span>
                    </label>
                    <div class="layer-title">{layer.name}</div>
                </div>
                <div class="layer-description">{layer.description}</div>
            </div>
        </div>
    {/each}

    <div class="legend">
        <div style="font-size: 0.7em;line-height: 1.15em;margin-bottom:10px;">
            LEGEND
        </div>
        <div class="legend-list">
            {#each legend as item}
                <div class="legend-item">
                    <div class="legend-icon-wrapper">
                        <span class="legend-icon">{@html item.svgHtml}</span>
                    </div>
                    <span class="legend-type">{item.type}</span>
                </div>
            {/each}
        </div>
    </div>
</div>

<style>
    .layer-panel {
        position: absolute;
        bottom: 100px;
        right: 24px;
        z-index: 9988;
        width: 220px;
        background: white;
        color: #213547;
        padding: 1rem;
        border-radius: 0.2rem;
        box-shadow: 0 2px 16px rgba(0, 0, 0, 0.1);
        min-width: 20vw;
        max-width: 30vw;
        max-height: 60vh;
        font-family: "Karla-Cedille", sans-serif;
        font-size: 1.3em !important;
        line-height: 1.3;
        background-image: url("../assets/orchard.svg");
        background-size: 100px 100px;
        text-align: start;
        overflow-y: scroll;
    }

    h3 {
        margin: 0 0 12px;
        font-size: 1.1em;
        font-weight: 600;
    }

    .layer-item {
        margin-bottom: 12px;
        padding-bottom: 10px;
    }

    .layer-title {
        font-size: 0.85em;
        line-height: 1.15em;
        font-weight: 600;
    }

    .layer-description {
        font-size: 0.7em;
        line-height: 1.15em;
        color: #aa9f9f;
        margin-top: 0px;
    }

    /* SWITCH STYLE */
    .switch {
        position: relative;
        display: inline-block;
        min-width: 32px;
        width: 32px;
        height: 12px;
        margin-left: 10px;
    }

    .switch input {
        opacity: 0;
        width: 0;
        height: 0;
    }

    .slider {
        position: absolute;
        cursor: pointer;
        inset: 0;
        background-color: #ccc;
        border-radius: 22px;
        transition: 0.3s;
    }

    .slider:before {
        position: absolute;
        content: "";
        width: 8px;
        height: 8px;
        left: 2px;
        bottom: 2px;
        background-color: white;
        border-radius: 50%;
        transition: 0.3s;
    }

    input:checked + .slider {
        background-color: #262626;
    }

    input:checked + .slider:before {
        transform: translateX(20px);
    }

    .legend-item {
        display: flex;
        gap: 20px;
        align-items: center;
    }

    .legend-type {
        font-size: 0.7em;
        line-height: 1.15em;
    }

    .legend-icon-wrapper {
        width: 22px;
        height: 22px;
        position: relative;
        overflow: hidden;
    }

    .legend-icon {
        display: block;
        width: 100%;
        height: 100%;
    }

    .legend-icon svg {
        width: 100%;
        height: 100%;
        max-width: 100%;
        max-height: 100%;
        display: block;
    }

    @media only screen and (max-width: 600px) {
        .legend {
            display: none;
        }

        .layer-title {
            font-size: 0.8em !important;
        }

        .layer-mobile {
            display: flow-root !important;
        }

        .layer-panel {
            top: unset !important;
            bottom: 100px !important;
            min-width: 45vw;
        }
    }
</style>
