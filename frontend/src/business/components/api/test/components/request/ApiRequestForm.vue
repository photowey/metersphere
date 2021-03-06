<template>
  <div class="request-form">
    <component @runDebug="runDebug" :is="component" :is-read-only="isReadOnly" :request="request" :scenario="scenario"/>
    <ms-request-result-tail v-loading="debugReportLoading" v-if="isCompleted" :request="request.debugRequestResult ? request.debugRequestResult : {responseResult: {}, subRequestResults: []}"
                            :scenario-name="request.debugScenario ? request.debugScenario.name : ''" ref="msDebugResult"/>
  </div>
</template>

<script>
import {Request, RequestFactory, Scenario} from "../../model/ScenarioModel";
import MsApiHttpRequestForm from "./ApiHttpRequestForm";
import MsApiDubboRequestForm from "./ApiDubboRequestForm";
import MsScenarioResults from "../../../report/components/ScenarioResults";
import MsRequestResultTail from "../../../report/components/RequestResultTail";

export default {
  name: "MsApiRequestForm",
  components: {MsRequestResultTail, MsScenarioResults, MsApiDubboRequestForm, MsApiHttpRequestForm},
  props: {
    scenario: Scenario,
    request: Request,
    isReadOnly: {
      type: Boolean,
      default: false
    },
    debugReportId: String
  },
    data() {
      return {
        reportId: "",
        content: {scenarios:[]},
        debugReportLoading: false,
        showDebugReport: false
      }
    },
    computed: {
      component({request: {type}}) {
        let name;
        switch (type) {
          case RequestFactory.TYPES.DUBBO:
            name = "MsApiDubboRequestForm";
            break;
          default:
            name = "MsApiHttpRequestForm";
        }
        return name;
      },
      isCompleted() {
        return !!this.request.debugReport;
      }
    },
    watch: {
      debugReportId() {
        this.getReport();
      }
    },
    methods: {
      getReport() {
        if (this.debugReportId) {
          this.debugReportLoading = true;
          this.showDebugReport = true;
          this.request.debugReport = {};
          let url = "/api/report/get/" + this.debugReportId;
          this.$get(url, response => {
            let report = response.data || {};
            let res = {};
            if (response.data) {
              try {
                res = JSON.parse(report.content);
              } catch (e) {
                throw e;
              }
              if (res) {
                this.debugReportLoading = false;
                this.request.debugReport = res;
                if (res.scenarios && res.scenarios.length > 0) {
                  this.request.debugScenario = res.scenarios[0];
                  this.request.debugRequestResult = this.request.debugScenario.requestResults[0];
                  this.deleteReport(this.debugReportId);
                } else {
                  this.request.debugScenario = new Scenario();
                  this.request.debugRequestResult = {responseResult: {}, subRequestResults: []};
                }
                this.$refs.msDebugResult.reload();
              } else {
                setTimeout(this.getReport, 2000)
              }
            } else {
              this.debugReportLoading = false;
            }
          });
        }
      },
      deleteReport(reportId) {
        this.$post('/api/report/delete', {id: reportId});
      },
      runDebug() {
        this.$emit('runDebug', this.request);
      }
    }
  }
</script>

<style scoped>

  .scenario-results {
    margin-top: 20px;
  }

  .request-form >>> .debug-button {
    margin-left: auto;
    display: block;
    margin-right: 10px;
  }

</style>
