<template>
  <aside class="media pb-component">
    <article class="pb">
      <div class="pb-label">
        <a :href="imageInfo"> {p. {{ canvasNum }}}</a>
      </div>
      <div><img class="pb-thumnbail" @click="goToCanvas" :src="thumbnail" /></div>
    </article>
  </aside>
</template>

<script>
import { inject } from "vue";

export default {
  name: "PageBreak",

  props: ["canvasId", "canvasNum", "image"],

  setup(props) {
    const mirador = inject("mirador");

    const goToCanvas = function () {
      if (mirador) {
        mirador.setCanvasId(props.canvasId);
      }
    };

    const thumbnail = `${props.image}/full/60,/0/default.jpg`;
    const imageInfo = `${props.image}`;

    return {
      goToCanvas,
      thumbnail,
      imageInfo,
    };
  },
};
</script>

<style>
.pb-component {
  position: relative;
  float: left;
}
.pb {
  position: absolute;
  left: -80px;
}
.pb-component:hover {
  cursor: pointer;
}
.pb-thumnbail {
  max-width: 60px;
}
.pb-label {
  font-size: large;
  text-align: center;
  margin-top: 16px;
  margin-bottom: 16px;
}
</style>
