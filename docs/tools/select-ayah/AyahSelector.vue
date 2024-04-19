<script lang="ts" setup>
import { computed, ref } from 'vue'
import { loadText } from '@ghoran/text'
import { pageList } from '@ghoran/metadata'
import { Ayah } from '@ghoran/entity'

const pageNumber = ref(1)

const isValidPageNumber = computed(
  () =>
    typeof pageNumber.value === 'number' &&
    pageNumber.value >= 1 &&
    pageNumber.value <= 604,
)

const text = await loadText('tanzil-simple-min')
const startIndex = computed(() => pageList[pageNumber.value - 1])
const endIndex = computed(() => pageList[pageNumber.value] || text.length)

const pageText = computed(() =>
  text.slice(startIndex.value, endIndex.value).map((ayah) => ayah.split(' ')),
)

const pageAyat = computed(() =>
  pageText.value.map((ayah, i) => Ayah.get(startIndex.value + i)),
)
</script>

<template>
  <div>
    <div class="mb-3 text-lg">
      صفحه‌ی
      <input
        type="number"
        min="1"
        max="604"
        dir="ltr"
        v-model.number="pageNumber"
        class="border-gray-500 border-solid border-px px-1 rounded"
      />
    </div>
    <div v-if="isValidPageNumber">
      <div v-for="(ayah, i) in pageText" class="mb-4">
        <div v-if="pageAyat[i].isFirstOfSurah" class="mb-3">
          سوره {{ pageAyat[i].surah.name }}
        </div>

        <div class="flex items-center">
          <div class="mr-3">
            {{ pageAyat[i].index }}
          </div>
          <div>
            <span
              v-for="(word, offset) in ayah"
              class="border-1 border-gray-500 group border-solid relative mx-1 rounded inline-block px-1 mb-1 py-0.5"
            >
              {{ word }}
              <span
                class="hidden cursor-default text-bold group-hover:flex absolute bg-green-500/80 items-center justify-center inset-0"
              >
                {{ offset }}
              </span>
            </span>
            <span class="text-xs">
              {{ pageAyat[i].ayahNumber }}&nbsp;{{ pageAyat[i].surah.name }}
            </span>
          </div>
        </div>
      </div>
    </div>
    <div v-else>شماره صفحه نامعتبر است</div>
  </div>
</template>