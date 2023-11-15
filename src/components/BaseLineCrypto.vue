<script setup lang="ts">
import { computed, ref, inject } from "vue";
import { storeToRefs } from "pinia";
import { TCryptoData } from "@/stores/crypto.types";
import { useCryptoStore } from "@/stores/crypto";
import { BaseCryptoChart, FavoriteStar, Spinner } from "@/app.organizer";
import { useIntersectionObserver } from "@vueuse/core";
import useCurrencySymbol from "@/composables/useCurrencySymbol";

import { ROUTE_CRYPTO_VIEW } from "@/app.routes";

const props = defineProps<{
    item: TCryptoData,
}>();

const cryptoStore = useCryptoStore();

const { currencyActive, cryptoList, cryptoFavorites } = storeToRefs(cryptoStore);
const { addFavorite, removeFavorite } = cryptoStore;

// const crypto = ref(cryptoList.value.get(props.itemId) as TCryptoData)

const currencySymbol = computed(() => useCurrencySymbol(currencyActive.value));

const chartElement = ref();
const chartIsVisible = ref(false);

const isInFavorites = computed(() =>
  props.item ? (cryptoFavorites.value.get(props.item.id) ? true : false): false
);

const toggleFavorite = () => {
  if (isInFavorites.value && props.item) {
    removeFavorite(props.item);
  } else if (props.item) addFavorite(props.item);
};

useIntersectionObserver(chartElement, ([{ isIntersecting }]) => {
  if (isIntersecting) chartIsVisible.value = true;
});

const calculatedSparkline = computed(() => {
  if (!props.item?.sparkline_in_7d?.length) return [] as number[];

  const toReduce = props.item.sparkline_in_7d;

  const reduced = toReduce.reduce((acc, val, index) => {
    if (index && index % 23 === 0) acc.push(val);
    return acc;
  }, new Array<number>());

  return reduced.length > 3 ? reduced : [] as number[];
});

const orderedSparkLabels = computed(() => {
  if (!calculatedSparkline.value) return [];
  return calculatedSparkline.value.map((_, index: number) => {
    if (calculatedSparkline.value) {
      return "J" + (index - calculatedSparkline.value.length);
    } else return "";
  });
});
</script>

<template>
  <div
    class="line-crypto w-100 flex flex-1 h-16 mb-1 cursor-pointer"
    @click="
      (event) =>
        $router.push({
          name: ROUTE_CRYPTO_VIEW.name,
          params: { id: props.item.id },
        })
    "
  >
    <div class="flex w-20 pl-2 pr-2 items-center">
      <img
        v-if="props.item.image"
        v-lazy="props.item.image"
        class="w-8 h-8 border-round rounded-full"
      />
      <Spinner v-else color="#DDD" size="small" class="inline-block mx-auto" />
    </div>
    <div
      class="flex w-48 pl-4 pr-4 items-center text-black dark:text-white p-2 font-bold"
    >
      {{
        props.item.name.length > 20 ? props.item.name.slice(0, 20) + "..." : props.item.name
      }}
    </div>
    <div class="flex pl-4 pr-4 w-44 items-center text-black dark:text-white">
      <template
        v-if="props.item?.pricesByCurrencies[currencyActive]?.current_price"
      >
        {{ props.item.pricesByCurrencies[currencyActive].current_price }}
        {{ currencySymbol }}
      </template>
      <div v-else class="text-sm border-1 text-gray-300">N/A</div>
    </div>
    <div class="flex pl-4 pr-4 w-36 items-center text-black dark:text-white">
      <template v-if="props.item?.pricesByCurrencies[currencyActive]?.market_cap">
        {{ props.item.pricesByCurrencies[currencyActive].market_cap }}
        {{ currencySymbol }}
      </template>
      <div v-else class="text-sm border-1 text-gray-300">N/A</div>
    </div>
    <div class="flex pl-4 pr-4 w-40 items-center text-black dark:text-white">
      <template v-if="props.item?.pricesByCurrencies[currencyActive]?.total_volume">
        {{ props.item.pricesByCurrencies[currencyActive].total_volume }}
        {{ currencySymbol }}
      </template>
      <div v-else class="text-sm border-1 text-gray-300">N/A</div>
    </div>
    <div
      class="flex flex-1 w-200 items-center text-black dark:text-white pr-3"
      :ref="(ref) => (chartElement = ref)"
    >
      <template v-if="calculatedSparkline && chartIsVisible">
        <BaseCryptoChart
          :sparkline="calculatedSparkline"
          :labels="orderedSparkLabels"
          :grid="false"
          :tooltip="false"
          :win="
            calculatedSparkline[0] <
            calculatedSparkline[calculatedSparkline.length - 1]
          "
        />
      </template>
      <div v-else class="text-sm border-1 text-gray-300">N/A</div>
    </div>
    <div
      class="flex w-14 items-center justify-center pr-3 cursor-pointer"
      @click.prevent.stop="toggleFavorite"
    >
      <FavoriteStar :active="isInFavorites" />
    </div>
  </div>
</template>

<style lang="scss">
.line-crypto {
  transition: all 0.2s;
}

#app.dark {
  .line-crypto:hover {
    background-color: rgba(255, 255, 255, 0.02);
  }
}

#app.light {
  .line-crypto:hover {
    background-color: rgba(0, 0, 0, 0.02);
  }
}
</style>
