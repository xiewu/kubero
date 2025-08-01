<template>
    <div>
    <v-container>
        <Breadcrumbs :items="breadcrumbItems"></Breadcrumbs>
        <v-spacer />
        <v-col class="text-right pt-0 pr-0">
            <v-btn
            elevation="2"
            color="primary"
            :disabled="!authStore.hasPermission('pipeline:write')"
            :to="{ name: 'Pipeline Form', params: { pipeline: pipeline }}"
            >{{ $t('pipeline.buttons.edit') }}</v-btn>
        </v-col>
    </v-container>
    <v-container :fluid="true">
        <!--<h1>{{ pipeline }}</h1>-->
        <v-layout>
                <v-row>
                    <v-col v-for="phase in activePhases" :key="phase.name">
                        <h4 class="text-uppercase ml-1">{{ $t(`pipeline.phases.${phase.name}`) }}</h4>
                        <v-chip
                            class="ma-1"
                            label
                            size="small"
                            >
                            <v-icon icon="mdi-kubernetes" start></v-icon>
                            {{phase.context}}
                        </v-chip>


                        <Appcard v-for="app in phase.apps" :key="app.name"
                            :pipeline="pipeline"
                            :phase="phase.name"
                            :app="app" />

                        <span v-if="phase.name == 'review'">
                            <PRcard v-for="pr in pullrequests" :key="pr.number"
                                :pipeline="pipeline"
                                :pullrequest="pr" />
                        </span>
                        <div>
                            <v-btn
                            elevation="2"
                            icon="mdi-plus"
                            :to="{ name: 'App Form', params: { phase: phase.name, pipeline: pipeline, app: 'new'}}"
                            class="mt-5 navBG"
                            color="secondary"
                            size="small"
                            style="margin-bottom: 5px;"
                            >
                            </v-btn>
                        </div>
                    </v-col>

                </v-row>
        </v-layout>
    </v-container>
    </div>
</template>

<script lang="ts">
import axios from "axios";
import Appcard from "./appcard.vue";
import PRcard from "./prcard.vue";
import Breadcrumbs from "../breadcrumbs.vue";
import { useKuberoStore } from '../../stores/kubero'
import { useAuthStore } from '../../stores/auth'
const authStore = useAuthStore();

import { reactive, ref, defineComponent } from 'vue'

type Phase = {
    name: string,
    context: string,
    enabled: boolean,
    apps: Array<App>,
}

type App = {
    name: string,
    enabled: boolean,
    autodeploy: boolean,
}

type Git = {
    ssh_url: string,
    provider: string,
}

type Pullrequest = {
    number: number,
    branch: string,
    title: string,
    ssh_url: string,
    created_at: string,
    updated_at: string,
}
const socket = useKuberoStore().kubero.socket as any;

const phases = ref([] as Array<Phase>);
const reviewapps = ref(false);
const git = reactive({} as Git);
const pullrequests = ref([] as Array<Pullrequest>);
const pipeline = ref("");


async function loadPipeline() {
    axios.get('/api/pipelines/' + pipeline.value + '/apps')
    .then(response => {
        //console.log("loadPipeline Phases", response.data.phases);
        phases.value = response.data.phases;
        reviewapps.value = response.data.reviewapps;
        git.ssh_url = response.data.git.repository.ssh_url;
        git.provider = response.data.git.provider;
        if (reviewapps.value) {
            loadPullrequests();
        }
        return response.data.phases;
    })
    .catch(error => {
        console.log(error);
    });
}

async function loadPullrequests() {
    if (git.provider == "") {
        return;
    }

    const gitrepoB64 = btoa(git.ssh_url);

    axios.get('/api/repo/'+git.provider+'/' + gitrepoB64 + '/pullrequests')
    .then(response => {

        pullrequests.value = [] as Array<Pullrequest>;

        // iterate over response.data and search in phases[0].name for a match
        // if not found, add the pullrequest to the phase.apps array
        response.data.forEach((pr: Pullrequest) => {
            let found = false;
            phases.value[0].apps.forEach((app: App) => {
                console.log(app.name, pr.branch);
                if (app.name == pr.branch) {
                    found = true;
                }
            });
            if (!found) {
                pullrequests.value.push(pr);
            }
        });

        //pullrequests.value = response.data;
        return response.data;
    })
    .catch(error => {
        console.log(error);
    });
}

socket.on('deleteApp', async (instances: Array<App>) => {
    //console.log("deleteApp", instances);
    // sleep 1 second to give the app time to start
    await new Promise(r => setTimeout(r, 1000));
    loadPipeline();
});

socket.on('updatedApps', async (instances: Array<App>) => {
    console.log("updatedApps", instances);
    // sleep 1 second to give the app time to start
    await new Promise(r => setTimeout(r, 1000));
    loadPipeline();
});


export default defineComponent({
    setup(props) {
        pipeline.value = props.pipeline;
        return {
            phases,
            reviewapps,
            git,
            pullrequests,
            pipeline,
            authStore,
        }
    },
    mounted() {
        loadPipeline();
    },
    unmounted() {
        socket.off('deleteApp');
        socket.off('updatedApps');

        // empty the phases array
        phases.value = [] as Array<Phase>;
        pullrequests.value = [] as Array<Pullrequest>;
    },
    props: {
      pipeline: {
        type: String,
        default: "MISSING"
      },
    },
    data () {return {
        breadcrumbItems: [
            {
                title: 'Dashboard.Pipelines',
                disabled: false,
                to: { name: 'Pipelines', params: {}}
            },
            {
                title: 'Pipeline.'+this.pipeline,
                disabled: true,
                to: { name: 'Pipeline Apps', params: { pipeline: this.pipeline }}
            }
        ],
        reviewapps: false,
        //phases: [] as Array<Phase>,
        git: {
            ssh_url: "",
            provider: ""
        },
        //pullrequests: [] as Array<Pullrequest>,
    }},
    computed: {
        activePhases() {
            let phases = [] as Array<Phase>;
            if (this.phases) {
                this.phases.forEach((phase: Phase) => {
                    if (phase.enabled) {
                        phases.push(phase);
                    }
                });
            }
            return phases;
        }
    },
    components: {
        PRcard,
        Appcard,
        Breadcrumbs,
    },
    
    methods: {
        loadPipeline,
        loadPullrequests,
    },
})
</script>