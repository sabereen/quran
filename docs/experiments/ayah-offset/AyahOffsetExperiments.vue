<script setup lang="ts">
import { loadText } from '@ghoran/text'
import { computed, ref } from 'vue'

const mode = ref<'word' | 'alphabet'>('word')
const autofix = ref(false)

const [hafsLikeText, imlaLikeText] = await Promise.all([
  loadText('hafs'),
  loadText('imla'),
])

const wordStrategy = {
  splitHafs: (ayah: string): string[] => ayah.split(' '),
  splitImla: (ayah: string): string[] => ayah.split(' '),
  match(hafsItem: string, imlaItem: string) {
    return (
      alphabetStrategy.splitHafs(hafsItem).length ===
      alphabetStrategy.splitImla(imlaItem).length
    )
  },
  test(textsToUse: { hafs: string[]; imla: string[] }) {
    const hafsWordCounts = textsToUse.hafs.map(wordStrategy.splitHafs)
    const imlaWordCounts = textsToUse.imla.map(wordStrategy.splitImla)
    const errors = []
    for (let i = 0; i < hafsWordCounts.length; i++) {
      if (hafsWordCounts[i].length !== imlaWordCounts[i].length) errors.push(i)
    }
    return errors
  },
  fixOneItem(index: number) {
    if (index === 2570) debugger
    const nbsp = '\u00A0'
    const hafsToSplit = hafsLikeText[index]
    const imlaToSplit = imlaLikeText[index]
      .replace(/ ۩/g, nbsp + '۩')
      .replace(/ ([ۖۛۗۜۚ])/g, nbsp + '$1')
      .replace(/۞ /g, '۞' + nbsp)
    const hafs = wordStrategy.splitHafs(hafsToSplit)
    const imla = wordStrategy.splitImla(imlaToSplit)
    const length = Math.max(hafs.length, imla.length)

    for (let i = 0; i < length; i++) {
      if (!hafs[i] || !imla[i]) continue
      if (!wordStrategy.match(hafs[i], imla[i])) {
        const hafsLength = alphabetStrategy.splitHafs(hafs[i]).length
        const imlaLength = alphabetStrategy.splitImla(imla[i]).length
        if (imlaLength < hafsLength) {
          imla[i] = imla[i] + nbsp + imla[i + 1]
          imla.splice(i + 1, 1)
        }
        // فقط ۵ مورد این شکلی داریم
        if (imlaLength > hafsLength) {
          // فقط دو مورد این شکلی داریم
          if (hafs[i].includes('ٱلّ')) continue
          // یک مورد این شکلی در رسم الخط کم‌علامت داریم
          if (hafs[i].includes('نُـۨجِي')) continue
          hafs[i] = hafs[i] + nbsp + hafs[i + 1]
          hafs.splice(i + 1, 1)
        }
      }
    }

    return { hafs, imla }
  },
  fix() {
    const fixedImlaText = [...imlaLikeText]
    const fixedHafsText = [...hafsLikeText]
    const errors = wordStrategy.test({ hafs: hafsLikeText, imla: imlaLikeText })

    errors.forEach((index) => {
      const fixResult = wordStrategy.fixOneItem(index)
      fixedHafsText[index] = fixResult.hafs.join(' ')
      fixedImlaText[index] = fixResult.imla.join(' ')
    })

    console.log('fix word', { fixedImlaText, fixedHafsText })

    return {
      imla: fixedImlaText,
      hafs: fixedHafsText,
    }
  },
  getText() {
    console.log('text word')
    if (autofix.value) {
      return wordStrategy.fix()
    }
    return {
      hafs: hafsLikeText,
      imla: imlaLikeText,
    }
  },
}

const alphabetStrategy = {
  splitHafs: (ayah: string): string[] =>
    Array.from((ayah || '').match(/\p{L}/gu) || []).filter(
      (item) => !item.match(/[وۛـۖؤءۗإآٱأائىۜیۚيۥۦ]/),
    ),
  splitImla: (ayah: string): string[] =>
    Array.from((ayah || '').match(/\p{L}/gu) || []).filter(
      (item) => !item.match(/[وۛـۖؤءۗإآٱأائىۜیۚيۥۦ]/),
    ),
  match(hafsItem: string, imlaItem: string) {
    return hafsItem === imlaItem
  },
  test(textsToUse: { hafs: string[]; imla: string[] }) {
    const hafsWordCounts = textsToUse.hafs.map(alphabetStrategy.splitHafs)
    const imlaWordCounts = textsToUse.imla.map(alphabetStrategy.splitImla)
    const errors = []
    for (let i = 0; i < hafsWordCounts.length; i++) {
      if (hafsWordCounts[i].length !== imlaWordCounts[i].length) {
        errors.push(i)
      }
    }
    return errors
  },
  fix() {
    return alphabetStrategy.getText()
  },
  getText() {
    return {
      hafs: hafsLikeText,
      imla: imlaLikeText,
    }
  },
}

const getStrategy = () =>
  mode.value === 'word' ? wordStrategy : alphabetStrategy

const texts = computed(() => getStrategy().getText())

const affectiveCountTestResult = computed(() => getStrategy().test(texts.value))

const expandCompare = ref({
  hafs: [] as string[],
  imla: [] as string[],
  index: -1,
})

function expand(i: number) {
  expandCompare.value = {
    hafs: getStrategy().splitHafs(texts.value.hafs[i]),
    imla: getStrategy().splitImla(texts.value.imla[i]),
    index: i,
  }
  previewAutofix(i)
}

function isItemError(i: number) {
  return !getStrategy().match(
    expandCompare.value.hafs[i] || '',
    expandCompare.value.imla[i] || '',
  )
}

const autofixPreviewCompare = ref({
  hafs: [] as string[],
  imla: [] as string[],
  index: -1,
})
function previewAutofix(index: number) {
  const strategy = getStrategy()
  if (strategy === alphabetStrategy) {
    autofixPreviewCompare.value = {
      hafs: [] as string[],
      imla: [] as string[],
      index: -1,
    }
    return
  }

  const { hafs, imla } = wordStrategy.fixOneItem(index)

  autofixPreviewCompare.value = { hafs, imla, index }
}
</script>

<template>
  <div>
    <div class="my-4">
      <label class="mr-3">
        <input type="radio" v-model="mode" value="word" />
        تفکیک واژه
      </label>
      <label>
        <input type="radio" v-model="mode" value="alphabet" />
        تفکیک حروف الفبا
      </label>
    </div>
    <div class="mb-4">
      <label>
        <input type="checkbox" v-model="autofix" />
        تلاش برای رفع اشکالات
      </label>
    </div>
    <div>تعداد آیات متفاوت: {{ affectiveCountTestResult.length }}</div>
    <div>
      <table v-if="affectiveCountTestResult.length">
        <thead>
          <th>ش</th>
          <th>عثمان‌طه</th>
          <th>رسم الاملاء</th>
        </thead>
        <tbody>
          <template v-for="index in affectiveCountTestResult">
            <tr @click="expand(index)">
              <td>{{ index }}</td>
              <td>{{ texts.hafs[index] }}</td>
              <td>{{ texts.imla[index] }}</td>
            </tr>
            <tr v-if="expandCompare.index === index">
              <td colspan="3" class="h-40 relative">
                <table class="absolute inset-0">
                  <tr>
                    <td
                      v-for="(item, i) in expandCompare.hafs"
                      :class="{ 'bg-red-500/15!': isItemError(i) }"
                    >
                      {{ item }}
                    </td>
                  </tr>
                  <tr>
                    <td
                      v-for="(item, i) in expandCompare.imla"
                      :class="{ 'bg-red-500/15!': isItemError(i) }"
                    >
                      {{ item }}
                    </td>
                  </tr>
                </table>
              </td>
            </tr>
            <tr v-if="!autofix && autofixPreviewCompare.index === index">
              <td colspan="3" class="h-40 relative">
                <table class="absolute inset-0">
                  <tr>
                    <td
                      v-for="(item, i) in autofixPreviewCompare.hafs"
                      :class="{ 'bg-red-500/15!': false }"
                    >
                      {{ item }}
                    </td>
                  </tr>
                  <tr>
                    <td
                      v-for="(item, i) in autofixPreviewCompare.imla"
                      :class="{ 'bg-red-500/15!': false }"
                    >
                      {{ item }}
                    </td>
                  </tr>
                </table>
              </td>
            </tr>
          </template>
        </tbody>
      </table>
      <div class="pt-1">موردی وجود ندارد.</div>
    </div>
  </div>
</template>
