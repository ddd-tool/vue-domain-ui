<script setup lang="ts">
import nomnoml from 'nomnoml'
import { computed, ref, watchEffect } from 'vue'
import style from './style'
import { useDiagramAgg } from '#lib/domain/diagram-agg'

const diagramAgg = useDiagramAgg()

const svgCode = computed(() => {
  return nomnoml.renderSvg(style + diagramAgg.states.code.value)
})

// ======================= focusOnWorkFlow/playWorkflow =======================
let currentAnimationTask = 0
function startWorkflowAnimation(
  animationTask: number,
  nodes: readonly string[]
) {
  diagramAgg.commands.setDownloadEnabled(false)
  let t = 0
  for (let i = 0; i < nodes.length; i++) {
    const el = document.querySelector(
      `[data-name="${nodes[i]}"]`
    ) as SVGGElement
    setTimeout(() => {
      if (animationTask !== currentAnimationTask) return
      el.style.transition = `opacity 0s`
      el.style.opacity = '0.0'
      setTimeout(() => {
        if (animationTask !== currentAnimationTask) return
        el.style.transition = `opacity 0.5s`
        el.style.opacity = '1'
        if (i === nodes.length - 1) {
          diagramAgg.commands.setDownloadEnabled(true)
        }
      }, 100)
    }, t)
    t += 500
  }
}
diagramAgg.events.onFocusFlow.watchPublish(({ data }) => {
  const idMap = diagramAgg.commands.getIdMap() as Record<
    string,
    {
      _attributes: {
        __code: string
        rule: string
      }
    }
  >
  let items: readonly string[] =
    data.workflow === null ? [] : diagramAgg.states.workflows[data.workflow]
  items = items.filter((i) => {
    let b = true
    if (!data.displayReadModel && idMap[i]._attributes.rule === 'ReadModel') {
      b = false
    } else if (!data.displaySystem && idMap[i]._attributes.rule === 'System') {
      b = false
    }
    return b
  })
  if (!data.displayReadModel || !data.displaySystem) {
    items = removeAdjacentDuplicates(items)
  }
  const map: Record<string, boolean> = {}

  const doms = document.querySelectorAll('g')
  if (items.length === 0) {
    for (const dom of doms) {
      dom.style.transition = 'opacity 0s'
      dom.style.opacity = '1'
    }
    return
  }
  for (const dom of doms) {
    const dataName = dom.dataset.name
    if (!dataName || map[dataName]) {
      continue
    }
    dom.style.transition = 'opacity 0s'
    dom.style.opacity = '0.1'
    map[dataName] = true
  }
  setTimeout(() => {
    startWorkflowAnimation(++currentAnimationTask, items)
  })
})
function removeAdjacentDuplicates(arr: readonly string[]): string[] {
  if (arr.length === 0) return []
  const result: string[] = [arr[0]]
  for (let i = 1; i < arr.length; i++) {
    if (arr[i] !== arr[i - 1]) {
      result.push(arr[i])
    }
  }
  return result
}

// ============================ 下载 ============================
diagramAgg.events.onDownloadSvg.watchPublish(() => {
  const el = document.querySelector('svg') as SVGSVGElement
  const svg = new XMLSerializer().serializeToString(el)
  const svgBlob = new Blob([svg], { type: 'image/svg+xml;charset=utf-8' })
  const svgUrl = URL.createObjectURL(svgBlob)
  const tempLink = document.createElement('a')
  tempLink.href = svgUrl
  tempLink.setAttribute('download', 'diagram.svg')
  tempLink.click()
  URL.revokeObjectURL(svgUrl)
})
</script>

<template>
  <div v-html="svgCode"></div>
</template>
