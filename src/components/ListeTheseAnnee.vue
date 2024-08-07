<template>
  <div class="list-content" :class="listCssClass">
    <div class="list-header is-flex">
      <p class="menu-label">
        <span @click="toggleContent">Thèses de l'année</span> :
        <input type="text" class="year" v-model="inputAnnee" />
      </p>
      <div class="slider-prev-next-buttons is-flex is-align-items-center">
        <nav>
          <button v-on:click="downOneAnne">-</button>
        </nav>
        <vue-slider
          v-model="annee"
          :vData="listProm"
          :lazy="true"
          :tooltip="'active'"
        ></vue-slider>
        <nav>
          <button v-on:click="addOneAnne">+</button>
        </nav>
      </div>
      <nav v-if="isNotInitialAnnee">
        <button v-on:click="reinitalise">
          Retour à l'année en cours <span>{{ id }}</span>
        </button>
      </nav>
      <a href="#" class="toggle-btn" @click="toggleContent"></a>
    </div>
    <div class="list-body">
      <div class="menu-list-scrollable thin-scroll">
        <ul class="menu-list" v-if="state.metadata">
          <li v-for="these in state.metadata" :key="these">
            <ul v-if="these[1]">
              <b v-if="these[0] === textid">
                <div class="thesis-author">{{ these[1] }}</div>
                <div class="thesis-title" v-html="these[2]"></div>
              </b>
              <router-link :to="these[0]" v-else>
                <div v-on:click="gotoTop">
                  <div class="thesis-author">{{ these[1] }}</div>
                  <div v-html="these[2]"></div>
                </div>
              </router-link>
            </ul>
            <ul v-else>
              <b v-if="these[0] === textid">{{ these[2] }}</b>
              <router-link :to="these[0]" v-on:click="gotoTop" v-else
                ><span v-html="these[2]"></span
              ></router-link>
            </ul>
          </li>
        </ul>
        <span v-if="Object.keys(state.metadataSupplement).length > 0" class="menu-list-supplement">Autres positions de thèses de {{ annee }} (supplément)</span>
        <ul class="menu-list" v-if="state.metadataSupplement">
          <li v-for="these in state.metadataSupplement" :key="these">
            <ul v-if="these[1]">
              <b v-if="these[0] === textid">
                <div class="thesis-author">{{ these[1] }}</div>
                <div class="thesis-title" v-html="these[2]"></div>
              </b>
              <router-link :to="these[0]" v-else>
                <div v-on:click="gotoTop">
                  <div class="thesis-author">{{ these[1] }}</div>
                  <div v-html="these[2]"></div>
                </div>
              </router-link>
            </ul>
            <ul v-else>
              <b v-if="these[0] === textid">{{ these[2] }}</b>
              <router-link :to="these[0]" v-on:click="gotoTop" v-else
                ><span v-html="these[2]"></span
              ></router-link>
            </ul>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, reactive, toRefs, watch, computed } from "vue";
import { getMetadataENCPOSFromApi, getPositionAnneeFromApi } from "@/api/document";
import VueSlider from "vue-slider-component";
import "vue-slider-component/theme/antd.css";

export default {
  name: "ListeTheseAnnee",

  props: ["id", "textid", "yearswithadditionalpositions"],
  components: {
    VueSlider,
  },
  async setup(props) {
    let state = reactive({
      isOpened: false,
    });
    const { id } = toRefs(props);
    var annee = ref(id.value.toString());

    var yearsWithSupplement = props.yearswithadditionalpositions.map(string => string.replace('b', ''));

    const listProm = ref([]);
    const getPositionsForCurrentYear = async () => {
      let metadata = {};
      let metadataSupplement = {};
      let [data, dataSupplement] = []
      if (yearsWithSupplement.includes(annee.value)) {
        [data, dataSupplement] = await getPositionAnneeFromApi(annee.value, true);
      } else {
        [data, dataSupplement] = await getPositionAnneeFromApi(annee.value, false);
      }
      var htmlnamespace = Object.keys(data["@context"]).find((k) =>
        data["@context"][k].includes("html")
      );
      var dcnamespace = Object.keys(data["@context"]).find((k) =>
        data["@context"][k].includes("dc/elements")
      );

      if (data && data["member"]) {
        for (var these of data["member"]) {
          if (these["@id"].includes("PREV") || these["@id"].includes("NEXT")) {
            continue;
          }
          var title = these["dts:extensions"][htmlnamespace + ":h1"];
          var author = these["dts:extensions"][dcnamespace + ":creator"];
          try {
            const page = these["dts:dublincore"]["dct:extend"].toString().split("-")[0];
            metadata[page] = [these["@id"], author, title];
          } catch {
            metadata[these["@id"].split("_")[2]] = [these["@id"], author, title];
          }
        }
      }
      if (dataSupplement && dataSupplement["member"]) {
        for (these of dataSupplement["member"]) {
          if (these["@id"].includes("PREV") || these["@id"].includes("NEXT")) {
            continue;
          }
          title = these["dts:extensions"][htmlnamespace + ":h1"];
          author = these["dts:extensions"][dcnamespace + ":creator"];
          try {
            const page = these["dts:dublincore"]["dct:extend"].toString().split("-")[0];
            metadataSupplement[page] = [these["@id"], author, title];
          } catch {
            metadataSupplement[these["@id"].split("_")[2]] = [these["@id"], author, title];
          }
        }
      }
      state.metadata = metadata;
      state.metadataSupplement = metadataSupplement;
    };

    const getAllPositionsYears = async () => {
      const data = await getMetadataENCPOSFromApi();
      let annees = [];
      for (var member of data.member) {
        let annee = member["@id"].replace("ENCPOS_", "").replace("b", "");
        annees.push(annee);
      }
      annees.sort();
      listProm.value = [...new Set(annees)];
    };

    watch(annee, getPositionsForCurrentYear);

    const listCssClass = computed(() => {
      return state.isOpened ? "is-opened" : "";
    });

    const toggleContent = function (event) {
      event.preventDefault();
      state.isOpened = !state.isOpened;
    };

    const isNotInitialAnnee = computed(() => {
      return annee.value.toString() !== id.value.toString();
    });

    const downOneAnne = function () {
      if (listProm.value.indexOf(annee.value.toString()) != "0") {
        annee.value = listProm.value[
          listProm.value.indexOf(annee.value.toString()) - 1
        ].toString();
      }
      return annee;
    };

    const reinitalise = function () {
      annee.value = id.value.toString();
      return annee;
    };

    const addOneAnne = function () {
      if (
        listProm.value.indexOf(annee.value.toString()) !=
        listProm.value.length.toString() - 1
      ) {
        annee.value = listProm.value[
          listProm.value.indexOf(annee.value.toString()) + 1
        ].toString();
      }
      return annee;
    };

    const inputAnnee = computed({
      get: () => annee.value,
      set: (val) => {
        const valStr = val.toString().toLowerCase(); // special string value : 1942b
        if (valStr.length >= 4) {
          if (listProm.value.indexOf(valStr) !== -1) {
            annee.value = valStr;
          } else {
            const valNum = parseInt(val);
            const minDate = listProm.value[0];
            const maxDate = listProm.value[listProm.value.length - 1];
            annee.value = Math.min(maxDate, Math.max(minDate, valNum)).toString();
            console.log(minDate, annee.value);
          }
        }
      },
    });

    watch(inputAnnee, () => {
      state.isOpened = true;
    });

    await Promise.all([getPositionsForCurrentYear(), getAllPositionsYears()]);
    const gotoTop = function () {
      scroll(0, 0);
    };

    return {
      state,
      listCssClass,
      toggleContent,
      isNotInitialAnnee,
      addOneAnne,
      annee,
      reinitalise,
      downOneAnne,
      inputAnnee,
      gotoTop,
      listProm,
    };
  },
};
</script>

<style scoped>
nav button {
  cursor: pointer;
}
nav button,
.list-header p.menu-label {
  font-family: "Barlow", sans-serif !important;
  font-size: 15px;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0;
}
.list-header {
  align-items: center;
  background-color: #b8b5ad;
  padding: 20px 0 20px;
  border-radius: 6px;
  position: relative;
}
.list-content.is-opened .list-header {
  border-radius: 6px 6px 0 0;
}
.list-header p.menu-label {
  color: #4a4a4a;
}
.list-body {
  display: none;
  padding: 30px 20px;
  background-color: #e5e3de;
  border-radius: 0 0 6px 6px;
}
.list-content.is-opened .list-body {
  display: block;
}
.list-header {
  padding-left: 20px;
}

p.menu-label {
  margin-bottom: 0;
  text-indent: 0;
}
.menu-label span:first-child {
  cursor: pointer;
}
.menu-label span.year {
  background-color: #fff;
  font-size: 14px;
  color: #8b8a7e;
  padding: 1px 20px;
  margin: 0 10px;
}
.menu-label input[type="text"].year,
.menu-label input[type="number"].year {
  inset: unset;
  border: none;
  text-shadow: none;
  -moz-appearance: textfield;
  background-color: #fff;

  max-width: 70px;
  padding: 2px 0;
  margin: 0 15px;

  font-family: "Barlow", sans-serif;
  font-weight: 500;
  font-size: 14px;
  color: #979797;
  text-transform: uppercase;
  text-align: center;
}
.menu-label input[type="text"].year:focus,
.menu-label input[type="number"].year:focus {
  outline: solid 2px #b9192f;
}
input[type="number"]::-webkit-outer-spin-button,
input[type="number"]::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

nav button {
  border: none;
  background: none;
  color: #ffffff;
}
nav button > span {
  border: #ffffff solid 1px;
  border-radius: 4px;
  padding: 4px 20px;
  margin-left: 10px;
}

.slider-prev-next-buttons {
  margin-right: 30px;
}

/* slider */
.vue-slider {
  min-width: 200px;
}
.vue-slider.vue-slider-ltr {
  margin-top: 2px !important;
  padding: 0 10px !important;
  height: 3px !important;
}
.vue-slider :deep(.vue-slider-dot) {
  width: 18px !important;
  height: 18px !important;
}
.vue-slider:hover :deep(.vue-slider-rail),
.vue-slider :deep(.vue-slider-rail) {
  background-color: #ffffff;
}
.vue-slider :deep(.vue-slider:hover .vue-slider-process),
.vue-slider :deep(.vue-slider-process) {
  background-color: #b9192f !important;
}
.vue-slider :deep(.vue-slider-dot-handle:hover),
.vue-slider :deep(.vue-slider-dot-handle-focus),
.vue-slider :deep(.vue-slider-dot-handle) {
  border-color: #b9192f !important;
}
.vue-slider :deep(.vue-slider-dot-handle-focus) {
  box-shadow: 0 0 0 5px rgba(185, 25, 47, 0.2);
}
.vue-slider :deep(.vue-slider-dot-tooltip-text) {
  font-family: "Barlow", sans-serif;
}

/* toogle */
.toggle-btn {
  position: absolute;
  right: 20px;
  top: 19px;
  width: 27px;
  height: 27px;
  background: url(../assets/images/chevron_blanc.svg) center top -3px / cover no-repeat;
  border: none;
  text-decoration: none;
}
.list-content.is-opened .toggle-btn {
  background: url(../assets/images/croix_blc.svg) center / cover no-repeat;
}
.menu-list-scrollable {
  padding: 10px 20px 10px 0;
  max-height: 50vh;
  overflow-y: auto;
}

*.thin-scroll {
  scrollbar-width: thin;
  scrollbar-color: #b9192f #ceccc8;
}
/* Works on Chrome/Edge/Safari */
*.thin-scroll::-webkit-scrollbar {
  width: 8px;
}
*.thin-scroll::-webkit-scrollbar-track {
  background: #ceccc8;
}
*.thin-scroll::-webkit-scrollbar-thumb {
  background-color: #b9192f;
}

.menu-list {
  counter-reset: thesis-counter;
  columns: 3;
}

.thesis-author {
  font-weight: 600;
}
.thesis-author::before {
  content: counter(thesis-counter);
  counter-increment: thesis-counter;
  color: #b9192f;
  margin-right: 5px;
}
.menu-list-supplement {
  display: inline-block;
  margin-bottom: 5px;
  color: #b9192f;
}
.menu-list li ul {
  padding-left: 0;
}
.menu-list > li {
  display: inline-block;
  margin-bottom: 20px;
  break-inside: avoid;
}
.menu-list li > ul {
  border: none;
  display: block;
  margin: 0;
}
.menu-list > li {
  flex: 33.333% 0 0;
}
.menu-list b,
.menu-list a {
  border: none;
  padding: 0;
  font-family: "Barlow Semi Condensed", sans-serif;
  font-size: 18px;
  font-weight: 400;
  line-height: 25px;
  color: #161616;
  text-transform: none;
}
.menu-list b {
  font-weight: 600;
}
ol,
ul {
  list-style: none;
}

@media screen and (max-width: 1150px) {
  .menu-list {
    columns: 2;
  }
}
@media screen and (max-width: 950px) {
  .list-header {
    flex-wrap: wrap;
  }
  .list-header > nav:last-of-type {
    margin-top: 20px;
    width: 100%;
  }
}
@media screen and (max-width: 800px) {
  .vue-slider.vue-slider-ltr {
    margin-top: 5px !important;
    padding: 0 !important;
    width: calc(100% - 70px) !important;
  }
  .list-body {
    padding: 8px 5px 30px 20px;
  }
}
@media screen and (max-width: 800px) {
}
@media screen and (max-width: 640px) {
  .list-header {
    flex-direction: column;
    justify-content: flex-start;
    align-items: flex-start;
    gap: 20px;
  }
  .slider-prev-next-buttons *:first-child {
    padding-right: 10px;
  }
  .slider-prev-next-buttons *:last-child {
    padding-left: 10px;
  }
  .list-header > nav:last-of-type {
    margin-top: 0px;
    width: auto;
  }
  .liste-theses-area {
    padding-top: 30px;
    padding-bottom: 30px;
  }
  .menu-list {
    columns: 1;
  }
  .toggle-btn {
    width: 20px;
    right: 15px;
  }
}
</style>
