<template>
    <div class="defi-borrow-confirmation">
        <tx-confirmation
            v-if="hasCorrectParams"
            :tx="tx"
            card-off
            :send-button-label="sendButtonLabel"
            :password-label="passwordLabel"
            :gas-limit="gasLimit"
            :on-send-transaction-success="onSendTransactionSuccess"
            :set-tmp-pwd="params.step === 1"
            :tmp-pwd-code="tmpPwdCode"
            @change-component="onChangeComponent"
        >
            <h1 class="with-back-btn">
                <f-back-button
                    v-if="!params.steps || params.step === 1"
                    :route-name="backButtonRoute"
                    :params="{ token }"
                />
                Confirmation
                <template v-if="params.steps">({{ params.step }}/{{ params.steps }})</template>
            </h1>

            <div class="confirmation-info">
                <div v-if="increasedDebt > 0">
                    You’re adding
                    <span class="inc-desc-collateral">
                        <f-token-value :token="token" :value="increasedDebt" no-currency /> {{ cTokenSymbol }}
                    </span>
                </div>
                <div v-else-if="decreasedDebt > 0">
                    <template v-if="params.step === 1">You’re allowing</template>
                    <template v-else>You’re removing</template>
                    <span class="inc-desc-collateral">
                        <f-token-value :token="token" :value="decreasedDebt" no-currency /> {{ cTokenSymbol }}
                    </span>
                </div>
            </div>

            <template #window-content>
                <ledger-confirmation-content :to="tx.to" :amount="0" />
            </template>
        </tx-confirmation>
        <template v-else>
            <f-message type="info" role="alert" class="big"> Adjust {{ cTokenSymbol }} first, please. </f-message>
        </template>
    </div>
</template>

<script>
import TxConfirmation from '../../components/TxConfirmation/TxConfirmation.vue';
import LedgerConfirmationContent from '../../components/LedgerConfirmationContent/LedgerConfirmationContent.vue';
import { GAS_LIMITS, Web3 } from '../../plugins/fantom-web3-wallet.js';
import { mapGetters } from 'vuex';
import { toFTM } from '../../utils/transactions.js';
import FBackButton from '../../components/core/FBackButton/FBackButton.vue';
import { getAppParentNode } from '../../app-structure.js';
import FMessage from '../../components/core/FMessage/FMessage.vue';
import FTokenValue from '@/components/core/FTokenValue/FTokenValue.vue';
import fMintUtils from 'fantom-ledgerjs/src/fmint-utils.js';
import erc20Utils from 'fantom-ledgerjs/src/erc20-utils.js';
import appConfig from '../../../app.config.js';
import { getUniqueId } from '@/utils';

/**
 * Common component for DefiBorrowFUSDConfirmation a DefiManageBorrowConfirmation
 */
export default {
    name: 'DefiBorrowConfirmation',

    components: { FTokenValue, FMessage, FBackButton, LedgerConfirmationContent, TxConfirmation },

    props: {
        /** Address of smart contract. */
        contractAddress: {
            type: String,
            default: '',
        },
        /**  */
        compName: {
            type: String,
            default: 'defi-borrow-fusd',
        },
        /** @type {DefiToken} */
        token: {
            type: Object,
            default() {
                return {};
            },
        },
        /** Router params */
        params: {
            type: Object,
            default() {
                return {};
            },
            required: true,
        },
    },

    data() {
        return {
            tx: {},
            gasLimit: GAS_LIMITS.default,
            tmpPwdCode: '',
        };
    },

    computed: {
        ...mapGetters(['currentAccount']),

        hasCorrectParams() {
            return this.params.currDebt !== this.params.debt;
        },

        increasedDebt() {
            return this.params.currDebt - this.params.debt;
        },

        decreasedDebt() {
            return this.params.debt - this.params.currDebt;
        },

        sendButtonLabel() {
            let label = '';

            if (this.params.step === 1) {
                label = 'Continue to the next step';
            } else {
                label = 'Submit';
            }
            /*
            } else if (this.increasedDebt > 0) {
                if (this.compName === 'defi-borrow-fusd') {
                    label = 'Mint now';
                } else {
                    label = 'Borrow now';
                }
            } else {
                label = 'Repay now';
            }
            */

            return label;
        },

        passwordLabel() {
            let label = '';

            if (this.params.step === 1) {
                label = 'Please enter your wallet password to allow your tokens';
            } else if (this.params.debt > 0) {
                label = 'Please enter your wallet password to rebalance your debt';
            } else {
                label = 'Please enter your wallet password to add your debt';
            }

            return label;
        },

        backButtonRoute() {
            const parentNode = getAppParentNode(`${this.compName}-confirmation`);

            return parentNode ? parentNode.route : '';
        },

        cTokenSymbol() {
            return this.$defi.getTokenSymbol({ symbol: this.token.symbol });
        },
    },

    created() {
        if (!this.hasCorrectParams) {
            // redirect to <this.compName>
            setTimeout(() => {
                this.$router.replace({ name: this.compName });
            }, 3000);
        } else {
            this.setTx();
        }
    },

    methods: {
        async setTx() {
            const { token } = this;
            const { params } = this;
            const repayMax = 0.995;
            const mintMax = repayMax;
            let { contractAddress } = this;
            let txToSign;

            if (!token) {
                return;
            }

            this.tmpPwdCode = params.tmpPwdCode || getUniqueId();

            if (!contractAddress) {
                contractAddress = this.$defi.contracts.fMint;
            }

            if (this.increasedDebt > 0) {
                // mint max
                if (params.ratio > mintMax) {
                    console.log('MINT MAX');
                    txToSign = fMintUtils.fMintMintTokenMaxTx(contractAddress, token.address);
                } else {
                    txToSign = fMintUtils.fMintMintTokenTx(
                        contractAddress,
                        token.address,
                        this.correctAmount(
                            Web3.utils.toHex(
                                this.$defi.shiftDecPointRight(this.increasedDebt.toString(), token.decimals)
                            ),
                            true
                        )
                    );
                }
            } else if (this.params.step === 1) {
                txToSign = erc20Utils.erc20IncreaseAllowanceTx(
                    token.address,
                    contractAddress,
                    this.correctAmount(
                        Web3.utils.toHex(
                            this.$defi.shiftDecPointRight((this.decreasedDebt * 1.051).toString(), token.decimals)
                        )
                    )
                );
            } else {
                // repay max
                if (params.ratio > repayMax) {
                    console.log('REPAY MAX');
                    txToSign = fMintUtils.fMintRepayTokenMaxTx(contractAddress, token.address);
                } else {
                    txToSign = fMintUtils.fMintRepayTokenTx(
                        contractAddress,
                        token.address,
                        this.correctAmount(
                            Web3.utils.toHex(
                                this.$defi.shiftDecPointRight(this.decreasedDebt.toString(), token.decimals)
                            )
                        )
                        // parseInt(this.decreasedDebt * Math.pow(10, token.decimals))
                    );
                }
            }

            this.tx = await this.$fWallet.getDefiTransactionToSign(
                txToSign,
                this.currentAccount.address,
                GAS_LIMITS.defi
            );
        },

        correctAmount(_amount, _borrow) {
            const { params } = this;
            /*
            if (!_borrow) {
                if (params.debtBalanceHex && this.$defi.compareBN(_amount, params.debtBalanceHex) === 1) {
                    return params.debtBalanceHex;
                }
            } else {
                if (params.borrowLimitHex && this.$defi.compareBN(_amount, params.borrowLimitHex) === 1) {
                    return params.borrowLimitHex;
                }
            }
            */
            console.log(_amount, params.debtBalanceHex, params.borrowLimitHex, _borrow);

            return _amount;
        },

        onSendTransactionSuccess(_data) {
            const params = {
                tx: _data.data.sendTransaction.hash,
                title: 'Success',
                continueTo: this.compName,
            };
            let transactionSuccessComp = `${this.compName}-transaction-success-message`;

            if (this.params.step === 1) {
                params.continueTo = `${this.compName}-confirmation2`;
                params.continueToParams = { ...this.params, step: 2, tmpPwdCode: this.tmpPwdCode };
                params.autoContinueToAfter = appConfig.settings.autoContinueToAfter;
                params.continueButtonLabel = 'Next Step';
                params.title = `${this.params.step}/${this.params.steps}  ${params.title}`;
            } else if (this.params.step === 2) {
                transactionSuccessComp = `${this.compName}-transaction-success-message2`;
                params.continueToParams = { token: { ...this.token } };
            }

            this.$router.replace({
                name: transactionSuccessComp,
                params,
            });
        },

        /**
         * Re-target `'change-component'` event.
         *
         * @param {object} _data
         */
        onChangeComponent(_data) {
            let transactionRejectComp = `${this.compName}-transaction-reject-message`;

            if (_data.to === 'transaction-reject-message') {
                if (this.params.step === 2) {
                    transactionRejectComp = `${this.compName}-transaction-reject-message2`;
                }

                this.$router.replace({
                    name: transactionRejectComp,
                    params: {
                        continueTo: this.compName,
                        continueToParams: { token: { ...this.token }, tmpPwdCode: this.tmpPwdCode },
                    },
                });
            }
        },

        toFTM,
    },
};
</script>
