<template>
  <div class="hv-root" ref="rootEl">

    <!-- Loading -->
    <div v-if="state === 'loading'" class="hv-status">
      <div class="hv-spinner"></div>
      <span>Loading…</span>
    </div>

    <!-- Error -->
    <div v-else-if="state === 'error'" class="hv-status hv-status--error">
      ⚠️ {{ errorMsg }}
    </div>

    <!-- Tree -->
    <div v-else-if="state === 'ready'" class="hv-tree-wrapper">
      <HvNode
        v-if="treeData"
        :node="treeData"
        :is-root="true"
        :depth="0"
        :expand-to-agent-id="expandToAgentId"
        @add-agent="handleAddAgent"
      />
    </div>

    <!-- Add Agent Overlay Panel -->
    <Transition name="hv-fade">
      <div v-if="panel.visible" class="hv-overlay" @click.self="closePanel">
        <div class="hv-panel">

          <div class="hv-panel__header">
            <div class="hv-panel__title">Add Agent</div>
            <button class="hv-panel__close" @click="closePanel">×</button>
          </div>

          <div class="hv-panel__sub">Reporting to: {{ panel.parentNode?.name }}</div>

          <!-- Panel loading -->
          <div v-if="panel.loading" class="hv-status hv-status--sm">
            <div class="hv-spinner hv-spinner--sm"></div>
            <span>Loading options…</span>
          </div>

          <!-- Panel load error -->
          <template v-else-if="panel.loadError">
            <div class="hv-status hv-status--error">⚠️ {{ panel.loadError }}</div>
          </template>

          <!-- Panel form -->
          <template v-else>
            <div class="hv-form-label">Hierarchy Level</div>
            <select v-model="panel.selectedLevel" class="hv-select">
              <option value="" disabled>Select a level…</option>
              <option v-for="l in panel.levels" :key="l.id" :value="l.id">{{ l.name }}</option>
            </select>

            <div class="hv-form-label">Agent</div>
            <select v-model="panel.selectedAgent" class="hv-select">
              <option value="" disabled>Select an agent…</option>
              <option v-for="a in panel.agents" :key="a.id" :value="a.id">
                {{ a.first_name }} {{ a.last_name }}
              </option>
            </select>

            <div
              v-if="panel.statusMsg"
              class="hv-panel__status"
              :class="panel.statusType === 'success' ? 'hv-panel__status--ok' : 'hv-panel__status--err'"
            >
              {{ panel.statusMsg }}
            </div>

            <div class="hv-btn-row">
              <button class="hv-btn hv-btn--secondary" :disabled="panel.saving" @click="closePanel">
                Cancel
              </button>
              <button class="hv-btn hv-btn--primary" :disabled="panel.saving" @click="saveAgent">
                {{ panel.saving ? 'Saving…' : 'Add Agent' }}
              </button>
            </div>
          </template>

        </div>
      </div>
    </Transition>

  </div>
</template>


<script>
import { ref, reactive, onMounted, defineComponent } from 'vue'

// ─── HvNode: recursive tree node component ────────────────────────────────────
const HvNode = defineComponent({
  name: 'HvNode',

  props: {
    node:            { type: Object,  required: true },
    isRoot:          { type: Boolean, default: false },
    depth:           { type: Number,  default: 0 },
    expandToAgentId: { type: Number,  default: null }
  },

  emits: ['add-agent'],

  data() {
    const shouldAutoExpand =
      this.isRoot ||
      !!(this.expandToAgentId && this.nodeContainsAgent(this.node, this.expandToAgentId))
    return {
      expanded: shouldAutoExpand,
      selected: shouldAutoExpand && !this.isRoot
    }
  },

  computed: {
    hasChildren() {
      return !!(this.node.children && this.node.children.length > 0)
    },
    isLeaf() {
      return this.node.level === 1
    },
    showPlus() {
      if (this.isLeaf)       return false
      if (this.isRoot)       return true
      if (!this.hasChildren) return true
      return this.expanded
    },
    showToggle() {
      return !this.isLeaf && !this.isRoot && this.hasChildren && !this.expanded
    }
  },

  methods: {
    nodeContainsAgent(node, targetId) {
      if (node.agent_id === targetId) return true
      if (!node.children) return false
      return node.children.some(c => this.nodeContainsAgent(c, targetId))
    },
    toggle() {
      if (this.isRoot || this.isLeaf || !this.hasChildren) return
      this.expanded = !this.expanded
      this.selected = this.expanded
    },
    onPlusClick(e) {
      e.stopPropagation()
      this.$emit('add-agent', this.node)
    },
    forwardAddAgent(node) {
      this.$emit('add-agent', node)
    }
  },

  template: `
    <div class="hv-node-wrap">
      <div v-if="!isRoot" class="hv-connector-v"></div>

      <div
        class="hv-card"
        :class="{ 'hv-card--root': isRoot, 'hv-card--selected': selected }"
        @click="toggle"
      >
        <div class="hv-badge">{{ node.levelName || '' }}</div>
        <div class="hv-name">{{ node.name || '' }}</div>
      </div>

      <div class="hv-below">
        <button v-if="showPlus" class="hv-plus-btn" @click="onPlusClick">+</button>
        <div v-if="showToggle" class="hv-toggle-chevron" @click.stop="toggle">&#9660;</div>
      </div>

      <div
        v-if="hasChildren"
        class="hv-children-area"
        :class="{ 'hv-children-area--open': expanded || isRoot }"
      >
        <div class="hv-connector-v"></div>
        <div class="hv-child-row">
          <div v-if="node.children.length > 1" class="hv-hbar"></div>
          <HvNode
            v-for="child in node.children"
            :key="child.agent_id || child.name"
            :node="child"
            :is-root="false"
            :depth="depth + 1"
            :expand-to-agent-id="expandToAgentId"
            @add-agent="forwardAddAgent"
          />
        </div>
      </div>
    </div>
  `
})

// ─── Main component ───────────────────────────────────────────────────────────
export default {
  name: 'HierarchyTree',

  components: { HvNode },

  props: {
    content: {
      type:    Object,
      default: () => ({})
    }
  },

  setup() {

    // ── Auth token ──────────────────────────────────────────────────────────
    function getToken() {
      return props.content?.authToken
}

    function authHeaders() {
      return { Authorization: 'Bearer ' + getToken() }
    }

    // ── State ───────────────────────────────────────────────────────────────
    const rootEl          = ref(null)
    const state           = ref('loading')
    const errorMsg        = ref('')
    const treeData        = ref(null)
    const expandToAgentId = ref(null)

    const panel = reactive({
      visible:       false,
      loading:       false,
      loadError:     '',
      saving:        false,
      parentNode:    null,
      levels:        [],
      agents:        [],
      selectedLevel: '',
      selectedAgent: '',
      statusMsg:     '',
      statusType:    ''
    })

    // ── Fetch tree ──────────────────────────────────────────────────────────
    async function fetchTree() {
      state.value = 'loading'
      try {
        const res = await fetch(
          'https://apiv1.myagencyview.com/api:4Hs1rh6K/getHierarchy',
          { headers: authHeaders() }
        )
        if (!res.ok) throw new Error('HTTP ' + res.status)
        treeData.value = await res.json()
        state.value    = 'ready'
      } catch (e) {
        state.value    = 'error'
        errorMsg.value = e.message || 'Failed to load hierarchy.'
      }
    }

    // ── Open Add Agent panel ────────────────────────────────────────────────
    async function handleAddAgent(parentNode) {
      panel.visible       = true
      panel.loading       = true
      panel.loadError     = ''
      panel.parentNode    = parentNode
      panel.levels        = []
      panel.agents        = []
      panel.selectedLevel = ''
      panel.selectedAgent = ''
      panel.statusMsg     = ''
      panel.saving        = false

      try {
        const res = await fetch(
          'https://apiv1.myagencyview.com/api:4Hs1rh6K/hierarchyOptions?level=' + encodeURIComponent(parentNode.level),
          { headers: authHeaders() }
        )
        if (!res.ok) throw new Error('HTTP ' + res.status)
        const data    = await res.json()
        panel.levels  = data.availableLevels || []
        panel.agents  = data.availableAgents || []
        panel.loading = false
      } catch (e) {
        panel.loading   = false
        panel.loadError = e.message || 'Failed to load options.'
      }
    }

    // ── Close panel ─────────────────────────────────────────────────────────
    function closePanel() {
      panel.visible = false
    }

    // ── Save agent ──────────────────────────────────────────────────────────
    async function saveAgent() {
      if (!panel.selectedLevel || !panel.selectedAgent) {
        panel.statusMsg  = 'Please select both a level and an agent.'
        panel.statusType = 'error'
        return
      }

      panel.saving    = true
      panel.statusMsg = ''

      try {
        const res = await fetch(
          'https://apiv1.myagencyview.com/api:4Hs1rh6K/updateHierarchy',
          {
            method:  'PATCH',
            headers: { ...authHeaders(), 'Content-Type': 'application/json' },
            body:    JSON.stringify({
              parent_id: panel.parentNode?.agent_id ?? null,
              child_id:  parseInt(panel.selectedAgent),
              level_id:  parseInt(panel.selectedLevel)
            })
          }
        )

        if (!res.ok) {
          const err = await res.json().catch(() => ({}))
          throw new Error(err.message || 'Save failed.')
        }

        panel.statusMsg  = '✓ Agent added successfully!'
        panel.statusType = 'success'

        setTimeout(() => {
          closePanel()
          expandToAgentId.value = parseInt(panel.selectedAgent)
          fetchTree()
        }, 1500)

      } catch (e) {
        panel.statusMsg  = '⚠️ ' + (e.message || 'Unexpected error.')
        panel.statusType = 'error'
        panel.saving     = false
      }
    }

    // ── Lifecycle ───────────────────────────────────────────────────────────
    onMounted(() => {
      if (!getToken()) {
        state.value    = 'error'
        errorMsg.value = 'No Xano auth token found. Please log in.'
        return
      }
      fetchTree()
    })

    return {
      rootEl,
      state,
      errorMsg,
      treeData,
      expandToAgentId,
      panel,
      handleAddAgent,
      closePanel,
      saveAgent
    }
  }
}
</script>


<style>
@keyframes hv-spin {
  to { transform: rotate(360deg); }
}

.hv-root {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  font-size: 13px;
  overflow: auto;
  box-sizing: border-box;
  width: 100%;
  height: 100%;
  background: #ffffff;
  position: relative;
}

.hv-tree-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  padding: 24px 16px;
  box-sizing: border-box;
  min-height: 100%;
}

.hv-node-wrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
  flex-shrink: 0;
}

.hv-connector-v {
  width: 2px;
  height: 24px;
  background: #040404;
  flex-shrink: 0;
}

.hv-child-row {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: flex-start;
  width: 100%;
  position: relative;
  box-sizing: border-box;
  gap: 12px;
}

.hv-hbar {
  position: absolute;
  top: 0;
  height: 2px;
  background: #040404;
  z-index: 0;
}

.hv-children-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  overflow: hidden;
  max-height: 0;
  opacity: 0;
  position: absolute;
  visibility: hidden;
}

.hv-children-area--open {
  max-height: 9999px;
  opacity: 1;
  position: relative;
  visibility: visible;
}

.hv-below {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  min-height: 28px;
}

.hv-card {
  background: #ffffff;
  border: 2px solid #040404;
  border-radius: 10px;
  padding: 12px 18px;
  width: 180px;
  min-width: 180px;
  max-width: 180px;
  text-align: center;
  cursor: pointer;
  user-select: none;
  box-sizing: border-box;
  margin: 0 6px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.10), 0 1px 3px rgba(0,0,0,0.08);
}

.hv-card--root     { cursor: default; }

.hv-card--selected {
  border-color: #e31d25;
  background: rgba(227,29,37,0.05);
  box-shadow: 0 8px 24px rgba(227,29,37,0.15), 0 2px 6px rgba(0,0,0,0.08);
}

.hv-badge {
  font-size: 9px;
  font-weight: 800;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: #e31d25;
  margin-bottom: 5px;
}

.hv-name {
  font-size: 13px;
  font-weight: 700;
  color: #040404;
  line-height: 1.3;
}

.hv-plus-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #e31d25;
  border: none;
  font-size: 16px;
  color: #ffffff;
  margin-top: 6px;
  cursor: pointer;
  flex-shrink: 0;
  line-height: 1;
  user-select: none;
}

.hv-toggle-chevron {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  font-weight: 900;
  color: #e31d25;
  margin-top: 4px;
  cursor: pointer;
  transform: rotate(180deg);
  position: relative;
  z-index: 1;
}

.hv-status {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 16px;
  font-size: 13px;
  color: #888888;
}

.hv-status--error { color: #e31d25; }
.hv-status--sm    { margin-bottom: 16px; }

.hv-spinner {
  width: 16px;
  height: 16px;
  border: 2px solid #e0e0e0;
  border-top-color: #e31d25;
  border-radius: 50%;
  animation: hv-spin 0.7s linear infinite;
  flex-shrink: 0;
}

.hv-spinner--sm { width: 14px; height: 14px; }

.hv-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(255,255,255,0.85);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
  box-sizing: border-box;
}

.hv-panel {
  background: #ffffff;
  border: 2px solid #040404;
  border-radius: 12px;
  padding: 24px;
  width: 320px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.15);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  box-sizing: border-box;
}

.hv-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.hv-panel__title  { font-size: 15px; font-weight: 700; color: #040404; }

.hv-panel__close {
  cursor: pointer;
  font-size: 18px;
  color: #040404;
  font-weight: 300;
  line-height: 1;
  padding: 2px 6px;
  border-radius: 4px;
  background: transparent;
  border: none;
}

.hv-panel__sub {
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #e31d25;
  margin-bottom: 16px;
}

.hv-form-label {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #040404;
  margin-bottom: 6px;
}

.hv-select {
  width: 100%;
  padding: 9px 12px;
  border: 2px solid #040404;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 500;
  color: #040404;
  background: #ffffff;
  appearance: none;
  -webkit-appearance: none;
  outline: none;
  cursor: pointer;
  box-sizing: border-box;
  margin-bottom: 16px;
}

.hv-panel__status      { font-size: 12px; min-height: 18px; margin-bottom: 12px; text-align: center; }
.hv-panel__status--ok  { color: #16a34a; }
.hv-panel__status--err { color: #e31d25; }

.hv-btn-row { display: flex; gap: 10px; }

.hv-btn {
  flex: 1;
  padding: 10px;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 700;
  cursor: pointer;
  box-sizing: border-box;
}

.hv-btn--secondary { border: 2px solid #040404; background: #ffffff; color: #040404; }
.hv-btn--primary   { border: 2px solid #e31d25; background: #e31d25; color: #ffffff; }

.hv-fade-enter-active,
.hv-fade-leave-active  { transition: opacity 0.18s ease; }
.hv-fade-enter-from,
.hv-fade-leave-to      { opacity: 0; }
</style>
