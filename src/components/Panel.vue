<template>
  <div class="upload-container">
    <label for="file_upload">Select files</label>
    <input
      type="file"
      id="file_upload"
      multiple
      @change="addFile"
      accept="application/pdf"
    />
  </div>
</template>

<script>
export default {
  name: 'Panel',
  props: {
    msg: String,
  },
  data() {
    return {
      file: [],
    };
  },
  methods: {
    async addFile(e) {
      var files = e.target.files || e.dataTransfer.files;

      for (let i = 0; i < files.length; i++) {
        if (files[i].type == 'application/pdf')
          this.$parent.addFile(await files[i].arrayBuffer());
      }
      e.target.value = '';
    },
  },
};
</script>

<style scoped>
.upload-container {
  position: sticky;
  width: 100%;
  height: 100vh;
}
:has(#file_list:not(:empty)) .upload-container {
  height: 50px;
  top: 0;
  z-index: 999;
}
.upload-container input {
  position: absolute;
  top: 0;
  left: 0;
  opacity: 0;
  z-index: 9999;
  width: 100%;
  height: 100%;
}
.upload-container label {
  display: flex;
  justify-content: center;
  align-items: center;
  background: #ffe1e1;
  border: 2px dashed #ff7474;
  height: 100%;
  box-sizing: border-box;
  font-size: 2em;
  color: #444;
}
</style>
