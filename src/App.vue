
<template>
  <v-app>
    <v-app-bar app color="blue darken-1" dark>
      <div class="d-flex align-start">
        <v-img
          alt="aiqimagens"
          class="shrink mr-2"
          contain
          src="@/assets/logoaiq.png"
          height="50"
          width="170"
        />
      </div>

      <v-spacer></v-spacer>

      <v-btn
        color="purple"
        dark
        class="mr-2"
        @click="showAdmin = !showAdmin"
      >
        <v-icon left>mdi-cog</v-icon>
        {{ showAdmin ? 'Voltar ao gerador' : 'Gerenciar artes' }}
      </v-btn>

      <v-btn
        href="https://github.com/mvfm1973-aiq/aiqfome-imagens"
        dark
        target="_blank"
        color="primary"
        elevation="1"
      >
        <v-icon>mdi-github</v-icon>
      </v-btn>
    </v-app-bar>

    <v-main>
      <v-container v-if="showAdmin" fluid>
        <AdminImages />
      </v-container>

      <template v-else>
        <v-container fluid class="grey lighten-5">
          <h1 class="grey--text light pa-4 pb-0 mb-0">
            1. Envie uma logo
          </h1>

          <v-row no-gutters>
            <v-col cols="12" sm="7">
              <v-container class="pa-2">
                <Uploader />
              </v-container>
            </v-col>

            <v-col cols="12" sm="5">
              <v-container class="pa-2">
                <ImageCrop />
              </v-container>
            </v-col>
          </v-row>
        </v-container>

        <Images />
      </template>
    </v-main>
  </v-app>
</template>

<script>
import Uploader from './components/Uploader';
import ImageCrop from './components/ImageCrop';
import Images from './components/Images';
import AdminImages from './components/AdminImages';

export default {
  name: 'App',

  components: {
    Uploader,
    ImageCrop,
    Images,
    AdminImages
  },

  data() {
    return {
      showAdmin: false
    };
  },

  created() {
    document.title = 'aiq imagens';
  }
};
</script>

<style>
.v-main {
  width: 100%;
  max-width: 1000px;
  margin: 0 auto;
}
</style>
