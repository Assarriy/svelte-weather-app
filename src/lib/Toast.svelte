<script context="module" lang="ts">
    import { writable } from 'svelte/store';

    // Store global untuk menampung notifikasi
    const toasts = writable<{id: number, msg: string, type: 'success'|'error'}[]>([]);

    export const toast = {
        success: (msg: string) => addToast(msg, 'success'),
        error: (msg: string) => addToast(msg, 'error')
    };

    function addToast(msg: string, type: 'success'|'error') {
        const id = Date.now();
        toasts.update(all => [...all, { id, msg, type }]);
        setTimeout(() => {
            toasts.update(all => all.filter(t => t.id !== id));
        }, 3000);
    }
</script>

<script lang="ts">
    import { fly, fade } from 'svelte/transition';
    import { flip } from 'svelte/animate';

    const icons = {
        success: "M9 12.75L11.25 15 15 9.75M21 12a9 9 0 11-18 0 9 9 0 0118 0z",
        error: "M12 9v3.75m9-.75a9 9 0 11-18 0 9 9 0 0118 0zm-9 3.75h.008v.008H12v-.008z"
    };
</script>

<div class="fixed top-6 left-1/2 -translate-x-1/2 z-[100] flex flex-col gap-2 w-full max-w-xs pointer-events-none">
    {#each $toasts as item (item.id)}
        <div 
            animate:flip
            in:fly={{ y: -20, duration: 300 }} 
            out:fade 
            class="pointer-events-auto flex items-center gap-3 px-4 py-3 rounded-2xl shadow-2xl border font-medium text-sm backdrop-blur-xl transition-all
            {item.type === 'success' 
                ? 'bg-white/90 text-slate-800 border-white/40 shadow-emerald-500/10' 
                : 'bg-red-500/90 text-white border-red-400 shadow-red-500/20'}"
        >
            <svg class="w-6 h-6 shrink-0 {item.type === 'success' ? 'text-emerald-500' : 'text-white'}" fill="none" stroke="currentColor" viewBox="0 0 24 24" stroke-width="2">
                <path stroke-linecap="round" stroke-linejoin="round" d={item.type === 'success' ? icons.success : icons.error} />
            </svg>

            <span>{item.msg}</span>
        </div>
    {/each}
</div>