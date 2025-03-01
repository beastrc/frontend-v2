<template>
  <BalForm ref="depositForm" @on-submit="submit">
    <div>
      <BalTextInput
        name="Deposit"
        v-model="amount"
        v-model:isValid="validInput"
        :rules="amountRules()"
        :disabled="loading"
        type="number"
        min="0"
        step="any"
        placeholder="0"
        :decimal-limit="18"
        validate-on="input"
        prepend-border
        append-shadow
      >
        <template v-slot:info>
          <div class="cursor-pointer flex" @click.prevent="amount = bptBalance">
            {{ $t('balance') }}:
            <BalLoadingBlock v-if="loading" class="h-4 w-24 ml-1" white />
            <span v-else>&nbsp;{{ bptBalance }}</span>
          </div>
        </template>
        <template v-slot:append>
          <div class="p-2">
            <BalBtn
              size="xs"
              color="white"
              @click.prevent="amount = bptBalance"
            >
              {{ $t('max') }}
            </BalBtn>
          </div>
        </template>
      </BalTextInput>
    </div>

    <div class="pt-4">
      <BalBtn
        v-if="!isWalletReady"
        :label="$t('connectWallet')"
        block
        @click.prevent="toggleWalletSelectModal"
      />
      <template v-else>
        <BalBtn
          v-if="approvalRequired"
          label="Approve BPT"
          :loading="approving || loading"
          :loading-label="loading ? 'Loading' : $t('approving')"
          :disabled="!validInput || parseFloat(amount) === 0 || amount === ''"
          block
          @click.prevent="approveToken"
        />
        <template v-else>
          <BalBtn
            type="submit"
            :loading-label="loading ? 'Loading' : $t('confirming')"
            color="gradient"
            :disabled="!validInput || parseFloat(amount) === 0 || amount === ''"
            :loading="depositing || loading"
            block
          >
            Mint fBEETS
          </BalBtn>
        </template>
      </template>
    </div>
  </BalForm>
</template>

<script lang="ts">
import {
  computed,
  defineComponent,
  onMounted,
  reactive,
  ref,
  toRefs,
  watch
} from 'vue';
import { FormRef } from '@/types';
import {
  isLessThanOrEqualTo,
  isPositive,
  isRequired
} from '@/lib/utils/validations';
import { useI18n } from 'vue-i18n';
import useNumbers from '@/composables/useNumbers';
import { scale, scaleDown, sleep } from '@/lib/utils';
import useWeb3 from '@/services/web3/useWeb3';
import useEthers from '@/composables/useEthers';
import BigNumber from 'bignumber.js';
import { useFreshBeets } from '@/beethovenx/composables/stake/useFreshBeets';
import useAllowanceAvailableQuery from '@/beethovenx/composables/farms/useAllowanceAvailableQuery';
import { governanceContractsService } from '@/beethovenx/services/governance/governance-contracts.service';
import useTokens from '@/composables/useTokens';
import useFarmUser from '@/beethovenx/composables/farms/useFarmUser';

type DataProps = {
  depositForm: FormRef;
  amount: string;
  propMax: string[];
  validInput: boolean;
  propToken: number;
};

export default defineComponent({
  name: 'FreshBeetsDepositForm',

  components: {},

  emits: ['success'],

  props: {
    loading: {
      type: Boolean,
      required: true
    }
  },

  setup(props, { emit }) {
    const data = reactive<DataProps>({
      depositForm: {} as FormRef,
      amount: '',
      propMax: [],
      validInput: true,
      propToken: 0
    });

    const {
      isWalletReady,
      account,
      toggleWalletSelectModal,
      appNetworkConfig
    } = useWeb3();
    const { fNum } = useNumbers();
    const { t } = useI18n();
    const {
      userBptTokenBalance,
      approve,
      stake,
      freshBeetsQuery,
      userAllowance,
      refetch
    } = useFreshBeets();
    const { farmUserRefetch } = useFarmUser(appNetworkConfig.fBeets.farmId);
    const { refetchAllowances } = useTokens();

    const { amount } = toRefs(data);
    const depositing = ref(false);
    const approving = ref(false);

    const bptBalance = computed(() => userBptTokenBalance.value.toString());

    const { txListener } = useEthers();

    function amountRules() {
      return isWalletReady.value
        ? [
            isPositive(),
            isLessThanOrEqualTo(bptBalance.value, t('exceedsBalance'))
          ]
        : [isPositive()];
    }

    async function approveToken(): Promise<void> {
      if (!data.depositForm.validate()) return;

      try {
        approving.value = true;
        const tx = await approve();

        if (!tx) {
          approving.value = false;
          return;
        }

        txListener(tx, {
          onTxConfirmed: async () => {
            await refetch.value();
            approving.value = false;
          },
          onTxFailed: () => {
            approving.value = false;
          }
        });
      } catch {
        approving.value = false;
      }
    }

    const approvalRequired = computed(() => {
      if (amount.value === '' || parseFloat(amount.value) === 0) {
        return false;
      }

      const availableAllowance = new BigNumber(
        userAllowance.value?.toString() || '0'
      );
      const amountScaled = scale(new BigNumber(amount.value), 18);

      return availableAllowance.lt(amountScaled);
    });

    async function submit(): Promise<void> {
      if (!data.depositForm.validate()) return;

      depositing.value = true;
      const amountScaled = scale(new BigNumber(amount.value), 18);

      try {
        const tx = await stake(amountScaled.toString());

        if (!tx) {
          depositing.value = false;
          return;
        }

        txListener(tx, {
          onTxConfirmed: async () => {
            await freshBeetsQuery.refetch.value();
            emit('success', tx);
            data.amount = '';
            depositing.value = false;
          },
          onTxFailed: () => {
            depositing.value = false;
          }
        });
      } catch {
        depositing.value = false;
      }
    }

    watch(isWalletReady, isAuth => {
      if (!isAuth) {
        data.amount = '0';
        data.propMax = [];
      }
    });

    watch(account, () => {
      //
    });

    onMounted(() => {
      //
    });

    return {
      // data
      ...toRefs(data),

      approvalRequired,
      approveToken,
      depositing,
      approving,

      appNetworkConfig,
      amountRules,
      isWalletReady,
      toggleWalletSelectModal,
      isRequired,
      // methods
      submit,
      fNum,
      bptBalance
    };
  }
});
</script>
