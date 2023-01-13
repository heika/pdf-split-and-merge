<template>
  <div>
    <input type="file" @change="addFile" accept="application/pdf" multiple />
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
        this.$parent.addFile(await files[i].arrayBuffer());
      }
      e.target.value = '';
    },
  },
};
</script>

<style scoped></style>
