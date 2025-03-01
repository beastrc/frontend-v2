<script setup lang="ts">
import { ref, computed } from 'vue';
import SelectTokenModal from '@/components/modals/SelectTokenModal/SelectTokenModal.vue';
import useTokens from '@/composables/useTokens';
import useNumbers from '@/composables/useNumbers';
import { TokenInfo } from '@/types/TokenList';
import { GqlLge } from '@/beethovenx/services/beethovenx/beethovenx-types';

/**
 * TYPES
 */
type Props = {
  modelValue: string;
  fixed?: boolean;
  weight?: number | string;
  excludedTokens?: string[];
  options?: string[];
  lge: GqlLge;
};

/**
 * PROPS & EMITS
 */
const props = withDefaults(defineProps<Props>(), {
  modelValue: '',
  fixed: false,
  weight: 0,
  excludedTokens: () => [],
  options: () => []
});

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void;
}>();

/**
 * STATE
 */
const openTokenModal = ref(false);

/**
 * COMPOSABLEs
 */
const { getToken } = useTokens();
const { fNum } = useNumbers();

/**
 * COMPUTED
 */
const hasToken = computed(() => !!props.modelValue);

const token = computed((): TokenInfo | null => {
  if (!hasToken.value) return null;
  return getToken(props.modelValue);
});

const isLaunchToken = computed((): boolean => {
  return (
    token.value?.address.toLowerCase() ===
    props.lge.tokenContractAddress.toLowerCase()
  );
});

/**
 * METHODS
 */
function toggleModal(): void {
  if (!props.fixed) openTokenModal.value = !openTokenModal.value;
}

function tokenFor(option: string): TokenInfo {
  return getToken(option);
}
</script>

<template>
  <div>
    <div
      v-if="hasToken && options.length === 0"
      :class="['token-select-input selected group', { selectable: !fixed }]"
      @click="toggleModal"
    >
      <div class="w-8">
        <BalAsset
          :address="token?.address"
          :iconURI="isLaunchToken ? props.lge.tokenIconUrl : undefined"
          class="shadow"
        />
      </div>
      <span class="text-base font-medium">
        {{ token?.symbol }}
      </span>
      <span v-if="Number(weight) > 0" class="text-gray-500 ml-2">
        {{ fNum(weight, 'percent_lg') }}
      </span>
      <BalIcon
        v-if="!fixed"
        name="chevron-down"
        size="sm"
        class="text-green-500 group-hover:text-pink-500 ml-2"
      />
    </div>
    <BalDropdown
      v-else-if="hasToken && fixed && options.length > 0"
      :options="options"
      minWidth="40"
      @selected="emit('update:modelValue', $event)"
    >
      <template #activator>
        <div class="token-select-input selected group selectable">
          <div class="w-8">
            <BalAsset :address="token?.address" class="shadow" />
          </div>
          <span class="text-base font-medium">
            {{ token?.symbol }}
          </span>
          <span v-if="Number(weight) > 0" class="text-gray-500 ml-2">
            {{ fNum(weight, 'percent_lg') }}
          </span>
          <BalIcon
            name="chevron-down"
            size="sm"
            class="text-green-500 group-hover:text-pink-500 ml-2"
          />
        </div>
      </template>
      <template #option="{ option: address }">
        <div
          :set="(optionToken = tokenFor(address) || {})"
          class="flex items-center justify-between"
        >
          <div class="flex items-center">
            <BalAsset :address="optionToken?.address" class="shadow" />
            <span class="ml-1 font-medium">
              {{ optionToken?.symbol }}
            </span>
          </div>
          <BalIcon
            v-if="optionToken.address === modelValue"
            name="check"
            class="text-green-500 ml-4"
          />
        </div>
      </template>
    </BalDropdown>

    <div
      v-else
      class="token-select-input unselected selectable"
      @click="toggleModal"
    >
      {{ $t('selectToken') }}
      <BalIcon name="chevron-down" size="sm" class="ml-2" />
    </div>

    <teleport to="#modal">
      <SelectTokenModal
        v-if="openTokenModal"
        :excludedTokens="[...excludedTokens, modelValue]"
        :includeEther="true"
        @close="openTokenModal = false"
        @select="emit('update:modelValue', $event)"
      />
    </teleport>
  </div>
</template>

<style scoped>
.token-select-input {
  @apply shadow rounded-lg flex items-center h-10 px-2 whitespace-nowrap;
  @apply text-sm;
  font-variation-settings: 'wght' 700;
}

.selectable {
  @apply cursor-pointer hover:shadow-none transition-shadow;
}

.unselected {
  @apply bg-green-500 text-white;
}

.selected {
  @apply bg-gray-50 dark:bg-gray-700 text-black dark:text-white;
}
</style>
