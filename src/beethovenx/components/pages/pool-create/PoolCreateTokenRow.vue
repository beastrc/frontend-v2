<template>
  <div>
    <BalTextInput
      name="pool-token-amount"
      :model-value="tokenAmountInput"
      @input="value => handleAmountChange(value)"
      type="number"
      :decimal-limit="tokenDecimals"
      min="0"
      step="any"
      placeholder="0"
      validate-on="input"
      prepend-border
      :no-margin="true"
      :title="title"
    >
      <template v-slot:prepend>
        <div class="flex items-center h-full mr-4">
          <div
            class="w-32 h-full cursor-pointer group flex items-center h-full"
            @click="openModalSelectToken('input')"
          >
            <BalAsset
              v-if="tokenAddressInput"
              :iconURI="tokenIconUrl"
              :address="tokenAddressInput"
              :size="28"
            />
            <div class="flex flex-col ml-3 w-24 leading-none truncate">
              <BalTooltip v-if="tokenSymbol.length > 5">
                <template v-slot:activator>
                  <span class="font-bold">
                    {{ tokenSymbol }}
                  </span>
                </template>
                <div>
                  {{ tokenSymbol }}
                </div>
              </BalTooltip>
              <span v-else class="font-bold">
                {{ tokenSymbol }}
              </span>
            </div>
            <BalIcon
              :name="'chevron-down'"
              :size="'sm'"
              :class="tokenChangeDisabled ? 'text-gray-500' : 'text-green-500'"
            />
          </div>
          <div
            v-if="hasTokenWeight"
            class="w-24 border-l border-r dark:border-gray-850 ml-4"
          >
            <BalTextInput
              name="pool-token-weight"
              :model-value="tokenWeightInput"
              @input="value => handleWeightChange(value)"
              type="number"
              :decimal-limit="4"
              min="0"
              step="any"
              placeholder="0"
              validate-on="input"
              prepend-border
              :no-margin="true"
            >
              <template v-slot:info>Weight (%)</template>
            </BalTextInput>
          </div>
          <div v-else class="border-r dark:border-gray-850 ml-4 h-20" />
        </div>
      </template>
      <template v-slot:info>
        <div class="cursor-pointer" @click="handleMax">
          {{ $t('balance') }}: {{ fNum(balanceLabel, 'token') }}
        </div>
      </template>
      <template v-slot:append>
        <div class="p-2 h-full flex items-center">
          <BalBtn
            size="xs"
            color="white"
            class="h-full"
            @click="approveToken()"
            v-if="requiresApproval"
            :loading="approving"
            loading-label="Approving..."
          >
            <span>Approve</span>
          </BalBtn>

          <BalBtn
            v-if="!hideDelete"
            size="xs"
            color="white"
            @click="handleTokenDelete"
            class="ml-4 h-full"
            :disabled="!canDelete"
          >
            <BalIcon
              name="trash"
              :class="[canDelete ? 'text-red-500' : 'text-gray-500']"
            />
          </BalBtn>
        </div>
      </template>
    </BalTextInput>
  </div>
  <teleport to="#modal">
    <SelectTokenModal
      v-if="modalSelectTokenIsOpen"
      :excludedTokens="[tokenAddressInput, nativeAssetAddress]"
      @close="modalSelectTokenIsOpen = false"
      @select="handleSelectToken"
      include-ether
      :included-tokens="allowedTokens"
    />
  </teleport>
</template>

<script lang="ts">
import { computed, defineComponent, PropType, ref, toRefs, watch } from 'vue';
import { NATIVE_ASSET_ADDRESS } from '@/constants/tokens';
import useTokens from '@/composables/useTokens';
import useNumbers from '@/composables/useNumbers';
import SelectTokenModal from '@/components/modals/SelectTokenModal/SelectTokenModal.vue';
import BalIcon from '@/components/_global/BalIcon/BalIcon.vue';
import useWeb3 from '@/services/web3/useWeb3';
import useTokenApproval from '@/composables/trade/useTokenApproval';
import { getAddress, isAddress } from '@ethersproject/address';

const ETH_BUFFER = 0.1;

export default defineComponent({
  components: { BalIcon, SelectTokenModal },
  props: {
    tokenAmountInput: {
      type: String,
      required: true
    },
    tokenWeightInput: {
      type: String,
      required: true
    },
    tokenAddressInput: {
      type: String,
      required: true
    },
    canDelete: {
      type: Boolean,
      required: true
    },
    hasTokenWeight: {
      type: Boolean,
      required: true
    },
    hideDelete: {
      type: Boolean
    },
    title: {
      type: String
    },
    tokenIconUrl: {
      type: String
    },
    allowedTokens: { type: Array as PropType<string[]> },
    spenderAddress: {
      type: String
    }
  },
  emits: [
    'tokenAddressChange',
    'tokenAmountChange',
    'tokenWeightChange',
    'tokenDelete',
    'tokenApproved',
    'requiresApproval'
  ],

  setup(props, { emit }) {
    const {
      tokens,
      balances,
      injectTokens,
      approvalRequired,
      refetchAllowances,
      allowances
    } = useTokens();
    const { fNum, toFiat } = useNumbers();
    const { appNetworkConfig } = useWeb3();
    const { tokenAmountInput, tokenAddressInput } = toRefs(props);
    const spenderAddress = computed(
      () => props.spenderAddress || appNetworkConfig.addresses.vault
    );
    const {
      approving,
      approved,
      approveSpender,
      isLoading: isLoadingApprovals
    } = useTokenApproval(tokenAddressInput, tokenAmountInput, tokens);

    const requiresApproval = computed(() => {
      if (approved.value) {
        return false;
      }

      return approvalRequired(
        tokenAddressInput.value,
        tokenAmountInput.value,
        spenderAddress.value
      );
    });

    const tokenValue = computed(() =>
      toFiat(tokenAmountInput.value, tokenAddressInput.value)
    );

    const tokenSymbol = computed(() => {
      const token = tokens.value[tokenAddressInput.value];
      const symbol = token ? token.symbol : '';
      return symbol;
    });

    const tokenDecimals = computed(() => {
      const decimals = tokens.value[tokenAddressInput.value]
        ? tokens.value[tokenAddressInput.value].decimals
        : 18;
      return decimals;
    });

    const tokenChangeDisabled = computed(
      () => props.allowedTokens && props.allowedTokens.length === 1
    );

    const isInRate = ref(true);
    const modalSelectTokenType = ref('input');
    const modalSelectTokenIsOpen = ref(false);

    function handleMax(): void {
      const balance = balances.value[tokenAddressInput.value] || '0';
      const balanceNumber = parseFloat(balance);
      const maxAmount =
        tokenAddressInput.value !== NATIVE_ASSET_ADDRESS
          ? balance
          : balanceNumber > ETH_BUFFER
          ? (balanceNumber - ETH_BUFFER).toString()
          : '0';
      handleAmountChange(maxAmount);
    }

    function handleAmountChange(value: string): void {
      emit('tokenAmountChange', value);
    }

    function handleWeightChange(value: string): void {
      emit('tokenWeightChange', value);
    }

    function handleSelectToken(address: string): void {
      emit('tokenAddressChange', address);
      injectTokens([address]);
    }

    function handleTokenDelete(): void {
      emit('tokenDelete');
    }

    function toggleRate(): void {
      isInRate.value = !isInRate.value;
    }

    function openModalSelectToken(type: string): void {
      if (tokenChangeDisabled.value) {
        return;
      }

      modalSelectTokenIsOpen.value = true;
      modalSelectTokenType.value = type;
    }

    async function approveToken() {
      await approveSpender(spenderAddress.value);
      await refetchAllowances.value();
    }

    const balanceLabel = computed(
      () => balances.value[tokenAddressInput.value]
    );

    if (requiresApproval.value) {
      emit('requiresApproval', props.tokenAddressInput);
    }

    watch(requiresApproval, newVal => {
      if (newVal) {
        emit('requiresApproval', props.tokenAddressInput);
      } else {
        emit('tokenApproved', props.tokenAddressInput);
      }
    });

    return {
      fNum,
      handleMax,
      balanceLabel,
      handleAmountChange,
      toggleRate,
      tokenValue,
      tokenSymbol,
      modalSelectTokenIsOpen,
      openModalSelectToken,
      handleSelectToken,
      handleWeightChange,
      handleTokenDelete,
      tokenDecimals,
      nativeAssetAddress: appNetworkConfig.nativeAsset.address,
      approved,
      approving,
      isLoadingApprovals,
      tokenChangeDisabled,
      requiresApproval,
      approveToken
    };
  }
});
</script>
