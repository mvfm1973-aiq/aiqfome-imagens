<template>
  <v-card class="mt-4 pa-4" outlined>
    <h2 class="mb-2">Gerenciar artes — versão 2.0</h2>
    <p class="caption">Selecione a região da logomarca na imagem. O token é mantido apenas na memória desta página.</p>
    <v-text-field v-model="token" type="password" label="Token GitHub (Contents: leitura e escrita)" autocomplete="off" outlined dense />
    <v-alert type="info" dense text>Utilize um token fine-grained restrito a este repositório. Não compartilhe o token.</v-alert>
    <v-btn color="purple" dark :loading="busy" @click="load">Carregar artes</v-btn>
    <v-alert v-if="message" :type="error ? 'error' : 'success'" class="mt-3" dense>{{ message }}</v-alert>
    <v-divider class="my-5" />
    <h3>Adicionar nova arte</h3>
    <v-text-field v-model="name" label="Nome da arte (ex.: promocao-pizza)" outlined dense />
    <v-file-input v-model="original" accept="image/png" label="Imagem original PNG" outlined dense @change="prepareImage" />
    <div v-if="previewUrl">
      <p class="caption">Arraste sobre a imagem para marcar a área da logomarca. A máscara será gerada com o retângulo verde #60FC1C.</p>
      <div class="selection-wrap" :style="{ maxWidth: imageWidth + 'px' }">
        <img ref="preview" :src="previewUrl" class="selection-image" draggable="false" @pointerdown="startSelection" @pointermove="moveSelection" @pointerup="finishSelection" @pointercancel="finishSelection" />
        <div v-if="selection" class="selection-rect" :style="selectionStyle"></div>
      </div>
      <p class="caption mt-2" v-if="selection">Área selecionada: {{ Math.round(selection.w) }} × {{ Math.round(selection.h) }} pixels</p>
      <v-btn small text color="primary" @click="selection=null">Refazer seleção</v-btn>
    </div>
    <v-btn class="mt-3" color="success" :loading="busy" :disabled="busy || !token || !original || !selection || !name.trim()" @click="add">Gerar máscara e adicionar arte</v-btn>
    <v-divider class="my-5" />
    <h3>Artes cadastradas ({{ files.length }})</h3>
    <v-text-field v-model="search" label="Pesquisar pelo nome" prepend-inner-icon="mdi-magnify" dense outlined />
    <v-row>
      <v-col v-for="item in filtered" :key="item.name" cols="6" sm="4" md="3">
        <v-card outlined>
          <v-img :src="item.download_url" aspect-ratio="1.5" contain />
          <v-card-text class="text-truncate">{{ item.name }}</v-card-text>
          <v-card-actions><v-btn color="red" text small :loading="busy" @click="remove(item)"><v-icon left small>mdi-delete</v-icon>Excluir</v-btn></v-card-actions>
        </v-card>
      </v-col>
    </v-row>
  </v-card>
</template>

<script>
const owner = 'mvfm1973-aiq'
const repo = 'aiqfome-imagens'
const base = `https://api.github.com/repos/${owner}/${repo}/contents/src/assets/resources/`

export default {
  name: 'AdminImages',
  data: () => ({
    token: '', name: '', original: null, files: [], search: '', busy: false,
    message: '', error: false, previewUrl: null, imageWidth: 0, imageHeight: 0,
    selection: null, dragging: false, anchor: null
  }),
  computed: {
    filtered () { return this.files.filter(f => f.name.toLowerCase().includes(this.search.toLowerCase())) },
    selectionStyle () {
      if (!this.selection || !this.$refs.preview) return {}
      const rect = this.$refs.preview.getBoundingClientRect()
      const sx = rect.width / this.imageWidth
      const sy = rect.height / this.imageHeight
      return {
        left: `${this.selection.x * sx}px`, top: `${this.selection.y * sy}px`,
        width: `${this.selection.w * sx}px`, height: `${this.selection.h * sy}px`
      }
    }
  },
  beforeDestroy () { this.releasePreview() },
  methods: {
    releasePreview () {
      if (this.previewUrl) URL.revokeObjectURL(this.previewUrl)
      this.previewUrl = null
    },
    async request (path, method = 'GET', body) {
      if (!this.token) throw new Error('Informe o token de acesso.')
      const res = await fetch(base + path, {
        method,
        headers: {
          Authorization: `Bearer ${this.token}`,
          Accept: 'application/vnd.github+json',
          'X-GitHub-Api-Version': '2022-11-28',
          ...(body ? { 'Content-Type': 'application/json' } : {})
        },
        ...(body ? { body: JSON.stringify(body) } : {})
      })
      if (!res.ok) {
        let info = {}
        try { info = await res.json() } catch (e) { info = {} }
        throw new Error(info.message || `GitHub: erro ${res.status}`)
      }
      return res.json()
    },
    async perform (fn) {
      this.busy = true
      this.message = ''
      try { await fn(); this.error = false } catch (e) { this.error = true; this.message = e.message } finally { this.busy = false }
    },
    async load () {
      await this.perform(async () => {
        const items = await this.request('original')
        this.files = items.filter(f => f.type === 'file' && f.name.toLowerCase().endsWith('.png'))
        this.message = `${this.files.length} artes carregadas.`
      })
    },
    prepareImage (file) {
      this.releasePreview()
      this.selection = null
      this.imageWidth = 0
      this.imageHeight = 0
      if (!file || Array.isArray(file)) return
      if (file.type !== 'image/png') { this.error = true; this.message = 'Selecione uma imagem PNG.'; return }
      if (file.size > 5000000) { this.error = true; this.message = 'O limite é 5 MB por imagem.'; return }
      const url = URL.createObjectURL(file)
      const img = new Image()
      img.onload = () => {
        if (this.previewUrl !== url) return
        this.imageWidth = img.naturalWidth
        this.imageHeight = img.naturalHeight
      }
      img.onerror = () => { if (this.previewUrl === url) { this.releasePreview(); this.error = true; this.message = 'PNG inválido.' } }
      this.previewUrl = url
      img.src = url
    },
    pointerPosition (event) {
      const rect = this.$refs.preview.getBoundingClientRect()
      return {
        x: Math.max(0, Math.min(this.imageWidth, (event.clientX - rect.left) * this.imageWidth / rect.width)),
        y: Math.max(0, Math.min(this.imageHeight, (event.clientY - rect.top) * this.imageHeight / rect.height))
      }
    },
    startSelection (event) {
      if (!this.imageWidth || !this.imageHeight) return
      event.preventDefault()
      this.$refs.preview.setPointerCapture(event.pointerId)
      this.dragging = true
      this.anchor = this.pointerPosition(event)
      this.selection = null
    },
    moveSelection (event) {
      if (!this.dragging) return
      const p = this.pointerPosition(event)
      this.selection = {
        x: Math.min(this.anchor.x, p.x), y: Math.min(this.anchor.y, p.y),
        w: Math.abs(p.x - this.anchor.x), h: Math.abs(p.y - this.anchor.y)
      }
    },
   
finishSelection (event) {
  if (!this.dragging) return

  this.moveSelection(event)
  this.dragging = false

  const preview = this.$refs.preview
  if (preview && preview.hasPointerCapture(event.pointerId)) {
    preview.releasePointerCapture(event.pointerId)
  }

  if (!this.selection ||
      this.selection.w < 2 ||
      this.selection.h < 2) {
    this.selection = null
    this.message = 'Seleção inválida. Arraste na diagonal para marcar a área.'
    this.error = true
    return
  }

  this.message = ''
},
      async fileBase64 (file) {
      const bytes = new Uint8Array(await file.arrayBuffer())
      let binary = ''
      for (let i = 0; i < bytes.length; i += 8192) binary += String.fromCharCode(...bytes.subarray(i, i + 8192))
      return btoa(binary)
    },
    async generateMask () {
      const canvas = document.createElement('canvas')
      canvas.width = this.imageWidth
      canvas.height = this.imageHeight
      const ctx = canvas.getContext('2d')
      // White background with the selected logo region painted green.
      ctx.fillStyle = '#FFFFFF'
      ctx.fillRect(0, 0, canvas.width, canvas.height)
      ctx.fillStyle = '#60FC1C'
      const s = this.selection
      ctx.fillRect(Math.round(s.x), Math.round(s.y), Math.round(s.w), Math.round(s.h))
      const blob = await new Promise(resolve => canvas.toBlob(resolve, 'image/png'))
      if (!blob) throw new Error('Não foi possível gerar a máscara.')
      return new File([blob], 'mask.png', { type: 'image/png' })
    },
    async add () {
      await this.perform(async () => {
        if (!this.original || this.original.type !== 'image/png' || this.original.size > 5000000) throw new Error('Selecione um PNG de até 5 MB.')
        if (!this.selection || !this.imageWidth || !this.imageHeight) throw new Error('Selecione a área da logomarca.')
        const filename = this.name.trim().toLowerCase().replace(/[^a-z0-9_-]+/g, '-').replace(/^-|-$/g, '') + '.png'
        if (filename === '.png') throw new Error('Informe um nome válido.')
        const existing = await this.request('original')
        if (existing.some(f => f.name.toLowerCase() === filename.toLowerCase())) throw new Error('Já existe uma arte com esse nome.')
        const mask = await this.generateMask()
        const originalContent = await this.fileBase64(this.original)
        const maskContent = await this.fileBase64(mask)
        const path = encodeURIComponent(filename)
        const saved = await this.request('original/' + path, 'PUT', { message: `Adicionar arte ${filename}`, content: originalContent })
        try {
          await this.request('masks/' + path, 'PUT', { message: `Adicionar máscara ${filename}`, content: maskContent })
        } catch (e) {
          // Roll back the original when the mask upload fails, so the library stays paired.
          try {
            await this.request('original/' + path, 'DELETE', { message: `Reverter arte incompleta ${filename}`, sha: saved.content.sha })
          } catch (rollbackError) {
            throw new Error(`Máscara não foi salva: ${e.message}. A arte original permaneceu no GitHub; exclua-a manualmente antes de tentar novamente.`)
          }
          throw new Error(`Não foi possível salvar a máscara: ${e.message}. A arte original foi revertida.`)
        }
        this.original = null
        this.name = ''
        this.selection = null
        this.releasePreview()
        const items = await this.request('original')
        this.files = items.filter(f => f.type === 'file' && f.name.toLowerCase().endsWith('.png'))
        this.message = 'Arte e máscara cadastradas. Aguarde a atualização automática do site.'
      })
    },
    async remove (item) {
      if (!confirm(`Excluir definitivamente ${item.name} e sua máscara?`)) return
      await this.perform(async () => {
        const path = encodeURIComponent(item.name)
        const orig = await this.request('original/' + path)
        let mask = null
        try { mask = await this.request('masks/' + path) } catch (e) { if (!e.message.includes('Not Found') && !e.message.includes('404')) throw e }
        if (mask) await this.request('masks/' + path, 'DELETE', { message: `Excluir máscara ${item.name}`, sha: mask.sha })
        await this.request('original/' + path, 'DELETE', { message: `Excluir arte ${item.name}`, sha: orig.sha })
        this.files = this.files.filter(f => f.name !== item.name)
        this.message = 'Arte excluída. Aguarde a atualização automática do site.'
      })
    }
  }
}
</script>

<style scoped>
.selection-wrap { position: relative; width: 100%; user-select: none; }
.selection-image { display: block; width: 100%; height: auto; touch-action: none; cursor: crosshair; }
.selection-rect { position: absolute; border: 2px solid #60FC1C; background: rgba(96,252,28,.28); pointer-events: none; box-sizing: border-box; }
</style>
