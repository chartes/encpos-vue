<template>
  <div class="is-flex is-flex-direction-column" :class="viewModeCssClass">
    <div v-if="metadata">
      <liste-these-annee
        v-if="metadata.date"
        class="liste-theses-area app-width-padding"
        :id="metadata.date"
        :textid="$route.params.docId"
        :yearswithadditionalpositions="yearsWithAdditionalPositions"
      />
      <document-metadata :metadata="metadata" class="metadata-area app-width-margin" />
    </div>
    <div class="toc-area app-width-margin" :class="tocCssClass">
      <div class="toc-area-header">
        <a href="#" v-on:click="toggleTOCContent">SOMMAIRE</a>
        <a href="#" class="toggle-btn" v-on:click="toggleTOCContent"></a>
      </div>
      <div id="toc-area" class="toc-area-content toc-content" />
    </div>
    <nav class="controls is-flex app-width-margin">
      <a
        href=""
        @click="toggleTOCMenu"
        class="toc-menu-toggle"
        :class="TOCMenuBtnCssClass"
        >Sommaire</a
      >
      <ul class="is-flex">
        <li>
          <a
            href=""
            class="text-btn"
            aria-label="texte seul"
            @click.prevent="changeViewMode('text-mode')"
          ></a>
        </li>
        <li>
          <a
            href=""
            class="text-images-btn"
            aria-label="texte et images"
            @click.prevent="changeViewMode('text-and-images-mode')"
          ></a>
        </li>
        <li>
          <a
            href=""
            class="images-btn"
            aria-label="images seules"
            @click.prevent="changeViewMode('images-mode')"
          ></a>
        </li>
      </ul>
      <ul class="is-flex">
        <li><a v-if="metadata.downloadPDF" target="_blank" v-bind:href="metadata.downloadPDF" class="pdf-btn" aria-label="Télécharger le PDF"></a></li>
        <li><a v-if="metadata.downloadXML" target="_blank" v-bind:href="metadata.downloadXML" class="xml-btn" aria-label="Télécharger le XML"></a></li>
        <li><a v-if="metadata.thenca" target="_blank" v-bind:href="metadata.thenca" class="access_link">Accès à la thèse</a></li>
      </ul>
    </nav>
    <div class="document-area is-flex app-width-margin" :class="tocMenuCssClass">
      <div id="toc-area-aside" class="toc-area-aside toc-content" />
      <div class="document-views is-flex">
        <div class="text-view" id="text-view">
          <document :id="$route.params.docId" :key="$route.params.docId" />
        </div>
        <div class="mirador-view" id="mirador-view" :style="miradorViewCssStyle">
          <div id="vue-mirador-container" ref="miradorContainer"/>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Document from '@/components/Document.vue'
import DocumentMetadata from '../components/DocumentMetadata.vue'
import { getMetadataENCPOSFromApi, getMetadataFromApi } from '@/api/document'

import {
  computed,
  onMounted,
  onUnmounted,
  watch,
  reactive,
  provide,
  ref,
  inject,
  watchEffect
} from 'vue/dist/vue.esm-bundler.js'
import { onBeforeRouteUpdate, useRoute } from 'vue-router'

import ListeTheseAnnee from '@/components/ListeTheseAnnee.vue'
import useMirador from '@/composables/use-mirador'

const sources = [
  { name: 'data_bnf', ext: 'data.bnf.fr' },
  { name: 'dbpedia', ext: 'dbpedia.org' },
  { name: 'idref', ext: 'idref.fr' },
  { name: 'catalogue_bnf', ext: 'catalogue.bnf.fr' },
  { name: 'wikidata', ext: 'wikidata' },
  { name: 'wikipedia', ext: 'wikipedia' },
  { name: 'thenca', ext: 'thenca' },
  { name: 'hal', ext: 'hal' },
  { name: 'benc', ext: 'koha' },
  {}
]

function findSource (id) {
  let i = 0
  let source = null

  do {
    source = id.includes(sources[i].ext) ? sources[i] : null
    i++
  } while (i < sources.length && source === null)

  if (source) {
    return source.name
  }

  return null
}

export default {
  name: 'DocumentPage',
  components: {
    Document,
    DocumentMetadata,
    ListeTheseAnnee
  },
  async setup () {
    const manifestIsAvailable = ref(false)
    let yearsWithAdditionalPositions = []

    // Mirador view sticky behavior
    const miradorViewBoundingTop = ref(0)
    const miradorViewCssStyle = computed(() => {
      return { marginTop: miradorViewBoundingTop.value + 'px' }
    })

    const updateMiradorTopPosition = function () {
      const textView = document.getElementById('text-view')
      if (textView) {
        const top = textView.getBoundingClientRect().top
        miradorViewBoundingTop.value = top < 0 ? -Math.floor(top) : 0
      }
    }
    const metadata = reactive({
      sudoc: null,
      benc: null,
      thenca: null,
      iiifManifestUrl: null,
      hal: null,
      downloadPDF: null,
      downloadXML: null,

      author: null,
      data_bnf: null,
      dbpedia: null,
      idref: null,
      catalogue_bnf: null,
      wikidata: null,
      wikipedia: null
    })
    const manifest = ref(null)
    const miradorContainer = ref(null)

    const miradorInstance = useMirador(miradorContainer, manifest)
    // provide an uninitialized instance of Mirador
    provide('mirador', miradorInstance)

    const layout = inject('variable-layout')

    const getMetadata = async (docId) => {
      const listmetadata = await getMetadataFromApi(docId)

      const dcnamespace = Object.keys(listmetadata['@context']).find((k) =>
        listmetadata['@context'][k].includes('dc/elements')
      )
      const htmlnamespace = Object.keys(listmetadata['@context']).find((k) =>
        listmetadata['@context'][k].includes('html')
      )
      console.log(htmlnamespace)
      console.log(listmetadata)
      metadata.author = listmetadata['dts:extensions'][dcnamespace + ':creator']
      if (Array.isArray(listmetadata['dts:download'])) {
        for (const meta of listmetadata['dts:download']) {
          if (meta.includes('.PDF')) {
            metadata.downloadPDF = meta
          }
          if (meta.includes('document')) {
            metadata.downloadXML = meta
          }
        }
      } else {
        metadata.downloadXML = listmetadata['dts:download']
        metadata.downloadPDF = null
      }
      metadata.download = listmetadata['dts:download']

      const dublincore = listmetadata['dts:dublincore']
      console.log('---------')
      console.log('dublincore', dublincore)
      try {
        metadata.iiifManifestUrl = dublincore['dct:source'][0]['@id']
        layout.imageIsAvailable.value = true
      } catch {
        metadata.iiifManifestUrl = ''
        layout.imageIsAvailable.value = false
      }
      metadata.date = dublincore['dct:date']
      metadata.page = dublincore['dct:extend']
      metadata.coverage = dublincore['dct:coverage']
      metadata.rights = dublincore['dct:rights'][0]['@id']
      metadata.title = listmetadata['dts:extensions'][htmlnamespace + ':h1']

      console.log('metadata.iiifManifestUrl', metadata.iiifManifestUrl)
      console.log('metadata', metadata)

      if (dublincore) {
        // reset the sources
        for (const s of sources) {
          metadata[s.name] = null
        }

        // benc & sudoc & thenca
        if (dublincore['dct:isVersionOf']) {
          for (const member of dublincore['dct:isVersionOf']) {
            if (member['@id']) {
              const source = findSource(member['@id'])
              if (source) {
                metadata[source] = member['@id']
                console.log('source found:', source, member['@id'])
              }
            }
          }
        }

        // creators
        if (Array.isArray(dublincore['dct:creator'])) {
          for (const aut of dublincore['dct:creator']) {
            if (aut['@id']) {
              // find the source name from its extension
              const source = findSource(aut['@id'])
              if (source) {
                metadata[source] = aut['@id']
                console.log('source found:', source, aut['@id'])
              }
            }
          }
        }
      }
    }
    const getAllPositionsYears = async () => {
      const data = await getMetadataENCPOSFromApi()
      const annees = []
      let listProm = []
      for (const member of data.member) {
        const annee = member['@id'].replace('ENCPOS_', '')
        annees.push(annee)
      }
      annees.sort()
      listProm = annees
      const yearsIdwithSupplementalIndicator = listProm.filter(a => a.includes('b'))
      console.log('yearsIdwithSupplementalIndicator : ', yearsIdwithSupplementalIndicator)
      yearsWithAdditionalPositions = yearsIdwithSupplementalIndicator.map(string => string.replace('b', ''))
      console.log('yearsWithAdditionalPositions : ', yearsWithAdditionalPositions)
    }
    const setMirador = function () {
      fetch(metadata.iiifManifestUrl, {
        method: 'GET'
      })
        .then((r) => {
          manifestIsAvailable.value = r.ok
          return r.json()
        })
        .then((loadedManifest) => {
          manifest.value = loadedManifest
        })
        .catch((error) => {
          console.log('setMirador error : ', error)
          manifestIsAvailable.value = false
        })
    }

    watch(
      () => metadata.iiifManifestUrl,
      async () => {
        setMirador()
      }
    )

    onBeforeRouteUpdate(async (to) => {
      getMetadata(to.params.docId)
    })

    onMounted(() => {
      const appView = document.getElementById('app')
      appView.addEventListener('scroll', updateMiradorTopPosition)
      window.addEventListener('scroll', updateMiradorTopPosition)
    })

    onUnmounted(() => {
      const appView = document.getElementById('app')
      layout.changeViewMode('text-mode')
      appView.removeEventListener('scroll', updateMiradorTopPosition)
      window.removeEventListener('scroll', updateMiradorTopPosition)
    })

    const route = useRoute()
    await getMetadata(route.params.docId)

    await getAllPositionsYears()

    return {
      tocCssClass: layout.tocCssClass,
      toggleTOCContent: layout.toggleTOCContent,
      tocMenuCssClass: layout.tocMenuCssClass,
      toggleTOCMenu: layout.toggleTOCMenu,
      TOCMenuBtnCssClass: layout.TOCMenuBtnCssClass,
      changeViewMode: layout.changeViewMode,
      viewModeCssClass: layout.viewModeCssClass,
      miradorViewCssStyle,
      metadata,
      manifestIsAvailable,
      layout,
      yearsWithAdditionalPositions,
      miradorContainer
    }
  }
}
</script>

<style>
.liste-theses-area {
  background-color: #fbf8f4;
  padding-top: 50px;
  padding-bottom: 50px;
  margin-bottom: 28px;
}
.metadata-area {
  margin-bottom: 15px !important;
}
.metadata-area .columns {
  margin: 0;
}
.toc-area {
  width: 100%;
  font-family: "Barlow", sans-serif !important;
  margin-bottom: 30px !important;
}
.toc-area-header {
  display: flex;
  width: 100%;
  padding: 20px;
  background-color: #f1f1f1;
  border-radius: 6px;
  position: relative;
}
.toc-area-header > a {
  text-decoration: none;
  border: none;
}
.toc-area-content {
  background-color: #e4e4e4;
  border-radius: 0 0 6px 6px;
  display: none;
}
.toc-area.is-opened .toc-area-header {
  background-color: #f1f1f1;
  border-radius: 6px 6px 0 0;
}
.toc-area.is-opened .toc-area-content {
  display: block;
}
.toc-area .toc-area-content aside {
  width: 100% !important;
  padding: 20px 50px !important;
}
.toc-area .toc-area-content nav > ol.tree {
  columns: 4;
  gap: 40px;
}
.toc-content > aside > nav > nav > ol.tree > li {
  text-transform: uppercase;
  margin-bottom: 20px;
  padding: 0;
}
.toc-content > aside > nav > nav > ol.tree > li.more > a {
  display: inline-block;
  margin-bottom: 8px;
}
.toc-content > aside > nav > nav > ol.tree > li li {
  padding: 0;
  margin: 0 0 6px;
  text-transform: none;
}
.toc-content > aside > nav > nav > ol.tree > li ol {
  margin: 0;
}
.toc-content nav > ol.tree > li {
  break-inside: avoid;
}
.toc-content nav > ol.tree li::before {
  display: none;
}
.toc-content nav > .tree ol,
.tree ul {
  border: none !important;
}
.toc-content a:hover {
  background-color: transparent !important;
  border-radius: unset !important;
  color: #000;
  text-decoration: underline dotted !important;
}
.toc-area-aside a,
.toc-area-content a {
  font-family: "Barlow Semi Condensed", sans-serif !important;
  font-weight: 400;
  text-align: left;
  line-height: 20px;
  letter-spacing: 0;
  border: none;
  box-shadow: none;
}
.toc-area-content a {
  font-size: 17px;
  color: #252525;
}
.toc-area-aside a {
  font-size: 16px;
  color: #000;
}

/* toogle */
.toggle-btn {
  position: absolute;
  right: 20px;
  width: 27px;
  height: 27px;
  background: url(../assets/images/chevron_rouge.svg) center top -7px / cover no-repeat;
  border: none;
  text-decoration: none;
}
.toc-area.is-opened .toggle-btn {
  background: url(../assets/images/croix.svg) center / cover no-repeat;
}

.controls {
  justify-content: space-between;
  align-items: center;
  width: 100%;
  border-top: #b9192f 1px dashed;
  border-bottom: #b8b8b8 1px solid;
  padding: 12px 0 9px;
}
.controls a {
  display: inline;
  font-family: "Barlow", sans-serif;
  font-size: 15px;
  font-weight: 500;
  text-transform: uppercase;
  color: #4a4a4a;
  margin: 0;
}
.controls > a.toc-menu-toggle {
  font-size: 14px;
  color: #aeaeae;
  padding: 6px 10px;
  border: #aeaeae 1px solid;
  border-radius: 4px;
}
.controls > a.toc-menu-toggle.is-opened {
  color: #b9192f;
  border-color: #b9192f;
}
.controls ul {
  align-items: center;
}
.controls ul > li {
  margin: 0;
}
.controls ul > li > a {
  display: inline-block;
  width: 42px;
  height: 42px;
  margin-right: 10px;
}
.controls ul > li > a.access_link {
  vertical-align: center;
  display: inline;
  margin-left: 15px;
}
.controls a.text-btn {
  background: url(../assets/images/b_text_off.svg) center / cover no-repeat;
}
.text-mode .controls a.text-btn {
  background-image: url(../assets/images/b_text_on.svg);
}
.controls a.text-images-btn {
  width: 80px;
  background: url(../assets/images/b_text-image_off.svg) center / cover no-repeat;
  margin: 0 25px 0 15px;
}
.text-and-images-mode .controls a.text-images-btn {
  background-image: url(../assets/images/b_text-image_on.svg);
}
.controls a.images-btn {
  background: url(../assets/images/b_image_off.svg) center / cover no-repeat;
}
.images-mode .controls a.images-btn {
  background-image: url(../assets/images/b_image_on.svg);
}
.text-mode-only .controls a.text-btn {
  pointer-events: none;
}
.text-mode-only .controls a.text-images-btn,
.text-mode-only .controls a.images-btn {
  pointer-events: none;
  opacity: 0.2;
}
.controls a.pdf-btn {
  background: url(../assets/images/b_PDF.svg) center / cover no-repeat;
}
.controls a.xml-btn {
  background: url(../assets/images/b_XML.svg) center / cover no-repeat;
}
.document-area {
  width: 100%;
}
.document-area #aside,
.toc-area #aside {
  position: unset;
  float: none;
  margin: 0;
  background: none;
  border: none;
}
.document-area #aside header,
.toc-area #aside header {
  display: none;
}
.document-views {
  width: 100%;
}
.toc-area-aside {
  display: none;
}
.toc-aside-is-opened .toc-area-aside {
  display: flex;
  width: 220px;
}
.toc-aside-is-opened .document-views {
  flex: calc(100% - 220px);
}
.mirador-view {
  position: relative;
  min-height: 80vh;
  max-height: 100vh;
}
.text-mode .text-view,
.images-mode .mirador-view {
  flex: 100% 0 0;
  width: 100%;
}
.images-mode .text-view,
.text-mode .mirador-view {
  position: absolute;
  width: 500px;
  height: 700px;
  visibility: hidden;
}
.text-mode .mirador-view {
  flex: 100% 0 0;
}
.text-and-images-mode .text-view,
.text-and-images-mode .mirador-view {
  flex: 50% 0 0;
}

#center {
  width: 100%;
  margin: 0 !important;
}

#article {
  padding: 40px 10% 120px;
  border-bottom: 1px dotted #ffffff;
  min-height: 100%;
}

#article article {
  margin: 0;
}

#article h1.head.textpart,
#article .titlepage {
  font-family: "Barlow", sans-serif !important;
}

#article h1.head.textpart {
  padding: 0;
  font-size: 25px;
  font-weight: 500;
  line-height: 33px;
  text-transform: none;
  color: #971716;
}

#article .titlepage {
  font-size: 18px;
  line-height: 25px;
}

#article .titlepage hr {
  width: 100%;
  margin: 60px 0 45px;
  border: dashed #b9192f 1px;
}

#article .titlepage,
#article .titlepage .forename {
  font-variant: small-caps;
  text-transform: none;
}

#article .titlepage .surname {
  text-transform: uppercase;
}

#article .titlepage .forename,
#article .titlepage .surname {
  font-size: 20px;
  font-weight: 500;
}
#article .titlepage .name {
  margin-bottom: 30px;
}
#article .titlepage .roleName {
  font-size: 16px;
  line-height: 22px;
  text-transform: uppercase;
  font-style: italic;
  color: #777;
}

#article section.div {
  border: none;
  padding-bottom: 0;
  padding-top: 40px;
  font-family: "Libre Baskerville", serif !important;
  font-size: 16px;
  text-align: left;
  font-weight: 400;
  line-height: 28px;
  color: #5f5f5f;
}

#article section.div h2.head {
  line-height: 115%;
  color: #971716;
  border-bottom: none;
  padding: 1em 0 0 0;
  margin: 35px 0 43px 0px;
  text-align: center;
  font-variant: small-caps;
}

#article section.div h3.head {
  color: #222222;
  margin: 35px 0 28px 0;
  border-bottom: 0px dotted;
  text-align: center;
  padding: 1em 0 0 1ex;
  font-weight: bold;
  text-transform: none;
}

#article section.div h4.head {
  text-align: center;
  color: #999;
  font-size: 15px;
  font-weight: bold;
  padding: 1em 0 1ex 2ex;
}

.toc-area-header a {
  color: inherit;
}

@media screen and (max-width: 1150px) {
  .toc-area .toc-area-content nav > ol.tree {
    columns: 2;
  }
  .controls > a.toc-menu-toggle {
    margin-left: 20px;
  }
  .controls ul > li > a.access_link {
    margin-right: 20px;
  }
}
@media screen and (max-width: 800px) {
  #article {
    padding: 40px 4% 120px;
  }
  .toc-area .toc-area-content aside {
    padding: 20px 20px !important;
  }
}
@media screen and (max-width: 640px) {
  .pb,
  .controls > a.toc-menu-toggle,
  .toc-area-aside {
    display: none !important;
  }
  .toc-area .toc-area-content nav > ol.tree {
    columns: 1;
  }
  .toggle-btn {
    width: 20px;
    right: 15px;
  }
  .controls {
    flex-wrap: wrap;
  }
  .controls ul:first-of-type {
    order: 3;
    flex: 100% 0 0;
    width: 100%;
    justify-content: center;
    padding: 20px 0 10px;
  }
  .controls ul:last-of-type {
    flex: 100% 0 0;
    justify-content: center;
  }
  .controls ul:last-of-type > li > a {
    width: 30px;
    height: 30px;
  }
  .controls > ul:first-of-type > li:nth-child(2) {
    display: none;
  }
  #article section.div {
    font-size: 14px;
    line-height: 24px;
  }
  #article h1.head.textpart {
    font-size: 20px;
    line-height: 25px;
  }
  #article section.div h2.head,
  #article section.div h3.head {
    font-size: 14px;
    line-height: 24px;
  }
  #article section.div {
    padding-top: 10px;
  }
  #article p.p {
    text-align: left;
  }

  .text-and-images-mode .document-views {
    display: block !important;
  }
  .text-and-images-mode .text-view,
  .text-and-images-mode .mirador-view {
  }
}
* [class*="mirador-window-top-bar"] {
  border-top: none !important;
}
</style>
