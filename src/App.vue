<template>
  <v-app>
    <v-main>
      <GroupJoin />
    </v-main>
    <v-footer>
      <v-row justify="center" no-gutters>
        <v-btn v-for="link in links" :key="link.href" :href="link.href" color="primary" variant="outlined" class="mx-2"
          rounded="0" :prepend-icon="link.icon">
          {{ link.text }}
        </v-btn>
        <v-col class="text-center mt-4" cols="12">
          <p>
            {{ new Date().getFullYear() }} — <strong>Bellingcat</strong> 
          </p>
          <small>No data is stored on our servers. This is a client-side application.</small>
        </v-col>
      </v-row></v-footer>
  </v-app>
</template>

<script setup>
import GroupJoin from '@/components/GroupJoin.vue'
const links = [
  { text: 'Code on GitHub', href: 'https://github.com/bellingcat/telegram-group-joiner', icon: 'mdi-github'},
  { text: 'Bellingcat.com', href: 'https://www.bellingcat.com', icon: 'mdi-earth' },
]
</script>
