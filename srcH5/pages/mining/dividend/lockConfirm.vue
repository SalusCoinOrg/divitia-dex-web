<template>
    <confirm v-show="isShow" :showMask="true"
             :title="confirmText.title"
             :closeIcon="true" :close="close"
             :leftBtnClick="submit" :leftBtnTxt="confirmText.submit"
             :singleBtn="true" :btnUnuse="!canSubmit">
        <div class="__row">
            <div class="__row_t">{{ confirmText.available }}</div>
            <div class="__input_row __unuse_input __bold">
                <img :src="dslsTokenInfo.icon" class="__icon" />dSLS
                <span class="__right">{{ basicAvailableAmount }}</span>
            </div>
        </div>

        <div class="__row">
            <div class="__row_t">
                {{ confirmText.amountText }}
                <span v-show="amountErr" class="__err">{{ amountErr }}</span>
            </div>
            <vite-input v-model="amount" :valid="validAmount" @input="noAll"
                        :placeholder="confirmText.amountPlaceholder">
                <span slot="after" @click="all" class="all-wrapper">
                    <span class="all">{{ $t('tradeAssets.all') }}</span>
                </span>
            </vite-input>
        </div>

        <div v-if="isLockDSLS">
            <div class="__hint distance"><span>{{ $t('tradeDividend.addHint1') }}</span></div>
            <div class="__hint"><span>{{ $t('tradeDividend.addHint2') }}</span></div>
            <div class="__hint"><span>{{ $t('tradeDividend.addHint3') }}</span></div>
        </div>
        <div v-else>
            <div class="__hint distance"><span>{{ $t('tradeDividend.unlockVXConfirm.hint') }}</span></div>
        </div>
    </confirm>
</template>

<script>
import { VITE_TOKENID } from 'utils/constant';
import viteInput from 'components/viteInput';
import confirm from 'h5Components/confirm/confirm.vue';
import bigNumber from 'utils/bigNumber';
import sendTx from 'h5Utils/sendTx';
import { verifyAmount } from 'h5Utils/validations';

export default {
    components: { confirm, viteInput },
    data() {
        return {
            isLockDSLS: true,
            isShow: false,
            amount: '',
            amountErr: '',
            isAll: false
        };
    },
    computed: {
        dslsTokenInfo() {
            return this.$store.state.env.tokenMap[VITE_TOKENID];
        },
        dslsTokenDecimals() {
            return this.dslsTokenInfo.decimals;
        },
        confirmText() {
            if (this.isLockDSLS) {
                return {
                    title: this.$t('tradeDividend.lockVXConfirm.title'),
                    submit: this.$t('tradeDividend.lockVXConfirm.submit'),
                    available: this.$t('tradeDividend.lockVXConfirm.available'),
                    notEnough: this.$t('tradeDividend.lockVXConfirm.notEnough'),
                    amountPlaceholder: this.$t('tradeDividend.lockVXConfirm.amountPlaceholder'),
                    amountText: this.$t('tradeDividend.lockVXConfirm.amountText')
                };
            }
            return {
                title: this.$t('tradeDividend.unlockVXConfirm.title'),
                submit: this.$t('tradeDividend.unlockVXConfirm.submit'),
                available: this.$t('tradeDividend.unlockVXConfirm.available'),
                notEnough: this.$t('tradeDividend.unlockVXConfirm.notEnough'),
                amountPlaceholder: this.$t('tradeDividend.unlockVXConfirm.amountPlaceholder'),
                amountText: this.$t('tradeDividend.unlockVXConfirm.amountText')
            };
        },
        dslsBalanceInfo() {
            return this.$store.getters.exVXBalanceInfo || {};
        },
        availableAmount() {
            return this.isLockDSLS
                ? this.dslsBalanceInfo.available || 0
                : this.dslsBalanceInfo.vxLocked || 0;
        },
        basicAvailableAmount() {
            return bigNumber.toBasic(this.availableAmount, this.dslsTokenDecimals);
        },
        canSubmit() {
            return +this.amount && !this.amountErr;
        }
    },
    methods: {
        validAmount() {
            this.amountErr = verifyAmount({
                minAmount: bigNumber.toMin(1, this.dslsTokenDecimals),
                formatDecimals: 8,
                decimals: this.dslsTokenDecimals,
                balance: this.availableAmount,
                errorMap: {
                    notEnough: this.confirmText.notEnough,
                    lessMin: this.$t('tradeDividend.lessMin')
                }
            })(this.amount);
            return !this.amountErr;
        },
        noAll() {
            this.isAll = false;
        },
        all() {
            if (!+this.availableAmount) {
                return;
            }
            this.isAll = true;
            this.amount = bigNumber.toBasic(this.availableAmount, this.dslsTokenDecimals);
        },
        show(isLockDSLS) {
            this.isLockDSLS = isLockDSLS;
            this.isShow = true;
        },
        close() {
            this.isShow = false;
            this.amount = '';
            this.isAll = false;
        },
        submit() {
            const actionType = this.isLockDSLS ? 1 : 2;
            const amount = this.isAll ? this.availableAmount : bigNumber.toMin(this.amount, this.dslsTokenDecimals);

            sendTx({
                methodName: 'dexLockVxForDividend',
                data: { actionType, amount }
            }).then(() => {
                this.close();
            }).catch(err => {
                console.warn(err);
            });
        }
    }
};
</script>

<style lang="scss" scoped>
.__hint {
    &.distance {
        margin-top: 20px;
    }
    margin-top: 6px;
}
.all-wrapper {
    color: #007AFF;
    font-size: 12px;
    margin: 0 15px;
    .all {
        border-bottom: 1px dashed #007AFF;
    }
}
</style>
