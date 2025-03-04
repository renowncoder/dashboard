<script>
import Banner from '@/components/Banner';
import Loading from '@/components/Loading';
import MessageLink from '@/components/MessageLink';
import ResourceTable from '@/components/ResourceTable';

import { NAME, AGE, NAMESPACE } from '@/config/table-headers';
import { HCI } from '@/config/types';

import { findBy } from '@/utils/array';
import { allHash } from '@/utils/promise';

export default {
  name:       'HarvesterListNetworks',
  components: {
    ResourceTable, Banner, Loading, MessageLink
  },

  props: {
    schema: {
      type:     Object,
      required: true,
    }
  },

  async fetch() {
    const _hash = { rows: this.$store.dispatch('harvester/findAll', { type: HCI.NETWORK_ATTACHMENT }) };

    if (this.$store.getters['harvester/schemaFor'](HCI.NODE_NETWORK)) {
      _hash.hostNetworks = this.$store.dispatch('harvester/findAll', { type: HCI.NODE_NETWORK });
    }

    if (this.$store.getters['harvester/schemaFor'](HCI.CLUSTER_NETWORK)) {
      _hash.clusterNetworkSetting = this.$store.dispatch('harvester/findAll', { type: HCI.CLUSTER_NETWORK });
    }

    const hash = await allHash(_hash);

    this.rows = hash.rows;
    this.hostNetworks = hash.hostNetworks || [];
    this.clusterNetworkSetting = hash.clusterNetworkSetting || [];
  },

  data() {
    return {
      hash:                  {},
      rows:                  [],
      hosts:                 [],
      hostNetworks:          [],
      clusterNetworkSetting: [],
      to:                    `${ HCI.CLUSTER_NETWORK }/vlan?mode=edit`
    };
  },

  computed: {
    headers() {
      return [
        NAME,
        NAMESPACE,
        {
          name:      'type',
          value:     'vlanType',
          sort:      'spec.config',
          labelKey:  'tableHeaders.networkType'
        },
        {
          name:      'vlan',
          value:     'vlanId',
          sort:      'spec.config',
          labelKey:  'tableHeaders.networkVlan'
        },
        AGE
      ];
    },

    isVlanDisable() {
      const vlan = findBy((this.clusterNetworkSetting || []), 'metadata.name', 'vlan') || {};

      return !vlan.canUseVlan;
    },

    hasClusterNetwork() {
      return this.$store.getters['harvester/schemaFor'](HCI.CLUSTER_NETWORK);
    },

    abnormalNetwork() {
      const notReadyCrd = this.hostNetworks.filter( O => !O.isReady);

      return notReadyCrd.map( O => O.linkMessage);
    },
  },
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <div v-else>
    <template v-if="isVlanDisable && hasClusterNetwork">
      <Banner color="error">
        <MessageLink
          :to="to"
          prefix-label="harvester.network.message.premise.prefix"
          middle-label="harvester.network.message.premise.middle"
          suffic-label="harvester.network.message.premise.suffic"
        />
      </Banner>
    </template>

    <template v-else>
      <Banner v-if="hasClusterNetwork" color="info">
        <MessageLink
          :to="to"
          prefix-label="harvester.network.message.viewSetting.prefix"
          middle-label="harvester.network.message.viewSetting.middle"
          suffic-label="harvester.network.message.viewSetting.suffic"
        />
      </Banner>
    </template>

    <template v-if="abnormalNetwork.length">
      <Banner
        v-for="item in abnormalNetwork"
        :key="item.name"
        color="error"
      >
        <nuxt-link :to="item.to">
          {{ item.name }}:
        </nuxt-link>
        {{ item.message }}
      </Banner>
    </template>

    <ResourceTable
      v-bind="$attrs"
      :headers="headers"
      default-sort-by="age"
      :schema="schema"
      :groupable="true"
      :rows="rows"
      key-field="_key"
      v-on="$listeners"
    />
  </div>
</template>
