<script setup lang="ts">
import useNumbers from '@/composables/useNumbers';
import { isStablePhantom, isWstETH } from '@/composables/usePool';
import useTokens from '@/composables/useTokens';
import useUserSettings from '@/composables/useUserSettings';
import { bnum } from '@/lib/utils';
import { FullPool } from '@/services/balancer/subgraph/types';
import { computed } from 'vue';
import { useI18n } from 'vue-i18n';

/**
 * TYPES
 */
type Props = {
  pool: FullPool;
  fiatTotal: string;
  priceImpact: number;
};

/**
 * PROPS & EMITS
 */
const props = defineProps<Props>();

/**
 * COMPOSABLES
 */
const { fNum } = useNumbers();
const { currency } = useUserSettings();
const { t } = useI18n();
const { getTokens } = useTokens();

/**
 * COMPUTED
 */
const totalWeeklyYield = computed((): string =>
  weeklyYieldForAPR(`${props.pool.apr.total}`)
);

const swapFeeWeeklyYield = computed((): string =>
  weeklyYieldForAPR(props.pool.apr.swapApr)
);

/*const thirdPartyWeeklyYield = computed((): string =>
  weeklyYieldForAPR(props.pool.apr.thirdPartyApr)
);

const lmWeeklyYield = computed((): string =>
  weeklyYieldForAPR(props.pool.apr.beetsApr)
);

const lmBreakdown = computed(
  () => props.pool.dynamic.apr.liquidityMiningBreakdown
);

const lmTokens = computed(() => getTokens(Object.keys(lmBreakdown.value)));

const lmMultiRewardPool = computed(
  () => Object.keys(lmTokens.value).length > 1
);*/

const hasThirdPartyAPR = computed(() =>
  bnum(props.pool.apr.thirdPartyApr).gt(0)
);

/*const thirdPartyBreakdown = computed(
  () => props.pool.dynamic.apr.thirdPartyBreakdown
);*/

/*const thirdPartyTokens = computed(() =>
  getTokens(Object.keys(thirdPartyBreakdown.value))
);*/

/*const thirdPartyMultiRewardPool = computed(
  () => Object.keys(thirdPartyTokens.value).length > 1
);

const thirdPartyFiatLabel = computed(() => {
  if (isWstETH(props.pool)) return t('thirdPartyRewards.fiat.steth');
  if (isStablePhantom(props.pool.poolType))
    return t('thirdPartyRewards.fiat.aaveBoosted');

  return '';
});*/

/**
 * METHODS
 */
function weeklyYieldForAPR(apr: string): string {
  return bnum(apr)
    .times(props.fiatTotal)
    .div(52)
    .toString();
}
</script>

<template>
  <div class="summary-table">
    <h6 class="p-2">
      {{ $t('summary') }}
    </h6>
    <div class="flex flex-col py-2">
      <div class="summary-table-row">
        <div class="summary-table-label">
          {{ $t('total') }}
        </div>
        <div class="summary-table-number">
          {{ fNum(fiatTotal, currency) }}
          <BalTooltip
            :text="$t('tooltips.invest.total', [currency.toUpperCase()])"
            icon-size="sm"
            class="ml-2"
          />
        </div>
      </div>
      <div class="summary-table-row">
        <div class="summary-table-label">
          {{ $t('priceImpact') }}
        </div>
        <div class="summary-table-number">
          {{ fNum(priceImpact, 'percent') }}
          <BalTooltip
            :text="$t('tooltips.invest.priceImpact')"
            icon-size="sm"
            width="72"
            class="ml-2"
          />
        </div>
      </div>
      <div class="summary-table-row">
        <div class="summary-table-label" v-text="$t('potentialWeeklyYield')" />
        <div class="summary-table-number">
          {{ fNum(totalWeeklyYield, currency) }}
          <BalTooltip icon-size="sm" width="72" noPad>
            <template v-slot:activator>
              <StarsIcon
                v-if="props.pool.apr.hasRewardApr || hasThirdPartyAPR"
                class="h-4 text-yellow-300"
              />
              <BalIcon
                v-else
                name="info"
                size="sm"
                class="text-gray-300 ml-2"
              />
            </template>
            <div
              class="p-2 bg-gray-50 dark:bg-gray-700 rounded-t-lg border-b dark:border-gray-700"
            >
              <span class="text-sm" v-text="$t('yieldEarnings')" />
              <span class="ml-1 text-gray-500">
                ({{ $t('basedOnLast24h') }})
              </span>
              <div class="text-base font-semibold mt-1">
                {{ fNum(totalWeeklyYield, currency) }}
                {{ $t('perWeek') }}
              </div>
            </div>
            <div class="p-2">
              <div class="whitespace-nowrap flex items-center mb-1">
                {{ fNum(swapFeeWeeklyYield, currency) }}
                <span class="ml-1 text-gray-500 text-xs">
                  {{ $t('swapFee') }}
                </span>
              </div>
              <!--              <BalBreakdown
                :items="Object.entries(thirdPartyBreakdown)"
                v-if="hasThirdPartyAPR"
                :hideItems="!thirdPartyMultiRewardPool"
              >
                <div class="flex items-center">
                  {{ fNum(thirdPartyWeeklyYield, currency) }}
                  <span class="ml-1 text-gray-500 text-xs">
                    {{ thirdPartyFiatLabel }}
                  </span>
                </div>
                <template v-if="thirdPartyMultiRewardPool" #item="{ item }">
                  {{ fNum(weeklyYieldForAPR(item[1]), currency) }}
                  <span class="text-gray-500 text-xs ml-1">
                    {{ thirdPartyTokens[item[0]].symbol }}
                  </span>
                </template>
              </BalBreakdown>
              <BalBreakdown
                v-if="props.pool.hasLiquidityMiningRewards"
                :items="Object.entries(lmBreakdown)"
                :hideItems="!lmMultiRewardPool"
              >
                <div class="flex items-center">
                  <span>{{ fNum(lmWeeklyYield, currency) }}</span>
                  <span class="ml-1 text-gray-500">
                    {{ $t('liquidityMining') }}
                  </span>
                </div>
                <template v-if="lmMultiRewardPool" v-slot:item="{ item }">
                  {{ fNum(weeklyYieldForAPR(item[1]), currency) }}
                  <span class="text-gray-500 ml-1">
                    {{ lmTokens[item[0]].symbol }}
                  </span>
                </template>
              </BalBreakdown>-->
            </div>
          </BalTooltip>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.summary-table {
  @apply border dark:border-gray-700 divide-y dark:divide-gray-700 rounded-lg mt-4;
}

.summary-table-row {
  @apply grid grid-cols-2 px-2 py-1;
}

.summary-table-label {
  @apply flex items-center;
}

.summary-table-number {
  @apply flex items-center justify-end;
}
</style>
