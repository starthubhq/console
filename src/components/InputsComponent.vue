<script setup lang="ts">
import { ref, reactive, onMounted, watch, computed } from 'vue'
import { useRoute } from 'vue-router'
import { supabase } from '@/lib/supabase'

type PortDirection = 'INPUT' | 'OUTPUT'
type PortType = 'STRING' | 'NUMBER' | 'BOOLEAN' | 'JSON' | string

type ActionPort = {
  id: string
  name: string
  created_at: string
  rls_owner_id: string | null
  action_port_type: PortType
  action_version_id: string
  action_port_direction: PortDirection
}

type ActionResponse = {
  action_id: string
  name: string | null
  description: string | null
  created_at: string
  action_version_id: string
  version_number: string
  version_created_at: string
  commit_sha: string | null
  inputs: ActionPort[]
  outputs: ActionPort[]
}

const route = useRoute()
const data = ref<ActionResponse | null>(null)
const errorMsg = ref<string | null>(null)
const loading = ref(false)
const submitting = ref(false)
const runner = ref<'github' | 'local'>('github')  // matches #[serde(rename_all="lowercase")]
const envName = ref<string | null>(null)
// simple secrets dictionary; wire to UI later if you want
const secrets = reactive<Record<string, string>>({})


// form state keyed by input port name
const form = reactive<Record<string, any>>({})

// Build sensible defaults per port type
function defaultForType(t: PortType) {
  switch (t) {
    case 'STRING': return ''
    case 'NUMBER': return null
    case 'BOOLEAN': return false
    case 'JSON': return '{}'
    default: return ''
  }
}

async function fetchData() {
  loading.value = true
  errorMsg.value = null
  data.value = null
  Object.keys(form).forEach(k => delete (form as any)[k])

  const namespace = String(route.params.namespace ?? '')
  const actionSlug = String(route.params.slug ?? '')
  const version =
    (route.params.version as string | undefined)
    ?? (route.query.v as string | undefined)
    ?? null

  console.info({
      p_namespace: namespace,
      p_action_slug: actionSlug,
      p_version_number: version
    })
  const { data: res, error } = await supabase.rpc(
    // make sure this matches your DB function name
    'get_action_version_by_namespace_slug_version',
    {
      p_namespace: 'tgirotto',
      p_action_slug: 'tom-action-4',
      p_version_number: '0.1.0'
    }
  )

  if (error) {
    errorMsg.value = error.message
  } else {
    console.info(res)
    // res is a single JSON object (or {}), normalize to null if empty
    const obj = (res && Object.keys(res).length) ? (res as ActionResponse) : null
    data.value = obj

    // initialize form defaults from inputs
    if (obj?.inputs?.length) {
      for (const p of obj.inputs) {
        // avoid collisions: port names should be unique per action version
        form[p.name] = defaultForType(p.action_port_type)
      }
    }
  }

  loading.value = false
}

// Derived lists for rendering
const inputs = computed(() => data.value?.inputs ?? [])
const outputs = computed(() => data.value?.outputs ?? [])

// Optional: simple per-field hint from type
function placeholderFor(p: ActionPort) {
  switch (p.action_port_type) {
    case 'STRING': return `Enter ${p.name}…`
    case 'NUMBER': return `Enter number for ${p.name}…`
    case 'BOOLEAN': return ''
    case 'JSON': return `Paste JSON for ${p.name}…`
    default: return `Enter ${p.name}…`
  }
}

function coerceValue(port: ActionPort, raw: any) {
  switch (port.action_port_type) {
    case 'NUMBER':
      // Allow empty -> null, else parse float
      if (raw === '' || raw === null || typeof raw === 'undefined') return null
      const n = Number(raw)
      return Number.isFinite(n) ? n : null
    case 'BOOLEAN':
      return Boolean(raw)
    case 'JSON':
      // Do not parse here; keep as string. Parse on submit with try/catch.
      return String(raw ?? '')
    case 'STRING':
    default:
      return String(raw ?? '')
  }
}

async function onSubmit() {
  if (!data.value) return

  // Build payload { portName: value }
  const payload: Record<string, any> = {}
  const errors: string[] = []

  for (const p of inputs.value) {
    const raw = form[p.name]
    const val = coerceValue(p, raw)

    if (p.action_port_type === 'JSON') {
      // Validate JSON (optional)
      try {
        payload[p.name] = JSON.parse(val as string)
      } catch {
        errors.push(`"${p.name}" must be valid JSON`)
      }
    } else {
      payload[p.name] = val
    }
  }

  if (errors.length) {
    errorMsg.value = errors.join('\n')
    return
  }

  // TODO: wire this to your runner/execution path.
  // For now, just log the payload.
  console.log('Execute action', {
    action_id: data.value.action_id,
    action_version_id: data.value.action_version_id,
    version: data.value.version_number,
    inputs: payload,
  })

  // Compose action ref like "namespace/slug@version"
  const namespace = String(route.params.namespace ?? '')
  const actionSlug = String(route.params.slug ?? '')
  const version =
    (route.params.version as string | undefined)
    ?? (route.query.v as string | undefined)
    ?? null

  const actionRef = version
    ? `${namespace}/${actionSlug}@${version}`
    : `${namespace}/${actionSlug}`

  const body = {
    action: actionRef,        // Rust expects String
    inputs: payload,          // arbitrary JSON object
  }

  try {
    
    submitting.value = true
    errorMsg.value = null

    const resp = await fetch('/api/run', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    })

    if (!resp.ok) {
      const txt = await resp.text()
      throw new Error(`Run failed: ${resp.status} ${txt}`)
    }

    console.info(`here`)
    // success UX — swap with your toast system if you have one
    // alert('Dispatched! Check your workflow.')
  } catch (e: any) {
    console.info(e)
    errorMsg.value = String(e?.message ?? e)
  } finally {
    submitting.value = false
  }
}

onMounted(fetchData)

watch(
  () => [route.params.namespace, route.params.name, route.params.version, route.query.v],
  fetchData
)
</script>

<template>
  <div class="container">
    <div v-if="loading">Loading…</div>
    <div v-else-if="errorMsg" class="text-red-600 whitespace-pre-line">{{ errorMsg }}</div>
    <div v-else-if="data">
      <h2 class="text-xl font-semibold">
        {{ data.name ?? '(unnamed action)' }}@{{ data.version_number }}
      </h2>
      <p class="text-gray-600 mb-4">{{ data.description }}</p>

      <form class="space-y-4" @submit.prevent="onSubmit">
        <h3 class="font-medium">Inputs</h3>

        <div v-if="!inputs.length" class="text-gray-500">No inputs.</div>

        <div v-for="p in inputs" :key="p.id" class="flex flex-col gap-1">
          <label class="font-medium">
            {{ p.name }}
            <span class="text-xs text-gray-500">({{ p.action_port_type }})</span>
          </label>

          <!-- STRING -->
          <input
            v-if="p.action_port_type === 'STRING'"
            type="text"
            class="border rounded px-3 py-2"
            :placeholder="placeholderFor(p)"
            v-model="form[p.name]"
          />

          <!-- NUMBER -->
          <input
            v-else-if="p.action_port_type === 'NUMBER'"
            type="number"
            class="border rounded px-3 py-2"
            :placeholder="placeholderFor(p)"
            :value="form[p.name]"
            @input="form[p.name] = ($event.target as HTMLInputElement).value"
          />

          <!-- BOOLEAN -->
          <label v-else-if="p.action_port_type === 'BOOLEAN'" class="inline-flex items-center gap-2">
            <input
              type="checkbox"
              class="h-4 w-4"
              v-model="form[p.name]"
            />
            <span>True?</span>
          </label>

          <!-- JSON -->
          <textarea
            v-else-if="p.action_port_type === 'JSON'"
            class="border rounded px-3 py-2 font-mono"
            rows="6"
            :placeholder="placeholderFor(p)"
            v-model="form[p.name]"
          ></textarea>

          <!-- Fallback (treat as STRING) -->
          <input
            v-else
            type="text"
            class="border rounded px-3 py-2"
            :placeholder="placeholderFor(p)"
            v-model="form[p.name]"
          />
        </div>

        <button
          type="submit"
          class="mt-2 rounded px-4 py-2 bg-black text-white"
        >
          Run
        </button>
      </form>

      <h3 class="font-medium mt-8">Outputs</h3>
      <div v-if="!outputs.length" class="text-gray-500">No outputs.</div>
      <ul v-else class="list-disc ml-6">
        <li v-for="o in outputs" :key="o.id">
          {{ o.name }} <span class="text-xs text-gray-500">({{ o.action_port_type }})</span>
        </li>
      </ul>
    </div>
    <div v-else>No action/version found.</div>
  </div>
</template>

<style scoped>
/* minimal styles; replace with your design system */
.container {
  padding: 10px;
}
</style>
