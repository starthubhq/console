<script setup lang="ts">
import { ref, reactive, onMounted, watch, computed } from 'vue'
import { useRoute } from 'vue-router'

type PortType = 'string' | 'number' | 'boolean' | 'json' | 'type' | string

type LockFilePort = {
  description: string
  type: PortType
  required: boolean
  default: any
}

type LockFileResponse = {
  name: string
  description: string
  version: string
  kind: string
  manifest_version: number
  repository: string
  license: string
  inputs: Record<string, LockFilePort> | Array<LockFilePort & { name: string }>
  outputs: Record<string, LockFilePort> | Array<LockFilePort & { name: string }>
  types: Record<string, any>
  distribution?: {
    primary: string
  }
  digest?: string
}

const route = useRoute()
const data = ref<LockFileResponse | null>(null)
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
    case 'string': return ''
    case 'number': return null
    case 'boolean': return false
    case 'json': return '{}'
    case 'type': return ''
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

  if (!namespace || !actionSlug || !version) {
    errorMsg.value = 'Missing required parameters: namespace, slug, or version'
    loading.value = false
    return
  }

  // Construct the lock file URL
  const lockFileUrl = `https://api.starthub.so/storage/v1/object/public/artifacts/${namespace}/${actionSlug}/${version}/starthub-lock.json`
  
  console.info('Fetching lock file from:', lockFileUrl)

  try {
    const response = await fetch(lockFileUrl)
    
    if (!response.ok) {
      throw new Error(`Failed to fetch lock file: ${response.status} ${response.statusText}`)
    }

    const lockData = await response.json() as LockFileResponse
    console.info('Lock file data:', lockData)
    
    data.value = lockData

    // initialize form defaults from inputs
    if (lockData?.inputs) {
      // Handle both array and object formats for inputs
      if (Array.isArray(lockData.inputs)) {
        // Inputs is an array format
        for (const port of lockData.inputs) {
          const name = port.name
          // Check if this type is defined in the types field (custom type)
          if (lockData.types && port.type && lockData.types[port.type]) {
            // Extract the schema from types and use it as the default value
            const schema = lockData.types[port.type]
            // Stringify the schema for display in the textarea
            form[name] = JSON.stringify(schema, null, 2)
          } else {
            // Use the default value from the lock file if available, otherwise use type-based default
            let defaultValue = port.default !== null && port.default !== undefined ? port.default : defaultForType(port.type)
            
            // For JSON types, convert objects/arrays to JSON strings for the textarea
            if (port.type === 'json' && typeof defaultValue === 'object' && defaultValue !== null) {
              defaultValue = JSON.stringify(defaultValue, null, 2)
            }
            
            form[name] = defaultValue
          }
        }
      } else {
        // Inputs is an object format
        for (const [name, port] of Object.entries(lockData.inputs)) {
          // Check if this type is defined in the types field (custom type)
          if (lockData.types && port.type && lockData.types[port.type]) {
            // Extract the schema from types and use it as the default value
            const schema = lockData.types[port.type]
            // Stringify the schema for display in the textarea
            form[name] = JSON.stringify(schema, null, 2)
          } else {
            // Use the default value from the lock file if available, otherwise use type-based default
            let defaultValue = port.default !== null && port.default !== undefined ? port.default : defaultForType(port.type)
            
            // For JSON types, convert objects/arrays to JSON strings for the textarea
            if (port.type === 'json' && typeof defaultValue === 'object' && defaultValue !== null) {
              defaultValue = JSON.stringify(defaultValue, null, 2)
            }
            
            form[name] = defaultValue
          }
        }
      }
    }
  } catch (error) {
    console.error('Error fetching lock file:', error)
    errorMsg.value = error instanceof Error ? error.message : 'Failed to fetch lock file'
  }

  loading.value = false
}

// Derived lists for rendering
const inputs = computed(() => {
  if (!data.value?.inputs) return []
  // Handle both array and object formats for inputs
  if (Array.isArray(data.value.inputs)) {
    return data.value.inputs.map((port) => ({
      name: port.name,
      description: port.description,
      type: port.type,
      required: port.required,
      default: port.default
    }))
  }
  return Object.entries(data.value.inputs).map(([name, port]) => ({
    name,
    description: port.description,
    type: port.type,
    required: port.required,
    default: port.default
  }))
})
const outputs = computed(() => {
  if (!data.value?.outputs) return []
  return Object.entries(data.value.outputs).map(([name, port]) => ({
    name,
    ...port
  }))
})

// Check if a type is defined in the lock file's types
function isCustomType(type: string): boolean {
  return !!(data.value?.types && data.value.types[type])
}

// Optional: simple per-field hint from type
function placeholderFor(p: LockFilePort & { name: string }) {
  // If it's a custom type, use a specific placeholder
  if (isCustomType(p.type)) {
    return `Enter ${p.name} (custom type: ${p.type})…`
  }
  
  switch (p.type) {
    case 'string': return `Enter ${p.name}…`
    case 'number': return `Enter number for ${p.name}…`
    case 'boolean': return ''
    case 'json': return `Paste JSON for ${p.name}…`
    case 'type': return `Enter type for ${p.name}…`
    default: return `Enter ${p.name}…`
  }
}

function coerceValue(port: LockFilePort, raw: any) {
  // If already an object/array, return as-is
  if (typeof raw === 'object' && raw !== null) return raw
  
  // Try to parse as JSON first (if it's a string)
  if (typeof raw === 'string') {
    try {
      return JSON.parse(raw)
    } catch {
      // JSON parse failed, continue to type-specific parsing
    }
  }
  
  // Try to parse as number
  if (port.type === 'number') {
    if (raw === '' || raw === null || raw === undefined) return null
    const n = Number(raw)
    if (Number.isFinite(n)) return n
    return raw // Keep original if not a valid number
  }
  
  // Try to parse as boolean
  if (port.type === 'boolean') {
    if (raw === true || raw === false) return raw
    if (raw === 'true' || raw === '1') return true
    if (raw === 'false' || raw === '0') return false
    return raw // Keep original if not a valid boolean
  }
  
  // For everything else, return as-is
  return raw
}

async function onSubmit() {
  if (!data.value) return

  // Build payload as array of properly typed input values
  const payload: Array<any> = []
  const errors: string[] = []

  // Process inputs in order to match server expectations
  for (const p of inputs.value) {
    const raw = form[p.name]
    
    // Validate required fields
    if (p.required && (raw === null || raw === undefined || raw === '')) {
      errors.push(`Required field '${p.name}' is missing`)
      continue
    }
    
    // Coerce value to proper type based on port type
    const coercedValue = coerceValue(p, raw)
    
    // For required fields, ensure we have a valid value after coercion
    if (p.required && coercedValue === null && p.type !== 'number') {
      errors.push(`Required field '${p.name}' has an invalid value`)
      continue
    }
    
    payload.push(coercedValue)
  }

  if (errors.length) {
    errorMsg.value = errors.join('\n')
    return
  }

  // Compose action ref like "namespace/slug:version"
  const namespace = String(route.params.namespace ?? '')
  const actionSlug = String(route.params.slug ?? '')
  const version =
    (route.params.version as string | undefined)
    ?? (route.query.v as string | undefined)
    ?? null

  const actionRef = version
    ? `${namespace}/${actionSlug}:${version}`
    : `${namespace}/${actionSlug}`

  // Build request body matching server's expected format
  // Server expects: { action: string, inputs: Vec<Value> }
  // where inputs is an array of properly typed JSON values
  const body = {
    action: actionRef,
    inputs: payload,  // Array of properly typed JSON values (not strings)
  }

  console.log('🔍 Sending to /api/run:', {
    action: actionRef,
    inputs: payload,
    inputsType: payload.map(v => typeof v),
  })

  try {
    submitting.value = true
    errorMsg.value = null

    console.log('🔍 Sending to /api/run:', body)
    const resp = await fetch('http://localhost:3000/api/run', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    })

    if (!resp.ok) {
      const txt = await resp.text()
      throw new Error(`Run failed: ${resp.status} ${txt}`)
    }

    const result = await resp.json()
    console.info('✅ Action submitted successfully:', result)
    // success UX — swap with your toast system if you have one
    // alert('Dispatched! Check your workflow.')
  } catch (e: any) {
    console.error('❌ Error submitting action:', e)
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
        {{ data.name ?? '(unnamed action)' }}:{{ data.version }}
      </h2>
      <p class="text-gray-600 mb-4">{{ data.description }}</p>

      <form class="space-y-4" @submit.prevent="onSubmit">
        <h3 class="font-medium">Inputs</h3>

        <div v-if="!inputs.length" class="text-gray-500">No inputs.</div>

        <div v-for="p in inputs" :key="p.name" class="flex flex-col gap-1">
          <label class="font-medium">
            {{ p.name }}
            <span class="text-xs text-gray-500">({{ p.type }})</span>
            <span v-if="p.required" class="text-xs text-red-500">*</span>
          </label>
          <p v-if="p.description" class="text-xs text-gray-600">{{ p.description }}</p>

          <!-- STRING -->
          <input
            v-if="p.type === 'string'"
            type="text"
            class="border rounded px-3 py-2"
            :placeholder="placeholderFor(p)"
            v-model="form[p.name]"
          />

          <!-- NUMBER -->
          <input
            v-else-if="p.type === 'number'"
            type="number"
            class="border rounded px-3 py-2"
            :placeholder="placeholderFor(p)"
            :value="form[p.name]"
            @input="form[p.name] = ($event.target as HTMLInputElement).value === '' ? null : ($event.target as HTMLInputElement).value"
          />

          <!-- BOOLEAN -->
          <label v-else-if="p.type === 'boolean'" class="inline-flex items-center gap-2">
            <input
              type="checkbox"
              class="h-4 w-4"
              v-model="form[p.name]"
            />
            <span>True?</span>
          </label>

          <!-- JSON -->
          <textarea
            v-else-if="p.type === 'json'"
            class="border rounded px-3 py-2 font-mono"
            rows="6"
            :placeholder="placeholderFor(p)"
            v-model="form[p.name]"
          ></textarea>

          <!-- TYPE -->
          <input
            v-else-if="p.type === 'type'"
            type="text"
            class="border rounded px-3 py-2"
            :placeholder="placeholderFor(p)"
            v-model="form[p.name]"
          />

          <!-- CUSTOM TYPE (from lock file types) -->
          <textarea
            v-else-if="isCustomType(p.type)"
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
        <li v-for="o in outputs" :key="o.name">
          <div class="flex flex-col">
            <span class="font-medium">{{ o.name }}</span>
            <span class="text-xs text-gray-500">({{ o.type }})</span>
            <span v-if="o.description" class="text-xs text-gray-600">{{ o.description }}</span>
          </div>
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
