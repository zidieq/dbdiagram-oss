<template>
  <svg
    ref="root"
    class="db-chart"
    @mousemove.passive.capture="updateCursorPosition"
  >
    <defs>
      <pattern id="db-chart__bg-grid-base"
               :width="bgGrid.pattern.width"
               :height="bgGrid.pattern.height"
               patternUnits="userSpaceOnUse"
               :viewBox="`0 0 ${bgGrid.pattern.width} ${bgGrid.pattern.height}`"
               class="db-chart__bg-grid"
               ref="bgGridRect">
        <g class="db-chart__bg-grid-small">
          <path :d="bgGrid.pattern.path" fill="none"/>
        </g>
        <path :d="`M ${bgGrid.pattern.width} 0 L 0 0 0 ${bgGrid.pattern.height}`" fill="none"/>
      </pattern>

      <pattern id="db-chart__bg-grid"
               x="0" y="0"
               :width="bgGrid.pattern.width"
               :height="bgGrid.pattern.height"
               patternUnits="userSpaceOnUse"
               :viewBox="`${bgGrid.pattern.x} ${bgGrid.pattern.y} ${bgGrid.pattern.width} ${bgGrid.pattern.height}`">
        <rect
          :x="`-${bgGrid.pattern.width}`"
          :y="`-${bgGrid.pattern.height}`"
          :width="`${bgGrid.pattern.width*3}`"
          :height="`${bgGrid.pattern.height*3}`"
          fill="url(#db-chart__bg-grid-base)"/>
      </pattern>
    </defs>

    <g id="background-layer">
      <rect ref="bgRef" class="db-chart__bg"
            @mousedown="panZoom.enablePan()"
            @mouseup="panZoom.disablePan()"
      />
      <rect class="db-chart__bg-grid"
            x="0" y="0"
            width="100%" height="100%"
            fill="url(#db-chart__bg-grid)"/>
    </g>
    <g id="viewport-layer">
      <g id="tablegroups-layer"
         v-if="store.loaded">
        <v-db-table-group v-for="tableGroup of tableGroups"
                          :key="tableGroup.id"
                          v-bind="tableGroup"
                          :container-ref="root"
                          @click.passive="dblclickHelper(onTableGroupDblClick, $event, tableGroup)"
                          @mouseenter.passive="onTableGroupMouseEnter"
                          @mouseleave.passive="onTableGroupMouseLeave"
        >

        </v-db-table-group>
      </g>
      <g id="refs-layer"
         v-if="store.loaded">
        <v-db-ref v-for="ref of refs"
                  :key="ref.id"
                  v-bind="ref"
                  :container-ref="root"
                  @click.passive="dblclickHelper(onRefDblClick, $event, ref)"
                  @mouseenter.passive="onRefMouseEnter"
                  @mouseleave.passive="onRefMouseLeave"
        />
      </g>
      <g id="tables-layer"
         v-if="store.loaded">
        <v-db-table v-for="table of tables"
                    v-bind="table"
                    :key="table.id"
                    :container-ref="root"
                    @click:header="dblclickHelper(onTableDblClick, $event, table)"
                    @click:field="(...e) => dblclickHelper(onFieldDblClick, ...e)"
                    @mouseenter.passive="onTableMouseEnter($event,table)"
                    @mouseleave.passive="onTableMouseLeave"
        />
      </g>
      <g id="overlays-layer"
         v-if="store.loaded">
        <v-db-tooltip/>
      </g>
    </g>
    <g id="tools-layer">
      <svg x="10" y="10" width="150" height="36" class="db-tools">
        <rect class="db-tools__bg"/>
        <text x="0" class="db-tools__header">position</text>
        <text x="0">x:
          <v-number :value="position.x" decimals="1"/>
        </text>
        <text x="75">y:
          <v-number :value="position.y" decimals="1"/>
        </text>
      </svg>

      <svg x="170" y="10" width="150" height="36" class="db-tools">
        <rect class="db-tools__bg"/>
        <text x="0" class="db-tools__header">pan</text>
        <text x="0">x:
          <v-number :value="store.pan.x" decimals="1"/>
        </text>
        <text x="75">y:
          <v-number :value="store.pan.y" decimals="1"/>
        </text>
      </svg>

      <svg x="330" y="10" width="100" height="36" class="db-tools">
        <rect class="db-tools__bg"/>
        <text x="0" class="db-tools__header">zoom</text>
        <text x="0">
          <v-number :value="store.zoom" decimals="3"/>
        </text>
      </svg>
    </g>
  </svg>
</template>

<script setup>
  import { computed, nextTick, onMounted, reactive, ref, watch, watchEffect } from 'vue'
  import VDbTable from './VDbTable'
  import VDbRef from './VDbRef'
  import svgPanZoom from 'svg-pan-zoom'
  import { useChartStore } from '../../store/chart'
  import VDbTooltip from './VDbTooltip'
  import VDbTableGroup from './VDbTableGroup'

  const store = useChartStore()

  const props = defineProps({
    tableGroups: {
      type: Array,
      default: () => ([])
    },
    tables: {
      type: Array,
      default: () => ([])
    },
    refs: {
      type: Array,
      default: () => ([])
    }
  })

  const emit = defineEmits([
    'dblclick:table-group',
    'dblclick:table',
    'dblclick:ref',
    'dblclick:field',
  ])

  const root = ref(null)
  const bgGrid2 = ref(null)
  const bgGridRect = ref(null)

  const bgGrid = reactive({
    pattern: {
      viewport: {
        x: 0,
        y: 0,
        width: 100,
        height: 100
      },
      rect: {
        x: -100,
        y: -100,
        width: 300,
        height: 300
      },
      path: '',
      x: 0,
      y: 0,
      width: 100,
      height: 100
    },
    offset: {
      x: 0,
      y: 0
    }

  })
  const panZoom = ref({})
  const position = reactive({
    x: 0,
    y: 0
  },)
  let initialized = false

  const updateCursorPosition = (e) => {
    const p = store.inverseCtm.transformPoint({
      x: e.offsetX,
      y: e.offsetY
    })
    position.x = p.x
    position.y = p.y
  }

  const saveSizes = () => {
    const s = panZoom.value.getSizes()
    const p = panZoom.value.getPan()
    const z = panZoom.value.getZoom()
    const pan = {
      x: p.x - (s.width / 2),
      y: p.y - (s.height / 2)
    }
    store.$patch({
      pan: pan,
      zoom: z
    })
  }

  const loadSizes = () => {
    const s = panZoom.value.getSizes()
    const p = store.pan
    const z = store.zoom
    const pan = {
      x: p.x,
      y: p.y
    }
    panZoom.value.resize()
    panZoom.value.center()
    panZoom.value.zoom(z)
    panZoom.value.panBy(pan)
  }

  function updateGrid (matrix) {
    let p = ''
    const {
      size: c,
      divisions: d
    } = store.grid
    const e = c / d

    const restrainedMatrix = DOMMatrix.fromMatrix(matrix)
    const minPos = restrainedMatrix.transformPoint({
      x: 0,
      y: 0
    })
    const maxPos = restrainedMatrix.transformPoint({
      x: c,
      y: c
    })

    const cx = Math.abs(maxPos.x - minPos.x)
    const cy = Math.abs(maxPos.y - minPos.y)
    const dx = cx / d
    const dy = cy / d

    const tx = minPos.x
    const ty = minPos.y
    const mx = ((tx % cx) + cx) % cx
    const my = ((ty % cy) + cy) % cy

    p += 'M 0 0'
    for (let i = 1; i < d; i++) {
      p += ` m ${dx * i} 0 l 0 ${cy} m -${dx * i} -${cy}`
    }
    p += 'M 0 0'
    for (let i = 1; i < d; i++) {
      p += ` m 0 ${dy * i} l ${cx} 0 m -${cx} -${dy * i}`
    }

    bgGrid.pattern.x = -mx
    bgGrid.pattern.y = -my
    bgGrid.pattern.width = cx
    bgGrid.pattern.height = cy
    bgGrid.pattern.path = p
  }

  const updateCTM = (newCTM) => {
    store.updateCTM(newCTM)
    updateGrid(newCTM)
  }

  const updateZoom = () => {
    saveSizes()

  }

  onMounted(() => {
    panZoom.value = svgPanZoom(root.value, {
      viewportSelector: '#viewport-layer',
      panEnabled: false,
      fit: false,
      center: false,
      dblClickZoomEnabled: false,
      zoomScaleSensitivity: 0.2,
      minZoom: 0.1,
      maxZoom: 2.0,
      // onPan: (newPan) => {
      //   saveSizes()
      // },
      // onZoom: (newZoom) => {
      //   saveSizes()
      // },
      // onUpdatedCTM: (newCTM) => {
      //   store.updateCTM(newCTM)
      // }
    })
    nextTick(() => {
      loadSizes()
      panZoom.value.disablePan()
      panZoom.value.setOnPan(() => saveSizes())
      panZoom.value.setOnZoom(() => updateZoom())
      panZoom.value.setOnUpdatedCTM((newCTM) => updateCTM(newCTM))
    })
    initialized = true
  })

  watch(() => props.tables, () => {
    panZoom.value.updateBBox()
  })

  watch(() => props.refs, () => {
    panZoom.value.updateBBox()
  })

  watch(() => store.zoom, (newZoom) => {
    panZoom.value.zoom(newZoom)
  })

  function onRefDblClick (e, ref) {
    console.log("onRefDblClick", e, ref);
    emit('dblclick:ref', e, ref);
  }
  function onFieldDblClick (e, field) {
    console.log("onFieldDblClick", e, field);
    emit('dblclick:field', e, field);
  }
  function onTableDblClick (e, table) {
    console.log("onTableDblClick", e, table);
    emit('dblclick:table', e, table);
  }
  function onTableGroupDblClick (e, tableGroup) {
    console.log("onTableGroupDblClick", e, tableGroup);
    emit('dblclick:table-group', e, tableGroup);
  }

  function onRefMouseEnter (e) {
    e.target.parentElement.appendChild(e.target)
  }

  function onRefMouseLeave (e) {
  }

  function onTableMouseEnter (e,curTable) {
    console.log('--------------------------------------------')
    // console.log('tables')
    // console.dir(props.tables)
    // console.log('refs')
    // console.dir(props.refs)
    // console.log('e')
    // console.dir(e);
    console.log('table')
    console.dir(curTable)
    window.myTable = curTable;
    console.log('--------------------------------------------')
    e.target.parentElement.appendChild(e.target)
  }

  function onTableMouseLeave (e) {
  }

  function onTableGroupMouseEnter (e) {
    e.target.parentElement.appendChild(e.target)
  }

  function onTableGroupMouseLeave (e) {
  }

  let lastClick = Date.now();
  let lastClicked = null;
  function dblclickHelper(fn, e, ...args) {
    console.log("dblclickHelper", e, ...args)
    const nowClick = Date.now();

    if (((nowClick - lastClick) < 500) && lastClicked === e.target) {
      console.log("dblclickHelperYES", e, ...args)
      fn(e, ...args);
    }
    lastClicked = e.target;
    lastClick = nowClick;
  }

</script>
