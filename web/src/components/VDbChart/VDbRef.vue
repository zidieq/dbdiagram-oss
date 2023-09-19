<template>
  <g ref="root" :id="`ref-${id}`" :class="{
    'db-ref': true,
    'db-ref__highlight': (highlight || props.selfHighLight)
  }" @mouseenter.passive="onMouseEnter" @mouseleave.passive="onMouseLeave">
    <defs>
      <filter id="glow">
        <feGaussianBlur stdDeviation="3" result="glow" />
        <feMerge>
          <feMergeNode in="glow" />
          <feMergeNode in="SourceGraphic" />
        </feMerge>
      </filter>
    </defs>
    <path class="db-ref__hitbox" :d="path" />
    <path class="db-ref__path" :d="path" filter="url(#glow)" />
    <defs>
      <marker id="markerArrow" markerWidth="5" markerHeight="5" refx="2" refy="6" orient="auto">
        <path d="M2,2 L2,11 L10,6 L2,2" style="fill: #000000;" />
      </marker>
    </defs>
    <g class="db-ref__control-points">
      <!-- <circle v-for="(v, i) of controlPoints" :key="i" :cx="v.x" :cy="v.y" :class="{
        'db-ref__control-point': true,
        'db-ref__control-point__highlight': i === controlPoint_highlighted,
        'db-ref__control-point__dragging': i === controlPoint_dragging,
      }" :data-id="i" @dblclick.passive="controlPoint_onDblClick" @mousedown.passive="controlPoint_startDrag"
        @mouseenter.passive="controlPoint_onMouseEnter" @mouseleave.passive="controlPoint_onMouseLeave" /> -->
    </g>

  </g>
</template>

<style scoped>
/* .db-ref__path {
  stroke-dasharray: 15 2;
  animation: move 4s linear infinite;

}

@keyframes move {
  0% {
    stroke-dashoffset: 0;
  }

  100% {
    stroke-dashoffset: 300;
  }
} */
</style>

<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, onUpdated, reactive, ref, watch, watchEffect } from 'vue'
import { useChartStore } from '../../store/chart'
import { snap } from '../../utils/MathUtil'

const props = defineProps({
  id: Number,
  name: String,
  endpoints: Array,
  onUpdate: [String, Object, undefined],
  onDelete: [String, Object, undefined],
  schema: Object,
  dbState: Object,
  database: Object,
  token: Object,
  containerRef: Object,
  selfHighLight: Boolean
})

const store = useChartStore()
let s = store.getRef(props.id)

const gridSize = store.subGridSize
const gridSnap = store.grid.snap

const highlight = ref(false)
const affectedTables = ref([])
const d = ref('')

const getPositionAnchors = (endpoint) => {
  const s = store.getTable(endpoint.fields[0].table.id)
  const fieldIndex = endpoint.fields[0].table.fields.findIndex(f => f.id === endpoint.fields[0].id)

  return [
    {
      x: s.x,
      y: s.y + 25 + (20 * fieldIndex) + (20 / 2.0)
    },
    {
      x: s.x + s.width,
      y: s.y + 25 + (20 * fieldIndex) + (20 / 2.0)
    }
  ]
}

const getClosestAnchor = (point, anchors) => {
  const withDistances = anchors.map(a => ({
    distanceXY: {
      x: (a.x - point.x),
      y: (a.y - point.y)
    },
    distance: Math.sqrt(
      ((a.x - point.x) * (a.x - point.x))
      + ((a.y - point.y) * (a.y - point.y))
    ),
    anchor: a
  })
  )

  let smallest = withDistances[0]
  for (const withDistance of withDistances) {
    if (withDistance.distance < smallest.distance) {
      smallest = withDistance
    }
  }

  return smallest.anchor
}
/**
 * 两个参数里面分别有两个点，该函数会找出两个参数里面相距最近的点
 * @param {Array} anchorsA 
 * @param {Array} anchorsB 
 */
const getClosest = (anchorsA, anchorsB) => {
  const withDistances = anchorsA.flatMap(a => anchorsB.map(b => ({
    distanceXY: {
      x: (a.x - b.x),
      y: (a.y - b.y)
    },
    distance: Math.sqrt(
      ((a.x - b.x) * (a.x - b.x))
      + ((a.y - b.y) * (a.y - b.y))
    ),
    a: a,
    b: b
  })
  ))
  let smallest = withDistances[0]
  for (const withDistance of withDistances) {
    if (withDistance.distance < smallest.distance) {
      smallest = withDistance
    }
  }

  return [smallest.a, smallest.b]
}

const startAnchors = computed(() => {
  return getPositionAnchors(props.endpoints[0])
})
const endAnchors = computed(() => {
  return getPositionAnchors(props.endpoints[1])
})

const controlPoints = computed(() => {
  if (!s.vertices.length || s.vertices.some(v => Number.isNaN(v.x) || Number.isNaN(v.y))) {
    updateControlPoints()
  }
  if (!s.vertices.length || s.vertices.some(v => Number.isNaN(v.x) || Number.isNaN(v.y))) {
    return []
  }
  return s.vertices
})

const updateControlPoints = ({startPointArray,endPointArray}={}) => {
  const startElAnchors = startAnchors.value || startPointArray;
  const endElAnchors = endAnchors.value || endPointArray;

  if (!s.vertices.length || s.vertices.some(v => Number.isNaN(v.x) || Number.isNaN(v.y))) {
    s.auto = true
  } else if (!s.auto) {
    return
  }

  // 用于确定起始点，起始与终止字段的可连接点都有两个，这里用来确定最终使用这两个字段的哪一个点用以进行连接
  let [start, end] = getClosest(startElAnchors, endElAnchors)
  console.log('updateControlPoints', start, end, startElAnchors, endElAnchors)
  // 这里是计算两个字段之间连线的中间点的坐标的，其实最重要的是中间点的x坐标的确定，因为y坐标与起点、终点的y是一致的
  const minX = Math.min(start.x, end.x)
  const minY = Math.min(start.y, end.y)
  const maxX = Math.max(start.x, end.x)
  const maxY = Math.max(start.y, end.y)
  const midX = (minX + (((maxX - minX) || 2) / 2))
  const midY = (minY + (((maxY - minY) || 2) / 2))
  let mid = {
    x: midX,
    y: midY
  }
  // 以上是原来的计算方法，
  // --------------------------------------------------------------------------------------------------
  // 这里把两个字段的左边的x、右边的x都放到同一个数组里面，然后自己进行中间点X的选择
  // 中间点离表本体的最小距离，防止连线的中间的竖线贴到表格上
  const separatorLength = 20;
  // [开头字段的左边点的x, 开头字段的右边点的x, 结尾字段的左边点的x, 结尾字段的右边点的x]
  const loc = [startElAnchors[0].x - separatorLength, startElAnchors[1].x + separatorLength, endElAnchors[0].x - separatorLength, endElAnchors[1].x + separatorLength]
  if (loc[1] <= loc[2]) {
    mid.x = (loc[1]+loc[2])/2
  } else if (loc[1] > loc[2] && loc[3] > loc[1]) {
    mid.x = loc[3];
    start = startElAnchors[1];
    end = endElAnchors[1];
  } else if (loc[1] >= loc[2] && loc[3] <= loc[1] && loc[2] > loc[0]) {
    if (loc[2] - loc[0] > loc[1] - loc[3]) {
      mid.x = loc[1];
      start = startElAnchors[1];
    } else {
      mid.x = loc[0];
      start = startElAnchors[0];
    }
    end = endElAnchors[1];
  } else if (loc[0] >= loc[2] && loc[1] >= loc[3] && loc[3] > loc[0]) {
    mid.x = loc[2];
    start = startElAnchors[0];
    end = endElAnchors[0];
  } else if (loc[3] < loc[0]) {
    mid.x = (loc[3]+loc[0]) / 2;
  }

  // --------------------------------------------------------------------------------------------------


  s.vertices = [
    {
      x: mid.x,
      y: start.y
    },
    {
      x: mid.x,
      y: end.y
    }
  ]
  console.log('linePoint:',s.vertices)
}

const path = computed(() => {
  const startElAnchors = startAnchors.value
  const endElAnchors = endAnchors.value

  const points = s.vertices
  if (points.length == 0 || points.some(p => Number.isNaN(p.x) || Number.isNaN(p.y))) return ``
  const start = getClosestAnchor(points[0], startElAnchors)
  const end = getClosestAnchor(points[points.length - 1], endElAnchors)
  // 直线相连
  // const html = `M ${start.x},${start.y} L ${points.map(p => (`${p.x},${p.y}`)).join(' ')} L ${end.x} ${end.y}`;
  // 曲线相连
  const html = `M ${start.x},${start.y} C ${points.map(p => (`${p.x},${p.y}`)).join(' ')}  ${end.x} ${end.y}`;
  return html;
})

const onMouseEnter = (e) => {
  highlight.value = true
}
const onMouseLeave = (e) => {
  highlight.value = false
}

const controlPoint_highlighted = ref(null)
const controlPoint_dragging = ref(null)
const controlPoint_dragOffset = reactive({
  x: 0,
  y: 0
})

const controlPoint_onMouseEnter = ({ target }) => {
  const controlPointId = Number(target.getAttribute('data-id'))
  controlPoint_highlighted.value = controlPointId
}
const controlPoint_onMouseLeave = ({ target }) => {
  controlPoint_highlighted.value = null
  controlPoint_highlighted.value = null
}
const controlPoint_startDrag = ({
  target,
  offsetX,
  offsetY
}) => {
  const controlPointId = Number(target.getAttribute('data-id'))
  const v = s.vertices[controlPointId]

  controlPoint_dragging.value = controlPointId

  const p = store.inverseCtm.transformPoint({
    x: offsetX,
    y: offsetY
  })

  controlPoint_dragOffset.x = p.x - v.x
  controlPoint_dragOffset.y = p.y - v.y
  props.containerRef.addEventListener('mousemove', controlPoint_drag, { passive: true })
  props.containerRef.addEventListener('mouseup', controlPoint_drop, { passive: true })
  props.containerRef.addEventListener('mouseleave', controlPoint_onMouseLeave, { passive: true })
}
const controlPoint_drag = ({
  target,
  offsetX,
  offsetY
}) => {
  const p = store.inverseCtm.transformPoint({
    x: offsetX,
    y: offsetY
  })
  const controlPointId = controlPoint_dragging.value
  if (s.auto) {
    s.auto = false
  }

  const v = s.vertices[controlPointId]
  v.x = snap((p.x - controlPoint_dragOffset.x), gridSnap)
  v.y = snap((p.y - controlPoint_dragOffset.y), gridSnap)

}
const controlPoint_drop = (e) => {
  controlPoint_dragOffset.x = 0
  controlPoint_dragOffset.y = 0
  controlPoint_dragging.value = null
  controlPoint_highlighted.value = null

  props.containerRef.removeEventListener('mousemove', controlPoint_drag, { passive: true })
  props.containerRef.removeEventListener('mouseup', controlPoint_drop, { passive: true })
  props.containerRef.removeEventListener('mouseleave', controlPoint_onMouseLeave, { passive: true })
}

const controlPoint_onDblClick = ({ target }) => {
  s.auto = true;
  updateControlPoints()
}

onMounted(() => {
  affectedTables.value = props.endpoints.map(e => store.getTable(e.fields[0].table.id))
  nextTick(() => {
    updateControlPoints()
  })
})

watch(() => props.id, (newId) => {
  s = store.getRef(newId)
})

watch(props.endpoints, () => {
  affectedTables.value = props.endpoints.map(e => store.getTable(e.fields[0].table.id))
}, {
  deep: true
})

watch(affectedTables, () => {
  updateControlPoints()
}, {
  deep: true
})

</script>
