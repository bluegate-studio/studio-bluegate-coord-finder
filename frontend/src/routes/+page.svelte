<script>
    import { ImageUp, ClipboardCopy, Trash2, MousePointer2, Minus, Plus } from '@lucide/svelte';

    let image_src = $state(null);
    let file_name = $state('');
    let is_dragging = $state(false);
    let started = $state(false);

    /* ── Custom cursor image (no-op for now) ──────── */
    let cursor_src = $state(null);
    let cursor_name = $state('');
    let is_dragging_cursor = $state(false);
    let cursor_needs_centering = false;

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

    /* ── Custom cursor overlay state (real-pixel-integer space) ── */
    let cursor_natural_w = $state(0);
    let cursor_natural_h = $state(0);
    let cursor_pos_x = $state(0); // real-pixel X (top-left of overlay)
    let cursor_pos_y = $state(0); // real-pixel Y (top-left of overlay)

    let scale_factor = $derived(rendered_content_w > 0 ? rendered_content_w / natural_w : 1);
    let cursor_display_w = $derived(cursor_natural_w * scale_factor);
    let cursor_display_h = $derived(cursor_natural_h * scale_factor);
    // Real center — exact integers, no rounding
    let cursor_real_center_x = $derived(cursor_pos_x + Math.round(cursor_natural_w / 2));
    let cursor_real_center_y = $derived(cursor_pos_y + Math.round(cursor_natural_h / 2));
    // Rendered center — derived from real, for display only
    let cursor_center_x = $derived(Math.round(cursor_real_center_x * scale_factor));
    let cursor_center_y = $derived(Math.round(cursor_real_center_y * scale_factor));

    let has_overlay = $derived(!!cursor_src && cursor_natural_w > 0);

    let hover_display = $derived(
        has_overlay
            ? `${cursor_center_x}, ${cursor_center_y}  |  ${cursor_real_center_x}, ${cursor_real_center_y}`
            : hover_x !== null
                ? `${hover_x}, ${hover_y}  |  ${real_hover_x}, ${real_hover_y}`
                : '—'
    );

    /** @type {HTMLImageElement|null} */
    let img_el = $state(null);

    /** @type {HTMLDivElement|null} */
    let canvas_el = $state(null);

    /* ── Zoom & Pan ─────────────────────────────── */
    let zoom = $state(1);
    let manual_pan_x = $state(0);
    let manual_pan_y = $state(0);

    function clamp_zoom(v) {
        return Math.max(1, Math.min(20, v));
    }

    function compute_auto_pan_x() {
        if (!img_el || !canvas_el) return 0;
        const rect = get_content_rect();
        if (!rect) return 0;
        const center_in_wrapper = img_el.offsetLeft + rect.offset_x + cursor_pos_x * scale_factor + cursor_display_w / 2;
        return (canvas_el.clientWidth / 2) - center_in_wrapper * zoom;
    }

    function compute_auto_pan_y() {
        if (!img_el || !canvas_el) return 0;
        const rect = get_content_rect();
        if (!rect) return 0;
        const center_in_wrapper = img_el.offsetTop + rect.offset_y + cursor_pos_y * scale_factor + cursor_display_h / 2;
        return (canvas_el.clientHeight / 2) - center_in_wrapper * zoom;
    }

    let pan_x = $derived(has_overlay && zoom > 1 ? compute_auto_pan_x() : manual_pan_x);
    let pan_y = $derived(has_overlay && zoom > 1 ? compute_auto_pan_y() : manual_pan_y);

    let zoom_display = $derived(zoom.toFixed(1) + '×');

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

    function handle_cursor_drop(e) {
        e.preventDefault();
        is_dragging_cursor = false;
        const file = e.dataTransfer?.files?.[0];
        if (file) load_cursor_file(file);
    }

    function handle_cursor_drag_over(e) {
        e.preventDefault();
        is_dragging_cursor = true;
    }

    function handle_cursor_drag_leave() {
        is_dragging_cursor = false;
    }

    function load_cursor_file(file) {
        if (!file || !file.type.startsWith('image/')) return;
        cursor_name = file.name;
        const reader = new FileReader();
        reader.onload = (e) => {
            const data_url = e.target.result;
            cursor_src = data_url;
            // Probe natural dimensions
            const probe = new Image();
            probe.onload = () => {
                cursor_natural_w = probe.naturalWidth;
                cursor_natural_h = probe.naturalHeight;
                if (rendered_content_w > 0) {
                    // Workspace already rendered — center now
                    center_cursor_overlay();
                } else {
                    // Workspace not yet rendered — defer
                    cursor_pos_x = 0;
                    cursor_pos_y = 0;
                    cursor_needs_centering = true;
                }
            };
            probe.src = data_url;
        };
        reader.readAsDataURL(file);
    }

    function handle_cursor_file_select(e) {
        const file = e.target.files?.[0];
        if (file) load_cursor_file(file);
    }

    function handle_begin() {
        if (!image_src) return;
        started = true;
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

        // Keep content dimensions in sync for overlay positioning
        const rect = get_content_rect();
        if (rect) {
            rendered_content_w = rect.content_w;
            rendered_content_h = rect.content_h;

            // Deferred centering (cursor loaded before workspace rendered)
            if (cursor_needs_centering && cursor_natural_w > 0) {
                center_cursor_overlay();
                cursor_needs_centering = false;
            }
        }
    }

    function center_cursor_overlay() {
        cursor_pos_x = Math.round((natural_w - cursor_natural_w) / 2);
        cursor_pos_y = Math.round((natural_h - cursor_natural_h) / 2);
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
        if (has_overlay) return;
        const rect_info = get_content_rect();
        if (!rect_info) return;

        const { content_w, content_h, offset_x, offset_y } = rect_info;
        const bounds = img_el.getBoundingClientRect();

        const cx = ((e.clientX - bounds.left) / zoom) - offset_x;
        const cy = ((e.clientY - bounds.top) / zoom) - offset_y;

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
        if (has_overlay) return;
        hover_x = null;
        hover_y = null;
    }

    function handle_click() {
        if (has_overlay) return; // in overlay mode, only overlay saves
        if (hover_x === null || real_hover_x === null) return;
        saved_coords.push({
            x: hover_x,
            y: hover_y,
            rx: real_hover_x,
            ry: real_hover_y,
        });
    }

    /* ── Overlay drag ─────────────────────────────── */
    let is_overlay_dragging = $state(false);
    let drag_start_mouse_x = 0;
    let drag_start_mouse_y = 0;
    let drag_start_pos_x = 0;
    let drag_start_pos_y = 0;

    function clamp_cursor() {
        const max_x = natural_w - cursor_natural_w;
        const max_y = natural_h - cursor_natural_h;
        cursor_pos_x = Math.max(0, Math.min(max_x, cursor_pos_x));
        cursor_pos_y = Math.max(0, Math.min(max_y, cursor_pos_y));
    }

    function start_overlay_drag(e) {
        e.preventDefault();
        is_overlay_dragging = true;
        drag_start_mouse_x = e.clientX;
        drag_start_mouse_y = e.clientY;
        drag_start_pos_x = cursor_pos_x;
        drag_start_pos_y = cursor_pos_y;
        document.addEventListener('mousemove', do_overlay_drag);
        document.addEventListener('mouseup', end_overlay_drag);
    }

    function do_overlay_drag(e) {
        cursor_pos_x = Math.round(drag_start_pos_x + (e.clientX - drag_start_mouse_x) / (zoom * scale_factor));
        cursor_pos_y = Math.round(drag_start_pos_y + (e.clientY - drag_start_mouse_y) / (zoom * scale_factor));
        clamp_cursor();
    }

    function end_overlay_drag() {
        is_overlay_dragging = false;
        document.removeEventListener('mousemove', do_overlay_drag);
        document.removeEventListener('mouseup', end_overlay_drag);
    }

    /* ── Overlay keyboard ─────────────────────────── */
    /** @type {HTMLDivElement|null} */
    let overlay_el = $state(null);

    function handle_overlay_keydown(e) {
        const step = e.shiftKey ? 10 : 1;

        switch (e.key) {
            case 'ArrowLeft':
                e.preventDefault();
                cursor_pos_x -= step;
                break;
            case 'ArrowRight':
                e.preventDefault();
                cursor_pos_x += step;
                break;
            case 'ArrowUp':
                e.preventDefault();
                cursor_pos_y -= step;
                break;
            case 'ArrowDown':
                e.preventDefault();
                cursor_pos_y += step;
                break;
            case 'Enter':
            case ' ':
                e.preventDefault();
                save_overlay_coords();
                break;
        }
    }

    function save_overlay_coords() {
        saved_coords.push({
            x: cursor_center_x,
            y: cursor_center_y,
            rx: cursor_real_center_x,
            ry: cursor_real_center_y,
        });
    }

    /**
     * Compute the overlay's absolute CSS left/top within the canvas.
     */
    function get_overlay_left() {
        if (!img_el) return 0;
        const rect = get_content_rect();
        if (!rect) return 0;
        return img_el.offsetLeft + rect.offset_x + cursor_pos_x * scale_factor;
    }

    function get_overlay_top() {
        if (!img_el) return 0;
        const rect = get_content_rect();
        if (!rect) return 0;
        return img_el.offsetTop + rect.offset_y + cursor_pos_y * scale_factor;
    }

    /* ── Delete confirmation ──────────────────────── */
    let delete_index = $state(null);
    let show_delete_modal = $derived(delete_index !== null);

    function request_delete(i) {
        delete_index = i;
    }

    function confirm_delete() {
        if (delete_index !== null) {
            saved_coords.splice(delete_index, 1);
        }
        delete_index = null;
    }

    function cancel_delete() {
        delete_index = null;
    }

    function handle_modal_keyup(e) {
        if (e.key === 'Enter') {
            e.preventDefault();
            confirm_delete();
        } else if (e.key === 'Escape') {
            cancel_delete();
        }
    }

    /** @type {HTMLDivElement|null} */
    let veil_el = $state(null);

    $effect(() => {
        if (show_delete_modal && veil_el) {
            veil_el.focus();
        }
    });

    /* ── Copy to clipboard ───────────────────────── */
    let show_copy_modal = $state(false);

    async function handle_copy() {
        if (!saved_coords.length) return;

        const payload = {
            real: saved_coords.map(c => ({ x: c.rx, y: c.ry })),
            resized: saved_coords.map(c => ({ x: c.x, y: c.y })),
        };

        try {
            await navigator.clipboard.writeText(JSON.stringify(payload, null, 2));
            show_copy_modal = true;
        } catch (_) {
            /* clipboard access denied — fail silently */
        }
    }

    function dismiss_copy_modal() {
        show_copy_modal = false;
    }

    function handle_copy_modal_keyup(e) {
        if (e.key === 'Enter' || e.key === 'Escape') {
            dismiss_copy_modal();
        }
    }

    /** @type {HTMLDivElement|null} */
    let copy_veil_el = $state(null);

    $effect(() => {
        if (show_copy_modal && copy_veil_el) {
            copy_veil_el.focus();
        }
    });

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

    /* ── Zoom handlers ───────────────────────────── */
    function handle_wheel(e) {
        e.preventDefault();
        const factor = e.deltaY > 0 ? 0.97 : 1.03;
        const new_zoom = clamp_zoom(zoom * factor);

        if (has_overlay) {
            // Overlay mode: just zoom, pan auto-adjusts to keep overlay centered
            zoom = new_zoom;
        } else {
            // Non-overlay mode: zoom toward cursor position
            const cr = canvas_el.getBoundingClientRect();
            const mx = e.clientX - cr.left;
            const my = e.clientY - cr.top;
            manual_pan_x = mx - (mx - manual_pan_x) * (new_zoom / zoom);
            manual_pan_y = my - (my - manual_pan_y) * (new_zoom / zoom);
            zoom = new_zoom;
        }
    }

    function zoom_in_sidebar() {
        const new_zoom = clamp_zoom(zoom * 1.1);
        if (!has_overlay && canvas_el) {
            const cx = canvas_el.clientWidth / 2;
            const cy = canvas_el.clientHeight / 2;
            manual_pan_x = cx - (cx - manual_pan_x) * (new_zoom / zoom);
            manual_pan_y = cy - (cy - manual_pan_y) * (new_zoom / zoom);
        }
        zoom = new_zoom;
    }

    function zoom_out_sidebar() {
        const new_zoom = clamp_zoom(zoom * 0.9);
        if (!has_overlay && canvas_el) {
            const cx = canvas_el.clientWidth / 2;
            const cy = canvas_el.clientHeight / 2;
            manual_pan_x = cx - (cx - manual_pan_x) * (new_zoom / zoom);
            manual_pan_y = cy - (cy - manual_pan_y) * (new_zoom / zoom);
        }
        zoom = new_zoom;
    }

    function handle_canvas_keydown(e) {
        if (has_overlay) return; // overlay handles its own keys
        const step = e.shiftKey ? 100 : 10;

        switch (e.key) {
            case 'ArrowLeft':
                e.preventDefault();
                manual_pan_x += step;
                break;
            case 'ArrowRight':
                e.preventDefault();
                manual_pan_x -= step;
                break;
            case 'ArrowUp':
                e.preventDefault();
                manual_pan_y += step;
                break;
            case 'ArrowDown':
                e.preventDefault();
                manual_pan_y -= step;
                break;
        }
    }

    function handle_canvas_dblclick() {
        zoom = 1;
        manual_pan_x = 0;
        manual_pan_y = 0;
    }

    function handle_body_keydown(e) {
        if (show_delete_modal || show_copy_modal) return;
        if (has_overlay) {
            const step = e.ctrlKey && e.shiftKey ? 100 : e.shiftKey ? 10 : 1;
            switch (e.key) {
                case 'ArrowLeft':
                    e.preventDefault();
                    e.stopPropagation();
                    cursor_pos_x -= step;
                    break;
                case 'ArrowRight':
                    e.preventDefault();
                    e.stopPropagation();
                    cursor_pos_x += step;
                    break;
                case 'ArrowUp':
                    e.preventDefault();
                    e.stopPropagation();
                    cursor_pos_y -= step;
                    break;
                case 'ArrowDown':
                    e.preventDefault();
                    e.stopPropagation();
                    cursor_pos_y += step;
                    break;
                case 'Enter':
                case ' ':
                    e.preventDefault();
                    e.stopPropagation();
                    save_overlay_coords();
                    break;
            }
            clamp_cursor();
        }
    }
</script>

<svelte:body onkeydown={handle_body_keydown} />

{#if !started}
    <!-- ── Landing ─────────────────────────────────── -->
    <div class="drop-zone-wrapper">
        <!-- Main image drop zone -->
        <div
            class="drop-zone"
            class:dragging={is_dragging}
            class:done={image_src}
            role="button"
            tabindex="0"
            ondrop={handle_drop}
            ondragover={handle_drag_over}
            ondragleave={handle_drag_leave}
        >
            <div class="drop-zone__icon">
                <ImageUp size={48} strokeWidth={1.5} />
            </div>
            {#if image_src}
                <p class="drop-zone__title">{file_name}</p>
                <p class="drop-zone__subtitle">Main image selected ✓</p>
            {:else}
                <p class="drop-zone__title">Drop your image here</p>
                <p class="drop-zone__subtitle">or</p>
            {/if}
            <label class="drop-zone__btn">
                {image_src ? 'Change file' : 'Browse files'}
                <input
                    type="file"
                    accept="image/*"
                    onchange={handle_file_select}
                />
            </label>
            <p class="drop-zone__hint">PNG, JPG, SVG, WebP</p>
        </div>

        <!-- Custom cursor drop zone -->
        <div
            class="drop-zone drop-zone--small"
            class:dragging={is_dragging_cursor}
            class:done={cursor_src}
            role="button"
            tabindex="0"
            ondrop={handle_cursor_drop}
            ondragover={handle_cursor_drag_over}
            ondragleave={handle_cursor_drag_leave}
        >
            <div class="drop-zone__icon">
                <MousePointer2 size={32} strokeWidth={1.5} />
            </div>
            {#if cursor_src}
                <p class="drop-zone__title drop-zone__title--sm">{cursor_name}</p>
                <p class="drop-zone__subtitle">Custom image selected ✓</p>
            {:else}
                <p class="drop-zone__subtitle">You can use a custom image to match elements in your main image or as a custom cursor. It will be resized proportionally to the main image.</p>
                <p class="drop-zone__title drop-zone__title--sm">Drop your image here</p>
                <p class="drop-zone__subtitle">or</p>
            {/if}
            <label class="drop-zone__btn drop-zone__btn--secondary">
                {cursor_src ? 'Change file' : 'Browse files'}
                <input
                    type="file"
                    accept="image/*"
                    onchange={handle_cursor_file_select}
                />
            </label>
        </div>

        <!-- Begin button -->
        <button
            class="begin-btn"
            class:disabled={!image_src}
            onclick={handle_begin}
        >
            Begin
        </button>
    </div>
{:else}
    <!-- ── Workspace ──────────────────────────────── -->
    <div class="workspace">
        <!-- svelte-ignore a11y_no_static_element_interactions -->
        <!-- svelte-ignore a11y_no_noninteractive_tabindex -->
        <div
            class="workspace__canvas"
            bind:this={canvas_el}
            tabindex="0"
            onwheel={handle_wheel}
            onkeydown={handle_canvas_keydown}
            ondblclick={handle_canvas_dblclick}
        >
            <div
                class="zoom-wrapper"
                style:transform="translate({pan_x}px, {pan_y}px) scale({zoom})"
            >
                <!-- svelte-ignore a11y_click_events_have_key_events -->
                <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
                <img
                    bind:this={img_el}
                    src={image_src}
                    alt={file_name}
                    loading="eager"
                    decoding="async"
                    role="application"
                    class:has-overlay={has_overlay}
                    onload={handle_image_load}
                    onmousemove={handle_mouse_move}
                    onmouseleave={handle_mouse_leave}
                    onclick={handle_click}
                />

                {#if has_overlay}
                    <!-- svelte-ignore a11y_no_static_element_interactions -->
                    <!-- svelte-ignore a11y_no_noninteractive_tabindex -->
                    <div
                        class="cursor-overlay"
                        class:dragging={is_overlay_dragging}
                        bind:this={overlay_el}
                        tabindex="0"
                        style:left="{get_overlay_left()}px"
                        style:top="{get_overlay_top()}px"
                        style:width="{cursor_display_w}px"
                        style:height="{cursor_display_h}px"
                        style:background-image="url({cursor_src})"
                        onmousedown={start_overlay_drag}
                        onkeydown={handle_overlay_keydown}
                    ></div>
                {/if}
            </div>
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
                    <span class="info-row__label">Zoom</span>
                    <span class="info-row__value info-row__value--zoom">
                        <button class="zoom-btn" title="Zoom out" onclick={zoom_out_sidebar}><Minus size={12} strokeWidth={2.5} /></button>
                        <span>{zoom_display}</span>
                        <button class="zoom-btn" title="Zoom in" onclick={zoom_in_sidebar}><Plus size={12} strokeWidth={2.5} /></button>
                    </span>
                </div>

                <div class="info-row">
                    <span class="info-row__label">Hover (X, Y)</span>
                    <span class="info-row__value">{hover_display}</span>
                </div>
            </div>

            <div class="sidebar-section sidebar-section--grow">
                <div class="sidebar-section__header">
                    <h3 class="sidebar-section__title">Saved Coordinates</h3>
                    <button class="sidebar-section__action" title="Copy to clipboard" onclick={handle_copy}>
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
                                    <button class="coord-table__delete" title="Remove" onclick={() => request_delete(i)}>
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

{#if show_delete_modal}
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <div
        class="modal-veil"
        bind:this={veil_el}
        tabindex="-1"
        onclick={cancel_delete}
        onkeyup={handle_modal_keyup}
    >
        <!-- svelte-ignore a11y_no_static_element_interactions -->
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <div class="modal-dialog" onclick={(e) => e.stopPropagation()}>
            <p class="modal-dialog__title">Delete coordinate #{delete_index + 1}?</p>
            <p class="modal-dialog__text">This action cannot be undone.</p>
            <div class="modal-dialog__actions">
                <button class="modal-dialog__btn modal-dialog__btn--cancel" onclick={cancel_delete}>Cancel</button>
                <button class="modal-dialog__btn modal-dialog__btn--delete" onclick={confirm_delete}>Delete</button>
            </div>
        </div>
    </div>
{/if}

{#if show_copy_modal}
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <div
        class="modal-veil"
        bind:this={copy_veil_el}
        tabindex="-1"
        onclick={dismiss_copy_modal}
        onkeyup={handle_copy_modal_keyup}
    >
        <!-- svelte-ignore a11y_no_static_element_interactions -->
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <div class="modal-dialog" onclick={(e) => e.stopPropagation()}>
            <p class="modal-dialog__title">Copied to clipboard</p>
            <p class="modal-dialog__text">All {saved_coords.length} saved coordinates are in your clipboard as JSON, ready to paste.</p>
            <div class="modal-dialog__actions">
                <button class="modal-dialog__btn modal-dialog__btn--ok" onclick={dismiss_copy_modal}>OK</button>
            </div>
        </div>
    </div>
{/if}

<style>
    /* ── Drop Zone Wrapper ───────────────────────── */
    .drop-zone-wrapper {
        width: 100vw;
        height: 100vh;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        gap: 1.5rem;
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

    .drop-zone.done {
        border-color: #3a7;
        border-style: solid;
    }

    .drop-zone--small {
        padding: 1.8rem 2rem;
        max-width: 34rem;
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

    .drop-zone__title--sm {
        font-size: 0.95rem;
    }

    .drop-zone__subtitle {
        margin: 0;
        font-size: 0.85rem;
        color: var(--clr-text-dim);
        text-align: center;
        line-height: 1.5;
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

    .drop-zone__btn--secondary {
        background: transparent;
        border: 1px solid var(--clr-border);
        color: var(--clr-text-dim);
    }

    .drop-zone__btn--secondary:hover {
        border-color: var(--clr-accent);
        color: var(--clr-text);
    }

    /* ── Begin Button ────────────────────────────── */
    .begin-btn {
        padding: 0.75rem 3rem;
        font-size: 1.05rem;
        font-weight: 600;
        color: #fff;
        background: linear-gradient(135deg, var(--clr-accent), #8b6cff);
        border: none;
        border-radius: 0.7rem;
        cursor: pointer;
        transition: filter 0.2s ease, transform 0.15s ease, opacity 0.2s ease;
    }

    .begin-btn:hover {
        filter: brightness(1.1);
        transform: translateY(-2px);
    }

    .begin-btn:active {
        transform: translateY(0);
    }

    .begin-btn.disabled {
        opacity: 0.35;
        pointer-events: none;
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
        overflow: hidden;
        outline: none;
    }

    .zoom-wrapper {
        width: 100%;
        height: 100%;
        position: relative;
        transform-origin: 0 0;
    }

    .workspace__canvas img {
        width: 100%;
        height: 100%;
        object-fit: contain;
        object-position: 50% 0%;
        cursor: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='32' height='32'%3E%3Cline x1='16' y1='0' x2='16' y2='32' stroke='%23fff' stroke-width='1.5'/%3E%3Cline x1='0' y1='16' x2='32' y2='16' stroke='%23fff' stroke-width='1.5'/%3E%3Cline x1='16' y1='0' x2='16' y2='32' stroke='%23000' stroke-width='0.75'/%3E%3Cline x1='0' y1='16' x2='32' y2='16' stroke='%23000' stroke-width='0.75'/%3E%3C/svg%3E") 16 16, crosshair;
    }

    .workspace__canvas img.has-overlay {
        cursor: default;
    }

    /* ── Cursor Overlay ──────────────────────────── */
    .cursor-overlay {
        position: absolute;
        background-size: 100% 100%;
        background-repeat: no-repeat;
        cursor: grab;
        outline: none;
        z-index: 10;
        user-select: none;
    }

    .cursor-overlay:focus {
        box-shadow: 0 0 0 2px var(--clr-accent);
    }

    .cursor-overlay.dragging {
        cursor: grabbing;
    }

    /* ── Zoom Buttons ────────────────────────────── */
    .info-row__value--zoom {
        display: flex;
        align-items: center;
        gap: 0.4rem;
    }

    .zoom-btn {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        width: 1.4rem;
        height: 1.4rem;
        padding: 0;
        border: 1px solid var(--clr-border);
        border-radius: 0.3rem;
        background: #fff;
        color: var(--clr-text-dim);
        cursor: pointer;
        transition: border-color 0.15s ease, color 0.15s ease;
    }

    .zoom-btn:hover {
        border-color: var(--clr-accent);
        color: var(--clr-accent);
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

    /* ── Modal ────────────────────────────────────── */
    .modal-veil {
        position: fixed;
        inset: 0;
        z-index: 1000;
        display: flex;
        align-items: center;
        justify-content: center;
        background: rgba(0, 0, 0, 0.35);
        backdrop-filter: blur(4px) saturate(0.4);
        -webkit-backdrop-filter: blur(4px) saturate(0.4);
    }

    .modal-dialog {
        background: #fff;
        border-radius: 0.85rem;
        padding: 1.6rem 1.8rem;
        width: 100%;
        max-width: 20rem;
        box-shadow: 0 12px 40px rgba(0, 0, 0, 0.25);
    }

    .modal-dialog__title {
        margin: 0 0 0.35rem;
        font-size: 1rem;
        font-weight: 600;
        color: #1a1a1f;
    }

    .modal-dialog__text {
        margin: 0 0 1.2rem;
        font-size: 0.8rem;
        color: #888;
    }

    .modal-dialog__actions {
        display: flex;
        justify-content: flex-end;
        gap: 0.5rem;
    }

    .modal-dialog__btn {
        padding: 0.45rem 1rem;
        font-size: 0.8rem;
        font-weight: 500;
        border: none;
        border-radius: 0.45rem;
        cursor: pointer;
        transition: filter 0.15s ease;
    }

    .modal-dialog__btn:hover {
        filter: brightness(0.92);
    }

    .modal-dialog__btn--cancel {
        background: #ececf0;
        color: #444;
    }

    .modal-dialog__btn--delete {
        background: #e44;
        color: #fff;
    }

    .modal-dialog__btn--ok {
        background: var(--clr-accent);
        color: #fff;
    }
</style>
