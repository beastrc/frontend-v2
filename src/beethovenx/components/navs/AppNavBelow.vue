<script setup lang="ts">
import useProtocolDataQuery from '@/beethovenx/composables/queries/useProtocolDataQuery';
import { computed } from 'vue';
import useNumbers from '@/composables/useNumbers';
import useWeb3 from '@/services/web3/useWeb3';
import usePortfolio from '@/beethovenx/composables/usePortfolio';
import BalLoadingBlock from '@/components/_global/BalLoadingBlock/BalLoadingBlock.vue';
import usePortfolioValueQuery from '@/beethovenx/composables/queries/usePortfolioValueQuery';

const { fNum } = useNumbers();
const { account } = useWeb3();
const protocolDataQuery = useProtocolDataQuery();

const { portfolio, isLoadingPortfolio } = usePortfolio();
const portfolioValueQuery = usePortfolioValueQuery();

const procotolDataLoading = computed(() => protocolDataQuery.isLoading.value);

const tvl = computed(() => protocolDataQuery.data?.value?.totalLiquidity || 0);
const swapFee24h = computed(
  () => protocolDataQuery.data?.value?.swapFee24h || 0
);
const swapVolume24h = computed(
  () => protocolDataQuery.data?.value?.swapVolume24h || 0
);

const portfolioValue = computed(() => portfolioValueQuery.data?.value || '0');
const isLoadingPortfolioValue = computed(
  () => portfolioValueQuery.isLoading.value || portfolioValueQuery.isIdle.value
);
</script>

<template>
  <div class="bg-black">
    <div class="px-4 lg:px-6 py-2 flex items-center">
      <div class="flex-1 items-center">
        <span class="mr-4">
          TVL:
          <BalLoadingBlock
            v-if="procotolDataLoading"
            class="w-20 h-4 inline-block"
          />
          <span v-else class="text-green-500">${{ fNum(tvl, 'usd_lg') }}</span>
        </span>
        <span class="mr-4 hidden md:inline">
          Volume (24h):
          <BalLoadingBlock
            v-if="procotolDataLoading"
            class="w-20 h-4 inline-block"
          />
          <span v-else class="text-green-500">
            ${{ fNum(swapVolume24h, 'usd_lg') }}
          </span>
        </span>
        <span class="mr-4 hidden md:inline">
          Fees (24h):
          <BalLoadingBlock
            v-if="procotolDataLoading"
            class="w-16 h-4 inline-block"
          />
          <span v-else class="text-green-500">
            ${{ fNum(swapFee24h, 'usd_lg') }}
          </span>
        </span>
      </div>
      <router-link
        :to="{ name: 'my-portfolio' }"
        :class="['flex items-center mr-3 text-base font-medium']"
        v-if="account"
        ><BalBtn size="sm" color="blue">
          <BalIcon name="bar-chart-2" class="mr-2" />
          My Portfolio:&nbsp;<span>
            <BalLoadingBlock class="w-16 h-4" v-if="isLoadingPortfolioValue" />
            <template v-else>
              {{ fNum(portfolioValue, 'usd') }}
            </template>
          </span>
        </BalBtn>
      </router-link>
    </div>
  </div>
</template>
