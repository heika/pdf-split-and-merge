<template>
  <div id="app">
    <Panel />
    <Result :file="file" />
  </div>
</template>

<script>
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
:root {
  --main-result-width: 500px;
}
body {
  margin: 0;
  padding: 0;
  font-family: Verdana, Tahoma;
  position: relative;
}
div:has(> #uploaded-files) .upload-container {
  height: 100px;
}
button {
  border: 0;
  padding: 10px 20px;
  border-radius: 10px;
  margin: 10px;
}
button.btn-primary {
  background-color: #ff7474;
  color: #fff;
}
</style>
