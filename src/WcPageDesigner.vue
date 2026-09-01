<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, ref, watch } from 'vue'
import { HolderOutlined, PlusOutlined, DeleteOutlined } from '@ant-design/icons-vue'
import Sortable from 'sortablejs'
import type {
    DateSearchPrecision,
    FieldInputType,
    FieldInputTypeOption,
    FieldSearchMode,
    PageSchema,
} from 'wc-core'
import type { PageField } from 'wc-core'
import {
    DEFAULT_INPUT_TYPE_OPTIONS,
    applyPrimaryKeyField,
    createFieldKey,
    renumberFieldSort,
    resolveFieldInputType,
} from 'wc-core'
import {
    defaultDatePrecision,
    getDatePrecisionOptions,
    getSearchModeOptions,
    isDateSearchMode,
    normalizeSearchMode,
} from 'wc-core'

const props = withDefaults(defineProps<{
    fields: PageField[]
    selectedTable?: string | null
    tableComment?: string
    loadingFields?: boolean
    inputTypeOptions?: FieldInputTypeOption[]
    extensionMode?: boolean
    pageKey?: string | null
}>(), {
    selectedTable: null,
    tableComment: '',
    loadingFields: false,
    inputTypeOptions: undefined,
    extensionMode: false,
    pageKey: null,
})

const designerReady = computed(() => (
    props.extensionMode ? true : !!props.selectedTable
))

const emit = defineEmits<{
    'update:fields': [fields: PageField[]]
    change: [schema: PageSchema]
}>()

const tableBodyElement = ref<HTMLElement | null>(null)
let sortableInstance: Sortable | null = null

const typeOptions = computed(() => props.inputTypeOptions ?? DEFAULT_INPUT_TYPE_OPTIONS)

const isExtensionCoreField = (fieldItem: PageField) => (
    props.extensionMode && fieldItem.lockReason === 'core'
)

const resolveDesignerInputType = (fieldItem: PageField) => resolveFieldInputType(fieldItem)

const localFields = computed({
    get: () => props.fields,
    set: (nextFields) => {
        emit('update:fields', nextFields)
        emitChange(nextFields)
    },
})

const fieldColumns = computed(() => {
    const columns = [
        { key: 'drag', title: '', width: 48 },
        { key: 'field', title: '???', width: 160 },
        { key: 'title', title: '???', width: 160 },
        { key: 'comment', title: '??', width: 180 },
        { key: 'dbType', title: '?????', width: 140 },
        { key: 'inputType', title: '????', width: 160 },
        { key: 'showInTable', title: '????', width: 88, align: 'center' as const },
        { key: 'searchHeader', title: '????', width: 220 },
        { key: 'primaryKey', title: '??', width: 72, align: 'center' as const },
        { key: 'required', title: '??', width: 80, align: 'center' as const },
        { key: 'locked', title: '??', width: 90, align: 'center' as const },
        { key: 'action', title: '??', width: 90, align: 'center' as const, fixed: 'right' as const },
    ]
    if (props.extensionMode) {
        return columns.filter((columnItem) => (
            columnItem.key !== 'dbType' && columnItem.key !== 'primaryKey'
        ))
    }
    return columns
})

const emitChange = (fields: PageField[]) => {
    if (props.extensionMode) {
        if (!props.pageKey) {
            return
        }
        emit('change', {
            tableName: props.pageKey,
            fields,
        })
        return
    }
    if (!props.selectedTable) {
        return
    }
    emit('change', {
        tableName: props.selectedTable,
        comment: props.tableComment,
        fields,
    })
}

const patchField = (fieldKey: string | undefined, patch: Partial<PageField>) => {
    if (!fieldKey) {
        return
    }
    const nextFields = localFields.value.map((fieldItem) => {
        if (fieldItem.key !== fieldKey) {
            return fieldItem
        }
        if (fieldItem.locked) {
            const lockedPatch: Partial<PageField> = {}
            if (patch.title !== undefined) {
                lockedPatch.title = patch.title
            }
            if (patch.required !== undefined && fieldItem.lockReason !== 'geometry' && fieldItem.lockReason !== 'core') {
                lockedPatch.required = patch.required
            }
            if (patch.comment !== undefined && fieldItem.lockReason !== 'geometry') {
                lockedPatch.comment = patch.comment
            }
            if (patch.showInTable !== undefined) {
                lockedPatch.showInTable = patch.showInTable
            }
            if (patch.searchMode !== undefined) {
                lockedPatch.searchMode = patch.searchMode
            }
            if (patch.dateSearchPrecision !== undefined) {
                lockedPatch.dateSearchPrecision = patch.dateSearchPrecision
            }
            if (fieldItem.lockReason === 'core') {
                const corePatch: Partial<PageField> = {}
                if (lockedPatch.showInTable !== undefined) {
                    corePatch.showInTable = lockedPatch.showInTable
                }
                if (lockedPatch.searchMode !== undefined) {
                    corePatch.searchMode = lockedPatch.searchMode
                }
                if (lockedPatch.dateSearchPrecision !== undefined) {
                    corePatch.dateSearchPrecision = lockedPatch.dateSearchPrecision
                }
                return Object.keys(corePatch).length
                    ? { ...fieldItem, ...corePatch }
                    : fieldItem
            }
            return { ...fieldItem, ...lockedPatch }
        }
        return { ...fieldItem, ...patch }
    })
    localFields.value = nextFields
}

const setPrimaryKeyField = (fieldKey: string | undefined, checked: boolean) => {
    if (!fieldKey) {
        return
    }
    const targetField = localFields.value.find((fieldItem) => fieldItem.key === fieldKey)
    if (!targetField) {
        return
    }
    if (
        targetField.lockReason === 'geometry'
        || targetField.lockReason === 'audit'
        || targetField.inputType === 'geom'
    ) {
        return
    }
    const nextFieldName = checked ? targetField.field : null
    localFields.value = applyPrimaryKeyField(localFields.value, nextFieldName)
}

const patchInputType = (fieldKey: string | undefined, inputType: FieldInputType | undefined) => {
    if (!fieldKey || !inputType) {
        return
    }
    const targetField = localFields.value.find((fieldItem) => fieldItem.key === fieldKey)
    const nextSearchMode = normalizeSearchMode(resolveFieldInputType(targetField || { field: '', title: '' }), targetField?.searchMode)
    const fieldPatch: Partial<PageField> = {
        inputType,
        searchMode: nextSearchMode,
    }
    if (inputType === 'date' || inputType === 'datetime') {
        fieldPatch.dateSearchPrecision = defaultDatePrecision(inputType)
    }
    patchField(fieldKey, fieldPatch)
}

const patchSearchMode = (fieldKey: string | undefined, searchMode: FieldSearchMode) => {
    if (!fieldKey) {
        return
    }
    const targetField = localFields.value.find((fieldItem) => fieldItem.key === fieldKey)
    const fieldPatch: Partial<PageField> = { searchMode }
    if (isDateSearchMode(searchMode) && targetField) {
        fieldPatch.dateSearchPrecision = targetField.dateSearchPrecision
            || defaultDatePrecision(resolveFieldInputType(targetField))
    }
    patchField(fieldKey, fieldPatch)
}

const handleAddField = () => {
    const nextSort = localFields.value.length + 1
    const nextFields = [
        ...localFields.value,
        {
            key: createFieldKey(),
            field: `field_${nextSort}`,
            title: `??${nextSort}`,
            comment: '',
            inputType: 'input' as FieldInputType,
            required: false,
            showInTable: true,
            searchMode: 'none' as FieldSearchMode,
            sort: nextSort,
            locked: false,
        },
    ]
    localFields.value = renumberFieldSort(nextFields)
}

const handleRemoveField = (fieldKey: string | undefined) => {
    if (!fieldKey) {
        return
    }
    const targetField = localFields.value.find((fieldItem) => fieldItem.key === fieldKey)
    if (!targetField || targetField.locked) {
        return
    }
    localFields.value = renumberFieldSort(
        localFields.value.filter((fieldItem) => fieldItem.key !== fieldKey),
    )
}

const destroySortable = () => {
    if (sortableInstance) {
        sortableInstance.destroy()
        sortableInstance = null
    }
}

const bindSortable = async () => {
    destroySortable()
    await nextTick()
    const tableElement = tableBodyElement.value
    if (!tableElement) {
        return
    }
    const tableBody = (
        tableElement.querySelector('.ant-table-body .ant-table-tbody')
        || tableElement.querySelector('.ant-table-tbody')
        || tableElement.querySelector('tbody')
    )
    if (!(tableBody instanceof HTMLElement)) {
        return
    }
    sortableInstance = Sortable.create(tableBody, {
        handle: '.wc-designer__drag-handle',
        animation: 150,
        draggable: '.ant-table-row:not(.ant-table-measure-row)',
        filter: '.ant-table-measure-row',
        preventOnFilter: false,
        dataIdAttr: 'data-row-key',
        onEnd: (event) => {
            const oldIndex = event.oldDraggableIndex
            const newIndex = event.newDraggableIndex
            if (
                oldIndex == null
                || newIndex == null
                || oldIndex === newIndex
                || oldIndex < 0
                || newIndex < 0
            ) {
                return
            }
            const nextFields = [...localFields.value]
            if (oldIndex >= nextFields.length || newIndex >= nextFields.length) {
                return
            }
            const [movedField] = nextFields.splice(oldIndex, 1)
            if (!movedField) {
                return
            }
            nextFields.splice(newIndex, 0, movedField)
            localFields.value = renumberFieldSort(nextFields)
        },
    })
}

watch(
    () => [props.fields, props.selectedTable, props.loadingFields],
    () => {
        if (!props.loadingFields) {
            bindSortable()
        }
    },
    { flush: 'post', immediate: true },
)

onBeforeUnmount(() => {
    destroySortable()
})

defineExpose({
    getSchema: (): PageSchema | null => {
        if (!props.selectedTable) {
            return null
        }
        return {
            tableName: props.selectedTable,
            comment: props.tableComment,
            fields: localFields.value,
        }
    },
})
</script>

<template>
    <div class="wc-designer">
        <div class="wc-designer__fields-header">
            <div>
                <div class="wc-designer__panel-title">
                    ????
                    <span v-if="selectedTable" class="wc-designer__selected-table">
                        ?{{ selectedTable }}?
                    </span>
                </div>
                <div v-if="tableComment" class="wc-designer__hint">
                    {{ tableComment }}
                </div>
            </div>
            <a-button
                type="primary"
                :disabled="!designerReady"
                @click="handleAddField"
            >
                <template #icon>
                    <PlusOutlined />
                </template>
                ????
            </a-button>
        </div>

        <a-spin class="wc-designer__body" :spinning="loadingFields">
            <div v-if="designerReady" ref="tableBodyElement">
                <a-table
                    :columns="fieldColumns"
                    :data-source="localFields"
                    :pagination="false"
                    row-key="key"
                    size="middle"
                    :scroll="{ x: 1560 }"
                >
                    <template #bodyCell="{ column, record }">
                        <template v-if="column.key === 'drag'">
                            <span
                                class="wc-designer__drag-handle"
                                title="????"
                            >
                                <HolderOutlined />
                            </span>
                        </template>
                        <template v-else-if="column.key === 'field'">
                            <a-input
                                :value="record.field"
                                :disabled="record.locked"
                                placeholder="???"
                                @update:value="(value) => patchField(record.key, { field: String(value || '') })"
                            />
                        </template>
                        <template v-else-if="column.key === 'title'">
                            <a-input
                                :value="record.title"
                                :disabled="isExtensionCoreField(record)"
                                placeholder="???"
                                @update:value="(value) => patchField(record.key, { title: String(value || '') })"
                            />
                        </template>
                        <template v-else-if="column.key === 'comment'">
                            <a-input
                                :value="record.comment || ''"
                                :disabled="isExtensionCoreField(record) || (record.locked && record.lockReason === 'geometry')"
                                placeholder="??"
                                @update:value="(value) => patchField(record.key, { comment: String(value || '') })"
                            />
                        </template>
                        <template v-else-if="column.key === 'dbType'">
                            <span class="wc-designer__db-type">{{ record.dbType || '-' }}</span>
                        </template>
                        <template v-else-if="column.key === 'inputType'">
                            <a-select
                                :value="resolveDesignerInputType(record)"
                                :options="typeOptions"
                                :disabled="record.locked"
                                style="width: 100%"
                                @update:value="(value) => patchInputType(record.key, value as FieldInputType)"
                            />
                        </template>
                        <template v-else-if="column.key === 'showInTable'">
                            <a-checkbox
                                :checked="record.showInTable !== false"
                                :disabled="record.lockReason === 'geometry' || record.inputType === 'geom'"
                                @update:checked="(checked) => patchField(record.key, { showInTable: !!checked })"
                            />
                        </template>
                        <template v-else-if="column.key === 'searchHeader'">
                            <div class="wc-designer__search-cell">
                                <a-select
                                    :value="record.searchMode || 'none'"
                                    :options="getSearchModeOptions(resolveDesignerInputType(record))"
                                    :disabled="record.lockReason === 'geometry' || record.inputType === 'geom'"
                                    style="width: 100%"
                                    @update:value="(value) => patchSearchMode(record.key, value as FieldSearchMode)"
                                />
                                <a-select
                                    v-if="isDateSearchMode(record.searchMode)"
                                    :value="record.dateSearchPrecision || defaultDatePrecision(resolveDesignerInputType(record))"
                                    :options="getDatePrecisionOptions(resolveDesignerInputType(record))"
                                    style="width: 100%"
                                    @update:value="(value) => patchField(record.key, { dateSearchPrecision: value as DateSearchPrecision })"
                                />
                            </div>
                        </template>
                        <template v-else-if="column.key === 'primaryKey'">
                            <a-radio
                                :checked="record.lockReason === 'pk'"
                                :disabled="
                                    record.lockReason === 'geometry'
                                    || record.lockReason === 'audit'
                                    || record.inputType === 'geom'
                                "
                                @click.prevent="setPrimaryKeyField(record.key, record.lockReason !== 'pk')"
                            />
                        </template>
                        <template v-else-if="column.key === 'required'">
                            <a-checkbox
                                :checked="!!record.required"
                                :disabled="isExtensionCoreField(record) || (record.locked && record.lockReason === 'geometry')"
                                @update:checked="(checked) => patchField(record.key, { required: !!checked })"
                            />
                        </template>
                        <template v-else-if="column.key === 'locked'">
                            <a-tag v-if="record.locked" color="orange">
                                {{
                                    record.lockReason === 'geometry'
                                        ? '??'
                                        : record.lockReason === 'pk'
                                            ? '??'
                                            : record.lockReason === 'audit'
                                                ? '??'
                                                : record.lockReason === 'core'
                                                    ? '??'
                                                    : '??'
                                }}
                            </a-tag>
                            <span v-else>-</span>
                        </template>
                        <template v-else-if="column.key === 'action'">
                            <a-button
                                type="link"
                                danger
                                :disabled="record.locked"
                                @click="handleRemoveField(record.key)"
                            >
                                <template #icon>
                                    <DeleteOutlined />
                                </template>
                                ??
                            </a-button>
                        </template>
                    </template>
                </a-table>
            </div>
            <a-empty v-else :description="extensionMode ? '????' : '????????'" />
        </a-spin>
    </div>
</template>

<style scoped lang="less">
.wc-designer {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;
    min-height: 0;
    overflow: hidden;
    box-sizing: border-box;
    border: 1px solid var(--theme-color-border, #f0f0f0);
    border-radius: 8px;
    padding: 12px;
    background: var(--theme-color-bg-container, #fff);

    &__panel-title {
        font-size: 14px;
        font-weight: 600;
        margin-bottom: 8px;
        color: var(--theme-color-text, rgba(0, 0, 0, 0.88));
    }

    &__selected-table {
        font-weight: 400;
        color: var(--theme-color-text-secondary, rgba(0, 0, 0, 0.65));
    }

    &__hint {
        font-size: 12px;
        color: var(--theme-color-text-secondary, rgba(0, 0, 0, 0.65));
        margin-bottom: 8px;
    }

    &__fields-header {
        display: flex;
        align-items: flex-start;
        justify-content: space-between;
        gap: 12px;
        margin-bottom: 12px;
        flex-shrink: 0;
    }

    &__body {
        flex: 1;
        min-height: 0;
        overflow: auto;
    }

    &__drag-handle {
        cursor: grab;
        color: var(--theme-color-text-secondary, rgba(0, 0, 0, 0.45));

        &:active {
            cursor: grabbing;
        }
    }

    &__db-type {
        font-size: 12px;
        color: var(--theme-color-text-secondary, rgba(0, 0, 0, 0.65));
        word-break: break-all;
    }

    &__search-cell {
        display: flex;
        flex-direction: column;
        gap: 6px;
        min-width: 0;
    }
}
</style>
