<template>
  <div id="spotifyMenu">
    <p class="help content" title="Plz don't sue me Spotify">
      I am not affiliated with Spotify and this tool is not endorsed by Spotify AB. Please follow the <a href="https://www.spotifycodes.com/assets/Terms_and_Conditions_for_Spotify_Codes.pdf" target="_blank" rel="nofollow noopener noreferrer">Terms and Conditions</a> for Spotify Codes.
    </p>
    <!-- Spotify Options -->
    <nav class="panel">
      <p class="panel-heading">{{ $t("spotifyOptions") }}</p>

      <!-- Text -->
      <div class="option-pane">
        <div class="field is-horizontal">
          <div class="field-label is-normal">
            <label class="label">{{$t('spotifyUri')}}</label>
          </div>
          <div class="field-body">
            <div class="field">
              <div class="control">
                <input
                  class="input"
                  type="text"
                  placeholder="spotify:track:4uLU6hMCjMI75M1A2tKUQC"
                  v-model="options.spotifyUri"
                  @change="downloadSpotifyCode"
                />
              </div>
            </div>
          </div>
        </div>
        <div class="content">
          <p class="help">
            <span class="help-icon icon has-text-info">
              <i class="fas fa-info-circle"></i>
            </span>
            {{$t('spotifyUriHelp')}}
          </p>
        </div>
        <div class="content" v-if="spotifyCodeUrl">
          <figure class="image">
            <object
              type="image/svg+xml"
              id="spotify-code-preview"
              :data="spotifyCodeUrl"
              v-if="validSpotifyCode"
              @load="validSpotifyCode = true"
              @error="validSpotifyCode = false"
            />
            <p class="has-text-weight-bold has-text-success" v-if="validSpotifyCode">
              <span class="icon">
                <i class="fa fa-check"></i>
              </span>
              Valid Spotify URI
            </p>
            <p class="has-text-weight-bold has-text-danger" v-if="!validSpotifyCode">
              <span class="icon">
                <i class="fa fa-exclamation-triangle"></i>
              </span>
              Invalid Spotify URI
            </p>
          </figure>
        </div>

      </div>
    </nav>
    <!-- 3D Options -->
    <SpotifyModelOptionsPanel :options="options" :unit="unit" />

    <div class="notification is-danger is-light" v-if="generateError" style="margin-top: 20px 0;">
      {{generateError}}
    </div>

    <button
      class="button is-success is-large"
      v-bind:class="{'is-loading': isGenerating}"
      @click="generate3dModel"
      v-if="validSpotifyCode"
    >
      <span class="icon">
        <i class="fa fa-cube"></i>
      </span>
      <span>{{$t('generateButton')}}</span>
    </button>
  </div>
</template>

<script>
import * as THREE from 'three';
import { SVGLoader } from 'three/examples/jsm/loaders/SVGLoader';
import pathThatSvg from 'path-that-svg';
import { diff } from 'deep-object-diff';
import merge from 'deepmerge';
import JSZip from 'jszip';
import modelWorker from '@/model-worker';
// 3D settings panel
import SpotifyModelOptionsPanel from './SpotifyModelOptionsPanel.vue';
import { save, saveAsString, saveAsArrayBuffer } from '../utils';
import { nextTick } from 'vue';

const defaultOptions = {
  spotifyUri: '',
  base: {
    shape: 'roundedRectangle',
    width: 100,
    height: 25,
    depth: 3,
    cornerRadius: 5,
    hasBorder: true,
    borderWidth: 2,
    borderDepth: 1,
    hasText: false,
    textPlacement: 'bottom',
    textMargin: 4,
    textSize: 10,
    textMessage: '',
    textDepth: 1,
    textAlign: 'center',
    hasKeychainAttachment: false,
    keychainPlacement: 'left',
    keychainHoleDiameter: 6,
    mirrorHoles: false,
    hasNfcIndentation: false,
    nfcIndentationShape: 'square',
    nfcIndentationSize: 20,
    nfcIndentationDepth: 1,
    nfcIndentationHidden: false,
  },
  code: {
    depth: 1,
    margin: 5,
    cityMode: false,
    depthMax: 5,
    invert: false,
  },
};

export default {
  name: 'SpotifyMenu',
  props: {
    initData: Object,
    scene: Object,
    exporter: Object,
  },
  components: {
    SpotifyModelOptionsPanel,
  },
  data() {
    return {
      options: JSON.parse(JSON.stringify(defaultOptions)),
      spotifyCodeUrl: '',
      validSpotifyCode: false,
      unit: 'mm',
      mesh: null,
      baseMesh: null,
      spotifyCodeMesh: null,
      borderMesh: null,
      subtitleMesh: null,
      keychainAttachmentMesh: null,
      stlType: 'binary',
      dualExtrusion: false,
      isGenerating: false,
      generateError: null,
    };
  },

  methods: {
    initWorker() {
      modelWorker.worker.onmessage = (event) => {
        if (event.data.type !== 'result') {
          return;
        }
        this.$emit('resetScene');
        const jsonLoader = new THREE.ObjectLoader();
        const { meshes } = event.data;
        let i = 0;
        Object.keys(meshes).forEach((key) => {
          jsonLoader.parse(meshes[key], (parsed) => {
            meshes[key] = parsed;
            i += 1;
            if (key !== 'combined') {
              this.scene.add(meshes[key]);
            }
            if (i === event.data.meshCount) {
              this.mesh = meshes.combined;
              this.baseMesh = meshes.base;
              this.spotifyCodeMesh = meshes.spotifyCode;
              this.borderMesh = meshes.border;
              this.iconMesh = meshes.icon;
              this.subtitleMesh = meshes.subtitle;
              this.keychainAttachmentMesh = meshes.keychainAttachment;
              this.isGenerating = false;
            }
          });
        });
        this.$emit('exportReady', diff(defaultOptions, this.options));
      };
    },
    async setup3dObject() {
      const loader = new SVGLoader();
      let svg = document.querySelector('#spotify-code-preview').contentDocument.querySelector('svg').outerHTML;
      svg = svg.replace('<rect x="0" y="0" width="400" height="100" fill="#000000"/>', '');
      const pathedSvg = await pathThatSvg(svg);
      const svgData = loader.parse(pathedSvg);
      let shapes = svgData.paths.map((p) => p.toShapes(true, false)).flat();
      shapes = shapes.map((s) => s.toJSON());

      modelWorker.send({
        mode: 'Spotify',
        spotifyCodeShapes: shapes,
        options: this.options,
      });
    },
    async generate3dModel() {
      this.$emit('generating');
      this.isGenerating = true;

      nextTick(() => {
        // this.init3d();
        this.setup3dObject();
        // this.startAnimation();
      });
    },
    exportSTL(stlType, multipleParts) {
      const timestamp = new Date().getTime();
      const exportAsBinary = (stlType === 'binary');

      if (multipleParts) {
        const zip = new JSZip();
        const filenameBase = `base-${timestamp}.stl`;
        const filenameQrcode = `qrcode-${timestamp}.stl`;
        const filenameBorder = `border-${timestamp}.stl`;
        const filenameText = `text-${timestamp}.stl`;
        const filenameKeychain = `attachment-${timestamp}.stl`;

        const baseSTL = this.exporter.parse(this.baseMesh, { binary: exportAsBinary });
        const qrcodeSTL = this.exporter.parse(this.spotifyCodeMesh, { binary: exportAsBinary });
        zip.file(filenameBase, baseSTL.buffer);
        zip.file(filenameQrcode, qrcodeSTL.buffer);

        if (this.borderMesh) {
          const borderSTL = this.exporter.parse(this.borderMesh, { binary: exportAsBinary });
          zip.file(filenameBorder, borderSTL.buffer);
        }

        if (this.subtitleMesh) {
          const textSTL = this.exporter.parse(this.subtitleMesh, { binary: exportAsBinary });
          zip.file(filenameText, textSTL.buffer);
        }

        if (this.keychainAttachmentMesh) {
          const kcaSTL = this.exporter.parse(this.keychainAttachmentMesh, { binary: exportAsBinary });
          zip.file(filenameKeychain, kcaSTL.buffer);
        }

        zip.generateAsync({ type: 'blob' })
          .then((content) => {
            save(new Blob([content]), `qrcode2stl-${timestamp}.zip`);
          });
      } else {
        const filename = `combined-${timestamp}.stl`;
        const result = this.exporter.parse(this.mesh, { binary: exportAsBinary });
        if (exportAsBinary) {
          saveAsArrayBuffer(result, filename);
        } else {
          saveAsString(result, filename);
        }
      }
    },
    async downloadSpotifyCode() {
      let uri = this.options.spotifyUri;
      if (!uri.startsWith('spotify:')) {
        const regex = /spotify\.com\/(?:.*\/)*([^/]+)\/([^?/]+)/gm;
        const parts = regex.exec(uri);
        if (parts.length !== 3) {
          console.error('Not a valid Spotify URI or Link');
          return;
        }
        uri = `spotify:${parts[1]}:${parts[2]}`;
      }
      const spotifyCodeSvgUrl = `https://scannables.scdn.co/uri/plain/svg/000000/white/640/${uri}`;
      this.validSpotifyCode = true;
      const response = await fetch(spotifyCodeSvgUrl);
      const utf8Decoder = new TextDecoder('utf-8');
      const reader = response.body.getReader();
      const data = await reader.read();
      const svgString = utf8Decoder.decode(data.value);
      const svgBlob = new Blob([svgString], { type: 'image/svg+xml' });
      this.spotifyCodeUrl = URL.createObjectURL(svgBlob);
    },
  },
  async mounted() {
    this.initWorker();
    if (this.initData && this.initData.mode === 'Spotify') {
      delete this.initData.mode;
      this.options = merge(this.options, this.initData);
      await this.downloadSpotifyCode();
      this.generate3dModel();
    }
  },
};
</script>

<style scoped>
#notifications {
  margin-top: 10px;
}

.field-label {
  text-align: left;
  flex-grow: 1.5;
  margin-right: 0;
}

#spotify-code-preview {
  max-width: 100%;
}
</style>
