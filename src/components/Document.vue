<template>
  <Suspense>
    <component :is="customDocument" />
  </Suspense>
</template>

<script>
import { defineAsyncComponent } from "vue/dist/vue.esm-bundler.js";
import { getDocumentFromApi } from "@/api/document";

const VUE_APP_IIIF_URL = `${process.env.VUE_APP_IIIF_URL}`;
const VUE_APP_IIIF_IMAGES_URL = `${process.env.VUE_APP_IIIF_IMAGES_URL}`;

export default {
  name: "Document",

  props: ["id"],

  async setup(props) {
    const customDocument = defineAsyncComponent(async () => {
      // fetch the initial template
      const data = await getDocumentFromApi(props.id);
      // build a temporary dom just to ease the navigation inside the document
      let tmpDom = document.createElement("div");
      tmpDom.innerHTML = data;

      // customize the template with some vue components and code
      let frameNum = 1;
      tmpDom.querySelectorAll("a.pb.facs").forEach((a) => {
        const container = document.createElement("div");
        // TODO: gérer ce lowercase un peu gênant
        const canvadId = `${VUE_APP_IIIF_URL}/${props.id.toLowerCase()}/canvas/f${frameNum}`;
        const imageName = a.href.split("/").pop();
        const imageUrl = `${VUE_APP_IIIF_IMAGES_URL}/${props.id}/${imageName}`;
        //console.log(a.href, imageUrl);
        container.innerHTML = `<page-break canvas-id="${canvadId}" canvas-num="${frameNum}" image="${imageUrl}"/>`;
        frameNum += 1;
        // replace the link with a PageBreak component
        a.parentNode.replaceChild(container.firstChild, a);
      });

      const toc = tmpDom.querySelector("#aside");
      const tocDest = document.querySelector("#toc-area");
      tocDest.appendChild(toc);
      // return what will make the async component
      return new Promise((resolve) => {
        resolve({
          template: tmpDom.innerHTML,
        });
      });
    });
    return {
      customDocument,
    };
  },
};
</script>

<style src="../assets/css/html.css" id="document-html-css">
header {
  clear: both;
  padding: 1ex;
  border: dashed #ccc 1px;
  -webkit-border-radius: 1ex;
  -moz-border-radius: 1ex;
  border-radius: 1ex;
}
</style>
<style src="../assets/css/postprod.css"></style>
