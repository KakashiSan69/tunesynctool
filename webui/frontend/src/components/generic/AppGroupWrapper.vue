<template>
    <div class="p-3 rounded-xl border border-zinc-700/50 flex flex-col gap-3">
        <div class="flex flex-col gap-1" v-if="$slots.label || $slots.description">
            <div class="block whitespace-nowrap text-sm font-semibold leading-5 text-zinc-300">
                <slot name="label" />
            </div>
            <span v-if="$slots.description" class="block whitespace-nowrap text-xs font-normal leading-4 text-zinc-500">
                <slot name="description" />
            </span>
        </div>
        <div>
            <slot name="content" />
        </div>
    </div>   
</template>