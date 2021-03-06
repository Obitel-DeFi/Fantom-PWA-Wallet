<template>
    <div class="delegations-info">
        <f-card class="f-card-double-padding f-data-layout">
            <staking-overview />
        </f-card>

        <f-tabs class="account-main-content-mt">
            <template #delegations>
                Wallet Delegations
                <span class="f-records-count">({{ delegationsRecordsCount }})</span>
            </template>
            <template #all-delegations>
                All Delegations
                <span class="f-records-count">({{ allDelegationsRecordsCount }})</span>
            </template>

            <f-tab title-slot="delegations">
                <f-card class="f-data-layout">
                    <delegation-list
                        :account-address="currentAccount.address"
                        @row-action="onDelegationsRowAction"
                        @records-count="onDelegationsRecordsCount"
                    />
                    <div class="add-delegation-cont">
                        <button class="btn large" type="button" @click="onAddDelegationBtnClick">
                            <icon data="@/assets/svg/plus.svg" width="16" height="16" aria-hidden="true" /> Add
                            delegation
                        </button>
                    </div>
                </f-card>
            </f-tab>
            <f-tab title-slot="all-delegations">
                <f-card>
                    <all-delegations-list
                        :account-address="currentAccount.address"
                        @row-action="onAllDelegationsRowAction"
                        @records-count="onAllDelegationsRecordsCount"
                    />
                </f-card>
            </f-tab>
        </f-tabs>
    </div>
</template>

<script>
import FCard from '@/components/core/FCard/FCard.vue';
import DelegationList from '@/components/data-tables/DelegationList/DelegationList.vue';
import { mapGetters } from 'vuex';
import StakingOverview from '@/components/StakingOverview/StakingOverview.vue';
import FTabs from '@/components/core/FTabs/FTabs.vue';
import FTab from '@/components/core/FTabs/FTab.vue';
import AllDelegationsList from '@/components/data-tables/AllDelegationsList/AllDelegationsList.vue';
import {
    DEACTIVATE_ACTIVE_ACCOUNT,
    SET_ACTIVE_ACCOUNT_ADDRESS,
    SET_ACTIVE_ACCOUNT_BY_ADDRESS,
} from '@/store/mutations.type.js';

export default {
    name: 'DelegationsInfo',

    components: { AllDelegationsList, FTab, FTabs, StakingOverview, DelegationList, FCard },

    data() {
        return {
            delegationsRecordsCount: 0,
            allDelegationsRecordsCount: 0,
        };
    },

    computed: {
        ...mapGetters(['currentAccount']),
    },

    methods: {
        onAddDelegationBtnClick() {
            this.$emit('change-component', {
                to: 'stake-form',
                from: 'delegations-info',
                data: {
                    increaseDelegation: false,
                    stakerInfo: {
                        stakerInfo: {
                            name: 'Unknown',
                        },
                    },
                    previousComponent: 'delegations-info',
                },
            });
        },

        /**
         * @param {object} _item
         */
        onDelegationsRowAction(_item) {
            this.$emit('change-component', {
                to: 'staking-info',
                from: 'delegations-info',
                data: {
                    stakerId: _item.delegation.toStakerId,
                },
            });
        },

        /**
         * @param {string} _address
         */
        setActiveAccount(_address) {
            this.$store.commit(DEACTIVATE_ACTIVE_ACCOUNT);
            this.$store.commit(SET_ACTIVE_ACCOUNT_BY_ADDRESS, _address);
            this.$store.commit(SET_ACTIVE_ACCOUNT_ADDRESS, _address);
        },

        /**
         * @param {object} _item
         */
        onAllDelegationsRowAction(_item) {
            const address = _item.accountAddress;

            if (this.currentAccount.address.toLowerCase() !== address.toLowerCase()) {
                this.setActiveAccount(address);
                this.$router.replace({
                    name: 'staking',
                    params: { address },
                });
            }

            this.$emit('change-component', {
                to: 'staking-info',
                from: 'delegations-info',
                data: {
                    stakerId: _item.delegation.toStakerId,
                },
            });
        },

        onDelegationsRecordsCount(_count) {
            this.delegationsRecordsCount = _count;
        },

        onAllDelegationsRecordsCount(_count) {
            this.allDelegationsRecordsCount = _count;
        },
    },
};
</script>

<style lang="scss">
@import 'style';
</style>
