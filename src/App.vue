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
    uuidv4() {
      return ([1e7] + -1e3 + -4e3 + -8e3 + -1e11).replace(/[018]/g, (c) =>
        (
          c ^
          (crypto.getRandomValues(new Uint8Array(1))[0] & (15 >> (c / 4)))
        ).toString(16)
      );
    },
  },
};
</script>

<style>
body {
  margin: 0;
  padding: 0;
  font-family: Verdana, Tahoma;
  position: relative;
}

div:has(> #uploaded-files) .upload-container {
  height: 100px;
}
</style>
