<template lang="pug">
  b-modal#amazon-payment(title="Amazon", :hide-footer="true", size='md')
    h2.text-center Continue with Amazon
    #AmazonPayButton
    | {{amazonPayments}}
    | {{amazonLoggedIn}}
    #AmazonPayWallet(v-if="amazonLoggedIn", style="width: 400px; height: 228px;")
    #AmazonPayRecurring(v-if="amazonLoggedIn && amazonPayments.type === 'subscription'",
                        style="width: 400px; height: 140px;")
    .modal-footer
      .text-center
        .btn.btn-primary(v-if="amazonPaymentsCanCheckout",
          @click="amazonCheckOut()") {{ $t('checkout') }}
</template>

<style scoped>
  #AmazonPayButton {
    margin: 0 auto;
    width: 150px;
  }

  #AmazonPayRecurring {
    height: 200px;
    width: 500px;
  }
</style>

<script>
import axios from 'axios';
import { mapState } from 'client/libs/store';
import bModal from 'bootstrap-vue/lib/components/modal';

const AMAZON_PAYMENTS = process.env.AMAZON_PAYMENTS; // eslint-disable-line

export default {
  components: {
    bModal,
  },
  props: ['amazonPaymentsProp'],
  data () {
    return {
      OffAmazonPayments: {},
      isAmazonSetup: false,
      amazonButtonEnabled: false,
      amazonPaymentsbillingAgreementId: '',
      amazonPaymentspaymentSelected: false,
      amazonPaymentsrecurringConsent: 'false',
      amazonLoggedIn: false,
    };
  },
  computed: {
    ...mapState({user: 'user.data'}),
    ...mapState(['isAmazonReady']),
    // @TODO: Eh, idk if we should move data props here or move these props to data. But we shouldn't have both
    amazonPayments () {
      let amazonPayments = {
        type: 'single',
        loggedIn: this.amazonLoggedIn,
      };
      amazonPayments = Object.assign({}, amazonPayments, this.amazonPaymentsProp);
      return amazonPayments;
    },
    amazonPaymentsCanCheckout () {
      if (this.amazonPayments.type === 'single') {
        return this.amazonPaymentspaymentSelected === true;
      } else if (this.amazonPayments.type === 'subscription') {
        return this.amazonPaymentspaymentSelected === true &&
                // Mah.. one is a boolean the other a string...
                this.amazonPaymentsrecurringConsent === 'true';
      } else {
        return false;
      }
    },
  },
  mounted () {
    if (this.isAmazonReady) return this.setupAmazon();

    this.$store.watch(state => state.isAmazonReady, (isAmazonReady) => {
      if (isAmazonReady) return this.setupAmazon();
    });
  },
  methods: {
    setupAmazon () {
      if (this.isAmazonSetup) return false;
      this.isAmazonSetup = true;
      this.OffAmazonPayments = window.OffAmazonPayments;
      this.showButton();
    },
    showButton () {
      // @TODO: prevent modal close form clicking outside
      let amazonButton = this.OffAmazonPayments.Button( // eslint-disable-line
        'AmazonPayButton',
        AMAZON_PAYMENTS.SELLER_ID,
        {
          type: 'PwA',
          color: 'Gold',
          size: 'small',
          agreementType: 'BillingAgreement',

          onSignIn: async (contract) => {
            this.amazonPaymentsbillingAgreementId = contract.getAmazonBillingAgreementId();
            this.amazonLoggedIn = true;
            this.$set(this.amazonPayments, 'loggedIn', true);

            if (this.amazonPayments.type === 'subscription') {
              this.amazonPaymentsinitWidgets();
            } else {
              let url = '/amazon/createOrderReferenceId';
              let response = await axios.post(url, {
                billingAgreementId: this.amazonPaymentsbillingAgreementId,
              });

              if (response.status <= 400) {
                this.amazonPayments.orderReferenceId = response.data.data.orderReferenceId;

                // @TODO: Clarify the deifference of these functions by renaming
                this.amazonPaymentsinitWidgets();
                this.amazonInitWidgets();
                return;
              }

              alert(response.message);
            }
          },

          authorization: () => {
            window.amazon.Login.authorize({
              scope: 'payments:widget',
              popup: true,
            }, function amazonSuccess (response) {
              if (response.error) return alert(response.error);

              let url = '/amazon/verifyAccessToken';
              axios.post(url, response)
                .catch((e) => {
                  alert(e.message);
                });
            });
          },

          onError: this.amazonOnError,
        });
    },
    amazonInitWidgets () {
      let walletParams = {
        sellerId: AMAZON_PAYMENTS.SELLER_ID, // @TODO: Import
        design: {
          designMode: 'responsive',
        },

        onPaymentSelect: () => {
          this.amazonPaymentspaymentSelected = true;
        },

        onError: this.amazonOnError,
      };

      // @TODO: Check if this is duplicated below
      if (this.amazonPayments.type === 'subscription') {
        walletParams.agreementType = 'BillingAgreement';

        walletParams.billingAgreementId = this.amazonPaymentsbillingAgreementId;
        walletParams.onReady = (billingAgreement) => {
          this.amazonPaymentsbillingAgreementId = billingAgreement.getAmazonBillingAgreementId();

          new this.OffAmazonPayments.Widgets.Consent({
            sellerId: AMAZON_PAYMENTS.SELLER_ID,
            amazonBillingAgreementId: this.amazonPaymentsbillingAgreementId,
            design: {
              designMode: 'responsive',
            },

            onReady: (consent) => {
              let getConsent = consent.getConsentStatus;
              this.amazonPaymentsrecurringConsent = getConsent ? getConsent() : false;
            },

            onConsent: (consent) => {
              this.amazonPaymentsrecurringConsent = consent.getConsentStatus();
            },

            onError: this.amazonOnError,
          }).bind('AmazonPayRecurring');
        };
      } else {
        walletParams.amazonOrderReferenceId = this.amazonPayments.orderReferenceId;
      }

      new this.OffAmazonPayments.Widgets.Wallet(walletParams).bind('AmazonPayWallet');
    },
    async amazonCheckOut () {
      this.amazonButtonEnabled = false;

      // @TODO: Create factory functions
      // @TODO: A gift should not read the same as buying gems for yourself.
      if (this.amazonPayments.type === 'single') {
        let url = '/amazon/checkout';
        let response = await axios.post(url, {
          orderReferenceId: this.amazonPayments.orderReferenceId,
          gift: this.amazonPaymentsgift,
        });

        if (response.status < 400) {
          this.reset();
          // @TODO: What are we syncing?
          window.location.reload(true);
          return;
        }

        alert(response.message);
        this.amazonPaymentsreset();
      } else if (this.amazonPayments.type === 'subscription') {
        let url = '/amazon/subscribe';

        if (this.amazonPayments.groupToCreate) {
          url = '/api/v3/groups/create-plan';
        }

        let response = await axios.post(url, {
          billingAgreementId: this.amazonPaymentsbillingAgreementId,
          subscription: this.amazonPayments.subscription,
          coupon: this.amazonPayments.coupon,
          groupId: this.amazonPayments.groupId,
          groupToCreate: this.amazonPayments.groupToCreate,
          paymentType: 'Amazon',
        });

        let responseStatus = response.status;
        if (responseStatus >= 400) {
          alert(`Error: ${response.message}`);
          // @TODO: do we need this? this.amazonPaymentsreset();
          return;
        }

        let newGroup = response.data.data;
        if (newGroup && newGroup._id) {
          // @TODO: Just append? or $emit?
          this.$router.push(`/group-plans/${newGroup._id}/task-information`);
          this.user.guilds.push(newGroup._id);
          return;
        }

        if (this.amazonPayments.groupId) {
          this.$router.push(`/group-plans/${this.amazonPayments.groupId}/task-information`);
          return;
        }

        window.location.reload(true);
        this.reset();
      }
    },
    amazonPaymentsinitWidgets () {
      let walletParams = {
        sellerId: AMAZON_PAYMENTS.SELLER_ID,
        design: {
          designMode: 'responsive',
        },

        onPaymentSelect: () => {
          this.amazonPayments.paymentSelected = true;
        },

        onError: this.amazonOnError,
      };

      if (this.amazonPayments.type === 'subscription') {
        walletParams.agreementType = 'BillingAgreement';

        walletParams.billingAgreementId = this.amazonPayments.billingAgreementId;
        walletParams.onReady = (billingAgreement) => {
          this.amazonPayments.billingAgreementId = billingAgreement.getAmazonBillingAgreementId();

          new this.OffAmazonPayments.Widgets.Consent({
            sellerId: AMAZON_PAYMENTS.SELLER_ID,
            amazonBillingAgreementId: this.amazonPayments.billingAgreementId,
            design: {
              designMode: 'responsive',
            },

            onReady: (consent) => {
              let getConsent = consent.getConsentStatus;
              this.amazonPayments.recurringConsent = getConsent ? getConsent() : false;
            },

            onConsent: (consent) => {
              this.amazonPayments.recurringConsent = consent.getConsentStatus();
            },

            onError: this.amazonOnError,
          }).bind('AmazonPayRecurring');
        };
      } else {
        walletParams.amazonOrderReferenceId = this.amazonPayments.orderReferenceId;
      }

      new this.OffAmazonPayments.Widgets.Wallet(walletParams).bind('AmazonPayWallet');
    },
    amazonOnError (error) {
      alert(error.getErrorMessage());
      this.reset();
    },
    reset () {
      this.amazonPaymentsmodal = null;
      this.amazonPayments.type = null;
      this.amazonLoggedIn = false;
      this.amazonPaymentsgift = null;
      this.amazonPaymentsbillingAgreementId = null;
      this.amazonPayments.orderReferenceId = null;
      this.amazonPaymentspaymentSelected = false;
      this.amazonPaymentsrecurringConsent = false;
      this.amazonPaymentssubscription = null;
      this.amazonPaymentscoupon = null;
    },
  },
};
</script>
