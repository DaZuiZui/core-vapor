<script setup lang="ts">
import Header from './Header.vue'
import {
  Repl,
  type SFCOptions,
  useStore,
  useVueImportMap,
  mergeImportMap,
  File,
  StoreState,
  ImportMap,
} from '@vue/repl'
import type Monaco from '@vue/repl/monaco-editor'
import type CodeMirror from '@vue/repl/codemirror-editor'
import { ref, watchEffect, onMounted, computed, shallowRef, watch } from 'vue'
import welcomeSFC from './welcome.vue?raw'

const EditorComponent = shallowRef<typeof Monaco | typeof CodeMirror>()

if (import.meta.env.DEV) {
  import('@vue/repl/codemirror-editor').then(
    mod => (EditorComponent.value = mod.default),
  )
} else {
  import('@vue/repl/monaco-editor').then(
    mod => (EditorComponent.value = mod.default),
  )
}

const replRef = ref<InstanceType<typeof Repl>>()

const setVH = () => {
  document.documentElement.style.setProperty('--vh', window.innerHeight + `px`)
}
window.addEventListener('resize', setVH)
setVH()

const useSSRMode = ref(false)
const useVaporMode = ref(true)

const {
  vueVersion,
  productionMode,
  importMap: vueImportMap,
} = useVueImportMap({
  runtimeDev: import.meta.env.PROD
    ? `${location.origin}/vue.runtime.esm-browser.js`
    : `${location.origin}/src/vue-dev-proxy`,
  runtimeProd: import.meta.env.PROD
    ? `${location.origin}/vue.runtime.esm-browser.prod.js`
    : `${location.origin}/src/vue-dev-proxy-prod`,
  serverRenderer: import.meta.env.PROD
    ? `${location.origin}/server-renderer.esm-browser.js`
    : `${location.origin}/src/vue-server-renderer-dev-proxy`,
})

const importMap = computed(() => {
  const vapor = import.meta.env.PROD
    ? `${location.origin}/vue-vapor.esm-browser.js`
    : `${location.origin}/src/vue-vapor-dev-proxy`

  const vaporImportMap: ImportMap = {
    imports: {
      'vue/vapor': vapor,
    },
  }
  if (useVaporMode.value) vaporImportMap.imports!.vue = vapor

  return mergeImportMap(vueImportMap.value, vaporImportMap)
})

let hash = location.hash.slice(1)
if (hash.startsWith('__DEV__')) {
  hash = hash.slice(7)
  productionMode.value = false
}
if (hash.startsWith('__PROD__')) {
  hash = hash.slice(8)
  productionMode.value = true
}
if (hash.startsWith('__SSR__')) {
  hash = hash.slice(7)
  useSSRMode.value = true
}
if (hash.startsWith('__VAPOR__')) {
  hash = hash.slice(9)
  useVaporMode.value = true
}

const files: StoreState['files'] = ref(Object.create(null))

// enable experimental features
const sfcOptions = computed(
  (): SFCOptions => ({
    script: {
      inlineTemplate: productionMode.value,
      isProd: productionMode.value,
      propsDestructure: true,
    },
    style: {
      isProd: productionMode.value,
    },
    template: {
      vapor: useVaporMode.value,
      isProd: productionMode.value,
      compilerOptions: {
        isCustomElement: (tag: string) => tag === 'mjx-container',
      },
    },
  }),
)

const store = useStore(
  {
    files,
    vueVersion,
    builtinImportMap: importMap,
    sfcOptions,
    template: ref({
      welcomeSFC: welcomeSFC,
    }),
  },
  hash,
)
// @ts-expect-error
globalThis.store = store

watch(
  useVaporMode,
  () => {
    if (useVaporMode.value) {
      files.value['src/index.html'] = new File(
        'src/index.html',
        `<script type="module">
        import { render } from 'vue/vapor'
        import App from './App.vue'
        render(App, {}, '#app')` +
          '<' +
          '/script>' +
          `<div id="app"></div>`,
        true,
      )
      store.mainFile = 'src/index.html'
    } else if (files.value['src/index.html']?.hidden) {
      delete files.value['src/index.html']
      store.mainFile = 'src/App.vue'
      if (store.activeFile.filename === 'src/index.html') {
        store.activeFile = files.value['src/App.vue']
      }
    }
  },
  { immediate: true },
)

// persist state
watchEffect(() => {
  const newHash = store
    .serialize()
    .replace(/^#/, useVaporMode.value ? `#__VAPOR__` : `#`)
    .replace(/^#/, useSSRMode.value ? `#__SSR__` : `#`)
    .replace(/^#/, productionMode.value ? `#__PROD__` : `#`)
  history.replaceState({}, '', newHash)
})

function toggleProdMode() {
  productionMode.value = !productionMode.value
}

function toggleSSR() {
  useSSRMode.value = !useSSRMode.value
}

function toggleVapor() {
  useVaporMode.value = !useVaporMode.value
}

function reloadPage() {
  replRef.value?.reload()
}

const theme = ref<'dark' | 'light'>('dark')
function toggleTheme(isDark: boolean) {
  theme.value = isDark ? 'dark' : 'light'
}
onMounted(() => {
  const cls = document.documentElement.classList
  toggleTheme(cls.contains('dark'))
})
</script>

<template>
  <Header
    :store="store"
    :prod="productionMode"
    :ssr="useSSRMode"
    :vapor="useVaporMode"
    @toggle-theme="toggleTheme"
    @toggle-prod="toggleProdMode"
    @toggle-ssr="toggleSSR"
    @toggle-vapor="toggleVapor"
    @reload-page="reloadPage"
  />
  <Repl
    v-if="EditorComponent"
    ref="replRef"
    :theme="theme"
    :editor="EditorComponent"
    @keydown.ctrl.s.prevent
    @keydown.meta.s.prevent
    :ssr="useSSRMode"
    :store="store"
    :showCompileOutput="true"
    :autoResize="true"
    :clearConsole="false"
    :preview-options="{
      customCode: {
        importCode: `import { initCustomFormatter } from 'vue'`,
        useCode: `initCustomFormatter()`,
      },
    }"
  />
</template>

<style>
.dark {
  color-scheme: dark;
}

body {
  font-size: 13px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  margin: 0;
  --base: #444;
  --nav-height: 50px;
}

.vue-repl {
  height: calc(var(--vh) - var(--nav-height)) !important;
}

button {
  border: none;
  outline: none;
  cursor: pointer;
  margin: 0;
  background-color: transparent;
}
</style>
