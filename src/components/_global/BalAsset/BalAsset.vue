<template>
  <img
    v-if="
      iconSRC &&
        !error &&
        !iconSRC.startsWith('https://raw.githubusercontent.com/')
    "
    :src="iconSRC"
    :style="{
      width: `${size}px`,
      height: `${size}px`,
      background: 'transparent'
    }"
    @error="error = true"
    class="rounded-full inline-block leading-none shadow-sm"
  />
  <Avatar v-else :address="address" :size="size" />
</template>

<script>
import { defineComponent, toRefs, ref, computed, watch } from 'vue';
import useTokens from '@/composables/useTokens';
import Avatar from '../../images/Avatar.vue';
import useUrls from '@/composables/useUrls';
import { find } from 'lodash';

export default defineComponent({
  name: 'BalAsset',

  components: {
    Avatar
  },

  props: {
    address: {
      type: String,
      required: true
    },
    iconURI: { type: String },
    size: {
      type: Number,
      default: 24
    }
  },
  setup(props) {
    /**
     * COMPOSABLES
     */
    const { tokens } = useTokens();
    const { resolve } = useUrls();

    /**
     * STATE
     */
    const { address } = toRefs(props);
    const error = ref(false);

    /**
     * COMPUTED
     */
    const iconSRC = computed(() => {
      if (props.iconURI) return resolve(props.iconURI);

      const token = find(tokens.value, (token, tokenAddress) => {
        return (
          address.value &&
          tokenAddress.toLowerCase() === address.value.toLowerCase()
        );
      });

      if (!token) {
        return '';
      }

      return resolve(token.logoURI);
    });

    /**
     * WATCHERS
     */
    watch(iconSRC, newURL => {
      if (newURL !== '') error.value = false;
    });

    return {
      iconSRC,
      error
    };
  }
});
</script>
