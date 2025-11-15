<script setup lang="ts">
import { type PropType } from 'vue'
import NexonI18nDataOutput from '@/components/genetic/NexonI18nDataOutput.vue'
import type { NexonL10nData } from '@/types/OutsourcedData'
import { checkIfScenarioIdIsMain, getScenarioExtraDataById } from '@/tool/StoryTool'
import { httpGetAsync } from '@/tool/HttpRequest'
import { getScenarioDataEntryCharName } from '@/script/ScenarioUiMt'
import { i18nLangAll } from '@/tool/ConstantComputed'
import { NexonLangMap } from '@/tool/Constant'
import ScenarioIsAfterBattleBadge from '@/components/genetic/ScenarioIsAfterBattleBadge.vue'
import { useSetting } from '@/stores/setting'
import { useI18n } from 'vue-i18n'

import PvTag from 'primevue/tag'
import PvButton from 'primevue/button'
import PvDivider from 'primevue/divider'

const setting = useSetting()
const { t } = useI18n()

const props = defineProps({
  data: {
    type: {} as PropType<{ id: string; name: NexonL10nData; desc: NexonL10nData }>,
    required: true
  },
  dataMt: {
    type: {} as PropType<Record<'name' | 'desc', NexonL10nData>>,
    required: true
  },
  data_no: {
    type: Number,
    required: true
  }
})
// const storyId = computed(() => props.data.id)
// const storyName = computed(() => props.data.name)
// const storyDesc = computed(() => props.data.desc)

const isScenarioMain = checkIfScenarioIdIsMain(props.data.id)
let scenarioIdExtraData: ReturnType<typeof getScenarioExtraDataById>,
  scenarioIdIsAfterBattleFlag: 'A' | 'B'
if (isScenarioMain || (props.data.id.startsWith('1100') && props.data.id.length === 5)) {
  scenarioIdExtraData = getScenarioExtraDataById(props.data.id)
  scenarioIdIsAfterBattleFlag = !scenarioIdExtraData.isAfterBattle ? 'A' : 'B'
} else {
  scenarioIdExtraData = { isAfterBattle: false, actualScenarioNo: props.data_no }
  scenarioIdIsAfterBattleFlag = 'A'
}

const exportScript = async () => {
  try {
    const storyId = props.data.id
    const setting = useSetting()

    const selectedLangs = i18nLangAll.value.filter((lang) => lang !== 'null')
    const targetLang = selectedLangs.length > 0 ? selectedLangs[0] : 'c_cn'
    const langFallbacks = ['c_cn', 'g_tw_cn', 'g_tw', 'j_ja', 'g_en'];
    const uiLocale = NexonLangMap[targetLang] || 'zh_CN';

    const getBestAvailableText = (textObject: Record<string, string> | undefined) => {
      if (!textObject) return '';
      let text = textObject[targetLang]
      if (text && !text.includes('not found') && !text.includes('LocalizeError')) {
        return text;
      }
      for (const lang of langFallbacks) {
        text = textObject[lang]
        if (text && !text.includes('not found') && !text.includes('LocalizeError')) {
          return text;
        }
      }
      return '';
    }

    const narratorTranslations: Record<string, string> = {
        en: 'Narrator',
        zh_CN: '旁白',
        zh_TW: '旁白',
        ja: 'ナレーション',
        ko: '내레이션',
        th: 'ผู้บรรยาย',
    };
    const titleTranslations: Record<string, string> = {
        en: 'Title',
        zh_CN: '标题',
        zh_TW: '標題',
        ja: 'タイトル',
        ko: '제목',
        th: 'ชื่อเรื่อง',
    };

    const responseText = await httpGetAsync(`/data/story/normal/${storyId}.json`)
    const storyData = JSON.parse(responseText)

    const firstTitleEntry = storyData.find((e: any) => e.DataType === 'title');
    let storyTitle = '';
    if (firstTitleEntry) {
        storyTitle = getBestAvailableText(firstTitleEntry.Message);
    }
    if (!storyTitle) {
        storyTitle = getBestAvailableText(props.data.name);
    }

    const cleanDialogue = (text: string) => {
      if (!text) return ''
      return text
        .replace(/\[USERNAME\]/g, setting.username)
        .replace(/\[\\n\]/g, '\n')
        .replace(/\[img:.*?\]/g, '')
        .replace(/<rt>.*?<\/rt>|<rp>.*?<\/rp>/g, '') // Remove ruby annotations
        .replace(/<[^>]*>/g, '') // Remove all other HTML tags
        .trim()
    }

    const narratorText = narratorTranslations[uiLocale] || narratorTranslations['zh_CN'];
    const titleText = titleTranslations[uiLocale] || titleTranslations['zh_CN'];
    let scriptText = `${titleText}: ${cleanDialogue(storyTitle)}\n\n`

    for (const entry of storyData) {
      if (['cmd', 'video', 'title'].includes(entry.DataType)) continue
      let dialogue = cleanDialogue(getBestAvailableText(entry.Message))
      if (!dialogue) continue

      if (entry.SelectionGroup !== 0) {
        dialogue += ` (SeleGroup: ${entry.SelectionGroup})`
      }
      if (entry.SelectionToGroup !== -1) {
        dialogue += ` (SeleToGroup: ${entry.SelectionToGroup})`
      }

      if (entry.DataType === 'speaker') {
        const charInfo = getScenarioDataEntryCharName(entry)
        let speaker = getBestAvailableText(charInfo.Name)
        if (!speaker) speaker = narratorText
        scriptText += `${speaker}: ${dialogue}\n`
      } else if (entry.DataType === 'option') {
        scriptText += `${setting.username}: ${dialogue}\n`
      } else if (['na', 'st', 'stm', 'place'].includes(entry.DataType)) {
        scriptText += `${narratorText}: ${dialogue}\n`
      } else {
        scriptText += `${dialogue}\n`
      }
    }

    const sanitizedTitle = (storyTitle ? cleanDialogue(storyTitle) : 'scenario').replace(
      /[\\?%*:|"<>]/g,
      '-'
    )
    const blob = new Blob([scriptText], { type: 'text/plain;charset=utf-8' })
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = `${storyId}_${sanitizedTitle}.txt`
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    URL.revokeObjectURL(url)
  } catch (error) {
    console.error('Error exporting script:', error)
    alert(t('export-error'))
  }
}
</script>

<template>
  <h3>
    <PvTag severity="warn">{{ data.id }}</PvTag>
    <span v-if="isScenarioMain">&nbsp;</span>
    <ScenarioIsAfterBattleBadge :story-id="data.id" />
    <span>&nbsp;&nbsp;</span>
    <span
      >{{ scenarioIdExtraData.actualScenarioNo
      }}<span v-if="isScenarioMain">{{ '-' + scenarioIdIsAfterBattleFlag }}</span
      >.&nbsp;</span
    >
    <NexonI18nDataOutput :data="data.name" :data-mt="dataMt.name" />
    <span>&nbsp;</span>
    <PvButton
      severity="primary"
      class="btn-view-story"
      size="small"
      as="RouterLink"
      :to="`/scenario/${data.id}`"
    >
      {{ $t('comp-search-result-btn-view') }}
    </PvButton>
    <span>&nbsp;</span>
    <PvButton severity="secondary" size="small" @click="exportScript">
      {{ $t('comp-search-result-btn-export') }}
    </PvButton>
  </h3>
  <div v-show="setting.show_story_desc">
    <p>{{ $t('comp-search-result-desc') }}</p>
    <ul>
      <NexonI18nDataOutput
        :data="data.desc"
        :data-mt="dataMt.desc"
        html-element-name="li"
        :enable-text-line-clamp="true"
      />
    </ul>
  </div>
  <PvDivider></PvDivider>
</template>

<style scoped>
.btn-view-story {
  text-decoration-line: none;
}
</style>
