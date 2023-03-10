<template>
  <div id="uploaded-files" v-if="pages.length > 0">
    <canvas ref="canvas" />
    <button
      class="btn-primary"
      v-if="pages.find((p) => p.selected) != undefined"
      v-on:click="merge"
    >
      Merge
    </button>
    <ul>
      <li v-for="(page, index) in pages" :ref="`page_${page.guid}`">
        <input
          type="checkbox"
          v-model="page.selected"
          :id="`cb_${page.guid}`"
        />
        <label :for="`cb_${page.guid}`" :class="`rotate-${page.rotation}`">
          <div class="rotation-wrapper-outer">
            <div class="rotation-wrapper-inner">
              <img :src="page.url" :class="`rotate-${page.rotation}`" />
            </div>
          </div>
        </label>
        <div class="button-set">
          <button v-on:click="move(-1, page.guid)" v-if="index > 0">↑</button>
          <button
            v-on:click="move(1, page.guid)"
            v-if="index < pages.length - 1"
          >
            ↓
          </button>
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
          rotation: 0,
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
        let page = selectedPages[i];
        let url = page.content;
        let blob = await fetch(url).then((r) => r.blob());
        let arrayBuffer = await new Response(blob).arrayBuffer();
        const selectedPdf = await PDFDocument.load(arrayBuffer);

        if (page.rotation > 0)
          selectedPdf
            .getPages()[0]
            .setRotation({ type: 'degrees', angle: page.rotation });
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
    move(step, guid) {
      let fromIndex = this.pages.findIndex((p) => p.guid == guid);
      let element = this.pages[fromIndex];
      this.pages.splice(fromIndex, 1);
      this.pages.splice(fromIndex + step, 0, element);
    },
    rotate(guid) {
      let page = this.pages.find((p) => p.guid == guid);
      page.rotation = (page.rotation + 90) % 360;
    },
  },
};
</script>

<style scoped>
div#uploaded-files {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
canvas {
  display: none;
}
img {
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
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #f3f3f3;
  box-sizing: border-box;
}
input:checked + label {
  border: 3px solid #ff7474;
}
ul {
  margin: 20px;
  padding: 0;
  max-width: var(--main-result-width);
  counter-reset: item 0;
}
li {
  list-style: none;
  margin: 0;
  padding: 0;
  position: relative;
}
li::before {
  width: 20px;
  display: inline-block;
}
li:has(input:checked)::before {
  content: counter(item) ' ';
  counter-increment: item 1;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 40px;
  height: 40px;
  background-color: #ff7474;
  color: #fff;
  position: absolute;
  top: 3px;
  left: -13px;
  border-radius: 50%;
  font-weight: bold;
}
li:not(:has(input:checked))::before {
  content: ' ';
  list-style: none;
}
li .button-set {
  position: absolute;
  right: 0;
  bottom: 0;
}
label div {
  width: 100%;
  overflow: hidden;
}
.button-set button {
  border: 0;
  width: 40px;
  height: 40px;
  display: inline-flex;
  justify-content: center;
  align-items: center;
  margin: 0 5px 5px 0;
  border-radius: 50%;
  background: #ffe1e1;
  cursor: pointer;
}
.button-set button:hover {
  background: #ff7474;
  color: #fff;
}
label img {
  width: 100%;
}
.btn-primary {
  position: fixed;
  right: 0;
  z-index: 999;
}

.rotate-90 .rotation-wrapper-outer,
.rotate-270 .rotation-wrapper-outer {
  display: table;
}
.rotate-90 .rotation-wrapper-inner,
.rotate-270 .rotation-wrapper-inner {
  padding: 50% 0;
  height: 0;
}
.rotate-90 img {
  transform-origin: top left;
  transform: rotate(-90deg) translate(-100%);
  margin-top: -50%;
}
.rotate-180 img {
  transform: rotate(180deg);
}
.rotate-270 img {
  transform-origin: top left;
  transform: rotate(90deg) translate(0, -100%);
  margin-top: -50%;
}
</style>
