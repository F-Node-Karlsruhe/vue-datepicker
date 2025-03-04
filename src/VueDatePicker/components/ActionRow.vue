<template>
    <div class="dp__action_row" :style="calendarWidth ? { width: `${calendarWidth}px` } : {}">
        <template v-if="$slots['action-row']">
            <slot
                name="action-row"
                v-bind="{
                    internalModelValue,
                    disabled,
                    selectDate: () => $emit('select-date'),
                    closePicker: () => $emit('close-picker'),
                }"
            />
        </template>
        <template v-else>
            <div v-if="defaults.actionRow.showPreview" class="dp__selection_preview" :title="formatValue">
                <slot name="action-preview" v-if="$slots['action-preview']" :value="internalModelValue" />
                <template v-if="!$slots['action-preview']">
                    {{ formatValue }}
                </template>
            </div>
            <div class="dp__action_buttons">
                <slot name="action-buttons" v-if="$slots['action-buttons']" :value="internalModelValue" />
                <template v-if="!$slots['action-buttons']">
                    <button
                        type="button"
                        ref="cancelButtonRef"
                        v-if="!inline && defaults.actionRow.showCancel"
                        class="dp__action_button dp__action_cancel"
                        @click="$emit('close-picker')"
                        @keydown.enter="$emit('close-picker')"
                        @keydown.space="$emit('close-picker')"
                    >
                        {{ cancelText }}
                    </button>
                    <button
                        type="button"
                        ref="cancelButtonRef"
                        v-if="showNowButton || defaults.actionRow.showNow"
                        class="dp__action_button dp__action_cancel"
                        @click="$emit('select-now')"
                        @keydown.enter="$emit('select-now')"
                        @keydown.space="$emit('select-now')"
                    >
                        {{ nowButtonLabel }}
                    </button>
                    <button
                        type="button"
                        v-if="defaults.actionRow.showSelect"
                        class="dp__action_button dp__action_select"
                        @keydown.enter="selectDate"
                        @keydown.space="selectDate"
                        @click="selectDate"
                        :disabled="disabled"
                        data-test="select-button"
                        ref="selectButtonRef"
                    >
                        {{ selectText }}
                    </button>
                </template>
            </div>
        </template>
    </div>
</template>

<script lang="ts" setup>
    import { computed, onMounted, ref } from 'vue';

    import { convertType, unrefElement } from '@/utils/util';
    import { useArrowNavigation, useUtils } from '@/composables';
    import { AllProps } from '@/utils/props';
    import { getDate, isDateAfter, isDateBefore, isDateEqual, resetDate } from '@/utils/date-utils';

    import type { PropType } from 'vue';
    import type { InternalModuleValue } from '@/interfaces';

    const emit = defineEmits(['close-picker', 'select-date', 'select-now', 'invalid-select']);

    const props = defineProps({
        menuMount: { type: Boolean as PropType<boolean>, default: false },
        internalModelValue: { type: [Date, Array] as PropType<InternalModuleValue>, default: null },
        calendarWidth: { type: Number as PropType<number>, default: 0 },

        ...AllProps,
    });

    const { formatDate, isValidTime, defaults } = useUtils(props);
    const { buildMatrix } = useArrowNavigation();

    const cancelButtonRef = ref(null);
    const selectButtonRef = ref(null);

    onMounted(() => {
        if (props.arrowNavigation) {
            buildMatrix([unrefElement(cancelButtonRef), unrefElement(selectButtonRef)] as HTMLElement[], 'actionRow');
        }
    });

    const validDateRange = computed(() => {
        return props.range && !props.partialRange && props.internalModelValue
            ? (props.internalModelValue as Date[]).length === 2
            : true;
    });

    const disabled = computed(() => !isTimeValid.value || !isMonthValid.value || !validDateRange.value);

    const isTimeValid = computed((): boolean => {
        if (!props.enableTimePicker || props.ignoreTimeValidation) return true;
        return isValidTime(props.internalModelValue);
    });

    const isMonthValid = computed((): boolean => {
        if (!props.monthPicker) return true;
        if (props.range && Array.isArray(props.internalModelValue)) {
            const invalid = props.internalModelValue.filter((value) => !isMonthWithinRange(value));
            return !invalid.length;
        }
        return isMonthWithinRange(props.internalModelValue as Date);
    });

    const handleCustomPreviewFormat = () => {
        const formatFn = defaults.value.previewFormat as (val: Date | Date[]) => string | string[];

        if (props.timePicker) return formatFn(convertType(props.internalModelValue));

        if (props.monthPicker) return formatFn(convertType(props.internalModelValue as Date));

        return formatFn(convertType(props.internalModelValue));
    };

    const formatRangeDate = () => {
        const dates = props.internalModelValue as Date[];
        if (defaults.value.multiCalendars > 0) {
            return `${formatDate(dates[0])} - ${formatDate(dates[1])}`;
        }
        return [formatDate(dates[0]), formatDate(dates[1])];
    };

    const previewValue = computed((): string | string[] => {
        if (!props.internalModelValue || !props.menuMount) return '';
        if (typeof defaults.value.previewFormat === 'string') {
            if (Array.isArray(props.internalModelValue)) {
                if (props.internalModelValue.length === 2 && props.internalModelValue[1]) {
                    return formatRangeDate();
                }
                if (props.multiDates) {
                    return props.internalModelValue.map((date) => `${formatDate(date)}`);
                }
                if (props.modelAuto) {
                    return `${formatDate(props.internalModelValue[0])}`;
                }
                return `${formatDate(props.internalModelValue[0])} -`;
            }
            return formatDate(props.internalModelValue);
        }
        return handleCustomPreviewFormat();
    });

    const formatValue = computed(() =>
        !Array.isArray(previewValue.value)
            ? previewValue.value
            : previewValue.value.join(props.multiDates ? '; ' : ' - '),
    );

    const isMonthWithinRange = (date: Date | string): boolean => {
        if (!props.monthPicker) return true;
        let valid = true;
        const dateToCompare = getDate(resetDate(date));
        if (props.minDate && props.maxDate) {
            const minDate = getDate(resetDate(props.minDate));
            const maxDate = getDate(resetDate(props.maxDate));
            return (
                (isDateAfter(dateToCompare, minDate) && isDateBefore(dateToCompare, maxDate)) ||
                isDateEqual(dateToCompare, minDate) ||
                isDateEqual(dateToCompare, maxDate)
            );
        }
        if (props.minDate) {
            const minDate = getDate(resetDate(props.minDate));

            valid = isDateAfter(dateToCompare, minDate) || isDateEqual(dateToCompare, minDate);
        }
        if (props.maxDate) {
            const maxDate = getDate(resetDate(props.maxDate));
            valid = isDateBefore(dateToCompare, maxDate) || isDateEqual(dateToCompare, maxDate);
        }

        return valid;
    };

    const selectDate = (): void => {
        if (isTimeValid.value && isMonthValid.value && validDateRange.value) {
            emit('select-date');
        } else {
            emit('invalid-select');
        }
    };
</script>
