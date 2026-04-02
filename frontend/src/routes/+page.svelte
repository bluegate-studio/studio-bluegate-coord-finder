<script>
    import { ImageUp, ClipboardCopy, Trash2 } from '@lucide/svelte';

    let image_src = $state(null);
    let file_name = $state('');
    let is_dragging = $state(false);

    let natural_w = $state(0);
    let natural_h = $state(0);
    let rendered_w = $state(0);
    let rendered_h = $state(0);

    let aspect_ratio = $derived(
        natural_w > 0 ? (natural_h / natural_w).toFixed(4) : '—'
    );

    let hover_x = $state(null);
    let hover_y = $state(null);

    let real_hover_x = $derived(
        hover_x !== null && rendered_w > 0
            ? Math.round((hover_x / rendered_content_w) * natural_w)
            : null
    );
    let real_hover_y = $derived(
        hover_y !== null && rendered_h > 0
            ? Math.round((hover_y / rendered_content_h) * natural_h)
            : null
    );

    let rendered_content_w = $state(0);
    let rendered_content_h = $state(0);

    let hover_display = $derived(
        hover_x !== null
            ? `${hover_x}, ${hover_y}  |  ${real_hover_x}, ${real_hover_y}`
            : '—'
    );

    /** @type {HTMLImageElement|null} */
    let img_el = $state(null);

    /** @type {Array<{x: number, y: number, rx: number, ry: number}>} */
    let saved_coords = $state([]);

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

    function handle_image_load() {
        if (!img_el) return;
        natural_w = img_el.naturalWidth;
        natural_h = img_el.naturalHeight;
        measure_rendered();
    }

    function measure_rendered() {
        if (!img_el) return;
        rendered_w = img_el.clientWidth;
        rendered_h = img_el.clientHeight;
    }

    /**
     * Calculates the actual visible content rect inside the <img> element,
     * accounting for object-fit: contain + object-position: 50% 0%.
     */
    function get_content_rect() {
        if (!img_el || !natural_w || !natural_h) return null;

        const el_w = img_el.clientWidth;
        const el_h = img_el.clientHeight;
        const img_ratio = natural_w / natural_h;
        const el_ratio = el_w / el_h;

        let content_w, content_h;

        if (img_ratio > el_ratio) {
            // width-constrained (horizontal dead zones = none, vertical dead zones at bottom)
            content_w = el_w;
            content_h = el_w / img_ratio;
        } else {
            // height-constrained (vertical dead zones = none, horizontal dead zones on sides)
            content_h = el_h;
            content_w = el_h * img_ratio;
        }

        // object-position: 50% 0% → centered horizontally, top-aligned vertically
        const offset_x = (el_w - content_w) / 2;
        const offset_y = 0;

        return { content_w, content_h, offset_x, offset_y };
    }

    function handle_mouse_move(e) {
        const rect_info = get_content_rect();
        if (!rect_info) return;

        const { content_w, content_h, offset_x, offset_y } = rect_info;
        const bounds = img_el.getBoundingClientRect();

        const cx = (e.clientX - bounds.left) - offset_x;
        const cy = (e.clientY - bounds.top) - offset_y;

        if (cx < 0 || cx > content_w || cy < 0 || cy > content_h) {
            hover_x = null;
            hover_y = null;
            return;
        }

        hover_x = Math.round(cx);
        hover_y = Math.round(cy);
        rendered_content_w = content_w;
        rendered_content_h = content_h;
    }

    function handle_mouse_leave() {
        hover_x = null;
        hover_y = null;
    }

    function handle_click() {
        if (hover_x === null || real_hover_x === null) return;
        saved_coords.push({
            x: hover_x,
            y: hover_y,
            rx: real_hover_x,
            ry: real_hover_y,
        });
    }

    let resize_timer;
    function measure_rendered_debounced() {
        clearTimeout(resize_timer);
        resize_timer = setTimeout(measure_rendered, 120);
    }

    $effect(() => {
        if (!img_el) return;
        const ro = new ResizeObserver(() => measure_rendered_debounced());
        ro.observe(img_el);
        return () => {
            clearTimeout(resize_timer);
            ro.disconnect();
        };
    });
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
    <!-- ── Workspace ──────────────────────────────── -->
    <div class="workspace">
        <div class="workspace__canvas">
            <img
                bind:this={img_el}
                src={image_src}
                alt={file_name}
                loading="eager"
                decoding="async"
                role="application"
                onload={handle_image_load}
                onmousemove={handle_mouse_move}
                onmouseleave={handle_mouse_leave}
                onclick={handle_click}
            />
        </div>

        <aside class="workspace__sidebar">
            <div class="sidebar-section">
                <h3 class="sidebar-section__title">Image Info</h3>

                <div class="info-row">
                    <span class="info-row__label">File</span>
                    <span class="info-row__value" title={file_name}>{file_name}</span>
                </div>

                <div class="info-row">
                    <span class="info-row__label">Real size</span>
                    <span class="info-row__value">{natural_w} × {natural_h} px</span>
                </div>

                <div class="info-row">
                    <span class="info-row__label">Actual size</span>
                    <span class="info-row__value">{rendered_w} × {rendered_h} px</span>
                </div>

                <div class="info-row">
                    <span class="info-row__label">Aspect ratio (h/w)</span>
                    <span class="info-row__value">{aspect_ratio}</span>
                </div>

                <div class="info-row">
                    <span class="info-row__label">Hover (X, Y)</span>
                    <span class="info-row__value">{hover_display}</span>
                </div>
            </div>

            <div class="sidebar-section sidebar-section--grow">
                <div class="sidebar-section__header">
                    <h3 class="sidebar-section__title">Saved Coordinates</h3>
                    <button class="sidebar-section__action" title="Copy to clipboard">
                        <ClipboardCopy size={14} strokeWidth={2} />
                    </button>
                </div>

                <table class="coord-table">
                    <thead>
                        <tr>
                            <th class="coord-table__th" rowspan="2">#</th>
                            <th class="coord-table__th coord-table__th--group" colspan="2">Resized</th>
                            <th class="coord-table__th coord-table__th--group" colspan="2">Real</th>
                            <th class="coord-table__th" rowspan="2"></th>
                        </tr>
                        <tr>
                            <th class="coord-table__th">X</th>
                            <th class="coord-table__th">Y</th>
                            <th class="coord-table__th">X</th>
                            <th class="coord-table__th">Y</th>
                        </tr>
                    </thead>
                    <tbody>
                        {#each saved_coords as coord, i}
                            <tr class="coord-table__row">
                                <td class="coord-table__td coord-table__td--index">{i + 1}</td>
                                <td class="coord-table__td">{coord.x}</td>
                                <td class="coord-table__td">{coord.y}</td>
                                <td class="coord-table__td">{coord.rx}</td>
                                <td class="coord-table__td">{coord.ry}</td>
                                <td class="coord-table__td coord-table__td--action">
                                    <button class="coord-table__delete" title="Remove">
                                        <Trash2 size={13} strokeWidth={2} />
                                    </button>
                                </td>
                            </tr>
                        {/each}
                    </tbody>
                </table>
            </div>
        </aside>
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

    /* ── Workspace ────────────────────────────────── */
    .workspace {
        width: 100vw;
        height: 100vh;
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 1rem;
        padding: 5vh 2vw;
    }

    .workspace__canvas {
        flex: 1;
        height: 90vh;
        min-width: 0;
        display: flex;
        align-items: flex-start;
        justify-content: center;
    }

    .workspace__canvas img {
        max-width: 100%;
        max-height: 100%;
        object-fit: contain;
        object-position: 50% 0%;
        cursor: crosshair;
    }

    /* ── Sidebar ──────────────────────────────────── */
    .workspace__sidebar {
        width: 20rem;
        flex-shrink: 0;
        height: 90vh;
        overflow-y: auto;
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .sidebar-section {
        background: #f2f2f5;
        border: 1px solid #dcdce0;
        border-radius: 0.75rem;
        padding: 1rem;
        display: flex;
        flex-direction: column;
        gap: 0.65rem;
    }

    .sidebar-section--grow {
        flex: 1;
        min-height: 0;
        overflow-y: auto;
    }

    .sidebar-section__header {
        display: flex;
        align-items: center;
        justify-content: space-between;
    }

    .sidebar-section__action {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 1.6rem;
        height: 1.6rem;
        padding: 0;
        border: 1px solid #dcdce0;
        border-radius: 0.35rem;
        background: #fff;
        color: #666;
        cursor: pointer;
        transition: color 0.15s ease, border-color 0.15s ease;
    }

    .sidebar-section__action:hover {
        color: #1a1a1f;
        border-color: #aaa;
    }

    .sidebar-section__title {
        margin: 0;
        font-size: 0.7rem;
        font-weight: 600;
        text-transform: uppercase !important;
        letter-spacing: 0.08em;
        color: #888;
    }

    .info-row {
        display: flex;
        flex-direction: column;
        gap: 0.15rem;
    }

    .info-row__label {
        font-size: 0.7rem;
        color: #888;
    }

    .info-row__value {
        font-size: 0.85rem;
        font-weight: 500;
        color: #1a1a1f;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    /* ── Coord Table ─────────────────────────────── */
    .coord-table {
        width: 100%;
        border-collapse: collapse;
        font-size: 0.72rem;
        font-variant-numeric: tabular-nums;
    }

    .coord-table__th {
        padding: 0.3rem 0.35rem;
        font-weight: 600;
        color: #999;
        text-align: center;
        border-bottom: 1px solid #dcdce0;
        font-size: 0.65rem;
        letter-spacing: 0.03em;
    }

    .coord-table__th--group {
        border-bottom: 1px solid #ccc;
    }

    .coord-table__row:hover {
        background: #eaeaef;
    }

    .coord-table__td {
        padding: 0.3rem 0.35rem;
        text-align: center;
        color: #1a1a1f;
        border-bottom: 1px solid #f0f0f3;
    }

    .coord-table__td--index {
        font-weight: 600;
        color: #999;
        width: 1.6rem;
    }

    .coord-table__td--action {
        width: 1.6rem;
        padding: 0.2rem;
    }

    .coord-table__delete {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 1.4rem;
        height: 1.4rem;
        padding: 0;
        border: none;
        border-radius: 0.25rem;
        background: transparent;
        color: #bbb;
        cursor: pointer;
        transition: color 0.15s ease, background 0.15s ease;
    }

    .coord-table__delete:hover {
        color: #e44;
        background: rgba(228, 68, 68, 0.08);
    }
</style>
