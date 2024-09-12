<template>
  <div :class="[inEditor && !shared.isMobile ? '' : 'full']">
    <div
      v-loading="loading"
      class="right-panel"
      id="chart-panel"
      ref="chartPanel"
      :style="{ background: backgroundColor }"
    ></div>
    <div id="tool-panel">
      <div class="left-panel">
        <el-switch
          class="dark-mode"
          v-model="shared.darkMode"
          active-color="#181432"
          :active-text="$t('editor.darkMode')"
          :inactive-text="''"
        >
        </el-switch>
        <el-switch
          v-if="!isGL"
          class="enable-decal"
          v-model="shared.enableDecal"
          :active-text="$t('editor.enableDecal')"
          :inactive-text="''"
        >
        </el-switch>
        <!-- Not display when random button is displayed on mobile devices. -->
        <el-popover
          placement="bottom"
          trigger="click"
          v-if="!isGL && !(shared.isMobile && hasRandomData)"
        >
          <div class="render-config-container">
            <div class="cfg-renderer">
              <label class="tool-label">{{ $t('editor.renderer') }}</label>
              <el-radio-group
                v-model="shared.renderer"
                size="mini"
                style="text-transform: uppercase"
              >
                <el-radio-button label="svg" />
                <el-radio-button label="canvas" />
              </el-radio-group>
            </div>
            <el-switch
              v-if="shared.renderer === 'canvas'"
              v-model="shared.useDirtyRect"
              :active-text="$t('editor.useDirtyRect')"
              :inactive-text="''"
            />
          </div>
          <span class="render-config-trigger" slot="reference">
            <el-button size="mini">
              {{ $t('editor.renderCfgTitle')
              }}<i class="el-icon-setting el-icon--right"></i>
            </el-button>
          </span>
        </el-popover>
        <el-button
          class="random"
          v-if="hasRandomData"
          size="mini"
          @click="changeRandomSeed"
          >{{ $t('editor.randomData') }}</el-button
        >
        <el-select
          v-if="shared.echartsVersion && !shared.isMobile"
          class="version-select"
          :class="{
            'is-nightly': nightly,
            'is-pr': shared.isPR || hasPRVersion
          }"
          size="mini"
          id="choose-echarts-version"
          v-model="shared.echartsVersion"
          @change="changeVersion"
        >
          <el-option
            v-for="version in versionList"
            :key="version"
            :label="version"
            :value="version"
          >
            {{ version }}
          </el-option>
        </el-select>
        <el-checkbox
          v-if="inEditor && !shared.isMobile"
          v-model="nightly"
          class="use-nightly"
          >Nightly</el-checkbox
        >
      </div>

      <a
        :href="editLink"
        target="_blank"
        v-if="!inEditor"
        class="edit btn btn-sm"
        >{{ $t('editor.edit') }}</a
      >
    </div>

    <div id="preview-status">
      <div class="left-buttons">
        <template v-if="inEditor && !shared.isMobile">
          <el-button
            icon="el-icon-download"
            size="mini"
            @click="download"
            :title="$t('editor.download') + ' (HTML)'"
          >
            {{ $t('editor.download') }}
          </el-button>
          <el-button
            @click="screenshot"
            icon="el-icon-camera-solid"
            size="mini"
          >
            {{ $t('editor.screenshot') }}
          </el-button>
          <el-button
            @click="share"
            icon="el-icon-share"
            size="mini"
            :title="$t('editor.share.tooltip')"
          >
            {{ $t('editor.share.title') }}
          </el-button>
        </template>
      </div>

      <div
        id="run-log"
        v-if="inEditor && !shared.isMobile && shared.editorStatus.message"
      >
        <span class="run-log-time">{{ shared.editorStatus.time }}</span>
        <span :class="'run-log-type-' + shared.editorStatus.type">{{
          shared.editorStatus.message
        }}</span>
      </div>
    </div>
  </div>
</template>

<script>
import {
  getExampleConfig,
  isGLExample,
  saveExampleCodeToLocal,
  store,
  updateRandomSeed,
  updateRunHash,
  isValidPRVersion
} from '../common/store';
import { getScriptURLs, URL_PARAMS } from '../common/config';
import { compressStr } from '../common/helper';
import { createSandbox } from './sandbox';
import debounce from 'lodash/debounce';
import { download } from './downloadExample';
import { gotoURL, getURL } from '../common/route';
import { gt, rcompare } from 'semver';

const example = getExampleConfig();
const isGL = 'gl' in URL_PARAMS || isGLExample();
const isLocal = 'local' in URL_PARAMS;
const isDebug = 'debug' in URL_PARAMS;
const hasBMap = example && example.tags.indexOf('bmap') >= 0;

function getScriptURL(link) {
  return isDebug || isLocal ? link.replace('.min.', '.') : link;
}

function getScripts(nightly) {
  const SCRIPT_URLS = getScriptURLs(store.locale);

  const echartsDirTpl =
    SCRIPT_URLS[
      isLocal
        ? 'localEChartsDir'
        : store.isPR
        ? 'prPreviewEChartsDir'
        : nightly
        ? 'echartsNightlyDir'
        : 'echartsDir'
    ];
  const echartsDir = store.isPR
    ? echartsDirTpl.replace('{{PR_NUMBER}}', store.prNumber)
    : echartsDirTpl.replace('{{version}}', store.echartsVersion);
  const code = store.runCode;

  return [
    // echarts
    echartsDir +
      getScriptURL(SCRIPT_URLS.echartsJS) +
      (store.isPR ? '?_=' + (store.prLatestCommit || Date.now()) : ''),
    // echarts-gl
    ...(isGL
      ? [
          isLocal
            ? SCRIPT_URLS.localEChartsGLDir + '/dist/echarts-gl.js'
            : getScriptURL(SCRIPT_URLS.echartsGLJS)
        ]
      : []),
    // echarts theme
    ...(!store.darkMode && store.theme
      ? [echartsDir + `/theme/${store.theme}.js`]
      : []),
    // echarts bmap extension
    ...(hasBMap || /coordinateSystem.*:.*['"]bmap['"]/g.test(code)
      ? [
          SCRIPT_URLS.bmapLibJS,
          echartsDir + getScriptURL(SCRIPT_URLS.echartsBMapJS)
        ]
      : []),
    // echarts stat
    ...(code.indexOf('ecStat') > -1
      ? [getScriptURL(SCRIPT_URLS.echartsStatJS)]
      : []),
    // echarts graph modularity
    ...(code.indexOf('graph') > -1 && code.indexOf('modularity') > -1
      ? [getScriptURL(SCRIPT_URLS.echartsGraphModularityJS)]
      : []),
    // echarts map
    ...(/map.*:.*['"]world['"]/g.test(code)
      ? [SCRIPT_URLS.echartsWorldMapJS]
      : []),
    // data gui
    ...(code.indexOf('app.config') > -1 ? [SCRIPT_URLS.datGUIMinJS] : [])
  ].map((url) => ({ src: url }));
}

function log(text, type) {
  if (type !== 'warn' && type !== 'error') {
    type = 'info';
  }
  const now = new Date();
  store.editorStatus.time = [now.getHours(), now.getMinutes(), now.getSeconds()]
    .map((t) => (t + '').padStart(2, '0'))
    .join(':');
  store.editorStatus.message = text;
  store.editorStatus.type = type;
}

function run(recreateInstance) {
  if (!store.runCode) {
    return;
  }

  const runCode = () => {
    this.sandbox.run(store, recreateInstance);

    // Update run hash to let others known chart has been changed.
    updateRunHash();
  };

  const scripts = getScripts(this.nightly);
  const ECScriptReg = /\/echarts(?:\.min)?\.js/;
  const scriptsChanged =
    !this.scripts ||
    scripts.some(
      (s) =>
        !ECScriptReg.test(s.src) &&
        this.scripts.findIndex((s1) => s1.src === s.src) === -1
    );

  if (!this.sandbox || scriptsChanged) {
    this.loading = true;
    let isFirstRun = true;
    this.dispose();
    this.sandbox = createSandbox(
      this.$refs.chartPanel,
      (this.scripts = scripts),
      store.isSharedCode,
      () => {
        runCode();
        this.loading = false;
      },
      () => {
        // TODO show error hints
        console.error('failed to run sandbox');
        this.loading = false;
        this.dispose();
      },
      (errMsg) => {
        const infiniteLoopInEditor =
          errMsg && errMsg.indexOf('loop executes') > -1;
        const potentialRedirection =
          errMsg && errMsg.indexOf('potential redirection') > -1;
        log(
          this.$t(
            `editor.${
              infiniteLoopInEditor
                ? 'infiniteLoopInEditor'
                : potentialRedirection
                ? 'potentialRedirectionInEditor'
                : 'errorInEditor'
            }`
          ),
          'error'
        );
      },
      (updateTime) => {
        const option = this.getOption();
        if (
          typeof option.backgroundColor === 'string' &&
          option.backgroundColor !== 'transparent'
        ) {
          this.backgroundColor = option.backgroundColor;
        } else {
          this.backgroundColor = '#fff';
        }

        log(this.$t('editor.chartOK') + updateTime.toFixed(2) + 'ms');

        // Find the appropriate throttle time
        const debounceTimeQuantities = [0, 500, 2000, 5000, 10000];
        for (let i = debounceTimeQuantities.length - 1; i >= 0; i--) {
          const quantity = debounceTimeQuantities[i];
          const preferredDebounceTime =
            debounceTimeQuantities[i + 1] || 1000000;
          if (
            updateTime >= quantity &&
            this.debouncedTime !== preferredDebounceTime
          ) {
            this.debouncedRun = debounce(run, preferredDebounceTime, {
              trailing: true
            });
            this.debouncedTime = preferredDebounceTime;
            break;
          }
        }

        if (isFirstRun) {
          this.$emit('ready');
          isFirstRun = false;
        }
      },
      (css) => {
        this.css = css;
      }
    );
  } else {
    runCode();
  }
}

export default {
  props: {
    inEditor: {
      type: Boolean
    }
  },

  data() {
    return {
      shared: store,
      debouncedTime: undefined,
      backgroundColor: '',
      autoRun: true,
      loading: false,

      isGL,

      allEChartsVersions: [],
      nightlyVersions: [],
      nightly: false,

      hasPRVersion: false
    };
  },

  mounted() {
    this.run();

    this.fetchVersionList();
  },

  computed: {
    hasRandomData() {
      return (
        this.shared.runCode && this.shared.runCode.indexOf('Math.random()') >= 0
      );
    },
    editLink() {
      const url = this.getSharableURL(true);
      return './editor.html' + url.search;
    },
    versionList() {
      return this.nightly ? this.nightlyVersions : this.allEChartsVersions;
    },
    isNightlyVersion() {
      return store.echartsVersion && store.echartsVersion.indexOf('dev') > -1;
    },
    toolOptions() {
      const isCanvas = store.renderer === 'canvas';
      return {
        renderer: isCanvas ? null : store.renderer,
        useDirtyRect: store.useDirtyRect && isCanvas ? 1 : null,
        decal: store.enableDecal ? 1 : null,
        theme: store.darkMode ? 'dark' : store.theme || null
      };
    }
  },

  watch: {
    'shared.runCode'() {
      if (this.autoRun || !this.sandbox) {
        if (!this.debouncedRun) {
          // First run
          this.run();
          // show share hint on first run if code is user-shared
          store.isSharedCode && this.showShareHint();
          // show PR hint on first run if it's based on PR
          store.isPR &&
            !this.prHintTimer &&
            (this.prHintTimer = setTimeout(
              this.showPRHint,
              store.isSharedCode ? 1e3 : 0
            ));
        } else {
          this.debouncedRun();
        }
      }
    },
    toolOptions: {
      handler: 'refresh',
      deep: true
    },
    isNightlyVersion: {
      handler(val) {
        this.nightly = val;
      },
      immediate: true
    }
  },

  methods: {
    run,
    // debouncedRun will be created at first run
    // debouncedRun: null,
    refresh() {
      this.run(true);
    },
    refreshAll() {
      // trigger reload
      this.scripts = null;
      this.run(true);
    },
    dispose() {
      if (this.sandbox) {
        this.sandbox.dispose();
        this.sandbox = null;
      }
    },
    download() {
      const url = this.getSharableURL(true);
      const isShared = store.isSharedCode || url.searchParams.has('code');
      const headers = [
        `\t${this.$t('editor.downloadSourceTip')} ${url.toString()}`
      ];
      isShared && headers.push('\t⚠ ' + this.$t('editor.share.hint'));
      download(headers.join('\n'));
    },
    screenshot() {
      this.sandbox &&
        this.sandbox.screenshot(
          (URL_PARAMS.c || Date.now()) +
            '.' +
            (store.renderer === 'svg' ? 'svg' : 'png')
        );
    },
    showPRHint() {
      this.$message({
        type: 'warning',
        message: this.$t('editor.pr.hint').replace('{{PR}}', store.prNumber),
        customClass: 'toast-declaration',
        duration: 8000,
        showClose: true
      });
    },
    showShareHint() {
      this.$message.closeAll();
      this.$message({
        type: 'warning',
        message: this.$t('editor.share.hint'),
        customClass: 'toast-declaration',
        duration: 8000,
        showClose: true
      });
    },
    getSharableURL(raw) {
      const params = {};
      if (store.initialCode !== store.sourceCode) {
        params.code = compressStr(store.sourceCode);
        params.enc = null;
      }
      return getURL(
        {
          ...this.toolOptions,
          ...params
        },
        raw
      );
    },
    share() {
      const ctx = this;
      if (ctx.isShareBusy) {
        return;
      }
      ctx.isShareBusy = true;
      const sharableURL = ctx.getSharableURL();
      if (sharableURL.length < 1e4) {
        return copyToClipboard(sharableURL);
      }
      // test whether the sharable URL is valid
      $.ajax({
        url: sharableURL,
        method: 'HEAD',
        complete(jqXHR) {
          const statusCode = jqXHR.status;
          if (statusCode === 413 || statusCode === 414 || statusCode === 431) {
            ctx.isShareBusy = false;
            ctx.$message.closeAll();
            ctx.$message({
              type: 'error',
              message: ctx.$t('editor.share.urlTooLong'),
              customClass: 'toast-declaration'
            });
          } else {
            copyToClipboard(sharableURL);
          }
        }
      });

      function copyToClipboard(url) {
        navigator.clipboard
          .writeText(url)
          .then(() => {
            ctx.$message.closeAll();
            ctx.$message({
              type: 'success',
              message: ctx.$t('editor.share.success'),
              customClass: 'toast-declaration'
            });
          })
          // PENDING
          .catch((e) => {
            console.error('failed to write share url to the clipboard', e);
            window.open(url, '_blank');
          })
          .finally(() => {
            ctx.isShareBusy = false;
          });
      }
    },
    getOption() {
      return this.sandbox && this.sandbox.getOption();
    },
    changeVersion() {
      saveExampleCodeToLocal();
      setTimeout(() => {
        // no confirmation as we have saved the code to local storage
        window.__EDITOR_NO_LEAVE_CONFIRMATION__ = true;
        gotoURL({
          version: store.echartsVersion,
          pv: store.isPR ? URL_PARAMS.version : void 0,
          ...this.toolOptions
        });
      });
    },
    changeRandomSeed() {
      updateRandomSeed();
      gotoURL({ random: store.randomSeed }, true);
      this.run();
    },
    fetchVersionList() {
      const isZH = store.locale === 'zh';
      const getVersionListAPI = (pkgName) =>
        isZH
          ? // speed up for China mainland. `data.jsdelivr.com` is very slow or unaccessible for some ISPs.
            // Another API is `/vendor-cdn/${pkgName}` with a request header `Accept: application/vnd.npm.install-v1+json`.
            `https://data.jsdelivr.com/v1/package/npm/${pkgName}`
          : `https://data.jsdelivr.com/v1/package/npm/${pkgName}`;

      // const handleData =
      //   isZH &&
      //   ((rawData) => {
      //     const versions = rawData.versions.sort(rcompare);
      //     const tags = rawData['dist-tags'];
      //     return {
      //       versions,
      //       tags
      //     };
      //   });

      const prVersion = URL_PARAMS.pv;
      const hasPRVersion = (this.hasPRVersion = isValidPRVersion(prVersion));

      (() => {
        let data = {
          tags: {
            alpha: '5.0.0-alpha.2',
            beta: '5.0.0-beta.2',
            latest: '5.5.1',
            rc: '5.5.1-rc.1'
          },
          versions: [
            '5.5.1',
            '5.5.1-rc.1',
            '5.5.0',
            '5.5.0-rc.2',
            '5.5.0-rc.1',
            '5.4.3',
            '5.4.3-rc.1',
            '5.4.2',
            '5.4.2-rc.1',
            '5.4.1',
            '5.4.1-rc.1',
            '5.4.0',
            '5.4.0-rc.1',
            '5.3.3',
            '5.3.3-rc.1',
            '5.3.2',
            '5.3.2-rc.1',
            '5.3.1',
            '5.3.1-rc.1',
            '5.3.0',
            '5.3.0-rc.1',
            '5.2.2',
            '5.2.1',
            '5.2.0',
            '5.1.2',
            '5.1.1',
            '5.1.0',
            '5.0.2',
            '5.0.1',
            '5.0.0',
            '5.0.0-rc.1',
            '5.0.0-beta.2',
            '5.0.0-beta.1',
            '5.0.0-alpha.2',
            '5.0.0-alpha.1',
            '4.9.0',
            '4.8.0',
            '4.7.0',
            '4.6.0',
            '4.5.0',
            '4.5.0-rc.2',
            '4.5.0-rc.1',
            '4.4.0',
            '4.4.0-rc.1',
            '4.3.0',
            '4.3.0-rc.2',
            '4.3.0-rc.1',
            '4.2.1',
            '4.2.1-rc.3',
            '4.2.1-rc.2',
            '4.2.1-rc.1',
            '4.2.0-rc.2',
            '4.2.0-rc.1',
            '4.1.0',
            '4.0.4',
            '4.0.3',
            '4.0.2',
            '4.0.1',
            '4.0.0',
            '3.8.5',
            '3.8.4',
            '3.8.3',
            '3.8.2',
            '3.8.1',
            '3.8.0',
            '3.7.2',
            '3.7.1',
            '3.7.0',
            '3.6.2',
            '3.6.1',
            '3.6.0',
            '3.5.4',
            '3.5.3',
            '3.5.2',
            '3.5.1',
            '3.5.0',
            '3.4.0',
            '3.3.2',
            '3.3.1',
            '3.3.0',
            '3.2.3',
            '3.2.2',
            '3.2.1',
            '3.2.0',
            '3.1.10',
            '3.1.9',
            '3.1.8',
            '3.1.7',
            '3.1.6',
            '3.1.5',
            '3.1.4',
            '3.1.3',
            '3.1.2',
            '3.1.1',
            '3.0.2-beta3',
            '3.0.2-beta2',
            '3.0.2-beta1',
            '3.0.1',
            '3.0.0',
            '3.0.0-beta4',
            '3.0.0-beta3',
            '3.0.0-beta2',
            '3.0.0-beta1',
            '2.2.8',
            '2.2.7-beta7',
            '2.2.7-amd-beta2',
            '2.2.7-amd-beta1',
            '2.2.1-amd-beta1'
          ]
        };
        //$.getJSON(getVersionListAPI('echarts')).done((data) => {
        // isZH && (data = handleData(data));

        if (isDebug) {
          console.log('echarts version data', data);
        }

        const versions = data.versions.filter(
          (version) =>
            version.indexOf('beta') < 0 &&
            version.indexOf('rc') < 0 &&
            version.indexOf('alpha') < 0 &&
            version.startsWith('5') // Only version 5.
        );
        this.allEChartsVersions = versions;

        // Use latest version
        if (
          !store.echartsVersion ||
          store.echartsVersion === '5' ||
          store.echartsVersion === 'latest'
        ) {
          store.echartsVersion = versions[0];
        }

        // put latest rc version for preview
        if (gt(data.tags.rc.split('-')[0], data.tags.latest)) {
          versions.unshift(data.tags.rc);
          store.echartsVersion === 'rc' && (store.echartsVersion = versions[0]);
        }

        hasPRVersion && versions.unshift(prVersion);
        //});
      })();

      (() => {
        let data = {
          tags: {
            next: '5.6.0-dev.20240912',
            latest: '5.5.2-dev.20240912'
          },
          versions: [
            '5.6.0-dev.20240912',
            '5.6.0-dev.20240911',
            '5.6.0-dev.20240910',
            '5.6.0-dev.20240909',
            '5.6.0-dev.20240908',
            '5.6.0-dev.20240907',
            '5.6.0-dev.20240906',
            '5.6.0-dev.20240905',
            '5.6.0-dev.20240904',
            '5.6.0-dev.20240903',
            '5.6.0-dev.20240902',
            '5.6.0-dev.20240901',
            '5.6.0-dev.20240831',
            '5.6.0-dev.20240830',
            '5.6.0-dev.20240829',
            '5.6.0-dev.20240828',
            '5.6.0-dev.20240827',
            '5.6.0-dev.20240826',
            '5.6.0-dev.20240825',
            '5.6.0-dev.20240824',
            '5.6.0-dev.20240823',
            '5.6.0-dev.20240822',
            '5.6.0-dev.20240821',
            '5.6.0-dev.20240820',
            '5.6.0-dev.20240819',
            '5.6.0-dev.20240818',
            '5.6.0-dev.20240817',
            '5.6.0-dev.20240816',
            '5.6.0-dev.20240815',
            '5.6.0-dev.20240814',
            '5.6.0-dev.20240813',
            '5.6.0-dev.20240812',
            '5.6.0-dev.20240811',
            '5.6.0-dev.20240810',
            '5.6.0-dev.20240809',
            '5.6.0-dev.20240808',
            '5.6.0-dev.20240807',
            '5.6.0-dev.20240806',
            '5.6.0-dev.20240805',
            '5.6.0-dev.20240804',
            '5.6.0-dev.20240803',
            '5.6.0-dev.20240802',
            '5.6.0-dev.20240801',
            '5.6.0-dev.20240731',
            '5.6.0-dev.20240730',
            '5.6.0-dev.20240729',
            '5.6.0-dev.20240728',
            '5.6.0-dev.20240727',
            '5.6.0-dev.20240726',
            '5.6.0-dev.20240725',
            '5.6.0-dev.20240724',
            '5.6.0-dev.20240723',
            '5.6.0-dev.20240722',
            '5.6.0-dev.20240721',
            '5.6.0-dev.20240720',
            '5.6.0-dev.20240719',
            '5.6.0-dev.20240718',
            '5.6.0-dev.20240630',
            '5.6.0-dev.20240629',
            '5.6.0-dev.20240628',
            '5.6.0-dev.20240627',
            '5.6.0-dev.20240626',
            '5.6.0-dev.20240625',
            '5.6.0-dev.20240624',
            '5.6.0-dev.20240623',
            '5.6.0-dev.20240622',
            '5.6.0-dev.20240621',
            '5.6.0-dev.20240620',
            '5.6.0-dev.20240619',
            '5.6.0-dev.20240618',
            '5.6.0-dev.20240617',
            '5.6.0-dev.20240616',
            '5.6.0-dev.20240615',
            '5.6.0-dev.20240614',
            '5.6.0-dev.20240613',
            '5.6.0-dev.20240612',
            '5.6.0-dev.20240611',
            '5.6.0-dev.20240610',
            '5.6.0-dev.20240609',
            '5.6.0-dev.20240608',
            '5.6.0-dev.20240607',
            '5.6.0-dev.20240606',
            '5.6.0-dev.20240605',
            '5.6.0-dev.20240604',
            '5.6.0-dev.20240603',
            '5.6.0-dev.20240602',
            '5.6.0-dev.20240601',
            '5.6.0-dev.20240531',
            '5.6.0-dev.20240530',
            '5.6.0-dev.20240529',
            '5.6.0-dev.20240528',
            '5.6.0-dev.20240527',
            '5.6.0-dev.20240526',
            '5.6.0-dev.20240525',
            '5.6.0-dev.20240524',
            '5.6.0-dev.20240523',
            '5.6.0-dev.20240522',
            '5.6.0-dev.20240521',
            '5.6.0-dev.20240520',
            '5.6.0-dev.20240519',
            '5.6.0-dev.20240518',
            '5.6.0-dev.20240517',
            '5.6.0-dev.20240516',
            '5.6.0-dev.20240515',
            '5.6.0-dev.20240514',
            '5.6.0-dev.20240513',
            '5.6.0-dev.20240512',
            '5.6.0-dev.20240511',
            '5.6.0-dev.20240510',
            '5.6.0-dev.20240509',
            '5.6.0-dev.20240508',
            '5.6.0-dev.20240507',
            '5.6.0-dev.20240506',
            '5.6.0-dev.20240505',
            '5.6.0-dev.20240504',
            '5.6.0-dev.20240503',
            '5.6.0-dev.20240502',
            '5.6.0-dev.20240501',
            '5.6.0-dev.20240430',
            '5.6.0-dev.20240429',
            '5.6.0-dev.20240428',
            '5.6.0-dev.20240427',
            '5.6.0-dev.20240426',
            '5.6.0-dev.20240425',
            '5.6.0-dev.20240424',
            '5.6.0-dev.20240423',
            '5.6.0-dev.20240422',
            '5.6.0-dev.20240421',
            '5.6.0-dev.20240420',
            '5.6.0-dev.20240419',
            '5.6.0-dev.20240418',
            '5.6.0-dev.20240417',
            '5.6.0-dev.20240416',
            '5.6.0-dev.20240415',
            '5.6.0-dev.20240414',
            '5.6.0-dev.20240413',
            '5.6.0-dev.20240412',
            '5.6.0-dev.20240411',
            '5.6.0-dev.20240410',
            '5.6.0-dev.20240409',
            '5.6.0-dev.20240408',
            '5.6.0-dev.20240407',
            '5.6.0-dev.20240406',
            '5.6.0-dev.20240405',
            '5.6.0-dev.20240404',
            '5.6.0-dev.20240403',
            '5.6.0-dev.20240402',
            '5.6.0-dev.20240401',
            '5.6.0-dev.20240331',
            '5.6.0-dev.20240330',
            '5.6.0-dev.20240329',
            '5.6.0-dev.20240328',
            '5.6.0-dev.20240327',
            '5.6.0-dev.20240326',
            '5.6.0-dev.20240325',
            '5.6.0-dev.20240324',
            '5.6.0-dev.20240323',
            '5.6.0-dev.20240322',
            '5.6.0-dev.20240321',
            '5.6.0-dev.20240320',
            '5.6.0-dev.20240319',
            '5.6.0-dev.20240318',
            '5.6.0-dev.20240317',
            '5.6.0-dev.20240316',
            '5.6.0-dev.20240315',
            '5.6.0-dev.20240314',
            '5.6.0-dev.20240313',
            '5.6.0-dev.20240312',
            '5.6.0-dev.20240311',
            '5.6.0-dev.20240310',
            '5.6.0-dev.20240309',
            '5.6.0-dev.20240308',
            '5.6.0-dev.20240307',
            '5.6.0-dev.20240306',
            '5.6.0-dev.20240305',
            '5.6.0-dev.20240304',
            '5.6.0-dev.20240303',
            '5.6.0-dev.20240302',
            '5.6.0-dev.20240301',
            '5.6.0-dev.20240229',
            '5.6.0-dev.20240228',
            '5.6.0-dev.20240227',
            '5.6.0-dev.20240226',
            '5.6.0-dev.20240225',
            '5.6.0-dev.20240224',
            '5.6.0-dev.20240223',
            '5.6.0-dev.20240222',
            '5.5.2-dev.20240912',
            '5.5.2-dev.20240911',
            '5.5.2-dev.20240910',
            '5.5.2-dev.20240909',
            '5.5.2-dev.20240908',
            '5.5.2-dev.20240907',
            '5.5.2-dev.20240906',
            '5.5.2-dev.20240905',
            '5.5.2-dev.20240904',
            '5.5.2-dev.20240903',
            '5.5.2-dev.20240902',
            '5.5.2-dev.20240901',
            '5.5.2-dev.20240831',
            '5.5.2-dev.20240830',
            '5.5.2-dev.20240829',
            '5.5.2-dev.20240828',
            '5.5.2-dev.20240827',
            '5.5.2-dev.20240826',
            '5.5.2-dev.20240825',
            '5.5.2-dev.20240824',
            '5.5.2-dev.20240823',
            '5.5.2-dev.20240822',
            '5.5.2-dev.20240821',
            '5.5.2-dev.20240820',
            '5.5.2-dev.20240819',
            '5.5.2-dev.20240818',
            '5.5.2-dev.20240817',
            '5.5.2-dev.20240816',
            '5.5.2-dev.20240815',
            '5.5.2-dev.20240814',
            '5.5.2-dev.20240813',
            '5.5.2-dev.20240812',
            '5.5.2-dev.20240811',
            '5.5.2-dev.20240810',
            '5.5.2-dev.20240809',
            '5.5.2-dev.20240808',
            '5.5.2-dev.20240807',
            '5.5.2-dev.20240806',
            '5.5.2-dev.20240805',
            '5.5.2-dev.20240804',
            '5.5.2-dev.20240803',
            '5.5.2-dev.20240802',
            '5.5.2-dev.20240801',
            '5.5.2-dev.20240731',
            '5.5.2-dev.20240730',
            '5.5.2-dev.20240729',
            '5.5.2-dev.20240728',
            '5.5.2-dev.20240727',
            '5.5.2-dev.20240726',
            '5.5.2-dev.20240725',
            '5.5.2-dev.20240724',
            '5.5.2-dev.20240723',
            '5.5.2-dev.20240722',
            '5.5.2-dev.20240721',
            '5.5.2-dev.20240720',
            '5.5.2-dev.20240719',
            '5.5.2-dev.20240718',
            '5.5.2-dev.20240627',
            '5.5.1-dev.20240626',
            '5.5.1-dev.20240625',
            '5.5.1-dev.20240624',
            '5.5.1-dev.20240623',
            '5.5.1-dev.20240622',
            '5.5.1-dev.20240621',
            '5.5.1-dev.20240620',
            '5.5.1-dev.20240619',
            '5.5.1-dev.20240618',
            '5.5.1-dev.20240617',
            '5.5.1-dev.20240616',
            '5.5.1-dev.20240615',
            '5.5.1-dev.20240614',
            '5.5.1-dev.20240613',
            '5.5.1-dev.20240612',
            '5.5.1-dev.20240611',
            '5.5.1-dev.20240610',
            '5.5.1-dev.20240609',
            '5.5.1-dev.20240608',
            '5.5.1-dev.20240607',
            '5.5.1-dev.20240606',
            '5.5.1-dev.20240605',
            '5.5.1-dev.20240604',
            '5.5.1-dev.20240603',
            '5.5.1-dev.20240602',
            '5.5.1-dev.20240601',
            '5.5.1-dev.20240531',
            '5.5.1-dev.20240530',
            '5.5.1-dev.20240529',
            '5.5.1-dev.20240528',
            '5.5.1-dev.20240527',
            '5.5.1-dev.20240526',
            '5.5.1-dev.20240525',
            '5.5.1-dev.20240524',
            '5.5.1-dev.20240523',
            '5.5.1-dev.20240522',
            '5.5.1-dev.20240521',
            '5.5.1-dev.20240520',
            '5.5.1-dev.20240519',
            '5.5.1-dev.20240518',
            '5.5.1-dev.20240517',
            '5.5.1-dev.20240516',
            '5.5.1-dev.20240515',
            '5.5.1-dev.20240514',
            '5.5.1-dev.20240513',
            '5.5.1-dev.20240512',
            '5.5.1-dev.20240511',
            '5.5.1-dev.20240510',
            '5.5.1-dev.20240509',
            '5.5.1-dev.20240508',
            '5.5.1-dev.20240507',
            '5.5.1-dev.20240506',
            '5.5.1-dev.20240505',
            '5.5.1-dev.20240504',
            '5.5.1-dev.20240503',
            '5.5.1-dev.20240502',
            '5.5.1-dev.20240501',
            '5.5.1-dev.20240430',
            '5.5.1-dev.20240429',
            '5.5.1-dev.20240428',
            '5.5.1-dev.20240427',
            '5.5.1-dev.20240426',
            '5.5.1-dev.20240425',
            '5.5.1-dev.20240424',
            '5.5.1-dev.20240423',
            '5.5.1-dev.20240422',
            '5.5.1-dev.20240421',
            '5.5.1-dev.20240420',
            '5.5.1-dev.20240419',
            '5.5.1-dev.20240418',
            '5.5.1-dev.20240417',
            '5.5.1-dev.20240416',
            '5.5.1-dev.20240415',
            '5.5.1-dev.20240414',
            '5.5.1-dev.20240413',
            '5.5.1-dev.20240412',
            '5.5.1-dev.20240411',
            '5.5.1-dev.20240410',
            '5.5.1-dev.20240409',
            '5.5.1-dev.20240408',
            '5.5.1-dev.20240407',
            '5.5.1-dev.20240406',
            '5.5.1-dev.20240405',
            '5.5.1-dev.20240404',
            '5.5.1-dev.20240403',
            '5.5.1-dev.20240402',
            '5.5.1-dev.20240401',
            '5.5.1-dev.20240331',
            '5.5.1-dev.20240330',
            '5.5.1-dev.20240329',
            '5.5.1-dev.20240328',
            '5.5.1-dev.20240327',
            '5.5.1-dev.20240326',
            '5.5.1-dev.20240325',
            '5.5.1-dev.20240324',
            '5.5.1-dev.20240323',
            '5.5.1-dev.20240322',
            '5.5.1-dev.20240321',
            '5.5.1-dev.20240320',
            '5.5.1-dev.20240319',
            '5.5.1-dev.20240318',
            '5.5.1-dev.20240317',
            '5.5.1-dev.20240316',
            '5.5.1-dev.20240315',
            '5.5.1-dev.20240314',
            '5.5.1-dev.20240313',
            '5.5.1-dev.20240312',
            '5.5.1-dev.20240311',
            '5.5.1-dev.20240310',
            '5.5.1-dev.20240309',
            '5.5.1-dev.20240308',
            '5.5.1-dev.20240307',
            '5.5.1-dev.20240306',
            '5.5.1-dev.20240305',
            '5.5.1-dev.20240304',
            '5.5.1-dev.20240303',
            '5.5.1-dev.20240302',
            '5.5.1-dev.20240301',
            '5.5.1-dev.20240229',
            '5.5.1-dev.20240228',
            '5.5.1-dev.20240227',
            '5.5.1-dev.20240226',
            '5.5.1-dev.20240225',
            '5.5.1-dev.20240224',
            '5.5.1-dev.20240223',
            '5.5.1-dev.20240222',
            '5.5.1-dev.20240221',
            '5.5.0-dev.20240201',
            '5.5.0-dev.20240131',
            '5.5.0-dev.20240130',
            '5.5.0-dev.20240129',
            '5.5.0-dev.20240128',
            '5.5.0-dev.20240127',
            '5.5.0-dev.20240126',
            '5.5.0-dev.20240125',
            '5.5.0-dev.20240124',
            '5.5.0-dev.20240123',
            '5.5.0-dev.20240122',
            '5.5.0-dev.20240121',
            '5.5.0-dev.20240120',
            '5.5.0-dev.20240119',
            '5.5.0-dev.20240118',
            '5.5.0-dev.20240117',
            '5.5.0-dev.20240116',
            '5.5.0-dev.20240115',
            '5.5.0-dev.20240114',
            '5.5.0-dev.20240113',
            '5.5.0-dev.20240112',
            '5.5.0-dev.20240111',
            '5.5.0-dev.20240110',
            '5.5.0-dev.20240109',
            '5.5.0-dev.20240108',
            '5.5.0-dev.20240107',
            '5.5.0-dev.20240106',
            '5.5.0-dev.20240105',
            '5.5.0-dev.20240104',
            '5.5.0-dev.20240103',
            '5.5.0-dev.20240102',
            '5.5.0-dev.20240101',
            '5.5.0-dev.20231231',
            '5.5.0-dev.20231230',
            '5.5.0-dev.20231229',
            '5.5.0-dev.20231228',
            '5.5.0-dev.20231227',
            '5.5.0-dev.20231226',
            '5.5.0-dev.20231225',
            '5.5.0-dev.20231224',
            '5.5.0-dev.20231223',
            '5.5.0-dev.20231222',
            '5.5.0-dev.20231221',
            '5.5.0-dev.20231220',
            '5.5.0-dev.20231219',
            '5.5.0-dev.20231218',
            '5.5.0-dev.20231217',
            '5.5.0-dev.20231216',
            '5.5.0-dev.20231215',
            '5.5.0-dev.20231214',
            '5.5.0-dev.20231213',
            '5.5.0-dev.20231212',
            '5.5.0-dev.20231211',
            '5.5.0-dev.20231210',
            '5.5.0-dev.20231209',
            '5.5.0-dev.20231208',
            '5.5.0-dev.20231207',
            '5.5.0-dev.20231206',
            '5.5.0-dev.20231205',
            '5.5.0-dev.20231204',
            '5.5.0-dev.20231203',
            '5.5.0-dev.20231202',
            '5.5.0-dev.20231201',
            '5.5.0-dev.20231130',
            '5.5.0-dev.20231129',
            '5.5.0-dev.20231128',
            '5.5.0-dev.20231127',
            '5.5.0-dev.20231126',
            '5.5.0-dev.20231125',
            '5.5.0-dev.20231124',
            '5.5.0-dev.20231123',
            '5.5.0-dev.20231122',
            '5.5.0-dev.20231121',
            '5.5.0-dev.20231120',
            '5.5.0-dev.20231119',
            '5.5.0-dev.20231118',
            '5.5.0-dev.20231117',
            '5.5.0-dev.20231116',
            '5.5.0-dev.20231115',
            '5.5.0-dev.20231114',
            '5.5.0-dev.20231113',
            '5.5.0-dev.20231112',
            '5.5.0-dev.20231111',
            '5.5.0-dev.20231110',
            '5.5.0-dev.20231109',
            '5.5.0-dev.20231108',
            '5.5.0-dev.20231107',
            '5.5.0-dev.20231106',
            '5.5.0-dev.20231105',
            '5.5.0-dev.20231104',
            '5.5.0-dev.20231103',
            '5.5.0-dev.20231102',
            '5.5.0-dev.20231101',
            '5.5.0-dev.20231031',
            '5.5.0-dev.20231030',
            '5.5.0-dev.20231029',
            '5.5.0-dev.20231028',
            '5.5.0-dev.20231027',
            '5.5.0-dev.20231026',
            '5.5.0-dev.20231025',
            '5.5.0-dev.20231024',
            '5.5.0-dev.20231023',
            '5.5.0-dev.20231022',
            '5.5.0-dev.20231021',
            '5.5.0-dev.20231020',
            '5.5.0-dev.20231019',
            '5.5.0-dev.20231018',
            '5.5.0-dev.20231017',
            '5.5.0-dev.20231016',
            '5.5.0-dev.20231015',
            '5.5.0-dev.20231014',
            '5.5.0-dev.20231013',
            '5.5.0-dev.20231012',
            '5.5.0-dev.20231011',
            '5.5.0-dev.20231010',
            '5.5.0-dev.20231009',
            '5.5.0-dev.20231008',
            '5.5.0-dev.20231007',
            '5.5.0-dev.20231006',
            '5.5.0-dev.20231005',
            '5.5.0-dev.20231004',
            '5.5.0-dev.20231003',
            '5.5.0-dev.20231002',
            '5.5.0-dev.20231001',
            '5.5.0-dev.20230930',
            '5.5.0-dev.20230929',
            '5.5.0-dev.20230928',
            '5.5.0-dev.20230927',
            '5.5.0-dev.20230926',
            '5.5.0-dev.20230925',
            '5.5.0-dev.20230924',
            '5.5.0-dev.20230923',
            '5.5.0-dev.20230922',
            '5.5.0-dev.20230921',
            '5.5.0-dev.20230920',
            '5.5.0-dev.20230919',
            '5.5.0-dev.20230918',
            '5.5.0-dev.20230917',
            '5.5.0-dev.20230916',
            '5.5.0-dev.20230915',
            '5.5.0-dev.20230914',
            '5.5.0-dev.20230913',
            '5.5.0-dev.20230912',
            '5.5.0-dev.20230911',
            '5.5.0-dev.20230910',
            '5.5.0-dev.20230909',
            '5.5.0-dev.20230908',
            '5.5.0-dev.20230907',
            '5.5.0-dev.20230906',
            '5.5.0-dev.20230905',
            '5.5.0-dev.20230904',
            '5.5.0-dev.20230903',
            '5.5.0-dev.20230902',
            '5.5.0-dev.20230901',
            '5.5.0-dev.20230831',
            '5.5.0-dev.20230830',
            '5.5.0-dev.20230829',
            '5.5.0-dev.20230828',
            '5.5.0-dev.20230827',
            '5.5.0-dev.20230826',
            '5.5.0-dev.20230825',
            '5.5.0-dev.20230824',
            '5.5.0-dev.20230823',
            '5.5.0-dev.20230822',
            '5.5.0-dev.20230821',
            '5.5.0-dev.20230820',
            '5.5.0-dev.20230819',
            '5.5.0-dev.20230818',
            '5.5.0-dev.20230817',
            '5.5.0-dev.20230816',
            '5.5.0-dev.20230815',
            '5.5.0-dev.20230814',
            '5.5.0-dev.20230813',
            '5.5.0-dev.20230812',
            '5.5.0-dev.20230811',
            '5.5.0-dev.20230810',
            '5.5.0-dev.20230809',
            '5.5.0-dev.20230808',
            '5.5.0-dev.20230807',
            '5.5.0-dev.20230806',
            '5.5.0-dev.20230805',
            '5.5.0-dev.20230804',
            '5.5.0-dev.20230803',
            '5.5.0-dev.20230802',
            '5.5.0-dev.20230801',
            '5.5.0-dev.20230731',
            '5.5.0-dev.20230730',
            '5.5.0-dev.20230729',
            '5.5.0-dev.20230728',
            '5.5.0-dev.20230727',
            '5.5.0-dev.20230726',
            '5.5.0-dev.20230725',
            '5.5.0-dev.20230724',
            '5.5.0-dev.20230723',
            '5.5.0-dev.20230722',
            '5.5.0-dev.20230721',
            '5.5.0-dev.20230720',
            '5.5.0-dev.20230719',
            '5.5.0-dev.20230718',
            '5.5.0-dev.20230717',
            '5.5.0-dev.20230716',
            '5.5.0-dev.20230715',
            '5.5.0-dev.20230714',
            '5.5.0-dev.20230713',
            '5.5.0-dev.20230712',
            '5.5.0-dev.20230711',
            '5.5.0-dev.20230710',
            '5.5.0-dev.20230709',
            '5.5.0-dev.20230708',
            '5.5.0-dev.20230707',
            '5.5.0-dev.20230706',
            '5.5.0-dev.20230705',
            '5.5.0-dev.20230704',
            '5.5.0-dev.20230703',
            '5.5.0-dev.20230702',
            '5.5.0-dev.20230701',
            '5.5.0-dev.20230630',
            '5.5.0-dev.20230629',
            '5.5.0-dev.20230628',
            '5.5.0-dev.20230627',
            '5.5.0-dev.20230626',
            '5.5.0-dev.20230625',
            '5.5.0-dev.20230624',
            '5.5.0-dev.20230623',
            '5.5.0-dev.20230622',
            '5.5.0-dev.20230621',
            '5.5.0-dev.20230620',
            '5.5.0-dev.20230619',
            '5.5.0-dev.20230618',
            '5.5.0-dev.20230617',
            '5.5.0-dev.20230616',
            '5.5.0-dev.20230615',
            '5.5.0-dev.20230614',
            '5.5.0-dev.20230613',
            '5.5.0-dev.20230612',
            '5.5.0-dev.20230611',
            '5.5.0-dev.20230610',
            '5.5.0-dev.20230609',
            '5.5.0-dev.20230608',
            '5.5.0-dev.20230607',
            '5.5.0-dev.20230606',
            '5.5.0-dev.20230605',
            '5.5.0-dev.20230604',
            '5.5.0-dev.20230603',
            '5.5.0-dev.20230602',
            '5.5.0-dev.20230601',
            '5.5.0-dev.20230531',
            '5.5.0-dev.20230530',
            '5.5.0-dev.20230529',
            '5.5.0-dev.20230528',
            '5.5.0-dev.20230527',
            '5.5.0-dev.20230526',
            '5.5.0-dev.20230525',
            '5.5.0-dev.20230524',
            '5.5.0-dev.20230523',
            '5.5.0-dev.20230522',
            '5.5.0-dev.20230521',
            '5.5.0-dev.20230520',
            '5.5.0-dev.20230519',
            '5.5.0-dev.20230518',
            '5.5.0-dev.20230517',
            '5.5.0-dev.20230516',
            '5.5.0-dev.20230515',
            '5.5.0-dev.20230514',
            '5.5.0-dev.20230513',
            '5.5.0-dev.20230512',
            '5.5.0-dev.20230511',
            '5.5.0-dev.20230510',
            '5.5.0-dev.20230509',
            '5.5.0-dev.20230508',
            '5.5.0-dev.20230507',
            '5.5.0-dev.20230506',
            '5.5.0-dev.20230505',
            '5.5.0-dev.20230504',
            '5.5.0-dev.20230503',
            '5.5.0-dev.20230502',
            '5.5.0-dev.20230501',
            '5.5.0-dev.20230430',
            '5.5.0-dev.20230429',
            '5.5.0-dev.20230428',
            '5.5.0-dev.20230427',
            '5.5.0-dev.20230426',
            '5.5.0-dev.20230425',
            '5.5.0-dev.20230419',
            '5.5.0-dev.20230418',
            '5.5.0-dev.20230417',
            '5.5.0-dev.20230416',
            '5.5.0-dev.20230415',
            '5.5.0-dev.20230414',
            '5.5.0-dev.20230413',
            '5.5.0-dev.20230412',
            '5.5.0-dev.20230411',
            '5.5.0-dev.20230410',
            '5.5.0-dev.20230409',
            '5.5.0-dev.20230408',
            '5.5.0-dev.20230407',
            '5.5.0-dev.20230406',
            '5.5.0-dev.20230405',
            '5.5.0-dev.20230404',
            '5.5.0-dev.20230403',
            '5.5.0-dev.20230402',
            '5.5.0-dev.20230401',
            '5.5.0-dev.20230331',
            '5.5.0-dev.20230330',
            '5.5.0-dev.20230329',
            '5.5.0-dev.20230328',
            '5.5.0-dev.20230327',
            '5.5.0-dev.20230326',
            '5.5.0-dev.20230325',
            '5.5.0-dev.20230324',
            '5.5.0-dev.20230323',
            '5.5.0-dev.20230322',
            '5.5.0-dev.20230321',
            '5.5.0-dev.20230320',
            '5.5.0-dev.20230319',
            '5.5.0-dev.20230318',
            '5.5.0-dev.20230317',
            '5.5.0-dev.20230316',
            '5.5.0-dev.20230315',
            '5.5.0-dev.20230314',
            '5.5.0-dev.20230313',
            '5.5.0-dev.20230312',
            '5.5.0-dev.20230311',
            '5.5.0-dev.20230310',
            '5.5.0-dev.20230309',
            '5.5.0-dev.20230308',
            '5.5.0-dev.20230307',
            '5.5.0-dev.20230306',
            '5.5.0-dev.20230305',
            '5.5.0-dev.20230304',
            '5.5.0-dev.20230303',
            '5.5.0-dev.20230302',
            '5.5.0-dev.20230301',
            '5.5.0-dev.20230228',
            '5.5.0-dev.20230227',
            '5.5.0-dev.20230226',
            '5.5.0-dev.20230225',
            '5.5.0-dev.20230224',
            '5.5.0-dev.20230223',
            '5.5.0-dev.20230222',
            '5.5.0-dev.20230221',
            '5.5.0-dev.20230220',
            '5.5.0-dev.20230219',
            '5.5.0-dev.20230218',
            '5.5.0-dev.20230217',
            '5.5.0-dev.20230216',
            '5.5.0-dev.20230215',
            '5.5.0-dev.20230214',
            '5.5.0-dev.20230213',
            '5.5.0-dev.20230212',
            '5.5.0-dev.20230211',
            '5.5.0-dev.20230210',
            '5.5.0-dev.20230209',
            '5.5.0-dev.20230208',
            '5.5.0-dev.20230207',
            '5.5.0-dev.20230206',
            '5.5.0-dev.20230205',
            '5.5.0-dev.20230204',
            '5.5.0-dev.20230203',
            '5.5.0-dev.20230202',
            '5.5.0-dev.20230201',
            '5.5.0-dev.20230131',
            '5.5.0-dev.20230130',
            '5.5.0-dev.20230129',
            '5.5.0-dev.20230128',
            '5.5.0-dev.20230127',
            '5.5.0-dev.20230126',
            '5.5.0-dev.20230125',
            '5.5.0-dev.20230124',
            '5.5.0-dev.20230123',
            '5.5.0-dev.20230122',
            '5.5.0-dev.20230121',
            '5.5.0-dev.20230120',
            '5.5.0-dev.20230119',
            '5.5.0-dev.20230118',
            '5.5.0-dev.20230117',
            '5.5.0-dev.20230116',
            '5.5.0-dev.20230115',
            '5.5.0-dev.20230114',
            '5.5.0-dev.20230113',
            '5.5.0-dev.20230112',
            '5.5.0-dev.20230111',
            '5.5.0-dev.20230110',
            '5.5.0-dev.20230109',
            '5.5.0-dev.20230108',
            '5.5.0-dev.20230107',
            '5.5.0-dev.20230106',
            '5.5.0-dev.20230105',
            '5.5.0-dev.20230104',
            '5.5.0-dev.20230103',
            '5.5.0-dev.20230102',
            '5.5.0-dev.20230101',
            '5.5.0-dev.20221231',
            '5.5.0-dev.20221230',
            '5.5.0-dev.20221229',
            '5.5.0-dev.20221228',
            '5.5.0-dev.20221227',
            '5.5.0-dev.20221226',
            '5.5.0-dev.20221225',
            '5.5.0-dev.20221224',
            '5.5.0-dev.20221223',
            '5.5.0-dev.20221222',
            '5.5.0-dev.20221221',
            '5.5.0-dev.20221220',
            '5.5.0-dev.20221219',
            '5.5.0-dev.20221218',
            '5.5.0-dev.20221217',
            '5.5.0-dev.20221216',
            '5.5.0-dev.20221215',
            '5.5.0-dev.20221214',
            '5.5.0-dev.20221213',
            '5.5.0-dev.20221212',
            '5.5.0-dev.20221211',
            '5.5.0-dev.20221210',
            '5.5.0-dev.20221128',
            '5.5.0-dev.20221127',
            '5.5.0-dev.20221126',
            '5.5.0-dev.20221125',
            '5.5.0-dev.20221124',
            '5.5.0-dev.20221123',
            '5.5.0-dev.20221122',
            '5.5.0-dev.20221121',
            '5.5.0-dev.20221120',
            '5.5.0-dev.20221119',
            '5.5.0-dev.20221118',
            '5.5.0-dev.20221117',
            '5.5.0-dev.20221116',
            '5.5.0-dev.20221115',
            '5.5.0-dev.20221114',
            '5.5.0-dev.20221113',
            '5.5.0-dev.20221112',
            '5.5.0-dev.20221111',
            '5.5.0-dev.20221110',
            '5.5.0-dev.20221109',
            '5.5.0-dev.20221108',
            '5.5.0-dev.20221107',
            '5.5.0-dev.20221106',
            '5.5.0-dev.20221105',
            '5.5.0-dev.20221104',
            '5.5.0-dev.20221103',
            '5.5.0-dev.20221102',
            '5.5.0-dev.20221101',
            '5.5.0-dev.20221031',
            '5.5.0-dev.20221030',
            '5.5.0-dev.20221029',
            '5.5.0-dev.20221027',
            '5.5.0-dev.20221026',
            '5.5.0-dev.20221025',
            '5.5.0-dev.20221024',
            '5.5.0-dev.20221023',
            '5.5.0-dev.20221022',
            '5.5.0-dev.20221021',
            '5.5.0-dev.20221020',
            '5.5.0-dev.20221019',
            '5.5.0-dev.20221018',
            '5.5.0-dev.20221017',
            '5.5.0-dev.20221016',
            '5.5.0-dev.20221015',
            '5.5.0-dev.20221014',
            '5.5.0-dev.20221013',
            '5.5.0-dev.20221012',
            '5.5.0-dev.20221011',
            '5.5.0-dev.20221010',
            '5.5.0-dev.20221009',
            '5.5.0-dev.20221008',
            '5.5.0-dev.20221007',
            '5.5.0-dev.20221006',
            '5.5.0-dev.20221005',
            '5.5.0-dev.20221004',
            '5.5.0-dev.20221003',
            '5.5.0-dev.20221002',
            '5.5.0-dev.20221001',
            '5.5.0-dev.20220930',
            '5.4.4-dev.20240129',
            '5.4.4-dev.20240128',
            '5.4.4-dev.20240127',
            '5.4.4-dev.20240126',
            '5.4.4-dev.20240125',
            '5.4.4-dev.20240124',
            '5.4.4-dev.20240123',
            '5.4.4-dev.20240122',
            '5.4.4-dev.20240121',
            '5.4.4-dev.20240120',
            '5.4.4-dev.20240119',
            '5.4.4-dev.20240118',
            '5.4.4-dev.20240117',
            '5.4.4-dev.20240116',
            '5.4.4-dev.20240115',
            '5.4.4-dev.20240114',
            '5.4.4-dev.20240113',
            '5.4.4-dev.20240112',
            '5.4.4-dev.20240111',
            '5.4.4-dev.20240110',
            '5.4.4-dev.20240109',
            '5.4.4-dev.20240108',
            '5.4.4-dev.20240107',
            '5.4.4-dev.20240106',
            '5.4.4-dev.20240105',
            '5.4.4-dev.20240104',
            '5.4.4-dev.20240103',
            '5.4.4-dev.20240102',
            '5.4.4-dev.20240101',
            '5.4.4-dev.20231231',
            '5.4.4-dev.20231230',
            '5.4.4-dev.20231229',
            '5.4.4-dev.20231228',
            '5.4.4-dev.20231227',
            '5.4.4-dev.20231226',
            '5.4.4-dev.20231225',
            '5.4.4-dev.20231224',
            '5.4.4-dev.20231223',
            '5.4.4-dev.20231222',
            '5.4.4-dev.20231221',
            '5.4.4-dev.20231220',
            '5.4.4-dev.20231219',
            '5.4.4-dev.20231218',
            '5.4.4-dev.20231217',
            '5.4.4-dev.20231216',
            '5.4.4-dev.20231215',
            '5.4.4-dev.20231214',
            '5.4.4-dev.20231213',
            '5.4.4-dev.20231212',
            '5.4.4-dev.20231211',
            '5.4.4-dev.20231210',
            '5.4.4-dev.20231209',
            '5.4.4-dev.20231208',
            '5.4.4-dev.20231207',
            '5.4.4-dev.20231206',
            '5.4.4-dev.20231205',
            '5.4.4-dev.20231204',
            '5.4.4-dev.20231203',
            '5.4.4-dev.20231202',
            '5.4.4-dev.20231201',
            '5.4.4-dev.20231130',
            '5.4.4-dev.20231129',
            '5.4.4-dev.20231128',
            '5.4.4-dev.20231127',
            '5.4.4-dev.20231126',
            '5.4.4-dev.20231125',
            '5.4.4-dev.20231124',
            '5.4.4-dev.20231123',
            '5.4.4-dev.20231122',
            '5.4.4-dev.20231121',
            '5.4.4-dev.20231120',
            '5.4.4-dev.20231119',
            '5.4.4-dev.20231118',
            '5.4.4-dev.20231117',
            '5.4.4-dev.20231116',
            '5.4.4-dev.20231115',
            '5.4.4-dev.20231114',
            '5.4.4-dev.20231113',
            '5.4.4-dev.20231112',
            '5.4.4-dev.20231111',
            '5.4.4-dev.20231110',
            '5.4.4-dev.20231109',
            '5.4.4-dev.20231108',
            '5.4.4-dev.20231107',
            '5.4.4-dev.20231106',
            '5.4.4-dev.20231105',
            '5.4.4-dev.20231104',
            '5.4.4-dev.20231103',
            '5.4.4-dev.20231102',
            '5.4.4-dev.20231101',
            '5.4.4-dev.20231031',
            '5.4.4-dev.20231030',
            '5.4.4-dev.20231029',
            '5.4.4-dev.20231028',
            '5.4.4-dev.20231027',
            '5.4.4-dev.20231026',
            '5.4.4-dev.20231025',
            '5.4.4-dev.20231024',
            '5.4.4-dev.20231023',
            '5.4.4-dev.20231022',
            '5.4.4-dev.20231021',
            '5.4.4-dev.20231020',
            '5.4.4-dev.20231019',
            '5.4.4-dev.20231018',
            '5.4.4-dev.20231017',
            '5.4.4-dev.20231016',
            '5.4.4-dev.20231015',
            '5.4.4-dev.20231014',
            '5.4.4-dev.20231013',
            '5.4.4-dev.20231012',
            '5.4.4-dev.20231011',
            '5.4.4-dev.20231010',
            '5.4.4-dev.20231009',
            '5.4.4-dev.20231008',
            '5.4.4-dev.20231007',
            '5.4.4-dev.20231006',
            '5.4.4-dev.20231005',
            '5.4.4-dev.20231004',
            '5.4.4-dev.20231003',
            '5.4.4-dev.20231002',
            '5.4.4-dev.20231001',
            '5.4.4-dev.20230930',
            '5.4.4-dev.20230929',
            '5.4.4-dev.20230928',
            '5.4.4-dev.20230927',
            '5.4.4-dev.20230926',
            '5.4.4-dev.20230925',
            '5.4.4-dev.20230924',
            '5.4.4-dev.20230923',
            '5.4.4-dev.20230922',
            '5.4.4-dev.20230921',
            '5.4.4-dev.20230920',
            '5.4.4-dev.20230919',
            '5.4.4-dev.20230918',
            '5.4.4-dev.20230917',
            '5.4.4-dev.20230916',
            '5.4.4-dev.20230915',
            '5.4.4-dev.20230914',
            '5.4.4-dev.20230913',
            '5.4.4-dev.20230912',
            '5.4.4-dev.20230911',
            '5.4.4-dev.20230910',
            '5.4.4-dev.20230909',
            '5.4.4-dev.20230908',
            '5.4.4-dev.20230907',
            '5.4.4-dev.20230906',
            '5.4.4-dev.20230905',
            '5.4.4-dev.20230904',
            '5.4.4-dev.20230903',
            '5.4.4-dev.20230902',
            '5.4.4-dev.20230901',
            '5.4.4-dev.20230831',
            '5.4.4-dev.20230830',
            '5.4.4-dev.20230829',
            '5.4.4-dev.20230828',
            '5.4.4-dev.20230827',
            '5.4.4-dev.20230826',
            '5.4.4-dev.20230825',
            '5.4.4-dev.20230824',
            '5.4.4-dev.20230823',
            '5.4.4-dev.20230822',
            '5.4.4-dev.20230821',
            '5.4.4-dev.20230820',
            '5.4.4-dev.20230819',
            '5.4.4-dev.20230818',
            '5.4.4-dev.20230817',
            '5.4.4-dev.20230816',
            '5.4.4-dev.20230815',
            '5.4.4-dev.20230814',
            '5.4.4-dev.20230813',
            '5.4.4-dev.20230812',
            '5.4.4-dev.20230811',
            '5.4.4-dev.20230810',
            '5.4.4-dev.20230809',
            '5.4.4-dev.20230808',
            '5.4.4-dev.20230807',
            '5.4.4-dev.20230806',
            '5.4.4-dev.20230805',
            '5.4.4-dev.20230804',
            '5.4.4-dev.20230803',
            '5.4.4-dev.20230802',
            '5.4.4-dev.20230801',
            '5.4.4-dev.20230731',
            '5.4.4-dev.20230730',
            '5.4.4-dev.20230729',
            '5.4.4-dev.20230728',
            '5.4.4-dev.20230727',
            '5.4.4-dev.20230726',
            '5.4.4-dev.20230725',
            '5.4.4-dev.20230724',
            '5.4.4-dev.20230723',
            '5.4.4-dev.20230722',
            '5.4.4-dev.20230721',
            '5.4.4-dev.20230720',
            '5.4.4-dev.20230719',
            '5.4.4-dev.20230718',
            '5.4.3-dev.20230717',
            '5.4.3-dev.20230716',
            '5.4.3-dev.20230715',
            '5.4.3-dev.20230714',
            '5.4.3-dev.20230713',
            '5.4.3-dev.20230712',
            '5.4.3-dev.20230711',
            '5.4.3-dev.20230710',
            '5.4.3-dev.20230709',
            '5.4.3-dev.20230708',
            '5.4.3-dev.20230707',
            '5.4.3-dev.20230706',
            '5.4.3-dev.20230705',
            '5.4.3-dev.20230704',
            '5.4.3-dev.20230703',
            '5.4.3-dev.20230702',
            '5.4.3-dev.20230701',
            '5.4.3-dev.20230630',
            '5.4.3-dev.20230629',
            '5.4.3-dev.20230628',
            '5.4.3-dev.20230627',
            '5.4.3-dev.20230626',
            '5.4.3-dev.20230625',
            '5.4.3-dev.20230624',
            '5.4.3-dev.20230623',
            '5.4.3-dev.20230622',
            '5.4.3-dev.20230621',
            '5.4.3-dev.20230620',
            '5.4.3-dev.20230619',
            '5.4.3-dev.20230618',
            '5.4.3-dev.20230617',
            '5.4.3-dev.20230616',
            '5.4.3-dev.20230615',
            '5.4.3-dev.20230614',
            '5.4.3-dev.20230613',
            '5.4.3-dev.20230612',
            '5.4.3-dev.20230611',
            '5.4.3-dev.20230610',
            '5.4.3-dev.20230609',
            '5.4.3-dev.20230608',
            '5.4.3-dev.20230607',
            '5.4.3-dev.20230606',
            '5.4.3-dev.20230605',
            '5.4.3-dev.20230604',
            '5.4.3-dev.20230603',
            '5.4.3-dev.20230602',
            '5.4.3-dev.20230601',
            '5.4.3-dev.20230531',
            '5.4.3-dev.20230530',
            '5.4.3-dev.20230529',
            '5.4.3-dev.20230528',
            '5.4.3-dev.20230527',
            '5.4.3-dev.20230526',
            '5.4.3-dev.20230525',
            '5.4.3-dev.20230524',
            '5.4.3-dev.20230523',
            '5.4.3-dev.20230522',
            '5.4.3-dev.20230521',
            '5.4.3-dev.20230520',
            '5.4.3-dev.20230519',
            '5.4.3-dev.20230518',
            '5.4.3-dev.20230517',
            '5.4.3-dev.20230516',
            '5.4.3-dev.20230515',
            '5.4.3-dev.20230514',
            '5.4.3-dev.20230513',
            '5.4.3-dev.20230512',
            '5.4.3-dev.20230511',
            '5.4.3-dev.20230510',
            '5.4.3-dev.20230509',
            '5.4.3-dev.20230508',
            '5.4.3-dev.20230507',
            '5.4.3-dev.20230506',
            '5.4.3-dev.20230505',
            '5.4.3-dev.20230504',
            '5.4.3-dev.20230503',
            '5.4.3-dev.20230502',
            '5.4.3-dev.20230501',
            '5.4.3-dev.20230430',
            '5.4.3-dev.20230429',
            '5.4.3-dev.20230428',
            '5.4.3-dev.20230427',
            '5.4.3-dev.20230426',
            '5.4.3-dev.20230425',
            '5.4.3-dev.20230424',
            '5.4.3-dev.20230418',
            '5.4.3-dev.20230417',
            '5.4.3-dev.20230416',
            '5.4.3-dev.20230415',
            '5.4.3-dev.20230414',
            '5.4.3-dev.20230413',
            '5.4.3-dev.20230412',
            '5.4.3-dev.20230411',
            '5.4.3-dev.20230410',
            '5.4.3-dev.20230409',
            '5.4.3-dev.20230408',
            '5.4.3-dev.20230407',
            '5.4.3-dev.20230406',
            '5.4.3-dev.20230405',
            '5.4.3-dev.20230404',
            '5.4.3-dev.20230403',
            '5.4.3-dev.20230402',
            '5.4.3-dev.20230401',
            '5.4.3-dev.20230331',
            '5.4.3-dev.20230330',
            '5.4.3-dev.20230329',
            '5.4.3-dev.20230328',
            '5.4.3-dev.20230327',
            '5.4.3-dev.20230326',
            '5.4.3-dev.20230325',
            '5.4.3-dev.20230324',
            '5.4.3-dev.20230323',
            '5.4.2-dev.20230322',
            '5.4.2-dev.20230321',
            '5.4.2-dev.20230320',
            '5.4.2-dev.20230319',
            '5.4.2-dev.20230318',
            '5.4.2-dev.20230317',
            '5.4.2-dev.20230316',
            '5.4.2-dev.20230315',
            '5.4.2-dev.20230314',
            '5.4.2-dev.20230313',
            '5.4.2-dev.20230312',
            '5.4.2-dev.20230311',
            '5.4.2-dev.20230310',
            '5.4.2-dev.20230309',
            '5.4.2-dev.20230308',
            '5.4.2-dev.20230307',
            '5.4.2-dev.20230306',
            '5.4.2-dev.20230305',
            '5.4.2-dev.20230304',
            '5.4.2-dev.20230303',
            '5.4.2-dev.20230302',
            '5.4.2-dev.20230301',
            '5.4.2-dev.20230228',
            '5.4.2-dev.20230227',
            '5.4.2-dev.20230226',
            '5.4.2-dev.20230225',
            '5.4.2-dev.20230224',
            '5.4.2-dev.20230223',
            '5.4.2-dev.20230222',
            '5.4.2-dev.20230221',
            '5.4.2-dev.20230220',
            '5.4.2-dev.20230219',
            '5.4.2-dev.20230218',
            '5.4.2-dev.20230217',
            '5.4.2-dev.20230216',
            '5.4.2-dev.20230215',
            '5.4.2-dev.20230214',
            '5.4.2-dev.20230213',
            '5.4.2-dev.20230212',
            '5.4.2-dev.20230211',
            '5.4.2-dev.20230210',
            '5.4.2-dev.20230209',
            '5.4.2-dev.20230208',
            '5.4.2-dev.20230207',
            '5.4.2-dev.20230206',
            '5.4.2-dev.20230205',
            '5.4.2-dev.20230204',
            '5.4.2-dev.20230203',
            '5.4.2-dev.20230202',
            '5.4.2-dev.20230201',
            '5.4.2-dev.20230131',
            '5.4.2-dev.20230130',
            '5.4.2-dev.20230129',
            '5.4.2-dev.20230128',
            '5.4.2-dev.20230127',
            '5.4.2-dev.20230126',
            '5.4.2-dev.20230125',
            '5.4.2-dev.20230124',
            '5.4.2-dev.20230123',
            '5.4.2-dev.20230122',
            '5.4.2-dev.20230121',
            '5.4.2-dev.20230120',
            '5.4.2-dev.20230119',
            '5.4.2-dev.20230118',
            '5.4.2-dev.20230117',
            '5.4.2-dev.20230116',
            '5.4.2-dev.20230115',
            '5.4.2-dev.20230114',
            '5.4.2-dev.20230113',
            '5.4.2-dev.20230112',
            '5.4.2-dev.20230111',
            '5.4.2-dev.20230110',
            '5.4.2-dev.20230109',
            '5.4.2-dev.20230108',
            '5.4.2-dev.20230107',
            '5.4.2-dev.20230106',
            '5.4.2-dev.20230105',
            '5.4.2-dev.20230104',
            '5.4.2-dev.20230103',
            '5.4.2-dev.20230102',
            '5.4.2-dev.20230101',
            '5.4.2-dev.20221231',
            '5.4.2-dev.20221230',
            '5.4.2-dev.20221229',
            '5.4.2-dev.20221228',
            '5.4.2-dev.20221227',
            '5.4.2-dev.20221226',
            '5.4.2-dev.20221225',
            '5.4.2-dev.20221224',
            '5.4.2-dev.20221223',
            '5.4.2-dev.20221222',
            '5.4.2-dev.20221221',
            '5.4.2-dev.20221220',
            '5.4.2-dev.20221219',
            '5.4.2-dev.20221218',
            '5.4.2-dev.20221217',
            '5.4.2-dev.20221216',
            '5.4.2-dev.20221215',
            '5.4.2-dev.20221214',
            '5.4.2-dev.20221213',
            '5.4.2-dev.20221212',
            '5.4.2-dev.20221211',
            '5.4.2-dev.20221210',
            '5.4.1-dev.20221209',
            '5.4.1-dev.20221208',
            '5.4.1-dev.20221207',
            '5.4.1-dev.20221206',
            '5.4.1-dev.20221205',
            '5.4.1-dev.20221204',
            '5.4.1-dev.20221203',
            '5.4.1-dev.20221202',
            '5.4.1-dev.20221201',
            '5.4.1-dev.20221130',
            '5.4.1-dev.20221129',
            '5.4.1-dev.20221128',
            '5.4.1-dev.20221127',
            '5.4.1-dev.20221126',
            '5.4.1-dev.20221125',
            '5.4.1-dev.20221124',
            '5.4.1-dev.20221123',
            '5.4.1-dev.20221122',
            '5.4.1-dev.20221121',
            '5.4.1-dev.20221120',
            '5.4.1-dev.20221119',
            '5.4.1-dev.20221118',
            '5.4.1-dev.20221117',
            '5.4.1-dev.20221116',
            '5.4.1-dev.20221115',
            '5.4.1-dev.20221114',
            '5.4.1-dev.20221113',
            '5.4.1-dev.20221112',
            '5.4.1-dev.20221111',
            '5.4.1-dev.20221110',
            '5.4.1-dev.20221109',
            '5.4.1-dev.20221108',
            '5.4.1-dev.20221107',
            '5.4.1-dev.20221106',
            '5.4.1-dev.20221031',
            '5.4.1-dev.20221030',
            '5.4.1-dev.20221029',
            '5.4.1-dev.20221028',
            '5.4.1-dev.20221027',
            '5.4.1-dev.20221026',
            '5.4.1-dev.20221025',
            '5.4.1-dev.20221024',
            '5.4.1-dev.20221023',
            '5.4.1-dev.20221022',
            '5.4.1-dev.20221021',
            '5.4.1-dev.20221020',
            '5.4.1-dev.20221019',
            '5.4.1-dev.20221018',
            '5.4.1-dev.20221017',
            '5.4.1-dev.20221016',
            '5.4.1-dev.20221015',
            '5.4.1-dev.20221014',
            '5.4.1-dev.20221013',
            '5.4.1-dev.20221012',
            '5.4.1-dev.20221011',
            '5.4.1-dev.20221010',
            '5.4.1-dev.20221009',
            '5.4.1-dev.20221008',
            '5.4.1-dev.20221007',
            '5.4.1-dev.20221006',
            '5.4.1-dev.20221005',
            '5.4.1-dev.20221004',
            '5.4.1-dev.20221003',
            '5.4.1-dev.20221002',
            '5.4.1-dev.20221001',
            '5.4.1-dev.20220930',
            '5.4.1-dev.20220929',
            '5.4.1-dev.20220928',
            '5.4.1-dev.20220927',
            '5.4.1-dev.20220926',
            '5.4.0-dev.20220929',
            '5.4.0-dev.20220928',
            '5.4.0-dev.20220927',
            '5.4.0-dev.20220926',
            '5.4.0-dev.20220925',
            '5.4.0-dev.20220924',
            '5.4.0-dev.20220923',
            '5.4.0-dev.20220922',
            '5.4.0-dev.20220921',
            '5.4.0-dev.20220920',
            '5.4.0-dev.20220919',
            '5.4.0-dev.20220918',
            '5.4.0-dev.20220917',
            '5.4.0-dev.20220916',
            '5.4.0-dev.20220915',
            '5.4.0-dev.20220914',
            '5.4.0-dev.20220913',
            '5.4.0-dev.20220912',
            '5.4.0-dev.20220911',
            '5.4.0-dev.20220910',
            '5.4.0-dev.20220909',
            '5.4.0-dev.20220908',
            '5.4.0-dev.20220907',
            '5.4.0-dev.20220906',
            '5.4.0-dev.20220905',
            '5.4.0-dev.20220904',
            '5.4.0-dev.20220903',
            '5.4.0-dev.20220902',
            '5.4.0-dev.20220901',
            '5.4.0-dev.20220831',
            '5.4.0-dev.20220830',
            '5.4.0-dev.20220829',
            '5.4.0-dev.20220828',
            '5.4.0-dev.20220827',
            '5.4.0-dev.20220826',
            '5.4.0-dev.20220825',
            '5.4.0-dev.20220824',
            '5.4.0-dev.20220823',
            '5.4.0-dev.20220822',
            '5.4.0-dev.20220821',
            '5.4.0-dev.20220820',
            '5.4.0-dev.20220819',
            '5.4.0-dev.20220818',
            '5.4.0-dev.20220817',
            '5.4.0-dev.20220816',
            '5.4.0-dev.20220815',
            '5.4.0-dev.20220814',
            '5.4.0-dev.20220813',
            '5.4.0-dev.20220812',
            '5.4.0-dev.20220811',
            '5.4.0-dev.20220810',
            '5.4.0-dev.20220809',
            '5.4.0-dev.20220808',
            '5.4.0-dev.20220807',
            '5.4.0-dev.20220806',
            '5.4.0-dev.20220805',
            '5.4.0-dev.20220804',
            '5.4.0-dev.20220803',
            '5.4.0-dev.20220802',
            '5.4.0-dev.20220801',
            '5.4.0-dev.20220731',
            '5.4.0-dev.20220730',
            '5.4.0-dev.20220729',
            '5.4.0-dev.20220728',
            '5.4.0-dev.20220727',
            '5.4.0-dev.20220726',
            '5.4.0-dev.20220725',
            '5.4.0-dev.20220724',
            '5.4.0-dev.20220723',
            '5.4.0-dev.20220722',
            '5.4.0-dev.20220721',
            '5.4.0-dev.20220720',
            '5.4.0-dev.20220719',
            '5.4.0-dev.20220718',
            '5.4.0-dev.20220717',
            '5.4.0-dev.20220716',
            '5.4.0-dev.20220715',
            '5.4.0-dev.20220714',
            '5.4.0-dev.20220713',
            '5.4.0-dev.20220712',
            '5.4.0-dev.20220711',
            '5.4.0-dev.20220710',
            '5.4.0-dev.20220709',
            '5.4.0-dev.20220708',
            '5.4.0-dev.20220707',
            '5.4.0-dev.20220706',
            '5.4.0-dev.20220705',
            '5.4.0-dev.20220704',
            '5.4.0-dev.20220703',
            '5.4.0-dev.20220702',
            '5.4.0-dev.20220701',
            '5.4.0-dev.20220630',
            '5.4.0-dev.20220629',
            '5.4.0-dev.20220628',
            '5.4.0-dev.20220627',
            '5.4.0-dev.20220626',
            '5.4.0-dev.20220625',
            '5.4.0-dev.20220624',
            '5.4.0-dev.20220623',
            '5.4.0-dev.20220622',
            '5.4.0-dev.20220621',
            '5.4.0-dev.20220620',
            '5.4.0-dev.20220619',
            '5.4.0-dev.20220618',
            '5.4.0-dev.20220617',
            '5.4.0-dev.20220616',
            '5.4.0-dev.20220615',
            '5.4.0-dev.20220614',
            '5.4.0-dev.20220613',
            '5.4.0-dev.20220612',
            '5.4.0-dev.20220611',
            '5.4.0-dev.20220610',
            '5.4.0-dev.20220609',
            '5.4.0-dev.20220608',
            '5.4.0-dev.20220607',
            '5.4.0-dev.20220606',
            '5.4.0-dev.20220605',
            '5.4.0-dev.20220604',
            '5.4.0-dev.20220603',
            '5.4.0-dev.20220602',
            '5.4.0-dev.20220601',
            '5.4.0-dev.20220531',
            '5.4.0-dev.20220530',
            '5.4.0-dev.20220529',
            '5.4.0-dev.20220528',
            '5.4.0-dev.20220527',
            '5.4.0-dev.20220526',
            '5.4.0-dev.20220525',
            '5.4.0-dev.20220524',
            '5.4.0-dev.20220523',
            '5.4.0-dev.20220522',
            '5.4.0-dev.20220521',
            '5.4.0-dev.20220520',
            '5.4.0-dev.20220519',
            '5.4.0-dev.20220518',
            '5.4.0-dev.20220517',
            '5.4.0-dev.20220516',
            '5.4.0-dev.20220515',
            '5.4.0-dev.20220514',
            '5.4.0-dev.20220513',
            '5.4.0-dev.20220512',
            '5.4.0-dev.20220511',
            '5.4.0-dev.20220510',
            '5.4.0-dev.20220509',
            '5.4.0-dev.20220508',
            '5.4.0-dev.20220507',
            '5.4.0-dev.20220506',
            '5.4.0-dev.20220505',
            '5.4.0-dev.20220504',
            '5.4.0-dev.20220503',
            '5.4.0-dev.20220502',
            '5.4.0-dev.20220501',
            '5.4.0-dev.20220430',
            '5.4.0-dev.20220429',
            '5.4.0-dev.20220428',
            '5.4.0-dev.20220427',
            '5.4.0-dev.20220426',
            '5.4.0-dev.20220425',
            '5.4.0-dev.20220424',
            '5.4.0-dev.20220423',
            '5.4.0-dev.20220422',
            '5.4.0-dev.20220421',
            '5.4.0-dev.20220420',
            '5.4.0-dev.20220419',
            '5.4.0-dev.20220418',
            '5.4.0-dev.20220417',
            '5.4.0-dev.20220416',
            '5.4.0-dev.20220415',
            '5.4.0-dev.20220414',
            '5.4.0-dev.20220413',
            '5.4.0-dev.20220412',
            '5.4.0-dev.20220411',
            '5.4.0-dev.20220410',
            '5.4.0-dev.20220409',
            '5.4.0-dev.20220408',
            '5.4.0-dev.20220407',
            '5.4.0-dev.20220406',
            '5.4.0-dev.20220405',
            '5.4.0-dev.20220404',
            '5.4.0-dev.20220403',
            '5.4.0-dev.20220402',
            '5.4.0-dev.20220401',
            '5.4.0-dev.20220331',
            '5.4.0-dev.20220330',
            '5.4.0-dev.20220328',
            '5.4.0-dev.20220327',
            '5.4.0-dev.20220326',
            '5.4.0-dev.20220325',
            '5.4.0-dev.20220324',
            '5.4.0-dev.20220323',
            '5.4.0-dev.20220322',
            '5.4.0-dev.20220321',
            '5.4.0-dev.20220320',
            '5.4.0-dev.20220319',
            '5.4.0-dev.20220318',
            '5.4.0-dev.20220317',
            '5.4.0-dev.20220316',
            '5.3.4-dev.20220925',
            '5.3.4-dev.20220924',
            '5.3.4-dev.20220923',
            '5.3.4-dev.20220922',
            '5.3.4-dev.20220921',
            '5.3.4-dev.20220920',
            '5.3.4-dev.20220919',
            '5.3.4-dev.20220918',
            '5.3.4-dev.20220917',
            '5.3.4-dev.20220916',
            '5.3.4-dev.20220915',
            '5.3.4-dev.20220914',
            '5.3.4-dev.20220913',
            '5.3.4-dev.20220912',
            '5.3.4-dev.20220911',
            '5.3.4-dev.20220910',
            '5.3.4-dev.20220909',
            '5.3.4-dev.20220908',
            '5.3.4-dev.20220907',
            '5.3.4-dev.20220906',
            '5.3.4-dev.20220905',
            '5.3.4-dev.20220904',
            '5.3.4-dev.20220903',
            '5.3.4-dev.20220902',
            '5.3.4-dev.20220901',
            '5.3.4-dev.20220831',
            '5.3.4-dev.20220830',
            '5.3.4-dev.20220829',
            '5.3.4-dev.20220828',
            '5.3.4-dev.20220827',
            '5.3.4-dev.20220826',
            '5.3.4-dev.20220825',
            '5.3.4-dev.20220824',
            '5.3.4-dev.20220823',
            '5.3.4-dev.20220822',
            '5.3.4-dev.20220821',
            '5.3.4-dev.20220820',
            '5.3.4-dev.20220819',
            '5.3.4-dev.20220818',
            '5.3.4-dev.20220817',
            '5.3.4-dev.20220816',
            '5.3.4-dev.20220815',
            '5.3.4-dev.20220814',
            '5.3.4-dev.20220813',
            '5.3.4-dev.20220812',
            '5.3.4-dev.20220811',
            '5.3.4-dev.20220810',
            '5.3.4-dev.20220809',
            '5.3.4-dev.20220808',
            '5.3.4-dev.20220807',
            '5.3.4-dev.20220806',
            '5.3.4-dev.20220805',
            '5.3.4-dev.20220804',
            '5.3.4-dev.20220803',
            '5.3.4-dev.20220802',
            '5.3.4-dev.20220801',
            '5.3.4-dev.20220731',
            '5.3.4-dev.20220730',
            '5.3.4-dev.20220729',
            '5.3.4-dev.20220728',
            '5.3.4-dev.20220727',
            '5.3.4-dev.20220726',
            '5.3.4-dev.20220725',
            '5.3.4-dev.20220724',
            '5.3.4-dev.20220723',
            '5.3.4-dev.20220722',
            '5.3.4-dev.20220721',
            '5.3.4-dev.20220720',
            '5.3.4-dev.20220719',
            '5.3.4-dev.20220718',
            '5.3.4-dev.20220717',
            '5.3.4-dev.20220716',
            '5.3.4-dev.20220715',
            '5.3.4-dev.20220714',
            '5.3.4-dev.20220713',
            '5.3.4-dev.20220712',
            '5.3.4-dev.20220711',
            '5.3.4-dev.20220710',
            '5.3.4-dev.20220709',
            '5.3.4-dev.20220708',
            '5.3.4-dev.20220707',
            '5.3.4-dev.20220706',
            '5.3.4-dev.20220705',
            '5.3.4-dev.20220704',
            '5.3.4-dev.20220703',
            '5.3.4-dev.20220702',
            '5.3.4-dev.20220701',
            '5.3.4-dev.20220630',
            '5.3.4-dev.20220629',
            '5.3.4-dev.20220628',
            '5.3.4-dev.20220627',
            '5.3.4-dev.20220626',
            '5.3.4-dev.20220625',
            '5.3.4-dev.20220624',
            '5.3.4-dev.20220623',
            '5.3.4-dev.20220622',
            '5.3.4-dev.20220621',
            '5.3.4-dev.20220620',
            '5.3.4-dev.20220619',
            '5.3.4-dev.20220618',
            '5.3.4-dev.20220617',
            '5.3.4-dev.20220616',
            '5.3.4-dev.20220615',
            '5.3.4-dev.20220614',
            '5.3.3-dev.20220613',
            '5.3.3-dev.20220612',
            '5.3.3-dev.20220611',
            '5.3.3-dev.20220610',
            '5.3.3-dev.20220609',
            '5.3.3-dev.20220608',
            '5.3.3-dev.20220607',
            '5.3.3-dev.20220606',
            '5.3.3-dev.20220605',
            '5.3.3-dev.20220604',
            '5.3.3-dev.20220603',
            '5.3.3-dev.20220602',
            '5.3.3-dev.20220601',
            '5.3.3-dev.20220530',
            '5.3.3-dev.20220529',
            '5.3.3-dev.20220527',
            '5.3.3-dev.20220526',
            '5.3.3-dev.20220525',
            '5.3.3-dev.20220524',
            '5.3.3-dev.20220523',
            '5.3.3-dev.20220522',
            '5.3.3-dev.20220521',
            '5.3.3-dev.20220520',
            '5.3.3-dev.20220519',
            '5.3.3-dev.20220518',
            '5.3.3-dev.20220517',
            '5.3.3-dev.20220516',
            '5.3.3-dev.20220515',
            '5.3.3-dev.20220514',
            '5.3.3-dev.20220513',
            '5.3.3-dev.20220512',
            '5.3.3-dev.20220511',
            '5.3.3-dev.20220510',
            '5.3.3-dev.20220509',
            '5.3.3-dev.20220508',
            '5.3.3-dev.20220507',
            '5.3.3-dev.20220506',
            '5.3.3-dev.20220505',
            '5.3.3-dev.20220504',
            '5.3.3-dev.20220503',
            '5.3.3-dev.20220502',
            '5.3.3-dev.20220501',
            '5.3.3-dev.20220430',
            '5.3.3-dev.20220429',
            '5.3.3-dev.20220428',
            '5.3.3-dev.20220427',
            '5.3.3-dev.20220426',
            '5.3.3-dev.20220425',
            '5.3.3-dev.20220424',
            '5.3.3-dev.20220423',
            '5.3.3-dev.20220422',
            '5.3.3-dev.20220421',
            '5.3.3-dev.20220420',
            '5.3.3-dev.20220419',
            '5.3.3-dev.20220418',
            '5.3.3-dev.20220417',
            '5.3.3-dev.20220416',
            '5.3.3-dev.20220415',
            '5.3.3-dev.20220414',
            '5.3.3-dev.20220413',
            '5.3.3-dev.20220412',
            '5.3.3-dev.20220411',
            '5.3.3-dev.20220410',
            '5.3.3-dev.20220409',
            '5.3.3-dev.20220408',
            '5.3.3-dev.20220407',
            '5.3.3-dev.20220406',
            '5.3.3-dev.20220405',
            '5.3.3-dev.20220404',
            '5.3.3-dev.20220403',
            '5.3.3-dev.20220402',
            '5.3.3-dev.20220401',
            '5.3.2-dev.20220331',
            '5.3.2-dev.20220330',
            '5.3.2-dev.20220329',
            '5.3.2-dev.20220328',
            '5.3.2-dev.20220327',
            '5.3.2-dev.20220326',
            '5.3.2-dev.20220325',
            '5.3.2-dev.20220324',
            '5.3.2-dev.20220323',
            '5.3.2-dev.20220322',
            '5.3.2-dev.20220321',
            '5.3.2-dev.20220320',
            '5.3.2-dev.20220319',
            '5.3.2-dev.20220318',
            '5.3.2-dev.20220317',
            '5.3.2-dev.20220316',
            '5.3.2-dev.20220315',
            '5.3.2-dev.20220314',
            '5.3.2-dev.20220313',
            '5.3.2-dev.20220312',
            '5.3.2-dev.20220311',
            '5.3.2-dev.20220310',
            '5.3.2-dev.20220309',
            '5.3.2-dev.20220308',
            '5.3.2-dev.20220307',
            '5.3.1-dev.20220306',
            '5.3.1-dev.20220304',
            '5.3.1-dev.20220303',
            '5.3.1-dev.20220302',
            '5.3.1-dev.20220301',
            '5.3.1-dev.20220228',
            '5.3.1-dev.20220227',
            '5.3.1-dev.20220226',
            '5.3.1-dev.20220225',
            '5.3.1-dev.20220224',
            '5.3.1-dev.20220223',
            '5.3.1-dev.20220222',
            '5.3.1-dev.20220221',
            '5.3.1-dev.20220220',
            '5.3.1-dev.20220219',
            '5.3.1-dev.20220218',
            '5.3.1-dev.20220217',
            '5.3.1-dev.20220216',
            '5.3.1-dev.20220215',
            '5.3.1-dev.20220214',
            '5.3.1-dev.20220213',
            '5.3.1-dev.20220212',
            '5.3.1-dev.20220211',
            '5.3.1-dev.20220210',
            '5.3.1-dev.20220209',
            '5.3.1-dev.20220208',
            '5.3.1-dev.20220207',
            '5.3.1-dev.20220206',
            '5.3.1-dev.20220205',
            '5.3.1-dev.20220204',
            '5.3.1-dev.20220203',
            '5.3.1-dev.20220202',
            '5.3.1-dev.20220201',
            '5.3.1-dev.20220131',
            '5.3.1-dev.20220130',
            '5.3.1-dev.20220129',
            '5.3.1-dev.20220128',
            '5.3.1-dev.20220127',
            '5.3.1-dev.20211123',
            '5.3.1-dev.20211122',
            '5.3.1-dev.20211120',
            '5.3.1-dev.20211119',
            '5.3.1-dev.20211118',
            '5.3.1-dev.20211117',
            '5.3.1-dev.20211116',
            '5.3.1-dev.20211115',
            '5.3.1-dev.20211114',
            '5.3.1-dev.20211113',
            '5.3.1-dev.20211112',
            '5.3.1-dev.20211111',
            '5.3.1-dev.20211110',
            '5.3.1-dev.20211109',
            '5.3.1-dev.20211108',
            '5.3.1-dev.20211107',
            '5.3.1-dev.20211106',
            '5.3.1-dev.20211105',
            '5.3.1-dev.20211104',
            '5.3.1-dev.20211103',
            '5.3.1-dev.20211102',
            '5.3.1-dev.20211101',
            '5.3.1-dev.20211031',
            '5.3.1-dev.20211030',
            '5.3.1-dev.20211029',
            '5.3.1-dev.20211028',
            '5.3.1-dev.20211027',
            '5.3.1-dev.20211026',
            '5.3.1-dev.20211025',
            '5.3.1-dev.20211024',
            '5.3.1-dev.20211023',
            '5.3.1-dev.20211022',
            '5.3.1-dev.20211021',
            '5.3.1-dev.20211020',
            '5.3.1-dev.20211019',
            '5.3.1-dev.20211018',
            '5.3.1-dev.20211017',
            '5.3.1-dev.20211016',
            '5.3.1-dev.20211015',
            '5.3.1-dev.20211014',
            '5.3.1-dev.20211013',
            '5.3.1-dev.20211012',
            '5.3.1-dev.20211011',
            '5.3.1-dev.20211010',
            '5.3.1-dev.20211009',
            '5.3.1-dev.20211008',
            '5.3.1-dev.20211007',
            '5.3.1-dev.20211006',
            '5.3.1-dev.20211005',
            '5.3.1-dev.20211004',
            '5.3.1-dev.20211003',
            '5.3.1-dev.20211002',
            '5.3.1-dev.20211001',
            '5.3.1-dev.20210930',
            '5.3.1-dev.20210929',
            '5.3.1-dev.20210928',
            '5.3.1-dev.20210927',
            '5.3.1-dev.20210926',
            '5.3.1-dev.20210925',
            '5.3.1-dev.20210924',
            '5.3.1-dev.20210923',
            '5.3.1-dev.20210922',
            '5.3.1-dev.20210921',
            '5.3.1-dev.20210920',
            '5.3.1-dev.20210919',
            '5.3.1-dev.20210918',
            '5.3.1-dev.20210917',
            '5.3.0-dev.20220315',
            '5.3.0-dev.20220314',
            '5.3.0-dev.20220313',
            '5.3.0-dev.20220312',
            '5.3.0-dev.20220311',
            '5.3.0-dev.20220310',
            '5.3.0-dev.20220309',
            '5.3.0-dev.20220308',
            '5.3.0-dev.20220307',
            '5.3.0-dev.20220306',
            '5.3.0-dev.20220305',
            '5.3.0-dev.20220304',
            '5.3.0-dev.20220303',
            '5.3.0-dev.20220302',
            '5.3.0-dev.20220301',
            '5.3.0-dev.20220228',
            '5.3.0-dev.20220227',
            '5.3.0-dev.20220226',
            '5.3.0-dev.20220225',
            '5.3.0-dev.20220224',
            '5.3.0-dev.20220223',
            '5.3.0-dev.20220222',
            '5.3.0-dev.20220220',
            '5.3.0-dev.20220219',
            '5.3.0-dev.20220218',
            '5.3.0-dev.20220217',
            '5.3.0-dev.20220216',
            '5.3.0-dev.20220215',
            '5.3.0-dev.20220214',
            '5.3.0-dev.20220213',
            '5.3.0-dev.20220212',
            '5.3.0-dev.20220211',
            '5.3.0-dev.20220210',
            '5.3.0-dev.20220209',
            '5.3.0-dev.20220208',
            '5.3.0-dev.20220207',
            '5.3.0-dev.20220206',
            '5.3.0-dev.20220205',
            '5.3.0-dev.20220204',
            '5.3.0-dev.20220203',
            '5.3.0-dev.20220202',
            '5.3.0-dev.20220201',
            '5.3.0-dev.20220131',
            '5.3.0-dev.20220130',
            '5.3.0-dev.20220129',
            '5.3.0-dev.20220128',
            '5.3.0-dev.20220127',
            '5.3.0-dev.20220126',
            '5.3.0-dev.20220125',
            '5.3.0-dev.20220124',
            '5.3.0-dev.20220123',
            '5.3.0-dev.20220122',
            '5.3.0-dev.20220121',
            '5.3.0-dev.20220120',
            '5.3.0-dev.20220119',
            '5.3.0-dev.20220118',
            '5.3.0-dev.20220117',
            '5.3.0-dev.20220116',
            '5.3.0-dev.20220115',
            '5.3.0-dev.20220114',
            '5.3.0-dev.20220113',
            '5.3.0-dev.20220112',
            '5.3.0-dev.20220111',
            '5.3.0-dev.20220110',
            '5.3.0-dev.20220109',
            '5.3.0-dev.20220108',
            '5.3.0-dev.20220107',
            '5.3.0-dev.20220106',
            '5.3.0-dev.20220105',
            '5.3.0-dev.20220104',
            '5.3.0-dev.20220103',
            '5.3.0-dev.20220102',
            '5.3.0-dev.20220101',
            '5.3.0-dev.20211231',
            '5.3.0-dev.20211230',
            '5.3.0-dev.20211229',
            '5.3.0-dev.20211228',
            '5.3.0-dev.20211227',
            '5.3.0-dev.20211226',
            '5.3.0-dev.20211225',
            '5.3.0-dev.20211224',
            '5.3.0-dev.20211223',
            '5.3.0-dev.20211222',
            '5.3.0-dev.20211221',
            '5.3.0-dev.20211220',
            '5.3.0-dev.20211219',
            '5.3.0-dev.20211218',
            '5.3.0-dev.20211217',
            '5.3.0-dev.20211216',
            '5.3.0-dev.20211215',
            '5.3.0-dev.20211214',
            '5.3.0-dev.20211213',
            '5.3.0-dev.20211212',
            '5.3.0-dev.20211211',
            '5.3.0-dev.20211210',
            '5.3.0-dev.20211209',
            '5.3.0-dev.20211208',
            '5.3.0-dev.20211207',
            '5.3.0-dev.20211206',
            '5.3.0-dev.20211205',
            '5.3.0-dev.20211204',
            '5.3.0-dev.20211203',
            '5.3.0-dev.20211202',
            '5.3.0-dev.20211201',
            '5.3.0-dev.20211130',
            '5.3.0-dev.20211129',
            '5.3.0-dev.20211128',
            '5.3.0-dev.20211127',
            '5.3.0-dev.20211126',
            '5.3.0-dev.20211125',
            '5.3.0-dev.20211124',
            '5.3.0-dev.20210916',
            '5.3.0-dev.20210915',
            '5.3.0-dev.20210914',
            '5.3.0-dev.20210913',
            '5.3.0-dev.20210912',
            '5.3.0-dev.20210911',
            '5.3.0-dev.20210910',
            '5.3.0-dev.20210909',
            '5.3.0-dev.20210908',
            '5.3.0-dev.20210907',
            '5.3.0-dev.20210906',
            '5.3.0-dev.20210905',
            '5.3.0-dev.20210904',
            '5.3.0-dev.20210903',
            '5.2.3-dev.20220126',
            '5.2.3-dev.20220125',
            '5.2.3-dev.20220124',
            '5.2.3-dev.20220123',
            '5.2.3-dev.20220122',
            '5.2.3-dev.20220121',
            '5.2.3-dev.20220120',
            '5.2.3-dev.20220119',
            '5.2.3-dev.20220118',
            '5.2.3-dev.20220117',
            '5.2.3-dev.20220116',
            '5.2.3-dev.20220115',
            '5.2.3-dev.20220114',
            '5.2.3-dev.20220113',
            '5.2.3-dev.20220112',
            '5.2.3-dev.20220111',
            '5.2.3-dev.20220110',
            '5.2.3-dev.20220109',
            '5.2.3-dev.20220108',
            '5.2.3-dev.20220107',
            '5.2.3-dev.20220106',
            '5.2.3-dev.20220105',
            '5.2.3-dev.20220104',
            '5.2.3-dev.20220103',
            '5.2.3-dev.20220102',
            '5.2.3-dev.20220101',
            '5.2.3-dev.20211231',
            '5.2.3-dev.20211230',
            '5.2.3-dev.20211229',
            '5.2.3-dev.20211228',
            '5.2.3-dev.20211227',
            '5.2.3-dev.20211226',
            '5.2.3-dev.20211225',
            '5.2.3-dev.20211224',
            '5.2.3-dev.20211223',
            '5.2.3-dev.20211222',
            '5.2.3-dev.20211221',
            '5.2.3-dev.20211220',
            '5.2.3-dev.20211219',
            '5.2.3-dev.20211218',
            '5.2.3-dev.20211217',
            '5.2.3-dev.20211216',
            '5.2.3-dev.20211215',
            '5.2.3-dev.20211214',
            '5.2.3-dev.20211213',
            '5.2.3-dev.20211212',
            '5.2.3-dev.20211211',
            '5.2.3-dev.20211210',
            '5.2.3-dev.20211209',
            '5.2.3-dev.20211208',
            '5.2.3-dev.20211207',
            '5.2.3-dev.20211206',
            '5.2.3-dev.20211205',
            '5.2.3-dev.20211204',
            '5.2.3-dev.20211203',
            '5.2.3-dev.20211202',
            '5.2.3-dev.20211201',
            '5.2.3-dev.20211130',
            '5.2.3-dev.20211129',
            '5.2.3-dev.20211128',
            '5.2.3-dev.20211127',
            '5.2.3-dev.20211126',
            '5.2.3-dev.20211125',
            '5.2.3-dev.20211124',
            '5.2.3-dev.20211123',
            '5.2.3-dev.20211122',
            '5.2.3-dev.20211121',
            '5.2.3-dev.20211120',
            '5.2.3-dev.20211119',
            '5.2.3-dev.20211118',
            '5.2.3-dev.20211117',
            '5.2.3-dev.20211116',
            '5.2.3-dev.20211115',
            '5.2.3-dev.20211114',
            '5.2.3-dev.20211113',
            '5.2.3-dev.20211112',
            '5.2.3-dev.20211111',
            '5.2.3-dev.20211110',
            '5.2.3-dev.20211109',
            '5.2.3-dev.20211108',
            '5.2.3-dev.20211107',
            '5.2.3-dev.20211106',
            '5.2.3-dev.20211105',
            '5.2.3-dev.20211104',
            '5.2.3-dev.20211103',
            '5.2.3-dev.20211102',
            '5.2.3-dev.20211101',
            '5.2.2-dev.20211031',
            '5.2.2-dev.20211030',
            '5.2.2-dev.20211029',
            '5.2.2-dev.20211028',
            '5.2.2-dev.20211027',
            '5.2.2-dev.20211026',
            '5.2.2-dev.20211025',
            '5.2.2-dev.20211024',
            '5.2.2-dev.20211023',
            '5.2.2-dev.20211022',
            '5.2.2-dev.20211021',
            '5.2.2-dev.20211020',
            '5.2.2-dev.20211019',
            '5.2.2-dev.20211018',
            '5.2.2-dev.20211017',
            '5.2.2-dev.20211016',
            '5.2.2-dev.20211015',
            '5.2.2-dev.20211014',
            '5.2.2-dev.20211013',
            '5.2.2-dev.20211012',
            '5.2.2-dev.20211011',
            '5.2.2-dev.20211010',
            '5.2.2-dev.20211009',
            '5.2.2-dev.20211008',
            '5.2.2-dev.20211007',
            '5.2.2-dev.20211006',
            '5.2.2-dev.20211005',
            '5.2.2-dev.20211004',
            '5.2.2-dev.20211003',
            '5.2.2-dev.20211002',
            '5.2.2-dev.20211001',
            '5.2.2-dev.20210930',
            '5.2.2-dev.20210929',
            '5.2.2-dev.20210928',
            '5.2.2-dev.20210927',
            '5.2.2-dev.20210926',
            '5.2.2-dev.20210925',
            '5.2.2-dev.20210924',
            '5.2.2-dev.20210923',
            '5.2.2-dev.20210922',
            '5.2.1-dev.20210921',
            '5.2.1-dev.20210920',
            '5.2.1-dev.20210919',
            '5.2.1-dev.20210918',
            '5.2.1-dev.20210917',
            '5.2.1-dev.20210916',
            '5.2.1-dev.20210915',
            '5.2.1-dev.20210914',
            '5.2.1-dev.20210913',
            '5.2.1-dev.20210912',
            '5.2.1-dev.20210911',
            '5.2.1-dev.20210910',
            '5.2.1-dev.20210909',
            '5.2.1-dev.20210908',
            '5.2.1-dev.20210907',
            '5.2.1-dev.20210906',
            '5.2.1-dev.20210905',
            '5.2.1-dev.20210904',
            '5.2.1-dev.20210903',
            '5.2.1-dev.20210902',
            '5.2.1-dev.20210901',
            '5.1.3-dev.20210831',
            '5.1.3-dev.20210830',
            '5.1.3-dev.20210829',
            '5.1.3-dev.20210828',
            '5.1.3-dev.20210827',
            '5.1.3-dev.20210826',
            '5.1.3-dev.20210825',
            '5.1.3-dev.20210824',
            '5.1.3-dev.20210823',
            '5.1.3-dev.20210822',
            '5.1.3-dev.20210821',
            '5.1.3-dev.20210820',
            '5.1.3-dev.20210819',
            '5.1.3-dev.20210818',
            '5.1.3-dev.20210817',
            '5.1.3-dev.20210816',
            '5.1.3-dev.20210815',
            '5.1.3-dev.20210814',
            '5.1.3-dev.20210813',
            '5.1.3-dev.20210812',
            '5.1.3-dev.20210811',
            '5.1.3-dev.20210810',
            '5.1.3-dev.20210809',
            '5.1.3-dev.20210808',
            '5.1.3-dev.20210807',
            '5.1.3-dev.20210806',
            '5.1.3-dev.20210805',
            '5.1.3-dev.20210804',
            '5.1.3-dev.20210803',
            '5.1.3-dev.20210802',
            '5.1.3-dev.20210801',
            '5.1.3-dev.20210731',
            '5.1.3-dev.20210730',
            '5.1.3-dev.20210729',
            '5.1.3-dev.20210728',
            '5.1.3-dev.20210727',
            '5.1.3-dev.20210726',
            '5.1.3-dev.20210725',
            '5.1.3-dev.20210724',
            '5.1.3-dev.20210723',
            '5.1.3-dev.20210722',
            '5.1.3-dev.20210721',
            '5.1.3-dev.20210720',
            '5.1.3-dev.20210719',
            '5.1.3-dev.20210718',
            '5.1.3-dev.20210717',
            '5.1.3-dev.20210716',
            '5.1.3-dev.20210715',
            '5.1.3-dev.20210714',
            '5.1.3-dev.20210713',
            '5.1.3-dev.20210712',
            '5.1.3-dev.20210711',
            '5.1.3-dev.20210710',
            '5.1.3-dev.20210709',
            '5.1.3-dev.20210708',
            '5.1.3-dev.20210707',
            '5.1.3-dev.20210706',
            '5.1.3-dev.20210705',
            '5.1.3-dev.20210704',
            '5.1.3-dev.20210703',
            '5.1.3-dev.20210702',
            '5.1.3-dev.20210701',
            '5.1.3-dev.20210630',
            '5.1.3-dev.20210629',
            '5.1.3-dev.20210628',
            '5.1.3-dev.20210627',
            '5.1.3-dev.20210626',
            '5.1.3-dev.20210625',
            '5.1.3-dev.20210624',
            '5.1.3-dev.20210623',
            '5.1.3-dev.20210622',
            '5.1.3-dev.20210621',
            '5.1.3-dev.20210620',
            '5.1.3-dev.20210619',
            '5.1.3-dev.20210618',
            '5.1.3-dev.20210617',
            '5.1.3-dev.20210616',
            '5.1.3-dev.20210615',
            '5.1.3-dev.20210614',
            '5.1.3-dev.20210613',
            '5.1.3-dev.20210612',
            '5.1.3-dev.20210611',
            '5.1.2-dev.20210610',
            '5.1.2-dev.20210609',
            '5.1.2-dev.20210608',
            '5.1.2-dev.20210607',
            '5.1.2-dev.20210606',
            '5.1.2-dev.20210605',
            '5.1.2-dev.20210604',
            '5.1.2-dev.20210603',
            '5.1.2-dev.20210602',
            '5.1.2-dev.20210601',
            '5.1.2-dev.20210531',
            '5.1.2-dev.20210530',
            '5.1.2-dev.20210529',
            '5.1.2-dev.20210528',
            '5.1.2-dev.20210527',
            '5.1.2-dev.20210526',
            '5.1.2-dev.20210525',
            '5.1.2-dev.20210524',
            '5.1.2-dev.20210523',
            '5.1.2-dev.20210522',
            '5.1.2-dev.20210521',
            '5.1.2-dev.20210520',
            '5.1.2-dev.20210519',
            '5.1.2-dev.20210518',
            '5.1.2-dev.20210517',
            '5.1.2-dev.20210515',
            '5.1.2-dev.20210514',
            '5.1.2-dev.20210513',
            '5.1.2-dev.20210512',
            '5.1.2-dev.20210511',
            '5.1.2-dev.20210510',
            '5.1.2-dev.20210509',
            '5.1.2-dev.20210508',
            '5.1.2-dev.20210507',
            '5.1.2-dev.20210506',
            '5.1.2-dev.20210505',
            '5.1.2-dev.20210504',
            '5.1.2-dev.20210503',
            '5.1.2-dev.20210502',
            '5.1.2-dev.20210501',
            '5.1.2-dev.20210430',
            '5.1.2-dev.20210429',
            '5.1.2-dev.20210428'
          ]
        };

        // TODO lazy load when needed
        //$.getJSON(getVersionListAPI('echarts-nightly')).done((data) => {
        // isZH && (data = handleData(data));

        if (isDebug) {
          console.log('echarts-nightly version data', data);
        }

        const {
          tags: { latest, next },
          versions
        } = data;
        const nextIdx = versions.indexOf(next);
        const latestIdx = versions.indexOf(latest);
        this.nightlyVersions = versions
          .slice(nextIdx, Math.min(nextIdx + 10, latestIdx))
          .concat(versions.slice(latestIdx, latestIdx + 10));

        hasPRVersion && this.nightlyVersions.unshift(prVersion);
        //});
      })();

      if (isDebug) {
        console.log(
          'echarts versions',
          this.allEChartsVersions,
          'nightly versions',
          this.nightlyVersions
        );
      }
    },
    getAssets() {
      return {
        scripts: this.scripts,
        css: this.css
      };
    }
  }
};
</script>

<style lang="scss">
@import '../style/color.scss';

@mixin flex-center {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

#chart-panel {
  position: absolute;
  // top: $control-panel-height;
  top: 42px;
  right: 15px;
  bottom: 50px;
  left: 15px;
  box-sizing: border-box;
  box-shadow: rgba(0, 0, 0, 0.1) 0px 0px 20px;
  border-radius: 5px;
  background: #fff;
  overflow: hidden;
  padding: 10px;
}

.render-config-container {
  @include flex-center;

  user-select: none;

  > * {
    margin-left: 10px;
    margin-right: 10px;
    flex-shrink: 0;
  }

  .cfg-renderer {
    @include flex-center;
  }

  .tool-label {
    font-weight: bold;
    margin-right: 5px;
    margin-bottom: 0;
  }

  .el-radio-group {
    label {
      margin-bottom: 0;
    }
  }
}

#tool-panel {
  position: absolute;
  top: 5px;
  right: 15px;
  left: 15px;
  @include flex-center;

  white-space: nowrap;
  overflow-x: auto;
  overflow-y: hidden;
  scrollbar-width: thin;

  user-select: none;

  * {
    font-size: 12px;
  }

  .render-config-trigger {
    cursor: pointer;
    font-weight: 500;
    margin-left: 10px;
  }
  .version-select {
    width: 80px;
    margin-left: 10px;

    &.is-nightly,
    &.is-pr {
      width: 160px;
    }
  }
  .random,
  .use-nightly {
    margin-left: 10px;
  }

  label {
    margin-bottom: 0;
  }

  .dark-mode {
    margin-right: 5px;
  }

  .edit {
    margin-left: 5px;
    cursor: pointer;
  }
}

.full {
  #chart-panel {
    top: 40px;
    right: 5px;
    bottom: 5px;
    left: 5px;
    box-shadow: rgba(10, 9, 9, 0.1) 0px 0px 5px;
  }
  #tool-panel {
    right: 5px;
    left: 5px;
  }
}

#preview-status {
  position: absolute;
  bottom: 10px;
  left: 0;
  right: 0;
  padding: 0 15px;
  font-size: 0.9rem;
  @include flex-center;
  white-space: nowrap;
  overflow-x: auto;
  overflow-y: hidden;
  scrollbar-width: thin;

  .left-buttons {
    flex-shrink: 0;
  }

  #run-log {
    @include flex-center;
    font-size: 12px;
    margin-left: 10px;
    text-align: right;

    .run-log-time {
      color: $clr-text;
      margin-right: 10px;
      white-space: nowrap;
    }

    .run-log-type-info {
      color: $clr-text;
    }

    .run-log-type-warn {
      color: $clr-warn;
    }

    .run-log-type-error {
      color: $clr-error;
    }
  }
}

.el-message {
  &.toast-declaration {
    min-width: auto;
    z-index: 9999999 !important;

    .el-message__icon {
      font-size: 20px;
    }

    .el-message__content {
      padding-right: 20px;
      line-height: 1.25;
    }
  }
}
</style>
