<template>
    <v-container>
        <Breadcrumbs :items="breadcrumbItems"></Breadcrumbs>

        <v-container class="d-flex justify-space-between align-center mb-2 pt-0">
            <v-tabs v-model="tab"  class="background">
                <v-tab class="background">{{ $t('app.nav.overview') }}</v-tab>
                <v-tab class="background" :disabled="!hasBuilds">{{ $t('app.nav.builds') }}</v-tab>
                <v-tab class="background">{{ $t('app.nav.metrics') }}</v-tab>
                <v-tab class="background" :disabled="!authStore.hasPermission('logs:ok')">{{ $t('app.nav.logs') }}</v-tab>
                <v-tab class="background">{{ $t('app.nav.events') }}</v-tab>
                <v-tab class="background" :disabled="!authStore.hasPermission('security:read') && !authStore.hasPermission('security:write')">{{ $t('app.nav.vulnerabilities') }}</v-tab>
                <v-spacer  class="background"></v-spacer>
            </v-tabs>


            <v-menu offset-y>
            <template v-slot:activator="{ props }">
                <v-btn
                color="primary"
                dark
                v-bind="props"
                >
                    <v-icon color="white">mdi-menu-open</v-icon>
                    {{ $t('app.actions.name') }}
                </v-btn>
            </template>
            <v-list>
                <v-list-item
                    @click="ActionEditApp"
                    prepend-icon="mdi-pencil"
                    :disabled="!authStore.hasPermission('app:write')"
                    :title="$t('app.actions.edit')">
                </v-list-item>
                <v-list-item
                    @click="ActionOpenApp"
                    prepend-icon="mdi-open-in-new"
                    :title="$t('app.actions.openApp')">
                </v-list-item>
                <v-list-item 
                    @click="restartApp"
                    prepend-icon="mdi-reload-alert"
                    :disabled="!authStore.hasPermission('reboot:ok')"
                    :title="$t('app.actions.restart')">
                </v-list-item>
                <v-list-item 
                    :disabled="appData.spec.deploymentstrategy != 'docker'"
                    @click="ActionStartDownload"
                    prepend-icon="mdi-download"
                    :title="$t('app.actions.downloadTemplate')">
                </v-list-item>
                <v-list-item 
                    @click="openConsole"
                    prepend-icon="mdi-console"
                    :disabled="!kubero.consoleEnabled || !authStore.hasPermission('console:ok')"
                    :title="$t('app.actions.openConsole')">
                </v-list-item>
                <v-divider class="my-3"></v-divider>
                <v-list-item 
                    @click="deleteApp"
                    prepend-icon="mdi-delete"
                    :disabled="!authStore.hasPermission('app:write')"
                    :title="$t('app.actions.delete')">
                </v-list-item>
            </v-list>
            </v-menu>
        </v-container>
        
        <v-window v-model="tab">
            <v-window-item transition="false" reverse-transition="false" class="background">
                <Overview :pipeline="pipeline" :phase="phase" :app="app" :appData="appData" :pipelineData="pipelineData"/>
            </v-window-item>
            <v-window-item transition="false" reverse-transition="false" class="background">
                <Builds :pipeline="pipeline" :phase="phase" :app="app" :appData="appData" :pipelineData="pipelineData"/>
            </v-window-item>
            <v-window-item transition="false" reverse-transition="false" class="background">
                <Metrics :pipeline="pipeline" :phase="phase" :app="app" :host="appData.spec.ingress.hosts[0].host"/>
            </v-window-item>
            <v-window-item transition="false" reverse-transition="false" class="background">
                <LogsTab :pipeline="pipeline" :phase="phase" :app="app" :deploymentstrategy="appData.spec.deploymentstrategy" :buildstrategy="appData.spec.buildstrategy"/>
            </v-window-item>
            <v-window-item transition="false" reverse-transition="false" class="background">
                <Events :pipeline="pipeline" :phase="phase" :app="app"/>
            </v-window-item>
            <v-window-item transition="false" reverse-transition="false" class="background">
                <Vulnerabilities :pipeline="pipeline" :phase="phase" :app="app"/>
            </v-window-item>
        </v-window>
    </v-container>
</template>

<script lang="ts">
import axios from "axios";
import { defineComponent } from 'vue'
import Breadcrumbs from "../breadcrumbs.vue";
import Overview from "./overview.vue";
import Events from "./events.vue";
import LogsTab from "./logstab.vue";
import Metrics from "./metrics.vue";
import Builds from "./builds.vue";
import Vulnerabilities from "./vulnerabilities.vue";
import Swal from 'sweetalert2';
import { useKuberoStore } from '../../stores/kubero'
import { mapState } from 'pinia'
import { useAuthStore } from '../../stores/auth'
const authStore = useAuthStore();


export default defineComponent({
    name: 'AppDetail',
    setup() {
        return {
            authStore,
        }
    },
    data () {
        return {
            loadingState: false,
            tab: null,
            breadcrumbItems: [
                {
                    title: 'Dashboard.Pipelines',
                    disabled: false,
                    to: { name: 'Pipelines', params: {}}
                },
                {
                    title: 'Pipeline.'+this.pipeline,
                    disabled: false,
                    to: { name: 'Pipeline Apps', params: { pipeline: this.pipeline }}
                },
                {
                    title: 'Phase.'+this.phase,
                    disabled: true,
                    href: `/pipeline/${this.pipeline}/${this.phase}/${this.app}/detail`,
                },
                {
                    title: 'App.'+this.app,
                    disabled: true,
                    href: `/pipeline/${this.pipeline}/${this.phase}/${this.app}/detail`,
                }
            ],
            pipelineData: {},
            appData: {
                spec: {
                    deploymentstrategy: "git",
                    buildstrategy: "plain",
                    ingress: {
                        hosts: [{
                            host: '',
                        }]
                    },
                }
            }
        }
    },
    computed: {
      ...mapState(useKuberoStore, ['kubero']),
      hasBuilds() {
        // disable the builds tab if the buildstrategy is plain or external
        return this.appData.spec.deploymentstrategy == 'git' && this.appData.spec.buildstrategy != 'plain' && this.appData.spec.buildstrategy != 'external';
      }
    },
    mounted() {
        this.loadPipeline();
        this.loadApp();
    },
    methods: {
        loadPipeline() {
            axios.get('/api/pipelines/'+this.pipeline).then(response => {
                this.pipelineData = response.data;
            });
        },
        loadApp() {
            axios.get('/api/apps/'+this.pipeline+'/'+this.phase+'/'+this.app).then(response => {
                this.appData = response.data;
                //console.log(this.appData);
            });
        },
        ActionOpenApp() {
            window.open(`https://${this.appData.spec.ingress.hosts[0].host}`, '_blank');
        },
        ActionEditApp() {
            this.$router.push(`/pipeline/${this.pipeline}/${this.phase}/apps/${this.app}`);
        },
        ActionStartDownload() {
            axios.get('/api/apps/'+this.pipeline+'/'+this.phase+'/'+this.app+'/download').then(response => {
                //console.log(response.data);
                const url = window.URL.createObjectURL(new Blob([response.data]));
                const link = document.createElement('a');
                link.href = url;
                link.setAttribute('download', this.app+'.yaml');
                document.body.appendChild(link);
                link.click();
            });
        },
        deleteApp() {
            Swal.fire({
                title: "Delete App ”" + this.app + "” ?",
                text: "Do you want to delete this App? This action cannot be undone. It will delete all the data associated with this app.",
                icon: "question",
                showCancelButton: true,
                confirmButtonText: "Delete",
                cancelButtonText: "Cancel",
                confirmButtonColor: "rgb(var(--v-theme-kubero))",
                background: "rgb(var(--v-theme-cardBackground))",
                /*background: "rgb(var(--v-theme-on-surface-variant))",*/
                color: "rgba(var(--v-theme-on-background),var(--v-high-emphasis-opacity));",
            })
            .then((result) => {
                if (result.isConfirmed) {
                    axios.delete(`/api/apps/${this.pipeline}/${this.phase}/${this.app}`)
                    .then(response => {
                        // sleep 1 second
                        setTimeout(() => {
                            this.$router.push(`/pipeline/${this.pipeline}/apps`);
                        }, 1000);
                        //console.log("deleteApp", response);
                    })
                    .catch(error => {
                        console.log(error);
                    });
                return;
                }
            });
        },
        async restartApp() {
            axios.get(`/api/apps/${this.pipeline}/${this.phase}/${this.app}/restart`)
            .then(response => {
                //console.log(response);
                this.loadingState = true;
            })
            .catch(error => {
                console.log(error);
            });

            // TODO - this is a hack to wait for the restart to complete. It is not so easy to get the status of the restart.
            await new Promise(r => setTimeout(r, 15000));
            this.loadingState = false;
        },
        openConsole() {
            window.open(`/popup/console/${this.pipeline}/${this.phase}/${this.app}`, '_blank', 'popup=yes,location=no,height=720,width=900,scrollbars=yes,status=no');
        },
    },

    components: {
        Breadcrumbs,
        Events,
        LogsTab,
        Vulnerabilities,
        Overview,
        Metrics,
        Builds
    },
    props: {
      pipeline: {
        type: String,
        default: "MISSING"
      },
      phase: {
        type: String,
        default: "MISSING"
      },
      app: {
        type: String,
        default: "new"
      }
    },
})
</script>

<style scoped>
.v-list-item__icon:first-child {
    margin-right: 0px;
    margin: 10px
}
.v-list-item {
    min-height: 32px;
}
</style>

<style>
.v-window__container {
    transition: none !important;
}
canvas {
        transition: none !important;
    }
</style>