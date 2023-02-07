<template>
  <div id="uploaded-files" v-if="pages.length > 0">
    <canvas ref="canvas" />
    <button v-if="pages.length > 0" v-on:click="merge">Merge</button>
    <ul>
      <li v-for="(page, index) in pages" :ref="`page_${page.guid}`">
        <input
          type="checkbox"
          v-model="page.selected"
          :id="`cb_${page.guid}`"
        />
        <label :for="`cb_${page.guid}`">
          <img :src="page.url" />
        </label>
        <div class="button-set">
          <button v-on:click="move(1, page.guid)">↑</button>
          <button v-on:click="move(0, page.guid)">↓</button>
          <button v-on:click="rotate(page.guid)">⭯</button>
          <button v-on:click="deletePage(page.guid)">X</button>
        </div>
      </li>
    </ul>
  </div>
</template>

<script>
import { PDFDocument } from 'pdf-lib';
import * as pdfjsLib from 'pdfjs-dist';

export default {
  name: 'Result',
  props: ['file'],
  data() {
    return {
      pages: [],
      scale: 1,
    };
  },
  watch: {
    file: {
      handler: async function (newValue, oldValue) {
        if (newValue != null && newValue != '') {
          let prevLength = this.pages.length;
          let _this = this;

          this.pages = [
            ...this.pages,
            ...(await this.getSrc(newValue, this.pages.length)),
          ];

          for (let i = prevLength; i < this.pages.length; i++) {
            const loadingTask = await pdfjsLib
              .getDocument(this.pages[i].content)
              .promise.then(async function (pdfDoc_) {
                let pdfDoc = pdfDoc_;

                await _this.renderPage(pdfDoc, 1, _this.pages[i]);
              });
          }
        }
      },
      deep: true,
    },
  },
  created() {
    pdfjsLib.GlobalWorkerOptions.workerSrc =
      '//cdn.jsdelivr.net/npm/pdfjs-dist@3.3.122/build/pdf.worker.js';
    pdfjsLib.disableWorker = true;
  },
  methods: {
    async extractPdfPage(arrayBuff) {
      const pdfSrcDoc = await PDFDocument.load(arrayBuff);

      const noOfPage = pdfSrcDoc.getPageCount();
      let pdfPages = [];

      for (let i = 0; i < noOfPage; i++) {
        const pdfNewDoc = await PDFDocument.create();
        const docPages = await pdfNewDoc.copyPages(pdfSrcDoc, [i]);
        docPages.forEach(function (page) {
          pdfNewDoc.addPage(page);
        });
        const newpdf = await pdfNewDoc.save();
        pdfPages.push({
          pdf: newpdf,
        });
      }

      return pdfPages;
    },
    async getSrc(file, prevLength) {
      const newPdfDoc = await this.extractPdfPage(file);
      let docUrls = [];
      let _this = this;

      for (let i = 0; i < newPdfDoc.length; i++) {
        const tempblob = new Blob([newPdfDoc[i].pdf], {
          type: 'application/pdf',
        });
        const docUrl = URL.createObjectURL(tempblob);
        docUrls.push({
          selected: true,
          content: `${docUrl}`,
          guid: this.$parent.uuidv4(),
        });
      }

      return docUrls;
    },
    deletePage(guid) {
      this.pages = this.pages.filter((p) => p.guid != guid);
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
    async renderPage(pdfDoc, num, curPage) {
      let _this = this;
      let canvas = _this.$refs['canvas'];
      if (Array.isArray(canvas)) canvas = canvas[0];
      let ctx = canvas.getContext('2d');

      // Using promise to fetch the page
      await pdfDoc.getPage(num).then(async function (page) {
        var viewport = page.getViewport({ scale: _this.scale });

        canvas.height = viewport.height;
        canvas.width = viewport.width;

        // Render PDF page into canvas context
        var renderContext = {
          canvasContext: ctx,
          viewport: viewport,
        };
        var renderTask = page.render(renderContext);

        // Wait for rendering to finish
        await renderTask.promise.then(function () {
          curPage.url = canvas.toDataURL();
        });
      });
    },
    move(direction, guid) {
      let index = this.pages.findIndex((p) => p.guid == guid);
    },
  },
};
</script>

<style scoped>
canvas {
  display: none;
}
img {
  width: 90%;
  box-shadow: rgba(149, 157, 165, 0.2) 0px 8px 24px;
}
input[type='checkbox'] {
  width: 0;
}
label {
  display: block;
}
input + label {
  border-radius: 10px;
  padding: 10px;
}
input:checked + label {
  background-color: #f3f3f3;
}
ul,
li {
  list-style: none;
  margin: 0;
  padding: 0;
  position: relative;
}
li .button-set {
  position: absolute;
  right: 0;
  bottom: 0;
}
</style>
