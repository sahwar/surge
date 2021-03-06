<template>
  <div class="file-chunks">
    <div class="file-chunks__title text_wrap_none">
      <template v-if="file.IsMissing">
        Missing
      </template>
      <template v-else-if="file.IsHashing">
        Hashing
      </template>
      <template
        v-else-if="!file.IsDownloading && !file.IsUploading && !file.IsPaused"
      >
        Finished
      </template>
      <template v-else-if="file.IsUploading">
        Seeding: <FileSharedBadge :ratio="shared.toFixed(2)" />
      </template>
      <template v-else-if="file.IsPaused">
        Paused: {{ progress.toFixed(2) }}%
      </template>
      <template v-else> Downloading: {{ progress.toFixed(2) }}% </template>
    </div>
    <canvas
      v-if="file.IsDownloading || file.IsPaused"
      class="file-chunks__progress"
      ref="canvas"
      width="156"
      height="12"
    ></canvas>
  </div>
</template>

<style lang="scss">
@import "./FileChunks.scss";
</style>

<script>
import { mapState } from "vuex";

import FileSharedBadge from "@/components/File/FileSharedBadge/FileSharedBadge";

export default {
  components: {
    FileSharedBadge,
  },
  props: {
    file: {
      type: Object,
      default: () => {},
    },
  },
  data: () => {
    return {
      progress: 0,
      shared: 0,
    };
  },
  computed: {
    ...mapState("globalBandwidth", ["statusBundle"]),
    baseColor() {
      return document.getElementById("app").classList.contains("theme_dark")
        ? "#ebebeb"
        : "#fcfcfc";
    },
  },
  watch: {
    statusBundle(newEvent) {
      const { FileHash } = this.file;
      const newFileHash = this._.find(newEvent, { FileHash });
      const isNewFileHash = !this._.isEmpty(newFileHash);

      if (isNewFileHash) {
        this.shared = newFileHash.ChunksShared / newFileHash.NumChunks;
        this.progress = newFileHash.Progress * 100;
        this.drawProgress(newFileHash.ChunkMap);
      }
    },
    progress(x) {
      if (x === 100) {
        this.$store.dispatch("files/fetchLocalFiles");
      }
    },
  },
  mounted() {
    this.getChunkMap();
    this.progress = this.file.Progress * 100;
    this.shared = this.file.ChunksShared / this.file.NumChunks;
  },
  methods: {
    getChunkMap() {
      window.backend.getFileChunkMap(this.file.FileHash, 156).then((bits) => {
        this.drawProgress(bits);
      });
    },
    drawProgress(bits) {
      if (this.file.IsDownloading || this.file.IsPaused) {
        const canvas = this.$refs.canvas;
        const ctx = canvas.getContext("2d");
        const colours = [this.baseColor, "#5EC1FF", "#02d2b3"];

        const bitmap = `${bits}`.split("");

        bitmap.forEach((val, i) => {
          ctx.beginPath();
          ctx.strokeStyle = colours[parseFloat(val)];
          ctx.lineWidth = 1;
          ctx.moveTo(i, 0);
          ctx.lineTo(i, 12);
          ctx.closePath();
          ctx.stroke();
        });
      }
    },
  },
};
</script>
