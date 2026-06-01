<template>
  <div
    @click="focusNewTag"
    v-click-outside="closeContextMenu"
    :class="[
      'relative flex min-h-[42px] cursor-text flex-wrap rounded-lg border bg-white text-left transition',
      isInputActive
        ? 'border-black ring-1 ring-black'
        : 'border-gray-300',
      isError ? 'border-red-400' : '',
      disabled ? 'opacity-50 pointer-events-none' : '',
    ]"
  >
    <div
      ref="inputBox"
      :class="[
        'flex w-full flex-wrap gap-2 p-2',
        select ? 'pr-8' : '',
      ]"
    >
      <span
        v-for="(tag, index) in innerTags"
        :key="index"
        class="inline-flex h-8 max-w-full items-center rounded-md bg-blue-500 px-2 text-sm text-white"
      >
        <slot
          v-if="isShot('item')"
          name="item"
          v-bind="{ name: tag, index, tag }"
        />

        <span
          v-else
          class="truncate"
        >
          {{ tag }}
        </span>

        <button
          v-if="!readOnly"
          @click.prevent.stop="remove(index)"
          type="button"
          class="ml-2 text-white/70 transition hover:text-white"
        >
          ×
        </button>
      </span>

      <input
        ref="inputTag"
        v-model="newTag"
        :placeholder="placeholder"
        :disabled="readOnly"
        @keydown="addNew"
        @blur="handleInputBlur"
        @focus="handleInputFocus"
        @input="makeItNormal"
        :class="[
          'min-w-full w-full flex-1 bg-transparent px-1 outline-none rounded form-input',
          isError ? 'text-red-400 underline' : '',
        ]"
      />
    </div>

    <section
      v-if="select"
      ref="contextMenu"
      :class="[
        'absolute left-0 right-0 top-full z-50 mt-1 hidden max-h-48 overflow-auto rounded-lg border border-gray-200 bg-white shadow-lg',
        !isShowNoData && selectItems.length === 0 ? 'p-0' : 'p-1',
      ]"
    >
      <div
        v-if="loading"
        class="p-3 text-center text-sm text-gray-500"
      >
        <slot v-if="isShot('loading')" />
        <div v-else>Loading...</div>
      </div>

      <div
        v-if="!loading && selectItems.length === 0 && isShowNoData"
        class="p-3 text-center text-sm text-gray-400"
      >
        <slot
          v-if="isShot('no-data')"
          name="no-data"
        />

        <span v-else>No data</span>
      </div>

      <div
        v-if="!loading && selectItems.length > 0"
        class="space-y-1"
      >
        <div
          v-for="(item, index) in selectItems"
          :key="index"
          @click.stop="handleSelect(item)"
          :class="[
            'flex cursor-pointer items-center rounded-md px-3 py-2 transition',
            isShowCheckmark(item)
              ? 'bg-blue-50 text-blue-600'
              : 'hover:bg-gray-100',
          ]"
        >
          <div class="min-w-0 flex-1">
            <slot
              name="select-item"
              v-bind="item"
            />
          </div>

          <svg
            v-if="isShowCheckmark(item)"
            class="ml-2 h-4 w-4"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M5 12l5 5L20 7"
            />
          </svg>
        </div>
      </div>
    </section>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from 'vue';
import vClickOutside from 'click-outside-vue3';

type TagItem = string;

export default defineComponent({
  name: 'TagsInput',

  directives: {
    clickOutside: vClickOutside.directive,
  },

  emits: [
    'update:modelValue',
    'update:tags',
    'on-limit',
    'on-tags-changed',
    'on-remove',
    'on-error',
    'on-focus',
    'on-blur',
    'on-select',
    'on-select-duplicate-tag',
    'on-new-tag',
  ],

  props: {
    readOnly: {
      type: Boolean,
      default: false,
    },

    disabled: {
      type: Boolean,
      default: false,
    },

    error: {
      type: Boolean,
      default: false,
    },

    modelValue: {
      type: String,
      default: '',
    },

    validate: {
      type: [String, Function, Object] as PropType<
        string | ((tag: string) => boolean) | object
      >,
      default: '',
    },

    addTagOnKeys: {
      type: Array as PropType<Array<number | string>>,
      default: () => [13, ',', 32],
    },

    placeholder: {
      type: String,
      default: '',
    },

    tags: {
      type: Array as PropType<TagItem[]>,
      default: () => [],
    },

    loading: {
      type: Boolean,
      default: false,
    },

    limit: {
      type: Number,
      default: -1,
    },

    allowDuplicates: {
      type: Boolean,
      default: false,
    },

    addTagOnBlur: {
      type: Boolean,
      default: false,
    },

    // these are separate from tags
    // select menu items can be any object/string
    selectItems: {
      type: Array as PropType<any[]>,
      default: () => [],
    },

    select: {
      type: Boolean,
      default: false,
    },

    duplicateSelectItem: {
      type: Boolean,
      default: true,
    },

    uniqueSelectField: {
      type: String,
      default: 'id',
    },

    addTagOnKeysWhenSelect: {
      type: Boolean,
      default: false,
    },

    isShowNoData: {
      type: Boolean,
      default: true,
    },
  },

  data() {
    return {
      isInputActive: false,
      isError: false,
      newTag: '',
      innerTags: [] as TagItem[],
      multiple: false,
    };
  },

  computed: {
    isLimit(): boolean {
      const isLimit =
        this.limit > 0 &&
        this.innerTags.length >= this.limit;

      if (isLimit) {
        this.$emit('on-limit');
      }

      return isLimit;
    },

    selectedItemsIds(): any[] {
      if (!this.duplicateSelectItem) {
        return this.tags;
      }

      return [];
    },
  },

  watch: {
    error: {
      immediate: true,
      handler(value: boolean) {
        this.isError = value;
      },
    },

    modelValue: {
      immediate: true,
      handler(value: string) {
        this.newTag = value;
      },
    },

    tags: {
      deep: true,
      immediate: true,
      handler(tags: TagItem[]) {
        this.innerTags = [...tags];
      },
    },
  },

  methods: {
    isShot(name: string): boolean {
      return !!this.$slots[name];
    },

    makeItNormal(event: Event): void {
      const target = event.target as HTMLInputElement;

      this.$emit('update:modelValue', target.value);

      const input = this.$refs.inputTag as
        | HTMLInputElement
        | undefined;

      if (input) {
        input.className = 'v3ti-new-tag';
        input.classList.add('form-input');
        input.classList.add('rounded');
        input.classList.add('min-w-full');
        input.style.textDecoration = 'none';
      }
    },

    resetData(): void {
      this.innerTags = [];
    },

    resetInputValue(): void {
      this.newTag = '';
      this.$emit('update:modelValue', '');
    },

    setPosition(): void {
      const el = this.$refs.inputBox as
        | HTMLElement
        | undefined;

      const menu = this.$refs.contextMenu as
        | HTMLElement
        | undefined;

      if (!el || !menu) {
        return;
      }

      menu.style.display = 'block';

      const elementHeight = el.clientHeight || 32;
      const borderHeight = 3;

      menu.style.top = `${elementHeight + borderHeight}px`;
    },

    closeContextMenu(): void {
      const menu = this.$refs.contextMenu as
        | HTMLElement
        | undefined;

      if (menu) {
        menu.style.display = 'none';
      }
    },

    handleSelect(tagData: any): void {
      if (this.isShowCheckmark(tagData)) {
        const tags = this.tags.filter(
          (tag) => tag !== tagData
        );

        this.$emit('update:tags', tags);
        this.$emit(
          'on-select-duplicate-tag',
          tagData
        );

        this.resetInputValue();
      } else {
        this.$emit('on-select', tagData);
      }

      this.$nextTick(() => {
        this.closeContextMenu();
      });
    },

    isShowCheckmark(tag: any): boolean {
      if (!this.duplicateSelectItem) {
        return this.selectedItemsIds.includes(tag);
      }

      return false;
    },

    focusNewTag(): void {
      if (this.select && !this.disabled) {
        this.setPosition();
      }

      if (this.readOnly) {
        return;
      }

      const input = (
        this.$el as HTMLElement
      ).querySelector('.v3ti-new-tag') as
        | HTMLInputElement
        | null;

      input?.focus();
    },

    handleInputFocus(event: FocusEvent): void {
      this.isInputActive = true;
      this.$emit('on-focus', event);
    },

    handleInputBlur(event: FocusEvent): void {
      this.isInputActive = false;

      this.addNew(event);

      this.$emit('on-blur', event);
    },

    addNew(event?: KeyboardEvent | FocusEvent): void {
      if (
        this.select &&
        !this.addTagOnKeysWhenSelect
      ) {
        return;
      }

      const keyboardEvent = event as KeyboardEvent;

      const keyShouldAddTag = event
        ? this.addTagOnKeys.includes(
            keyboardEvent.keyCode
          ) ||
          this.addTagOnKeys.includes(
            keyboardEvent.key
          )
        : true;

      const typeIsNotBlur =
        event?.type !== 'blur';

      if (
        (!keyShouldAddTag &&
          (typeIsNotBlur || !this.addTagOnBlur)) ||
        this.isLimit
      ) {
        return;
      }

      const trimmedTag = this.newTag.trim();

      if (
        trimmedTag &&
        (this.allowDuplicates ||
          !this.innerTags.includes(trimmedTag)) &&
        this.validateIfNeeded(trimmedTag)
      ) {
        this.innerTags.push(trimmedTag);

        if (this.addTagOnKeysWhenSelect) {
          this.$emit('on-new-tag', trimmedTag);

          this.updatePositionContextMenu();
        }

        this.resetInputValue();

        this.tagChange();

        event?.preventDefault();
      } else {
        this.makeItError(true);

        event?.preventDefault();
      }
    },

    updatePositionContextMenu(): void {
      this.$nextTick(() => {
        this.setPosition();
      });
    },

    makeItError(
      isDuplicatedOrMaxLength: boolean
    ): void {
      if (!this.newTag) {
        return;
      }

      const input = this.$refs.inputTag as
        | HTMLInputElement
        | undefined;

      if (input) {
        input.className =
          'v3ti-new-tag form-input w-full min-w-full rounded v3ti-new-tag--error';

        input.style.textDecoration =
          'underline';
      }

      this.$emit(
        'on-error',
        isDuplicatedOrMaxLength
      );
    },

    validateIfNeeded(tagValue: string): boolean {
      if (
        this.validate === '' ||
        this.validate === undefined
      ) {
        return true;
      }

      if (typeof this.validate === 'function') {
        return this.validate(tagValue);
      }

      return true;
    },

    removeLastTag(): void {
      if (this.newTag) {
        return;
      }

      this.innerTags.pop();

      this.tagChange();

      this.updatePositionContextMenu();
    },

    remove(index: number): void {
      this.innerTags.splice(index, 1);

      this.tagChange();

      this.$emit('on-remove', index);

      this.updatePositionContextMenu();
    },

    tagChange(): void {
      this.$emit(
        'on-tags-changed',
        [...this.innerTags]
      );
    },
  },
});
</script>
