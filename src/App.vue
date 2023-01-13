<template>
  <div id="app">
    <Panel />
    <Result :file="file" />
  </div>
</template>

<script>
import { PDFDocument } from 'pdf-lib';

import Panel from './components/Panel.vue';
import Result from './components/Result.vue';

export default {
  name: 'App',
  data() {
    return {
      file: null,
    };
  },
  components: {
    Panel,
    Result,
  },
  methods: {
    async addFile(file) {
      this.file = file;
    },
  },
};
</script>

<style></style>
