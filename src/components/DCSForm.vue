<template>
  <div class="container">
    <div class="row">
      <main role="main" property="mainContentOfPage" class="col-md-9 col-md-push-3">
        <h1>{{ currentRouteTitle }}</h1>

        <p>{{ introDatasetText.gridded.use }}</p>
        <p>{{ introDatasetText.gridded.instructions }}</p>

        <data-access-doc-link></data-access-doc-link>

        <details :open="toggleDetailsState">
          <summary @click="toggleDetails"
            v-translate>Dataset description, technical information and metadata</summary>
          <p v-translate>The Statistically downscaled climate scenarios dataset provides projected changes in temperature and precipitation, with respect to the reference period of 1986-2005, for three emission scenarios at a 10km resolution. Downscaled data are based on global climate model projections from the Coupled Model Intercomparison Project Phase 5 (CMIP5). The median projected changes across the downscaled multi-model ensemble are shown.</p>

          <p v-html="techDocHtml"></p>

          <open-portal-links
            :open-portal-list-html="openPortalListHtml"
            :open-portal-variables="datasetTitles[$route.name].openPortal.variables"></open-portal-links>
        </details>

        <info-contact-support></info-contact-support>

        <bbox-map
          v-model="ows_bbox"
          :allow-click-point="true"
          @change="splitBBOXString"></bbox-map>

        <var-select
          v-model="wcs_id_variable"
          :select-options="variableOptions"></var-select>

        <option-radio
          v-model="scenarioType"
          :radio-inline="true"
          :radio-options="scenarioTypeOptions"></option-radio>

        <scenario-select
          v-show="scenarioType === 'RCP'"
          v-model="wcs_id_scenario"
          :select-options="scenarioOptions"></scenario-select>

        <var-select
          v-model="wcs_id_timePeriod"
          :label="$gettext('Time interval / Time of year')"
          :info-text="[infoDailyData]"
          :select-options="timePeriodOptions"></var-select>

        <var-select
          v-model="valueType"
          :label="$gettext('Value type')"
          :select-options="valueTypeOptions"></var-select>

        <var-select
          v-model="percentile"
          :label="$gettext('Ensemble percentile')"
          :info-text="[infoModelOutput, infoPercentile]"
          :select-options="percentileOptions"></var-select>

        <fieldset v-show="!pointDownloadOn">
          <legend v-translate>Date range</legend>

          <option-radio
            v-model="rangeType"
            :label="$gettext('Time range type')"
            :radio-inline="true"
            :radio-options="rangeTypeOptions"></option-radio>

          <div v-show="scenarioType === 'HISTO' && rangeType !=='year20'">
            <date-select
              v-model="dateHistStart"
              :label="$gettext('Historical start date')"
              :minimum-view="dateConfigs.minimumView"
              :format="dateConfigs.format"
              :required="timePeriodIsMonthly"
              :min-date="dateConfigs.dateMin"
              :max-date="dateConfigs.dateMax"
              :custom-error-msg="dateRangeErrorMessage"
              :placeholder="dateConfigs.placeholder"></date-select>

            <date-select
              v-model="dateHistEnd"
              :label="$gettext('Historical end date')"
              :minimum-view="dateConfigs.minimumView"
              :format="dateConfigs.format"
              :required="timePeriodIsMonthly"
              :min-date="dateConfigs.dateMin"
              :max-date="dateConfigs.dateMax"
              :custom-error-msg="dateRangeErrorMessage"
              :placeholder="dateConfigs.placeholder"></date-select>

            <button
              class="btn btn-default"
              type="button"
              @click="clearDates"
              v-translate>Clear dates</button>
          </div>
          <div v-show="scenarioType === 'RCP' && rangeType !=='year20'">
            <date-select
              v-model="dateRcpStart"
              :label="$gettext('Start date')"
              :minimum-view="dateConfigs.minimumView"
              :format="dateConfigs.format"
              :required="timePeriodIsMonthly"
              :min-date="dateConfigs.dateMin"
              :max-date="dateConfigs.dateMax"
              :custom-error-msg="dateRangeErrorMessage"
              :placeholder="dateConfigs.placeholder"></date-select>

            <date-select
              v-model="dateRcpEnd"
              :label="$gettext('End date')"
              :minimum-view="dateConfigs.minimumView"
              :format="dateConfigs.format"
              :required="timePeriodIsMonthly"
              :min-date="dateConfigs.dateMin"
              :max-date="dateConfigs.dateMax"
              :custom-error-msg="dateRangeErrorMessage"
              :placeholder="dateConfigs.placeholder"></date-select>

            <button
              class="btn btn-default"
              type="button"
              @click="clearDates"
              v-translate>Clear dates</button>
          </div>

          <var-select
            v-show="rangeType === 'year20' && valueType === 'ANO'"
            v-model="avg20Year"
            :label="$gettext('20-Year average range')"
            :select-options="avg20YearOptions"></var-select>
        </fieldset>

        <format-select-raster
          class="mrgn-tp-md"
          v-show="!pointDownloadOn"
          v-model="wcs_format"
          :info-text="[infoSupportDeskGridPoint]"></format-select-raster>

        <format-select-vector
          class="mrgn-tp-md"
          v-show="pointDownloadOn"
          v-model="wps_format"></format-select-vector>

        <details
          :open="toggleDetailsAdvState"
          v-show="!pointDownloadOn">
          <summary @click="toggleDetailsAdv"
            v-translate>Advanced options</summary>
          <var-select
            v-model="ows_crs"
            :label="crsLabel"
            :select-options="crsOptions"></var-select>
        </details>

        <url-box
          v-show="!pointDownloadOn"
          :layer-options="selectedCoverageIdOption"
          :ows-url-formatter="wcs_download_url"
          :layer-format="wcs_format"
          :wcs-common-url="wcsCommonUrl"
          :wcs-band-chunks="chunkedBandsParam"
          :wcs-num-bands="dateRangeNumBands"
          :band-range-format="bandRangeFormat"
          :has-errors="hasErrors"
          :url-box-title="$gettext('Data download link')">
        </url-box>

        <point-download-box
          v-show="pointDownloadOn"
          :title="titlePointDownload"
          :hasErrors="invalidPointDownloadInputs"
          :point-inputs="pointInputs" />
      </main>
      <dataset-menu></dataset-menu>
    </div>
  </div>
</template>

<script>
import DatasetMenu from './DatasetMenu'
import BBOXMap from './BBOXMap'
import FormatSelectRaster from './FormatSelectRaster'
import FormatSelectVector from './FormatSelectVector'
import VarSelect from './VarSelect'
import ScenarioSelect from './ScenarioSelect'
import DateSelect from './DateSelect'
import OptionRadio from './OptionRadio'
import URLBox from './URLBox'
import InfoContactSupport from './InfoContactSupport'
import OpenPortalLinks from './OpenPortalLinks'
import DataAccessDocLink from './DataAccessDocLink'
import PointDownloadBox from './PointDownloadBox'
import { wcs } from './mixins/wcs'
import { ows } from './mixins/ows'
import { datasets } from './mixins/datasets'
import { DCSCMIP5 } from './mixins/dcs-cmip5'
import { wps } from './mixins/wps'

export default {
  name: 'DCSForm',
  mixins: [wcs, ows, datasets, DCSCMIP5, wps],
  components: {
    DatasetMenu,
    'bbox-map': BBOXMap,
    FormatSelectRaster,
    FormatSelectVector,
    VarSelect,
    ScenarioSelect,
    DateSelect,
    OptionRadio,
    'url-box': URLBox,
    InfoContactSupport,
    OpenPortalLinks,
    DataAccessDocLink,
    PointDownloadBox
  },
  data () {
    return {
      wcs_id_dataset: 'DCS',
      wcs_id_variable: 'TM',
      MAX_BANDS: 100
    }
  },
  watch: {
    rangeType: function (newVal, oldVal) {
      // Auto select 50th percentile for 20 year averages
      if (newVal === 'year20') {
        this.percentile = 'PCTL50'
      }
    }
  },
  computed: {
    percentileOptions: function () {
      if (this.rangeType === 'year20') {
        return {
          PCTL50: this.$gettext('50th percentile')
        }
      } else {
        return {
          PCTL5: this.$gettext('5th percentile'),
          PCTL25: this.$gettext('25th percentile'),
          PCTL50: this.$gettext('50th percentile'),
          PCTL75: this.$gettext('75th percentile'),
          PCTL95: this.$gettext('95th percentile')
        }
      }
    },
    variableOptions: function () {
      return {
        TM: this.$gettext('Mean temperature'),
        TN: this.$gettext('Minimum temperature'),
        TX: this.$gettext('Maximum temperature'),
        PR: this.$gettext('Total precipitation')
      }
    },
    avg20YearOptions: function () {
      return {
        // '1986-2005': '1986-2005',
        '2021-2040': '2021-2040',
        '2041-2060': '2041-2060',
        '2061-2080': '2061-2080',
        '2081-2100': '2081-2100'
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
</style>
