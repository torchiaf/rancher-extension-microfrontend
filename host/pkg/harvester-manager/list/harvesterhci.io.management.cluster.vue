<script>
import BrandImage from '@shell/components/BrandImage';
import TypeDescription from '@shell/components/TypeDescription';
import ResourceTable from '@shell/components/ResourceTable';
import Masthead from '@shell/components/ResourceList/Masthead';
import Loading from '@shell/components/Loading';
import { HARVESTER_NAME as VIRTUAL } from '@shell/config/features';
import { CAPI, HCI, MANAGEMENT, WORKLOAD_TYPES } from '@shell/config/types';
import { isHarvesterCluster } from '@shell/utils/cluster';
import { allHash } from '@shell/utils/promise';

const HARVESTER_MICRO_DEPLOYMENT_ID = 'default/harvester-microfrontend';

export default {
  components: {
    BrandImage,
    ResourceTable,
    Masthead,
    TypeDescription,
    Loading
  },

  props: {
    schema: {
      type:     Object,
      required: true,
    },
    useQueryParamsForSimpleFiltering: {
      type:    Boolean,
      default: false
    }
  },

  async fetch() {
    const inStore = this.$store.getters['currentProduct'].inStore;

    const hash = await allHash({
      hciClusters:  this.$store.dispatch(`${ inStore }/findAll`, { type: HCI.CLUSTER }),
      mgmtClusters: this.$store.dispatch(`${ inStore }/findAll`, { type: MANAGEMENT.CLUSTER }),
      deployments:  this.$store.dispatch(`${ inStore }/findAll`, { type: WORKLOAD_TYPES.DEPLOYMENT }),
    });

    this.hciClusters = hash.hciClusters;
    this.mgmtClusters = hash.mgmtClusters;
  },

  data() {
    const resource = CAPI.RANCHER_CLUSTER;

    return {
      navigating:   false,
      VIRTUAL,
      hciDashboard: HCI.DASHBOARD,
      resource,
      hResource:    HCI.CLUSTER,
      realSchema:   this.$store.getters['management/schemaFor'](CAPI.RANCHER_CLUSTER),
      hciClusters:  [],
      mgmtClusters: []
    };
  },

  computed: {
    importLocation() {
      return {
        name:   'c-cluster-product-resource-create',
        params: {
          product:  this.$store.getters['currentProduct'].name,
          resource: this.schema.id,
        },
      };
    },

    canCreateCluster() {
      const schema = this.$store.getters['management/schemaFor'](CAPI.RANCHER_CLUSTER);

      return !!schema?.collectionMethods.find((x) => x.toLowerCase() === 'post');
    },

    rows() {
      return this.hciClusters.filter((c) => {
        const cluster = this.mgmtClusters.find((cluster) => cluster?.metadata?.name === c?.status?.clusterName);

        return isHarvesterCluster(cluster);
      });
    },

    typeDisplay() {
      return '';
    },

    harvesterExtensionMicro() {
      const deployments = this.$store.getters['management/all'](WORKLOAD_TYPES.DEPLOYMENT);

      const harvesterMicro = deployments?.find((deploy) => deploy.id === HARVESTER_MICRO_DEPLOYMENT_ID);

      if (!harvesterMicro) {
        return 'not-installed';
      }
      if (harvesterMicro.state !== 'active') {
        return 'not-ready';
      }
      return 'active';
    }
  },

  methods: {
    async goToCluster(row) {
      const timeout = setTimeout(() => {
        // Don't show loading indicator for quickly fetched plugins
        this.navigating = row.id;
      }, 1000);

      try {
        await row.goToCluster();

        clearTimeout(timeout);
        this.navigating = false;
      } catch {
        // The error handling is carried out within goToCluster, but just in case something happens before the promise chain can catch it...
        clearTimeout(timeout);
        this.navigating = false;
      }
    },

    goToRemote(row) {    
      this.$router.push({
        name: 'c-cluster-resource-remote',
        params: {
          product:     'harvesterManager',
          cluster:     '_',
          clusterName: row.status.clusterName,
        }
      });
    }
  },
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <div v-else>
    <div v-if="harvesterExtensionMicro === 'active'">
      <Masthead
        :schema="realSchema"
        :resource="resource"
        :is-creatable="false"
        :type-display="typeDisplay"
      >
        <template #typeDescription>
          <TypeDescription :resource="hResource" />
        </template>

        <template
          v-if="canCreateCluster"
          #extraActions
        >
          <router-link
            :to="importLocation"
            class="btn role-primary"
          >
            {{ t('cluster.importAction') }}
          </router-link>
        </template>
      </Masthead>
      <ResourceTable
        v-if="rows && rows.length"
        :schema="schema"
        :rows="rows"
        :is-creatable="true"
        :namespaced="false"
        :use-query-params-for-simple-filtering="useQueryParamsForSimpleFiltering"
      >
        <template #col:name="{row}">
          <td>
            <span class="cluster-link">
              <a
                v-if="row.isReady"
                class="link"
                :disabled="navigating ? true : null"
                @click="goToRemote(row)"
              >{{ row.nameDisplay }}</a>
              <span v-else>
                {{ row.nameDisplay }}
              </span>
              <i
                class="icon icon-spinner icon-spin ml-5"
                :class="{'navigating': navigating === row.id}"
              />
            </span>
          </td>
        </template>

        <template #cell:harvester="{row}">
          <router-link
            class="btn btn-sm role-primary"
            :to="row.detailLocation"
          >
            {{ t('harvesterManager.manage') }}
          </router-link>
        </template>
      </ResourceTable>
      <div v-else>
        <div class="no-clusters">
          {{ t('harvesterManager.cluster.none') }}
        </div>
        <hr class="info-section">
      </div>
    </div>
    <template v-if="harvesterExtensionMicro !== 'active' || !rows || !rows.length">
      <div class="logo">
        <BrandImage
          file-name="harvester.png"
          height="64"
        />
      </div>
      <div class="tagline">
        <div>{{ t('harvesterManager.cluster.description') }}</div>
      </div>
      <div class="tagline">
        <div v-clean-html="t('harvesterManager.cluster.learnMore', {}, true)" />
      </div>
      <template v-if="harvesterExtensionMicro !== 'active'">
        <div class="tagline microfrontend-warning-panel">
          <div class="micro-separator"></div>
          <template v-if="harvesterExtensionMicro === 'not-installed'">
            <div
              v-clean-html="t('harvesterManager.microfrontend.install.warning.notInstalled', {}, true)"
              class="microfrontend-warning mb-15"
            />
            <i class="icon lg icon-x text-error" />
          </template>
          <template v-if="harvesterExtensionMicro === 'not-ready'">
            <div
              v-clean-html="t('harvesterManager.microfrontend.install.warning.notReady', {}, true)"
              class="microfrontend-warning mb-15"
            />
            <i class="icon lg icon-spinner icon-spin" />
          </template>
        </div>
      </template>
    </template>
  </div>
</template>

<style lang="scss" scoped>
  .cluster-link {
    display: flex;
    align-items: center;

    .icon {
      // Use visibility to avoid the columns re-adjusting when the icon is shown
      visibility: hidden;

      &.navigating {
        visibility: visible;
      }
    }

  }
  .no-clusters {
    text-align: center;
  }

  .info-section {
    margin-top: 60px;
  }

  .logo {
    display: flex;
    justify-content: center;
    margin: 60px 0 40px 0;
  }

  .tagline {
    display: flex;
    justify-content: center;
    margin-top: 30px;

    > div {
      font-size: 16px;
      line-height: 22px;
      max-width: 80%;
      text-align: center;
    }
  }

  .micro-separator {
    border: 1px solid var(--border);
    margin-bottom: 20px;
    width: 50%;
  }

  .microfrontend-warning-panel {
    align-items: center;
    flex-direction: column;
  }

  .microfrontend-warning {
    font-size: 24px !important;
    font-weight: 400;
  }

  .lg {
    font-size: 30px;
  }

 .link {
    cursor: pointer;
  }

</style>
