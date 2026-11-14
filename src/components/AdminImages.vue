
<template>
  <v-card class="mt-4 pa-4" outlined>
    <h2 class="mb-2">Gerenciar artes</h2>
    <p class="caption">Acesso exclusivo a quem possui autorização no GitHub. O token não é salvo no site nem no navegador.</p>
    <v-text-field v-model="token" type="password" label="Token de acesso GitHub (permissão Contents: leitura e escrita)" autocomplete="off" outlined dense />
    <v-alert type="info" dense text>Use um token de acesso refinado (fine-grained), limitado a este repositório. Nunca compartilhe o token.</v-alert>
    <v-btn color="purple" dark :loading="busy" @click="load">Carregar artes</v-btn>
    <v-alert v-if="message" :type="error ? 'error' : 'success'" class="mt-3" dense>{{ message }}</v-alert>
    <v-divider class="my-5" />
    <h3>Adicionar nova arte</h3>
    <p class="caption">Cada arte precisa de uma imagem original e sua máscara PNG, com o mesmo nome e dimensões. A máscara usa verde #60FC1C na área da logomarca.</p>
    <v-text-field v-model="name" label="Nome da arte (ex.: promocao-pizza)" outlined dense />
    <v-file-input v-model="original" accept="image/png" label="Imagem original PNG" outlined dense />
    <v-file-input v-model="mask" accept="image/png" label="Máscara PNG" outlined dense />
    <v-btn color="success" :loading="busy" :disabled="!token || !original || !mask || !name" @click="add">Adicionar arte</v-btn>
    <v-divider class="my-5" />
    <h3>Artes cadastradas ({{ files.length }})</h3>
    <v-text-field v-model="search" label="Pesquisar pelo nome" prepend-inner-icon="mdi-magnify" dense outlined />
    <v-row><v-col v-for="item in filtered" :key="item.name" cols="6" sm="4" md="3">
      <v-card outlined><v-img :src="item.download_url" aspect-ratio="1.5" contain /><v-card-text class="text-truncate">{{ item.name }}</v-card-text>
      <v-card-actions><v-btn color="red" text small :loading="busy" @click="remove(item)"><v-icon left small>mdi-delete</v-icon>Excluir</v-btn></v-card-actions></v-card>
    </v-col></v-row>
  </v-card>
</template>
<script>
const owner = 'mvfm1973-aiq', repo = 'aiqfome-imagens'
const base = `https://api.github.com/repos/${owner}/${repo}/contents/src/assets/resources/`
export default {
  data: () => ({token:'',name:'',original:null,mask:null,files:[],search:'',busy:false,message:'',error:false}),
  computed: {filtered(){return this.files.filter(f=>f.name.toLowerCase().includes(this.search.toLowerCase()))}},
  methods: {
    
    async request(path, method = 'GET', body) {
      if (!this.token) {
        throw Error('Informe o token de acesso.')
      }

      const headers = {
        Authorization: `Bearer ${this.token}`,
        Accept: 'application/vnd.github+json',
        'X-GitHub-Api-Version': '2022-11-28'
      }

      if (body) {
        headers['Content-Type'] = 'application/json'
      }

      const options = {
        method,
        headers
      }

      if (body) {
        options.body = JSON.stringify(body)
      }

      const res = await fetch(base + path, options)

      if (!res.ok) {
        let info = {}

        try {
          info = await res.json()
        } catch (e) {
          info = {
            message: 'Erro ao consultar GitHub'
          }
        }

        throw Error(
          info.message || `GitHub: erro ${res.status}`
        )
      }

      return res.json()
    },
    async perform(fn){this.busy=true;this.message='';try{await fn();this.error=false}catch(e){this.error=true;this.message=e.message}finally{this.busy=false}},
    async load(){await this.perform(async()=>{const items=await this.request('original');this.files=items.filter(f=>f.type==='file' && f.name.toLowerCase().endsWith('.png'));this.message=`${this.files.length} artes carregadas.`})},
    async fileBase64(file){const buffer=await file.arrayBuffer();const bytes=new Uint8Array(buffer);let binary='';for(let i=0;i<bytes.length;i+=8192)binary+=String.fromCharCode(...bytes.subarray(i,i+8192));return btoa(binary)},
    async dimensions(file){return new Promise((resolve,reject)=>{const u=URL.createObjectURL(file),img=new Image();img.onload=()=>{URL.revokeObjectURL(u);resolve([img.naturalWidth,img.naturalHeight])};img.onerror=()=>{URL.revokeObjectURL(u);reject(Error('PNG inválido'))};img.src=u})},
    async add(){await this.perform(async()=>{
      const filename=this.name.trim().toLowerCase().replace(/[^a-z0-9_-]+/g,'-').replace(/^-|-$/g,'')+'.png'
      if(filename==='.png')throw Error('Informe um nome válido.')
      if(this.original.type!=='image/png'||this.mask.type!=='image/png')throw Error('Os dois arquivos devem ser PNG.')
      if(this.original.size>5000000||this.mask.size>5000000)throw Error('Limite de 5 MB por imagem.')
      const [a,b]=await Promise.all([this.dimensions(this.original),this.dimensions(this.mask)])
      if(a[0]!==b[0]||a[1]!==b[1])throw Error('Original e máscara devem ter dimensões iguais.')
      const existing=await this.request('original');if(existing.some(f=>f.name===filename))throw Error('Já existe uma arte com esse nome.')
      const payload={message:`Adicionar arte ${filename}`,content:await this.fileBase64(this.original)}
      await this.request('original/'+encodeURIComponent(filename),'PUT',payload)
      try{await this.request('masks/'+encodeURIComponent(filename),'PUT',{message:`Adicionar máscara ${filename}`,content:await this.fileBase64(this.mask)})}
      catch(e){this.message=`Original salvo, mas a máscara falhou: ${e.message}. Complete a máscara antes de usar.`;throw Error(this.message)}
      this.original=null;this.mask=null;this.name='';const items=await this.request('original');this.files=items.filter(f=>f.type==='file'&&f.name.endsWith('.png'));this.message='Arte adicionada. Aguarde a atualização automática do site (Actions).'
    })},
    async remove(item){if(!confirm(`Excluir definitivamente ${item.name} e sua máscara?`))return
      await this.perform(async()=>{
        const orig=await this.request('original/'+encodeURIComponent(item.name))
        let mask=null;try{mask=await this.request('masks/'+encodeURIComponent(item.name))}catch(e){if(!e.message.includes('Not Found'))throw e}
        if(mask)await this.request('masks/'+encodeURIComponent(item.name),'DELETE',{message:`Excluir máscara ${item.name}`,sha:mask.sha})
        await this.request('original/'+encodeURIComponent(item.name),'DELETE',{message:`Excluir arte ${item.name}`,sha:orig.sha})
        this.files=this.files.filter(f=>f.name!==item.name);this.message='Arte excluída. Aguarde a atualização automática do site.'
      })
    }
  }
}
</script>
