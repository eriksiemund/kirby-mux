<template>
  <k-block-figure
    :is-empty="!playbackId"
    empty-icon="video"
    empty-text="No video uploaded to MUX yet …"
    @open="open"
    @update="update"
  >
    <div class="mux-video__container">
      <mux-video
        v-if="playbackId"
        muted
        :playback-id="playbackId"
        :controls="true"
      />
    </div>
  </k-block-figure>
</template>

<script>
import '@mux/mux-video'

export default {
  name: 'VideoBlock',
  inject: ['$api'],
  props: {
    content: {
      type: Object,
      default: () => ({}),
    },
  },
  data() {
    return {
      mux: null,
      fileData: null,
    }
  },
  computed: {
    video() {
      return this.content.video?.[0] || {}
    },
    id() {
      return this.mux?.playback_ids?.[0]?.id
    },
    src() {
      if (!this.id) return ''
      return `https://stream.mux.com/${this.id}.m3u8`
    },
    time() {
      return this.content.thumbnail || 0
    },
    thumb() {
      if (!this.id) return ''
      const url = `https://image.mux.com/${this.id}/thumbnail.jpg?time=${this.time}`
      const srcset = [300, 600, 900, 1200, 1600]
      return {
        src: url,
        srcset: srcset.map((w) => `${url}&width=${w} ${w}w`).join(', '),
      }
    },
    videoWidth() {
      return (
        this.fileData?.content?.resolutionx ||
        this.mux?.max_stored_resolution?.split('x')?.[0] ||
        null
      )
    },
    videoHeight() {
      return (
        this.fileData?.content?.resolutiony ||
        this.mux?.max_stored_resolution?.split('x')?.[1] ||
        null
      )
    },
    aspectRatio() {
      if (this.fileData?.content?.resaspect) {
        return this.fileData.content.resaspect
      }
      if (this.videoWidth && this.videoHeight) {
        return `${this.videoWidth}/${this.videoHeight}`
      }
      return null
    },
    assetId() {
      return this.fileData?.content?.asset_id || this.fileData?.content?.assetid
    },
    playbackId() {
      return (
        this.fileData?.content?.playback_id ||
        this.fileData?.content?.playbackid
      )
    },
    posterUrl() {
      const poster = this.fileData?.content?.poster?.[0]
      return poster?.url || null
    },
  },
  watch: {
    'video.link': {
      handler(link) {
        if (link && this.$api) {
          this.$api
            .get(link)
            .then((file) => {
              // Guard against missing or invalid file data
              if (!file || !file.content) {
                console.error('Invalid file data received')
                return
              }

              this.fileData = file

              // Guard against missing or invalid mux data
              if (!file.content.mux) {
                console.error('Missing mux data in file content')
                return
              }

              try {
                const parsedMux = JSON.parse(file.content.mux)
                // Validate that parsed data has expected structure
                if (!parsedMux || typeof parsedMux !== 'object') {
                  console.error('Invalid mux data structure')
                  return
                }
                this.mux = parsedMux
              } catch (error) {
                console.error('Failed to parse mux JSON:', error)
              }
            })
            .catch((error) => {
              console.error('Failed to load video metadata:', error)
            })
        }
      },
      immediate: true,
    },
  },
}
</script>

<style>
.mux-video__container {
  box-shadow: var(--item-shadow);
  border-radius: var(--rounded);
  overflow: hidden;
}

mux-video {
  width: 100%;
  display: block;
  aspect-ratio: 16 / 9;
}
</style>
