<template>
    <button
        type="button"
        class="dp__btn dp__month_year_col_nav"
        @click="$emit('activate')"
        @keydown.enter="$emit('activate')"
        @keydown.space="$emit('activate')"
        tabindex="0"
        :aria-label="ariaLabel"
        :aria-disabled="disabled"
        ref="elRef"
    >
        <span class="dp__inner_nav" :class="{ dp__inner_nav_disabled: disabled }">
            <slot />
        </span>
    </button>
</template>

<script lang="ts" setup>
    import { onMounted, ref } from 'vue';

    const emit = defineEmits(['activate', 'set-ref']);

    interface Props {
        ariaLabel?: string;
        disabled: boolean;
    }

    defineProps<Props>();

    const elRef = ref(null);

    onMounted(() => emit('set-ref', elRef));
</script>
