<script setup lang="ts">
import { type PropType } from 'vue'
import NexonI18nDataOutput from '@/components/genetic/NexonI18nDataOutput.vue'
import type { NexonL10nData } from '@/types/OutsourcedData'
import { checkIfScenarioIdIsMain, getScenarioExtraDataById } from '@/tool/StoryTool'
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

const exportScenario = async () => {
  try {
    // Fetch scenario data
    const response = await fetch(`/data/story/normal/${props.data.id}.json`)
    if (!response.ok) {
      throw new Error(`Failed to fetch scenario data: ${response.statusText}`)
    }
    const scenarioData = await response.json()
    
    // Create export content
    const exportContent = {
      id: props.data.id,
      name: props.data.name,
      description: props.data.desc,
      actualScenarioNo: scenarioIdExtraData.actualScenarioNo,
      isAfterBattle: scenarioIdExtraData.isAfterBattle,
      isAfterBattleFlag: scenarioIdIsAfterBattleFlag,
      data: scenarioData
    }
    
    // Convert to JSON string
    const jsonString = JSON.stringify(exportContent, null, 2)
    
    // Create blob and download
    const blob = new Blob([jsonString], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = `${props.data.id}_scenario.json`
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    URL.revokeObjectURL(url)
  } catch (error) {
    console.error('Error exporting scenario:', error)
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
    <PvButton
      severity="secondary"
      size="small"
      @click="exportScenario"
    >
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
