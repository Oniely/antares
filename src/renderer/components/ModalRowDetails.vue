<template>
   <div class="dummy-wrapper">
      <Teleport to="#window-content">
         <div class="modal active modal-row-details">
            <a class="modal-overlay" @click="emit('hide')" />
            <div class="modal-container">
               <div class="modal-header pl-2">
                  <div class="modal-title h6 d-flex align-items-center">
                     <BaseIcon
                        icon-name="mdiTableEye"
                        class="mr-2"
                        :size="22"
                     />
                     {{ t('database.viewDetails') }}
                     <span v-if="!isEditable" class="readonly-badge ml-2">
                        {{ t('connection.readOnlyMode') }}
                     </span>
                  </div>
                  <a class="btn btn-clear float-right" @click="emit('hide')" />
               </div>

               <div class="modal-body pb-0">
                  <div class="content">
                     <!-- Search bar -->
                     <div class="search-wrapper mb-2">
                        <div class="has-icon-left input-group">
                           <BaseIcon
                              icon-name="mdiMagnify"
                              class="form-icon"
                              :size="16"
                           />
                           <input
                              ref="searchInput"
                              v-model="searchQuery"
                              type="text"
                              class="form-input"
                              :placeholder="t('general.search') + '...'"
                              @keydown.esc.stop.prevent="emit('hide')"
                              @keydown.stop
                           >
                        </div>
                        <div class="search-meta mt-1">
                           <span class="filter-counter">
                              {{ filteredEntries.length }} / {{ allEntries.length }} {{ t('database.column', 2).toLowerCase() }}
                           </span>
                        </div>
                     </div>

                     <!-- Refresh button -->
                     <button
                        class="btn btn-refresh mb-3"
                        :disabled="isRefreshing"
                        @click="onRefresh"
                     >
                        <BaseIcon
                           :icon-name="isRefreshing ? 'mdiLoading' : 'mdiRefresh'"
                           :class="{ 'spin': isRefreshing }"
                           :size="15"
                           class="mr-1"
                        />
                        {{ t('general.refresh') }}
                     </button>

                     <!-- Column/Value rows -->
                     <div class="details-list">
                        <div
                           v-for="(entry, index) in filteredEntries"
                           :key="entry.key"
                           class="detail-row"
                           :class="{
                              'detail-row--odd': index % 2 === 0,
                              'detail-row--editing': editingKey === entry.key
                           }"
                        >
                           <!-- Column name + type -->
                           <div class="detail-col detail-col--name">
                              <span class="col-name">
                                 <template v-if="searchQuery.trim() && highlightedName(entry.key).match">
                                    {{ highlightedName(entry.key).before }}<mark class="col-match">{{ highlightedName(entry.key).match }}</mark>{{ highlightedName(entry.key).after }}
                                 </template>
                                 <template v-else>
                                    {{ displayName(entry.key) }}
                                 </template>
                              </span>
                              <span v-if="entry.field" class="col-type">{{ entry.field.type }}</span>
                           </div>

                           <!-- Value / Editor -->
                           <div
                              class="detail-col detail-col--value"
                              :class="{
                                 'is-editable': isEditable && canEdit(entry),
                                 'is-readonly': !isEditable || !canEdit(entry)
                              }"
                              @dblclick="editON(entry)"
                           >
                              <!-- Editing: inline textarea for long text -->
                              <template v-if="editingKey === entry.key && isTextareaMode">
                                 <textarea
                                    ref="editField"
                                    v-model="editingContentStr"
                                    class="detail-editor detail-textarea"
                                    @blur="editOFF"
                                    @keydown.esc.stop="cancelEdit"
                                    @keydown.stop
                                 />
                              </template>

                              <!-- Editing: foreign key select -->
                              <ForeignKeySelect
                                 v-else-if="editingKey === entry.key && isForeignKey(entry.key)"
                                 v-model="editingContentStr"
                                 class="detail-editor"
                                 :key-usage="getKeyUsage(entry.key)"
                                 size="small"
                                 @blur="editOFF"
                              />

                              <!-- Editing: boolean select -->
                              <BaseSelect
                                 v-else-if="editingKey === entry.key && inputProps.type === 'boolean'"
                                 v-model="editingContent"
                                 :options="['true', 'false']"
                                 class="detail-editor form-select small-select"
                                 @blur="editOFF"
                                 @keydown.stop
                              />

                              <!-- Editing: enum select -->
                              <BaseSelect
                                 v-else-if="editingKey === entry.key && enumArray"
                                 v-model="editingContent"
                                 :options="enumArray"
                                 class="detail-editor form-select small-select"
                                 dropdown-class="small-select"
                                 @blur="editOFF"
                                 @keydown.stop
                              />

                              <!-- Editing: masked input (time/date) -->
                              <input
                                 v-else-if="editingKey === entry.key && inputProps.mask"
                                 ref="editField"
                                 v-model="editingContent"
                                 v-mask="inputProps.mask"
                                 :type="inputProps.type"
                                 class="detail-editor form-input input-sm"
                                 @blur="editOFF"
                                 @keydown.enter.stop="editOFF"
                                 @keydown.esc.stop="cancelEdit"
                                 @keydown.stop
                              >

                              <!-- Editing: standard input -->
                              <input
                                 v-else-if="editingKey === entry.key"
                                 ref="editField"
                                 v-model="editingContent"
                                 :type="inputProps.type"
                                 class="detail-editor form-input input-sm"
                                 @blur="editOFF"
                                 @keydown.enter.stop="editOFF"
                                 @keydown.esc.stop="cancelEdit"
                                 @keydown.stop
                              >

                              <!-- Display mode -->
                              <template v-else>
                                 <span
                                    v-if="localRow[entry.key] === null"
                                    class="value-null"
                                 >NULL</span>
                                 <span
                                    v-else-if="localRow[entry.key] === ''"
                                    class="value-empty"
                                 >(empty string)</span>
                                 <span v-else class="value-text">
                                    {{ formatValue(localRow[entry.key], entry.field) }}
                                 </span>
                                 <BaseIcon
                                    v-if="isEditable && canEdit(entry)"
                                    icon-name="mdiPencil"
                                    class="edit-hint"
                                    :size="12"
                                 />
                              </template>
                           </div>
                        </div>

                        <div v-if="filteredEntries.length === 0" class="empty-state">
                           {{ t('general.filter') }}: no columns match "{{ searchQuery }}"
                        </div>
                     </div>
                  </div>
               </div>

               <div class="modal-footer">
                  <button class="btn btn-link" @click="emit('hide')">
                     {{ t('general.close') }}
                  </button>
               </div>
            </div>
         </div>
      </Teleport>
   </div>
</template>

<script setup lang="ts">
import {
   ARRAY,
   BINARY,
   BLOB,
   BOOLEAN,
   DATE,
   DATETIME,
   FLOAT,
   HAS_TIMEZONE,
   IS_BIGINT,
   LONG_TEXT,
   NUMBER,
   SPATIAL,
   TEXT,
   TEXT_SEARCH,
   TIME
} from 'common/fieldTypes';
import { QueryForeign, TableField } from 'common/interfaces/antares';
import { formatBytes } from 'common/libs/formatBytes';
import { mimeFromHex } from 'common/libs/mimeFromHex';
import * as moment from 'moment';
import { computed, nextTick, onBeforeUnmount, onMounted, Prop, reactive, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';

import BaseIcon from '@/components/BaseIcon.vue';
import BaseSelect from '@/components/BaseSelect.vue';
import ForeignKeySelect from '@/components/ForeignKeySelect.vue';

const { t } = useI18n();

const props = defineProps({
   row: Object as Prop<Record<string, unknown>>,
   fields: Object as Prop<Record<string, TableField>>,
   keyUsage: Array as Prop<QueryForeign[]>,
   elementType: { type: String, default: 'table' }
});

const emit = defineEmits(['hide', 'update-field', 'refresh']);

// ── Search ─────────────────────────────────────────────────────────────────
const searchQuery = ref('');
const searchInput = ref<HTMLInputElement>(null);

// ── Refresh state ──────────────────────────────────────────────────────────
const isRefreshing = ref(false);

const onRefresh = () => {
   isRefreshing.value = true;
   emit('refresh');
};

// ── Local row copy (stays in sync with edits) ──────────────────────────────
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const localRow = reactive<Record<string, any>>({});

watch(() => props.row, (row) => {
   if (!row) return;
   Object.assign(localRow, row);
   isRefreshing.value = false;
}, { immediate: true });

// ── Editing state ──────────────────────────────────────────────────────────
const editField = ref<HTMLInputElement | HTMLTextAreaElement>(null);
const editingKey = ref<string | null>(null); // raw object key (may have table prefix)
const editingContent = ref<unknown>(null);
const editingType = ref<string | null>(null);
const editingLength = ref<number | null>(null);
const originalContent = ref<unknown>(null);
const isTextareaMode = ref(false);

// ── Computed string accessor for v-model on textarea / ForeignKeySelect ──────
const editingContentStr = computed({
   get: () => (editingContent.value === null || editingContent.value === undefined)
      ? ''
      : String(editingContent.value),
   set: (val: string) => {
      editingContent.value = val;
   }
});
interface DetailEntry {
   key: string;
   value: unknown;
   field: TableField | null;
}

const allEntries = computed<DetailEntry[]>(() => {
   if (!props.row) return [];
   return Object.keys(props.row)
      .filter(key => key !== '_antares_id')
      .map(key => ({
         key,
         value: props.row[key],
         field: props.fields?.[key] ?? null
      }));
});

const filteredEntries = computed<DetailEntry[]>(() => {
   if (!searchQuery.value.trim()) return allEntries.value;
   const q = searchQuery.value.toLowerCase();
   return allEntries.value.filter(entry =>
      displayName(entry.key).toLowerCase().includes(q)
   );
});

const displayName = (key: string): string => {
   if (key.includes('.')) return key.split('.').pop() ?? key;
   return key;
};

const highlightedName = (key: string): { before: string; match: string; after: string } => {
   const name = displayName(key);
   const q = searchQuery.value.trim();
   if (!q) return { before: name, match: '', after: '' };

   const idx = name.toLowerCase().indexOf(q.toLowerCase());
   if (idx === -1) return { before: name, match: '', after: '' };

   return {
      before: name.slice(0, idx),
      match: name.slice(idx, idx + q.length),
      after: name.slice(idx + q.length)
   };
};

// ── Editability ────────────────────────────────────────────────────────────
const isEditable = computed(() => {
   if (props.elementType === 'view') return false;
   if (!props.fields) return false;

   const nElements = Object.keys(props.fields).reduce((acc, curr) => {
      acc.add(props.fields[curr].table);
      acc.add(props.fields[curr].schema);
      return acc;
   }, new Set([]));

   if (nElements.size > 2) return false;

   const firstKey = Object.keys(props.fields)[0];
   return !!(props.fields[firstKey]?.schema && props.fields[firstKey]?.table);
});

const canEdit = (entry: DetailEntry): boolean => {
   if (!entry.field) return false;
   const type = entry.field.type.toUpperCase();
   return ![...BLOB, ...SPATIAL, ...BINARY].includes(type);
};

// ── Foreign keys ───────────────────────────────────────────────────────────
const foreignKeys = computed(() => (props.keyUsage ?? []).map(k => k.field));

const isForeignKey = (key: string): boolean => {
   const bare = key.includes('.') ? key.split('.').pop() : key;
   return foreignKeys.value.includes(bare);
};

const getKeyUsage = (key: string): QueryForeign | undefined => {
   const bare = key.includes('.') ? key.split('.').pop() : key;
   return (props.keyUsage ?? []).find(k => k.field === bare);
};

// ── Input props (mask / type) ──────────────────────────────────────────────
const inputProps = computed(() => {
   const type = editingType.value;
   if (!type) return { type: 'text', mask: false };

   if ([...TEXT, ...LONG_TEXT].includes(type))
      return { type: 'text', mask: false };

   if ([...NUMBER, ...FLOAT].includes(type)) {
      if (IS_BIGINT.includes(type)) return { type: 'text', mask: false };
      return { type: 'number', mask: false };
   }

   if (TIME.includes(type)) {
      let mask = '##:##:##';
      const precision = editingLength.value;
      for (let i = 0; i < Number(precision); i++)
         mask += i === 0 ? '.#' : '#';
      if (HAS_TIMEZONE.includes(type)) mask += 'X##';
      return { type: 'text', mask };
   }

   if (DATE.includes(type))
      return { type: 'text', mask: '####-##-##' };

   if (DATETIME.includes(type)) {
      let mask = '####-##-## ##:##:##';
      const precision = editingLength.value;
      for (let i = 0; i < Number(precision); i++)
         mask += i === 0 ? '.#' : '#';
      if (HAS_TIMEZONE.includes(type)) mask += 'X##';
      return { type: 'text', mask };
   }

   if (BOOLEAN.includes(type)) return { type: 'boolean', mask: false };

   return { type: 'text', mask: false };
});

const enumArray = computed(() => {
   if (!editingKey.value) return false;
   const field = props.fields?.[editingKey.value];
   if (field?.enumValues && field.type !== 'SET')
      return field.enumValues.replaceAll('\'', '').split(',');
   return false;
});

// ── Edit lifecycle ─────────────────────────────────────────────────────────
const editON = (entry: DetailEntry) => {
   if (!isEditable.value || !canEdit(entry)) return;
   if (editingKey.value) editOFF(); // commit any previous edit first

   const field = entry.field;
   const type = field?.type?.toUpperCase() ?? 'TEXT';
   const content = localRow[entry.key];

   if (BINARY.includes(type)) return;

   editingKey.value = entry.key;
   editingType.value = type;
   editingLength.value = (field?.length as number) ?? null;
   originalContent.value = content;
   editingContent.value = formatForEditing(content, type, field);

   isTextareaMode.value = [...LONG_TEXT, ...ARRAY, ...TEXT_SEARCH].includes(type);

   nextTick(() => {
      if (editField.value) {
         if (Array.isArray(editField.value))
            (editField.value[0] as HTMLInputElement)?.focus();
         else
            (editField.value as HTMLInputElement)?.focus();
      }
   });
};

const editOFF = () => {
   if (!editingKey.value) return;

   const key = editingKey.value;
   const type = editingType.value;

   // Clean trailing dot for datetime/time
   if ([...DATETIME, ...TIME].includes(type)) {
      if (typeof editingContent.value === 'string' && editingContent.value.endsWith('.'))
         editingContent.value = (editingContent.value as string).slice(0, -1);
   }

   const formatted = formatForEditing(originalContent.value, type, props.fields?.[key] ?? null);

   // No change → just exit edit mode
   if (editingContent.value === formatted || editingContent.value === originalContent.value) {
      resetEdit();
      return;
   }

   const field = props.fields?.[key];
   const fieldName = field?.name ?? displayName(key);

   // Update local row immediately so modal reflects the change
   localRow[key] = editingContent.value;

   emit('update-field', {
      orgField: key,
      field: fieldName,
      type,
      content: editingContent.value
   });

   resetEdit();
};

const cancelEdit = () => {
   resetEdit();
};

const resetEdit = () => {
   editingKey.value = null;
   editingContent.value = null;
   editingType.value = null;
   editingLength.value = null;
   originalContent.value = null;
   isTextareaMode.value = false;
};

// ── Formatting helpers ─────────────────────────────────────────────────────
const formatForEditing = (value: unknown, type: string, field: TableField | null): unknown => {
   if (value === null || value === undefined) return value;

   type = type.toUpperCase();

   if (DATE.includes(type)) {
      return moment(value as string | Date).isValid()
         ? moment(value as string | Date).format('YYYY-MM-DD')
         : value;
   }

   if (DATETIME.includes(type)) {
      if (typeof value === 'string') return value;
      let datePrecision = '';
      const precision = field?.datePrecision ?? 0;
      for (let i = 0; i < precision; i++)
         datePrecision += i === 0 ? '.S' : 'S';
      return moment(value as Date).isValid()
         ? moment(value as Date).format(`YYYY-MM-DD HH:mm:ss${datePrecision}`)
         : value;
   }

   if (typeof value === 'object') return JSON.stringify(value);

   return value;
};

const formatValue = (value: unknown, field: TableField | null): string => {
   if (value === null || value === undefined) return '';

   const type = field?.type?.toUpperCase() ?? '';

   if (DATE.includes(type)) {
      return moment(value as string | Date).isValid()
         ? moment(value as string | Date).format('YYYY-MM-DD')
         : String(value);
   }

   if (DATETIME.includes(type)) {
      if (typeof value === 'string') return value;
      let datePrecision = '';
      const precision = field?.datePrecision ?? 0;
      for (let i = 0; i < precision; i++)
         datePrecision += i === 0 ? '.S' : 'S';
      return moment(value as Date).isValid()
         ? moment(value as Date).format(`YYYY-MM-DD HH:mm:ss${datePrecision}`)
         : String(value);
   }

   if (BLOB.includes(type)) {
      if (typeof value === 'string') return value;
      const buff = Buffer.from(value as ArrayBuffer);
      if (!buff.length) return '(empty blob)';
      const hex = buff.toString('hex').substring(0, 8).toUpperCase();
      const { mime } = mimeFromHex(hex);
      return `${mime} (${formatBytes(buff.length)})`;
   }

   if (typeof value === 'object') return JSON.stringify(value);

   return String(value);
};

// ── Keyboard handling ──────────────────────────────────────────────────────
const onKey = (e: KeyboardEvent) => {
   if (e.key === 'Escape') {
      if (editingKey.value) {
         // First ESC: cancel the current edit
         e.stopPropagation();
         cancelEdit();
      }
      else {
         // Second ESC (no active edit): close modal
         e.stopPropagation();
         emit('hide');
      }
   }
};

window.addEventListener('keydown', onKey);

onMounted(() => {
   nextTick(() => searchInput.value?.focus());
});

onBeforeUnmount(() => {
   window.removeEventListener('keydown', onKey);
});
</script>

<style scoped lang="scss">
.modal-row-details {
   z-index: 500;

   .modal-container {
      width: 720px;
      max-width: 95vw;
      max-height: 85vh;
      display: flex;
      flex-direction: column;
   }

   .modal-body {
      flex: 1;
      overflow: hidden;
      display: flex;
      flex-direction: column;

      .content {
         display: flex;
         flex-direction: column;
         height: 100%;
         max-height: calc(85vh - 130px);
      }
   }
}

.readonly-badge {
   font-size: 0.6rem;
   font-weight: normal;
   opacity: 0.5;
   text-transform: uppercase;
   letter-spacing: 0.06em;
   border: 1px solid currentColor;
   padding: 0.1rem 0.4rem;
   border-radius: 10px;
}

.search-wrapper {
   flex-shrink: 0;

   .has-icon-left {
      position: relative;
      display: flex;
      align-items: center;

      .form-icon {
         position: absolute;
         left: 0.5rem;
         z-index: 1;
         opacity: 0.5;
      }

      .form-input {
         padding-left: 2rem;
      }
   }

   .search-meta {
      display: flex;
      justify-content: flex-end;

      .filter-counter {
         font-size: 0.68rem;
         opacity: 0.55;
         letter-spacing: 0.02em;
      }
   }
}

.details-list {
   overflow-y: auto;
   flex: 1;
   border-radius: 4px;
   border: 1px solid rgba(255, 255, 255, 0.08);
}

.detail-row {
   display: flex;
   align-items: stretch;
   border-bottom: 1px solid rgba(255, 255, 255, 0.05);
   min-height: 36px;
   transition: background 0.1s;

   &:last-child {
      border-bottom: none;
   }

   &--odd {
      background: rgba(255, 255, 255, 0.03);
   }

   &:hover {
      background: rgba(255, 255, 255, 0.06);
   }

   &--editing {
      background: rgba(66, 153, 225, 0.08) !important;
      border-color: rgba(66, 153, 225, 0.2);
   }
}

.detail-col {
   padding: 0.3rem 0.75rem;
   display: flex;
   align-items: center;
   gap: 0.5rem;

   &--name {
      width: 36%;
      flex-shrink: 0;
      border-right: 1px solid rgba(255, 255, 255, 0.06);
      display: flex;
      align-items: center;
      justify-content: space-between;
      flex-wrap: wrap;
      gap: 0.3rem;
   }

   &--value {
      flex: 1;
      word-break: break-all;
      white-space: pre-wrap;
      position: relative;
      min-height: 30px;
      display: flex;
      align-items: center;

      &.is-editable {
         cursor: text;

         .edit-hint {
            position: absolute;
            right: 0.5rem;
            opacity: 0;
            transition: opacity 0.15s;
            flex-shrink: 0;
         }

         &:hover .edit-hint {
            opacity: 0.35;
         }

         &:hover {
            background: rgba(66, 153, 225, 0.05);
         }
      }

      &.is-readonly {
         cursor: default;
         opacity: 0.8;
      }
   }
}

.detail-editor {
   width: 100%;
   min-width: 0;
   font-size: 0.8rem;
   margin: 0;
   border-radius: 3px;

   &.form-input,
   &.form-select {
      height: 26px;
      padding: 0.1rem 0.4rem;
   }

   &.detail-textarea {
      height: auto;
      min-height: 80px;
      resize: vertical;
      padding: 0.3rem 0.4rem;
      line-height: 1.4;
      border: 1px solid rgba(66, 153, 225, 0.5);
      background: rgba(0, 0, 0, 0.3);
      border-radius: 3px;
      width: 100%;
      font-family: inherit;
      color: inherit;
   }
}

.col-name {
   font-weight: 600;
   font-size: 0.8rem;
   opacity: 0.9;
}

.col-match {
   background: rgba(248, 204, 91, 0.25);
   color: inherit;
   padding: 0 0.1rem;
   border-radius: 2px;
}

.col-type {
   font-size: 0.65rem;
   opacity: 0.45;
   text-transform: uppercase;
   letter-spacing: 0.04em;
   white-space: nowrap;
}

.value-null {
   font-style: italic;
   opacity: 0.35;
   font-size: 0.78rem;
}

.value-empty {
   font-style: italic;
   opacity: 0.4;
   font-size: 0.78rem;
}

.value-text {
   font-size: 0.8rem;
   flex: 1;
}

.empty-state {
   padding: 2rem;
   text-align: center;
   opacity: 0.4;
   font-style: italic;
}

.btn-refresh {
   display: flex;
   align-items: center;
   width: 20%;
   justify-content: center;
   gap: 0.25rem;
   align-self: flex-end;
   background-color: #272727;

   &:hover {
      opacity: 0.9;
   }

   .spin {
      animation: spin 0.8s linear infinite;
   }
}

@keyframes spin {
   from { transform: rotate(0deg); }
   to   { transform: rotate(360deg); }
}
</style>
