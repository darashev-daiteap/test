<template>
  <div class="row">
    <div class="card mb-4 w-100">
      <div class="card-header py-3 cardHeaderLight">
        <h6 class="m-0 font-weight-bold">Longhorn Storage:</h6>
      </div>
      <div class="card-body">

      <div class="table-responsive">
        <table class="table table-bordered" id="dataTable" width="100%" cellspacing="0">
          <thead>
            <tr>
              <th v-if="cluster.longhorn_username">Username</th>
              <th v-if="cluster.longhorn_password">Password</th>
              <th>Address</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td v-if="cluster.longhorn_username">{{ cluster.longhorn_username }}</td>
              <td v-if="cluster.longhorn_password" style="display: flex">
                <div style="flex: 1" id="pwd">
                  {{
                  "*".repeat(cluster.longhorn_password.length)
                  }}
                </div>
                <div style="margin-left: auto; margin: 5px" type="button" id="copy" class="child far fa-copy" alt="copy"
                  title="Double click to copy password" @click="$emit('copyLonghornPassword')">
                  <span v-if="longhornPWCopied" style="margin-left: auto; margin: 5px">Copied</span>
                </div>
              </td>
              <td>
                <a :href="'http://' + cluster.longhorn_address" target="_blank" rel="noopener noreferrer">{{
                  cluster.longhorn_address
                  }}</a>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

        <div v-if="loadingTable" class="d-flex justify-content-center">
          <div class="spinner-border" role="status">
            <span class="sr-only">Loading...</span>
          </div>
        </div>

        <div
          v-else-if="showStorage == false"
          class="d-flex justify-content-center"
        >
          Storage information not available
        </div>

        <div v-else >
          <div class="table-responsive">
            <table
              class="table table-bordered"
              id="dataTable"
              width="100%"
              cellspacing="0"
            >
              <thead>
                <tr>
                  <th>Node</th>
                  <th>Type</th>
                  <th>Storage</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(node, index) in allNodes" :key="index">
                  <td class="node-name col-2">
                    {{ node.name }}
                  </td>
                  <td class="col-2">
                    <span
                      v-if="
                        (!!cluster.machinesList &&
                          !!cluster.machinesList.length > 0 &&
                          cluster.machinesList.filter(
                            (el) => el.kube_name === node.name
                          )[0].kube_master) ||
                        node.name.includes('-control-plane-')
                      "
                      >Control plane</span
                    >
                    <span v-else>Worker</span>
                  </td>

                  <td>
                    <b-progress
                      :max="(node.storageMaximum / 1024 / 1024 / 1024).toFixed(2)"
                      height="1rem"
                      class="daiteap-progress"
                    >
                      <b-progress-bar
                        class="daiteap-progress-bar"
                        :value="
                          (node.storageMaximum - node.storageAvailable) /
                          1024 /
                          1024 /
                          1024
                        "
                      >
                      </b-progress-bar>
                    </b-progress>
                    <span
                      >Used:
                      <strong
                        >{{
                          (
                            (node.storageMaximum - node.storageAvailable) /
                            1024 /
                            1024 /
                            1024
                          ).toFixed(2)
                        }}
                        GB /
                        {{
                          (node.storageMaximum / 1024 / 1024 / 1024).toFixed(2)
                        }}
                        GB</strong
                      ></span
                    >
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
    <br />
  </div>
</template>

<script>

export default {
  name: 'ClusterStorage',
  mounted() {
    this.getClusterStorage();
  },
  props: {
    cluster: {
      type: Object,
      required: true,
    },
    clusterID: {
      type: String,
      required: true,
    },
    longhornPWCopied: {
      type: Boolean,
    },
  },
  data() {
    return {
      showStorage: true,
      loadingTable: true,
      interval: "",
      allNodes: [],
    };
  },
  created() {},
  methods: {
    gettaskmessage(taskId) {
      let self = this;
      self.axios
        .get("/server/task-message/" + taskId, this.get_axiosConfig())
        .then(function (response) {
          if (response.data.status !== "PENDING") {
            clearInterval(self.intervalGetTaskMessage);
            if (response.data.error) {
              self.showStorage = false;
              console.log(response.data.errorMessage);
            }
            self.allNodes = response.data.lcmStatuses[0].nodes;
            self.loadingTable = false;
          }
        })
        .catch(function (error) {
          clearInterval(self.intervalGetTaskMessage);
          self.loadingTable = false;
          self.showStorage = false;
          console.log(error);
          if (error.response && error.response.status == "403") {
            self.$notify({
              group: "msg",
              type: "error",
              title: "Notification:",
              text: "Access Denied",
            });
          }
        });
    },
    getClusterStorage() {
      let self = this;
      this.axios
        .get(
          "/server/tenants/" +
            self.computed_active_tenant_id +
            "/clusters/" +
            self.clusterID +
            "/storage",
          this.get_axiosConfig()
        )
        .then(function (response) {
          let taskId = response.data.taskId;

          self.intervalGetTaskMessage = setInterval(() => {
            self.gettaskmessage(taskId);
          }, 1000);
        })
        .catch(function (error) {
          self.loadingTable = false;
          self.showStorage = false;
          console.log(error);
          if (error.response && error.response.status == "403") {
            self.$notify({
              group: "msg",
              type: "error",
              title: "Notification:",
              text: "Access Denied",
            });
          }
        });
    },
  },
  components: {},
  destroyed() {
    if (window.intervals) {
      for (let i = 0; i < window.intervals.length; i++) {
        clearInterval(window.intervals[i]);
      }
    }
    clearInterval(this.interval);
    clearInterval(this.intervalGetTaskMessage);
  },
};
</script>

<style scoped>
a:not([href]):not([tabindex]) {
  color: #4e73df;
  text-decoration: none;
  cursor: pointer;
}

a:not([href]):not([tabindex]):hover {
  color: #224abe;
  text-decoration: underline;
  cursor: pointer;
}

.fa-user-plus {
  color: #000;
}

.fa-server {
  color: #000;
}

.card-header {
  display: block;
}

.card-header > h6 {
  display: inline-block;
}
* {
  white-space: inherit;
}

.row {
  display: flex;
  width: 100%;
  align-items: stretch;
}
.sidebar {
  position: fixed;
  padding-left: 1rem;
  margin-left: 1rem;
  top: 5rem;
}
.cardHeaderLight {
  background-color: #fff;
  color: #1d1e22;
  border-bottom: #1d1e22 2px solid;
  border: #d1d1d1 1px solid;
}
.card-body {
  border: #00708c10 1px solid;
}
</style>
