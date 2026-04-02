<script>
    import { ImageUp } from '@lucide/svelte';

    let image_src = $state(null);
    let file_name = $state('');
    let is_dragging = $state(false);

    /** @param {File} file */
    function load_file(file) {
        if (!file || !file.type.startsWith('image/')) return;
        file_name = file.name;
        const reader = new FileReader();
        reader.onload = (e) => {
            image_src = e.target.result;
        };
        reader.readAsDataURL(file);
    }

    function handle_drop(e) {
        e.preventDefault();
        is_dragging = false;
        const file = e.dataTransfer?.files?.[0];
        if (file) load_file(file);
    }

    function handle_drag_over(e) {
        e.preventDefault();
        is_dragging = true;
    }

    function handle_drag_leave() {
        is_dragging = false;
    }

    function handle_file_select(e) {
        const file = e.target.files?.[0];
        if (file) load_file(file);
    }

    function handle_reset() {
        image_src = null;
        file_name = '';
    }
</script>

{#if !image_src}
    <!-- ── Drop Zone ──────────────────────────────── -->
    <div class="drop-zone-wrapper">
        <div
            class="drop-zone"
            class:dragging={is_dragging}
            role="button"
            tabindex="0"
            ondrop={handle_drop}
            ondragover={handle_drag_over}
            ondragleave={handle_drag_leave}
        >
            <div class="drop-zone__icon">
                <ImageUp size={48} strokeWidth={1.5} />
            </div>
            <p class="drop-zone__title">Drop your image here</p>
            <p class="drop-zone__subtitle">or</p>
            <label class="drop-zone__btn">
                Browse files
                <input
                    type="file"
                    accept="image/*"
                    onchange={handle_file_select}
                />
            </label>
            <p class="drop-zone__hint">PNG, JPG, SVG, WebP</p>
        </div>
    </div>
{:else}
    <!-- ── Image Display ──────────────────────────── -->
    <div class="image-display">
        <img
            src={image_src}
            alt={file_name}
            loading="eager"
            decoding="async"
        />
    </div>
{/if}

<style>
    /* ── Drop Zone Wrapper ───────────────────────── */
    .drop-zone-wrapper {
        width: 100vw;
        height: 100vh;
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 2rem;
    }

    .drop-zone {
        position: relative;
        width: 100%;
        max-width: 34rem;
        padding: 3.5rem 2.5rem;
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 0.6rem;
        border: 2px dashed var(--clr-border);
        border-radius: 1.25rem;
        background: var(--clr-surface);
        cursor: pointer;
        transition: border-color 0.25s ease,
                    background 0.25s ease,
                    box-shadow 0.25s ease;
    }

    .drop-zone:hover,
    .drop-zone.dragging {
        border-color: var(--clr-accent);
        background: var(--clr-surface-hover);
        box-shadow: 0 0 40px var(--clr-accent-glow),
                    inset 0 0 40px var(--clr-accent-glow);
    }

    .drop-zone__icon {
        color: var(--clr-text-dim);
        margin-bottom: 0.5rem;
        transition: color 0.25s ease;
    }

    .drop-zone:hover .drop-zone__icon,
    .drop-zone.dragging .drop-zone__icon {
        color: var(--clr-accent);
    }

    .drop-zone__title {
        margin: 0;
        font-size: 1.2rem;
        font-weight: 600;
        color: var(--clr-text);
    }

    .drop-zone__subtitle {
        margin: 0;
        font-size: 0.85rem;
        color: var(--clr-text-dim);
    }

    .drop-zone__btn {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        padding: 0.6rem 1.6rem;
        margin-top: 0.4rem;
        font-size: 0.9rem;
        font-weight: 500;
        color: #fff;
        background: var(--clr-accent);
        border: none;
        border-radius: 0.6rem;
        cursor: pointer;
        transition: filter 0.2s ease, transform 0.15s ease;
    }

    .drop-zone__btn:hover {
        filter: brightness(1.15);
        transform: translateY(-1px);
    }

    .drop-zone__btn:active {
        transform: translateY(0);
    }

    .drop-zone__btn input {
        position: absolute;
        width: 0;
        height: 0;
        opacity: 0;
        pointer-events: none;
    }

    .drop-zone__hint {
        margin: 0.5rem 0 0;
        font-size: 0.75rem;
        color: var(--clr-text-dim);
        letter-spacing: 0.04em;
    }

    /* ── Image Display ───────────────────────────── */
    .image-display {
        width: 100vw;
        height: 100vh;
        display: flex;
        align-items: center;
        justify-content: center;
    }

    .image-display img {
        width: 90vw;
        height: 90vh;
        object-fit: contain;
    }
</style>
