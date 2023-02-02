<template>
  <div id="uploaded-files" v-if="pages.length > 0">
    <button v-if="pages.length > 0" v-on:click="merge">Merge</button>
    <ul>
      <li v-for="(page, index) in pages">
        <input type="checkbox" v-model="page.selected" />
        <iframe :src="page.content" />
        <!--iframe
          :src="page.content"
          :height="page.size.height"
          :width="page.size.width"
        /-->
        <button v-on:click="deletePage(index)">Delete</button>
      </li>
    </ul>
  </div>
</template>

<script>
import { PDFDocument } from 'pdf-lib';

export default {
  name: 'Result',
  props: ['file'],
  data() {
    return {
      pages: [],
    };
  },
  watch: {
    file: {
      handler: async function (newValue, oldValue) {
        if (newValue != null && newValue != '')
          this.pages = [...this.pages, ...(await this.getSrc(newValue))];
      },
      deep: true,
    },
  },
  methods: {
    async extractPdfPage(arrayBuff) {
      const pdfSrcDoc = await PDFDocument.load(arrayBuff);

      const noOfPage = pdfSrcDoc.getPageCount();
      let pdfPages = [];

      for (let i = 0; i < noOfPage; i++) {
        const pdfNewDoc = await PDFDocument.create();
        const docPages = await pdfNewDoc.copyPages(pdfSrcDoc, [i]);
        let size = {};
        docPages.forEach(function (page) {
          pdfNewDoc.addPage(page);
          size = page.getSize();
        });
        const newpdf = await pdfNewDoc.save();
        pdfPages.push({
          pdf: newpdf,
          size: size,
        });
      }

      return pdfPages;
    },
    async getSrc(file) {
      const newPdfDoc = await this.extractPdfPage(file);
      let docUrls = [];
      for (let i = 0; i < newPdfDoc.length; i++) {
        const tempblob = new Blob([newPdfDoc[i].pdf], {
          type: 'application/pdf',
        });
        const docUrl = URL.createObjectURL(tempblob);
        docUrls.push({
          selected: true,
          content: `${docUrl}#toolbar=0&navpanes=0&scrollbar=0&zoom=FitW`,
          size: newPdfDoc[i].size,
        });
      }

      return docUrls;
    },
    deletePage(index) {
      this.pages.splice(index, 1);
    },
    async merge() {
      const selectedPages = this.pages.filter((page) => {
        return page.selected == true;
      });

      const pdfNewDoc = await PDFDocument.create();

      for (let i = 0; i < selectedPages.length; i++) {
        let url = selectedPages[i].content;
        let blob = await fetch(url).then((r) => r.blob());
        let arrayBuffer = await new Response(blob).arrayBuffer();
        const selectedPdf = await PDFDocument.load(arrayBuffer);

        const [existingPage] = await pdfNewDoc.copyPages(selectedPdf, [0]);
        pdfNewDoc.addPage(existingPage);
      }

      const newpdf = await pdfNewDoc.save();

      const mergedBlob = new Blob([newpdf], {
        type: 'application/pdf',
      });

      const link = document.createElement('a');
      link.href = window.URL.createObjectURL(mergedBlob);
      link.download = 'merged.pdf';
      link.click();
      window.URL.revokeObjectURL(link.href);
    },
  },
};
</script>

<style scoped>
iframe {
  width: 100%;
}
</style>
