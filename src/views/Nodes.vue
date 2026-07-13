<template>
  <div class="nodes-page">
    <v-row class="mb-4" align="stretch">
      <v-col cols="12" lg="8">
        <v-card class="hero-card" rounded="xl" elevation="8">
          <v-card-text>
            <div class="d-flex flex-wrap align-center justify-space-between ga-4">
              <div>
                <div class="text-overline hero-kicker">{{ $t('node.cluster') }}</div>
                <h1 class="hero-title">{{ $t('pages.nodes') }}</h1>
                <p class="hero-subtitle">{{ $t('node.subtitle') }}</p>
              </div>
              <div class="d-flex ga-2 flex-wrap">
                <v-btn color="primary" prepend-icon="mdi-plus" @click="showEditor()">
                  {{ $t('actions.add') }}
                </v-btn>
                <v-btn variant="tonal" prepend-icon="mdi-refresh" :loading="refreshing" @click="refreshAllNodes">
                  {{ $t('node.refreshAll') }}
                </v-btn>
                <v-btn color="warning" variant="tonal" prepend-icon="mdi-cloud-upload" :loading="pushing" @click="pushConfigToAll">
                  {{ $t('node.syncConfig') }}
                </v-btn>
              </div>
            </div>
          </v-card-text>
        </v-card>
      </v-col>
      <v-col cols="12" lg="4">
        <v-card class="summary-card" rounded="xl" elevation="6">
          <v-card-text>
            <div class="d-flex align-center justify-space-between mb-4">
              <span class="text-subtitle-2">{{ $t('node.summary') }}</span>
              <v-icon icon="mdi-lan" color="primary" />
            </div>
            <v-row dense>
              <v-col cols="4">
                <div class="summary-number">{{ nodes.length }}</div>
                <div class="summary-label">{{ $t('node.total') }}</div>
              </v-col>
              <v-col cols="4">
                <div class="summary-number text-success">{{ onlineCount }}</div>
                <div class="summary-label">{{ $t('node.online') }}</div>
              </v-col>
              <v-col cols="4">
                <div class="summary-number text-info">{{ syncedCount }}</div>
                <div class="summary-label">{{ $t('node.configSynced') }}</div>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>

    <v-card rounded="xl" elevation="4">
      <v-data-table
        :headers="tableHeaders"
        :items="nodes"
        :items-per-page="-1"
        class="nodes-table"
        hover
      >
        <template v-slot:item.drag="{ item }">
          <v-icon
            icon="mdi-drag"
            class="drag-handle"
            :class="{ 'dragging': draggingIndex === findNodeIndex(item.id) }"
            draggable="true"
            @dragstart="onDragStart($event, item)"
            @dragover.prevent="onDragOver($event, item)"
            @dragleave="onDragLeave"
            @drop.prevent="onDrop($event, item)"
            @dragend="onDragEnd"
          />
        </template>

        <template v-slot:item.enabled="{ item }">
          <v-switch
            v-model="item.enabled"
            hide-details
            density="compact"
            color="primary"
            @update:model-value="toggleNodeEnabled(item)"
            class="ma-0"
            style="width: 48px;"
          />
        </template>

        <template v-slot:item.status="{ item }">
          <v-chip :color="statusColor(item.status)" size="small" variant="flat">
            <v-icon start size="x-small" :icon="statusIcon(item.status)" />
            {{ statusText(item.status) }}
          </v-chip>
        </template>

        <template v-slot:item.name="{ item }">
          <div class="d-flex align-center ga-2">
            <v-icon size="small" :icon="item.type === 'local' ? 'mdi-home-variant' : 'mdi-server-network'" />
            <strong>{{ item.name }}</strong>
          </div>
        </template>

        <template v-slot:item.type="{ item }">
          <v-chip size="x-small" :color="item.type === 'local' ? 'info' : 'primary'" variant="tonal">
            {{ item.type }}
          </v-chip>
        </template>

        <template v-slot:item.sort="{ item }">
          <div class="sort-index" :class="{ 'drag-over': dragOverIndex === findNodeIndex(item.id) }">
            {{ findNodeIndex(item.id) + 1 }}
          </div>
        </template>

        <template v-slot:item.autoSync="{ item }">
          <v-switch
            v-model="item.autoSync"
            hide-details
            density="compact"
            color="primary"
            @update:model-value="toggleNodeAutoSync(item)"
            class="ma-0"
            style="width: 48px;"
          />
        </template>

        <template v-slot:item.configStatus="{ item }">
          <v-chip v-if="item.type === 'local'" size="small" variant="text" color="grey">
            —
          </v-chip>
          <v-chip v-else :color="configSyncColor(item)" size="small" variant="tonal">
            <v-icon start size="x-small" :icon="configSyncIcon(item)" />
            {{ configSyncText(item) }}
          </v-chip>
        </template>

        <template v-slot:item.lastSeen="{ item }">
          {{ formatTime(item.lastSeen) }}
        </template>

        <template v-slot:item.publicIp="{ item }">
          {{ item.meta?.publicIp || '—' }}
        </template>

        <template v-slot:item.actions="{ item }">
          <div class="d-flex ga-1 align-center">
            <v-btn size="small" variant="text" icon="mdi-pencil" @click="showEditor(item)" />
            <v-menu v-if="item.type !== 'local'">
              <template v-slot:activator="{ props }">
                <v-btn size="small" variant="text" icon="mdi-dots-vertical" v-bind="props" />
              </template>
              <v-list density="compact">
                <v-list-item prepend-icon="mdi-play" :title="$t('node.startCore')" @click="postNodeAction('startNodeCore', item.id)" />
                <v-list-item prepend-icon="mdi-stop" :title="$t('node.stopCore')" @click="postNodeAction('stopNodeCore', item.id)" />
                <v-list-item prepend-icon="mdi-restart" :title="$t('node.restartCore')" @click="postNodeAction('restartNodeCore', item.id)" />
                <v-divider class="my-1" />
                <v-list-item prepend-icon="mdi-key-change" :title="$t('node.rotateToken')" @click="rotateToken(item)" />
                <v-list-item prepend-icon="mdi-information" :title="$t('node.details')" @click="loadNodeInfo(item)" />
                <v-list-item prepend-icon="mdi-chart-line" :title="$t('node.stats')" @click="loadNodeStats(item)" />
                <v-divider class="my-1" />
                <v-list-item prepend-icon="mdi-delete" class="text-error" :title="$t('actions.del')" @click="confirmDelete(item)" />
              </v-list>
            </v-menu>
            <v-btn v-if="item.type === 'local'" size="small" variant="text" icon="mdi-delete" color="error" @click="confirmDelete(item)" />
          </div>
        </template>

        <template v-slot:item.lastError="{ item }">
          <v-tooltip v-if="item.lastError" :text="item.lastError" location="top">
            <template v-slot:activator="{ props }">
              <v-icon v-bind="props" color="warning" size="small" icon="mdi-alert-circle" />
            </template>
          </v-tooltip>
        </template>

        <template v-slot:bottom />
      </v-data-table>
    </v-card>

    <!-- Editor Dialog -->
    <v-dialog v-model="editor.visible" max-width="760">
      <v-card rounded="xl">
        <v-card-title class="d-flex align-center justify-space-between">
          <span>{{ editor.form.id ? $t('actions.edit') : $t('actions.add') }} {{ $t('objects.node') }}</span>
          <v-btn icon="mdi-close" variant="text" @click="editor.visible = false" />
        </v-card-title>
        <v-divider />
        <v-card-text>
          <v-row>
            <v-col cols="12" md="6">
              <v-text-field v-model="editor.form.name" :label="$t('node.name')" variant="outlined" density="comfortable" />
            </v-col>
            <v-col cols="12" md="6">
              <v-select v-model="editor.form.type" :items="nodeTypes" :label="$t('type')" variant="outlined" density="comfortable" />
            </v-col>
            <v-col cols="12" v-if="editor.form.type === 'remote'">
              <v-divider class="mb-2">
                <v-chip size="small" color="primary">{{ $t('node.agentApi') }}</v-chip>
              </v-divider>
            </v-col>
            <v-col cols="12" md="4" v-if="editor.form.type === 'remote'">
              <v-select v-model="editor.form.apiScheme" :items="apiSchemes" :label="$t('node.apiScheme')" variant="outlined" density="comfortable" />
            </v-col>
            <v-col cols="12" md="5" v-if="editor.form.type === 'remote'">
              <v-text-field v-model="editor.form.apiHost" :label="$t('node.apiHost')" placeholder="127.0.0.1" variant="outlined" density="comfortable" />
            </v-col>
            <v-col cols="12" md="3" v-if="editor.form.type === 'remote'">
              <v-text-field v-model.number="editor.form.apiPort" :label="$t('node.apiPort')" type="number" variant="outlined" density="comfortable" />
            </v-col>
            <v-col cols="12" v-if="editor.form.type === 'remote'">
              <v-text-field v-model="editor.form.token" :label="$t('node.token')" variant="outlined" density="comfortable" type="password" />
            </v-col>
            <v-col cols="12">
              <v-divider class="mb-2">
                <v-chip size="small" color="info">{{ $t('node.publicAccess') }}</v-chip>
              </v-divider>
            </v-col>
            <v-col cols="12" md="6">
              <v-select v-model="editor.form.publicHostMode" :items="publicHostModes" :label="$t('node.publicHostMode')" variant="outlined" density="comfortable" />
            </v-col>
            <v-col cols="12" md="6">
              <v-text-field
                v-model="editor.form.publicHost"
                :label="$t('node.publicHost')"
                :disabled="editor.form.publicHostMode !== 'custom'"
                variant="outlined"
                density="comfortable"
              />
            </v-col>
            <v-col cols="12" md="6">
              <v-text-field v-model="editor.form.publicPortMap" :label="$t('node.publicPortMap')" placeholder='{"443":"8443"}' variant="outlined" density="comfortable" />
            </v-col>
          </v-row>
        </v-card-text>
        <v-divider />
        <v-card-actions>
          <v-spacer />
          <v-btn variant="text" @click="editor.visible = false">{{ $t('actions.close') }}</v-btn>
          <v-btn color="primary" @click="saveNode">{{ $t('actions.save') }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Details Dialog -->
    <v-dialog v-model="details.visible" max-width="900">
      <v-card rounded="xl">
        <v-card-title class="d-flex align-center justify-space-between">
          <span>{{ details.title }}</span>
          <v-btn icon="mdi-close" variant="text" @click="details.visible = false" />
        </v-card-title>
        <v-divider />
        <v-card-text>
          <pre class="details-pre">{{ details.content }}</pre>
        </v-card-text>
      </v-card>
    </v-dialog>

    <!-- Delete Confirm Dialog -->
    <v-dialog v-model="deleteDialog.visible" max-width="420">
      <v-card rounded="xl">
        <v-card-title>{{ $t('actions.del') }} {{ $t('objects.node') }}</v-card-title>
        <v-card-text>{{ $t('confirm') }}</v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn variant="text" @click="deleteDialog.visible = false">{{ $t('no') }}</v-btn>
          <v-btn color="error" @click="deleteNode">{{ $t('yes') }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Snackbar -->
    <v-snackbar v-model="snackbar.visible" :color="snackbar.color" :timeout="snackbar.timeout" location="top">
      {{ snackbar.message }}
      <template v-slot:actions>
        <v-btn variant="text" icon="mdi-close" @click="snackbar.visible = false" />
      </template>
    </v-snackbar>
  </div>
</template>

<script lang="ts" setup>
import HttpUtils from '@/plugins/httputil'
import { computed, inject, onMounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'

const { t } = useI18n()

type NodeStatus = 'unknown' | 'online' | 'offline' | 'disabled' | 'error'
type NodeType = 'local' | 'remote'

interface NodeItem {
  id: number
  name: string
  type: NodeType
  enabled: boolean
  sort: number
  autoSync: boolean
  apiHost: string
  apiPort: number
  apiScheme: string
  apiBaseUrl: string
  token: string
  publicHostMode: string
  publicHost: string
  publicPortMap: string | Record<string, string> | null
  status: NodeStatus
  lastSeen: number
  lastError: string
  meta: any
}

const loading = inject<any>('loading', ref(false))
const nodes = ref<NodeItem[]>([])
const nodeConfigVersion = ref(0)
const refreshing = ref(false)
const pushing = ref(false)
const draggingIndex = ref<number | null>(null)
const dragOverIndex = ref<number | null>(null)
const nodeTypes: NodeType[] = ['local', 'remote']
const publicHostModes = [
  { title: t('node.hostModeAgent'), value: 'agent' },
  { title: t('node.hostModeLocal'), value: 'local' },
  { title: t('node.hostModePublic'), value: 'public' },
  { title: t('node.hostModeCustom'), value: 'custom' },
]
const apiSchemes = ['http', 'https']

const tableHeaders = computed(() => [
  { title: t('node.statusLabel'), key: 'status', sortable: true, width: '120px' },
  { title: t('node.name'), key: 'name', sortable: true },
  { title: t('type'), key: 'type', sortable: true, width: '100px' },
  { title: t('node.publicIp'), key: 'publicIp', sortable: false, width: '140px' },
  { title: t('node.configStatusLabel'), key: 'configStatus', sortable: false, width: '130px' },
  { title: t('node.lastSeen'), key: 'lastSeen', sortable: true, width: '180px' },
  { title: '', key: 'lastError', sortable: false, width: '40px' },
  { title: t('enable'), key: 'enabled', sortable: false, width: '80px' },
  { title: '', key: 'drag', sortable: false, width: '40px' },
  { title: t('node.sort'), key: 'sort', sortable: false, width: '60px' },
  { title: t('node.autoSync'), key: 'autoSync', sortable: false, width: '100px' },
  { title: t('actions.action'), key: 'actions', sortable: false, width: '120px' },
])

const emptyNode = (): NodeItem => ({
  id: 0,
  name: '',
  type: 'remote',
  enabled: true,
  sort: 0,
  autoSync: true,
  apiHost: '',
  apiPort: 2097,
  apiScheme: 'http',
  apiBaseUrl: '',
  token: '',
  publicHostMode: 'agent',
  publicHost: '',
  publicPortMap: '',
  status: 'unknown',
  lastSeen: 0,
  lastError: '',
  meta: null,
})

const editor = ref({
  visible: false,
  form: emptyNode(),
})

const details = ref({
  visible: false,
  title: '',
  content: '',
})

const deleteDialog = ref({
  visible: false,
  node: null as NodeItem | null,
})

const snackbar = ref({
  visible: false,
  color: 'success',
  message: '',
  timeout: 5000,
})

const onlineCount = computed(() => nodes.value.filter((n) => n.status === 'online').length)
const syncedCount = computed(() => {
  if (!nodeConfigVersion.value) return 0
  return nodes.value.filter((n) => {
    if (n.type === 'local') return false
    const meta = n.meta as any
    if (!meta || !meta.lastDBVersion) return false
    return Number(meta.lastDBVersion) === nodeConfigVersion.value
  }).length
})

const showSnackbar = (color: string, message: string) => {
  snackbar.value = { visible: true, color, message, timeout: 5000 }
}

const statusColor = (status: NodeStatus) => {
  switch (status) {
    case 'online': return 'success'
    case 'offline': return 'grey'
    case 'disabled': return 'warning'
    case 'error': return 'error'
    default: return 'info'
  }
}

const statusIcon = (status: NodeStatus) => {
  switch (status) {
    case 'online': return 'mdi-check-circle'
    case 'offline': return 'mdi-close-circle'
    case 'disabled': return 'mdi-pause-circle'
    case 'error': return 'mdi-alert-circle'
    default: return 'mdi-help-circle'
  }
}

const statusText = (status: NodeStatus) => {
  return status ? status : 'unknown'
}

const isNodeConfigSynced = (node: NodeItem) => {
  const meta = node.meta as any
  if (!meta || !meta.lastDBVersion || !nodeConfigVersion.value) return false
  return Number(meta.lastDBVersion) === nodeConfigVersion.value
}

const configSyncColor = (node: NodeItem) => {
  if (node.status === 'error') return 'error'
  const meta = node.meta as any
  if (!meta || !meta.lastDBVersion) return 'grey'
  if (!nodeConfigVersion.value) return 'grey'
  return isNodeConfigSynced(node) ? 'success' : 'warning'
}

const configSyncIcon = (node: NodeItem) => {
  if (node.status === 'error') return 'mdi-alert-circle'
  const meta = node.meta as any
  if (!meta || !meta.lastDBVersion) return 'mdi-help-circle'
  if (!nodeConfigVersion.value) return 'mdi-help-circle'
  return isNodeConfigSynced(node) ? 'mdi-check-circle' : 'mdi-sync-alert'
}

const configSyncText = (node: NodeItem) => {
  if (node.status === 'error') return t('node.configError')
  const meta = node.meta as any
  if (!meta || !meta.lastDBVersion) return t('node.configUnknown')
  if (!nodeConfigVersion.value) return t('node.configUnknown')
  return isNodeConfigSynced(node) ? t('node.configSynced') : t('node.configNotSynced')
}

const normalizePortMap = (value: any) => {
  if (!value) return ''
  if (typeof value === 'string') return value
  return JSON.stringify(value)
}

const formatTime = (timestamp: number) => {
  if (!timestamp) return '—'
  return new Date(timestamp * 1000).toLocaleString()
}

const setLoading = (value: boolean) => {
  loading.value = value
}

const loadNodes = async () => {
  setLoading(true)
  const msg = await HttpUtils.get('api/nodes')
  if (msg.success) {
    const data = msg.obj ?? {}
    nodeConfigVersion.value = Number(data.nodeConfigVersion ?? 0)
    nodes.value = (data.nodes ?? []).map((node: NodeItem) => ({
      ...node,
      publicPortMap: normalizePortMap(node.publicPortMap),
    }))
  }
  setLoading(false)
}

const refreshAllNodes = async () => {
  refreshing.value = true
  const msg = await HttpUtils.post('api/syncAllNodes', null)
  if (msg.success) {
    const obj = msg.obj ?? {}
    showSnackbar('success', `${t('node.refreshAllResult')}: ${obj.successCount ?? 0} ${t('success')}, ${obj.failCount ?? 0} ${t('failed')}`)
  } else {
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
  await loadNodes()
  refreshing.value = false
}

const pushConfigToAll = async () => {
  pushing.value = true
  const msg = await HttpUtils.post('api/pushConfigToAll', null)
  if (msg.success) {
    const obj = msg.obj ?? {}
    showSnackbar('success', `${t('node.syncConfigResult')}: ${obj.successCount ?? 0} ${t('success')}, ${obj.failCount ?? 0} ${t('failed')}`)
  } else {
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
  await loadNodes()
  pushing.value = false
}

const showEditor = (node?: NodeItem) => {
  if (node) {
    editor.value.form = {
      ...node,
      publicPortMap: normalizePortMap(node.publicPortMap),
      publicHostMode: node.publicHostMode || (node.type === 'local' ? 'local' : 'agent'),
    }
  } else {
    const newNode = emptyNode()
    editor.value.form = newNode
  }
  editor.value.visible = true
}

const buildSaveData = () => {
  const form = { ...editor.value.form }
  if (typeof form.publicPortMap === 'string' && form.publicPortMap.trim()) {
    form.publicPortMap = JSON.parse(form.publicPortMap)
  } else {
    form.publicPortMap = null
  }
  // If publicHostMode is not custom, clear publicHost
  if (form.publicHostMode !== 'custom') {
    form.publicHost = ''
  }
  // For local nodes, clear remote-specific fields
  if (form.type === 'local') {
    form.apiHost = ''
    form.apiPort = 2097
    form.apiScheme = 'http'
    form.token = ''
  }
  return form
}

const toggleNodeEnabled = async (node: NodeItem) => {
  const msg = await HttpUtils.post('api/toggleNode', { id: node.id })
  if (msg.success) {
    node.enabled = msg.obj?.enabled ?? node.enabled
  } else {
    node.enabled = !node.enabled
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
}

const findNodeIndex = (id: number): number => {
  return nodes.value.findIndex(n => n.id === id)
}

const onDragStart = (event: DragEvent, item: NodeItem) => {
  draggingIndex.value = findNodeIndex(item.id)
  if (event.dataTransfer) {
    event.dataTransfer.effectAllowed = 'move'
    event.dataTransfer.setData('text/plain', '')
  }
}

const onDragOver = (event: DragEvent, item: NodeItem) => {
  const index = findNodeIndex(item.id)
  if (draggingIndex.value !== null && draggingIndex.value !== index) {
    dragOverIndex.value = index
  }
}

const onDragLeave = () => {
  dragOverIndex.value = null
}

const onDrop = (event: DragEvent, item: NodeItem) => {
  if (draggingIndex.value === null) return
  
  const targetIndex = findNodeIndex(item.id)
  if (draggingIndex.value === targetIndex) return
  
  // 重新排序数组
  const draggedItem = nodes.value[draggingIndex.value]
  nodes.value.splice(draggingIndex.value, 1)
  nodes.value.splice(targetIndex, 0, draggedItem)
  
  // 更新所有节点的 sort 值
  nodes.value.forEach((node, index) => {
    node.sort = index
  })
  
  // 批量保存排序
  saveAllSorts()
}

const onDragEnd = () => {
  draggingIndex.value = null
  dragOverIndex.value = null
}

const saveAllSorts = async () => {
  const sortData = nodes.value.map(node => ({ id: node.id, sort: node.sort }))
  const msg = await HttpUtils.post('api/updateNodeSort', sortData, {
    headers: { 'Content-Type': 'application/json' },
  })
  if (msg.success) {
    showSnackbar('success', t('node.sortUpdated'))
  } else {
    showSnackbar('error', msg.msg || t('node.actionFailed'))
    await loadNodes()
  }
}

const toggleNodeAutoSync = async (node: NodeItem) => {
  const msg = await HttpUtils.post('api/toggleNodeAutoSync', { id: node.id })
  if (msg.success) {
    node.autoSync = msg.obj?.autoSync ?? node.autoSync
    showSnackbar('success', t('node.autoSyncUpdated'))
  } else {
    node.autoSync = !node.autoSync
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
}

const saveNode = async () => {
  setLoading(true)
  try {
    const data = buildSaveData()
    const msg = await HttpUtils.post('api/save', {
      object: 'nodes',
      action: data.id ? 'edit' : 'new',
      data: JSON.stringify(data),
    })
    if (msg.success) {
      editor.value.visible = false
      showSnackbar('success', t('node.actionSuccess'))
      await loadNodes()
    } else {
      showSnackbar('error', msg.msg || t('node.actionFailed'))
    }
  } catch (e: any) {
    showSnackbar('error', e.toString())
  }
  setLoading(false)
}

const postNodeAction = async (action: string, id: number) => {
  setLoading(true)
  const msg = await HttpUtils.post(`api/${action}`, { id })
  if (msg.success) {
    showSnackbar('success', t('node.actionSuccess'))
  } else {
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
  await loadNodes()
  setLoading(false)
}

const rotateToken = async (node: NodeItem) => {
  setLoading(true)
  const msg = await HttpUtils.post('api/rotateNodeToken', { id: node.id })
  if (msg.success) {
    details.value = {
      visible: true,
      title: `${node.name} token`,
      content: msg.obj?.token ?? '',
    }
    showSnackbar('success', t('node.actionSuccess'))
    await loadNodes()
  } else {
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
  setLoading(false)
}

const loadNodeInfo = async (node: NodeItem) => {
  setLoading(true)
  const msg = await HttpUtils.get('api/nodeInfo', { id: node.id })
  if (msg.success) {
    showDetails(`${node.name} - ${t('node.details')}`, msg.obj)
  } else {
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
  setLoading(false)
}

const loadNodeStats = async (node: NodeItem) => {
  setLoading(true)
  const msg = await HttpUtils.get('api/nodeStats', { id: node.id })
  if (msg.success) {
    showDetails(`${node.name} - ${t('node.stats')}`, msg.obj)
  } else {
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
  setLoading(false)
}

const showDetails = (title: string, data: any) => {
  details.value = {
    visible: true,
    title,
    content: typeof data === 'string' ? data : JSON.stringify(data, null, 2),
  }
}

const confirmDelete = (node: NodeItem) => {
  deleteDialog.value.node = node
  deleteDialog.value.visible = true
}

const deleteNode = async () => {
  if (!deleteDialog.value.node) return
  setLoading(true)
  const msg = await HttpUtils.post('api/save', {
    object: 'nodes',
    action: 'del',
    data: JSON.stringify({ id: deleteDialog.value.node.id }),
  })
  if (msg.success) {
    deleteDialog.value.visible = false
    showSnackbar('success', t('node.actionSuccess'))
    await loadNodes()
  } else {
    showSnackbar('error', msg.msg || t('node.actionFailed'))
  }
  setLoading(false)
}

onMounted(loadNodes)
</script>

<style scoped>
.nodes-page {
  padding-bottom: 24px;
}

.hero-card {
  overflow: hidden;
  background:
    radial-gradient(circle at 12% 10%, rgba(var(--v-theme-primary), .28), transparent 30%),
    linear-gradient(135deg, rgba(var(--v-theme-surface), 1), rgba(var(--v-theme-surface-variant), .45));
  border: 1px solid rgba(var(--v-theme-primary), .18);
}

.hero-kicker {
  color: rgb(var(--v-theme-primary));
  letter-spacing: .18em;
}

.hero-title {
  font-size: clamp(2rem, 4vw, 3.4rem);
  line-height: 1;
  font-weight: 800;
  margin: 4px 0 12px;
}

.hero-subtitle {
  max-width: 680px;
  opacity: .72;
  margin: 0;
}

.summary-card {
  height: 100%;
  border: 1px solid rgba(var(--v-theme-outline), .14);
}

.summary-number {
  font-size: 2rem;
  font-weight: 800;
  line-height: 1;
}

.summary-label {
  opacity: .66;
  margin-top: 6px;
  font-size: .85rem;
}

.nodes-table :deep(th) {
  font-weight: 600 !important;
  white-space: nowrap;
}

.drag-handle {
  cursor: grab;
  opacity: 0.5;
  transition: opacity 0.2s;
}

.drag-handle:hover {
  opacity: 1;
}

.drag-handle.dragging {
  cursor: grabbing;
  opacity: 0.3;
}

.sort-index {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: rgba(var(--v-theme-surface-variant), 0.3);
  font-weight: 600;
  font-size: 0.9rem;
  transition: all 0.2s;
}

.sort-index.drag-over {
  background: rgba(var(--v-theme-primary), 0.2);
  transform: scale(1.1);
}

.nodes-table :deep(tr) {
  transition: background-color 0.2s;
}

.details-pre {
  max-height: 62vh;
  overflow: auto;
  padding: 16px;
  border-radius: 14px;
  background: rgba(var(--v-theme-surface-variant), .52);
  font-size: .86rem;
  line-height: 1.55;
}
</style>
