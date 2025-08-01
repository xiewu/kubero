<template>
    <div>
      <v-dialog
        v-model="dialog"
        max-width="600"
      >
        <template v-slot:activator="{ props: activatorProps }">
          <v-btn
            block
            prepend-icon="mdi-wrench"
            text="Build"
            color="secondary"
            v-bind="activatorProps"
          ></v-btn>
        </template>
  
        <v-card
          prepend-icon="mdi-wrench"
          title="New Build"
        >
          <v-card-text>

            <v-row dense>
              <v-col
                cols="12"
                sm="6"
              >
                <v-select
                  :items="['nixpacks', 'dockerfile', 'buildpacks']"
                  label="Build strategy*"
                  description="The strategy to build the source code"
                  auto-select-first
                  required
                  v-model="form.buildstrategy"
                  @update:modelValue="changedBuildStrategy"
                ></v-select>
              </v-col>
              <v-col
                cols="12"
                sm="6"
              >
                <v-combobox
                  label="Branch/Tag/Commit*"
                  description="The reference to the source code"
                  :items="references"
                  v-model="form.reference"
                ></v-combobox>
              </v-col>
              <v-col
                cols="12"
                sm="6"
              >
                <v-text-field
                  v-if="form.buildstrategy === 'dockerfile' || form.buildstrategy === 'nixpacks'"
                  label="Dockerfile Path"
                  description="Path to the Dockerfile in the source code"
                  v-model="form.dockerfilePath"
                ></v-text-field>
              </v-col>
            </v-row>
  
            <small class="text-caption text-medium-emphasis">*indicates required field</small>
          </v-card-text>
  
          <v-divider></v-divider>
  
          <v-card-actions>
            <v-spacer></v-spacer>
  
            <v-btn
              text="Cancel"
              variant="plain"
              @click="dialog = false"
            ></v-btn>
  
            <v-btn
              color="primary"
              text="Build"
              variant="tonal"
              @click="saveBuild"
            ></v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </div>
</template>



<script lang="ts">
import axios from "axios";
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'BuildsForm',
  
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
            default: "MISSING"
        },
        appData: {
            type: Object,
        }
  },
  data: () => ({
    dialog: false,
    form: {
        buildstrategy: 'dockerfile',
        reference: '',
        dockerfilePath: 'Dockerfile'
    },
    references: [] as string[],
  }),
  mounted() {
    this.loadReferences()
    console.log('BuildsForm', this.pipeline, this.phase, this.app, this.appData)
  },
  methods: {
    changedBuildStrategy() {
        console.log('changedBuildStrategy', this.form.buildstrategy)
        if (this.form.buildstrategy === 'nixpacks') {
            this.form.dockerfilePath = '.nixpacks/Dockerfile'
        } else {
            this.form.dockerfilePath = 'Dockerfile'
        }
    },
    loadReferences() {
        const repoB64 = btoa(this.appData?.spec.gitrepo.ssh_url)
        //const provider = this.appData?.spec.gitrepo.provider
        const provider = "github" // TODO: FIX: get provider from appData
        axios.get(`/api/repo/${provider}/${repoB64}/references`)
        .then(response => {
            this.references = response.data
        })
        .catch(error => {
            console.log('Error loading references', error)
        })
    },
    saveBuild() {
        //console.log('Build', this.pipeline, this.phase, this.app, this.form, this.appData)
        let repository = this.appData?.spec.gitrepo.ssh_url
        if (!this.appData?.spec.gitrepo.private) {
            //repository = this.appData?.spec.gitrepo.ssh_url.replace('git@', 'https://')
            repository = this.appData?.spec.gitrepo.clone_url
        }

        const body = {
            buildstrategy: this.form.buildstrategy,
            repository: repository,
            reference: this.form.reference,
            dockerfilePath: this.form.dockerfilePath
        }
        axios.post(`/api/deployments/build/${this.pipeline}/${this.phase}/${this.app}`, body)
        .then(response => {
            //console.log('Build submitted', response.data)
            this.dialog = false
        })
        .catch(error => {
            console.log('Error submitting build', error)
        })
    }
  }
})
</script>