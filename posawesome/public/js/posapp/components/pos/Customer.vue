<template>
  <div>
    <v-autocomplete
      id="customer-autocomplete"
      dense
      clearable
      color="primary"
      auto-select-first
      outlined
      :label="frappe._('Customer')"
      v-model="customer"
      :items="customers"
      item-text="customer_name"
      item-value="name"
      background-color="white"
      :no-data-text="__('Customer not found')"
      hide-details
      :filter="customFilter"
      :disabled="readonly"
      append-icon="mdi-plus"
      @click:append="new_customer"
      prepend-inner-icon="mdi-account-edit"
      @click:prepend-inner="edit_customer"
    >
       <template v-slot:item="data">
        <template>
          <v-list-item-content>
            <v-list-item-title
              class="primary--text subtitle-1"
              v-html="data.item.customer_name"
            ></v-list-item-title>
            <v-list-item-subtitle
              v-if="data.item.customer_name != data.item.name"
              v-html="`ID: ${data.item.name}`"
            ></v-list-item-subtitle>
            <v-list-item-subtitle
              v-if="data.item.tax_id"
              v-html="`TAX ID: ${data.item.tax_id}`"
            ></v-list-item-subtitle>
            <v-list-item-subtitle
              v-if="data.item.email_id"
              v-html="`Email: ${data.item.email_id}`"
            ></v-list-item-subtitle>
            <v-list-item-subtitle
              v-if="data.item.mobile_no"
              v-html="`Mobile No: ${data.item.mobile_no}`"
            ></v-list-item-subtitle>
            <v-list-item-subtitle
              v-if="data.item.primary_address"
              v-html="`Primary Address: ${data.item.primary_address}`"
            ></v-list-item-subtitle>
          </v-list-item-content>
        </template>
      </template>
    </v-autocomplete>

    <div class="mb-8">
      <UpdateCustomer></UpdateCustomer>
    </div>

    <v-autocomplete
      id="membershipcard-autocomplete"
      dense
      clearable
      auto-select-first
      outlined
      :label="frappe._('Membership Card')"
      v-model="membershipcard"
      :items="membershipcards"
      item-text="name"
      item-value="name"
      background-color="white"
      :no-data-text="__('Membership Card not found')"
      hide-details
      :filter="customMemberFilter"
      :disabled="readonly"
      append-icon="mdi-plus"
      @click:append="new_membership"
  
    >
      <template v-slot:item="data">
        <v-list-item-content>
          <v-list-item-title
            class="primary--text subtitle-1"
            v-html="data.item.name"
          ></v-list-item-title>
          <v-list-item-subtitle
            v-if="data.item.valid_from"
            v-html="`Valid From: ${data.item.valid_from}`"
          ></v-list-item-subtitle>
          <v-list-item-subtitle
            v-if="data.item.valid_upto"
            v-html="`Valid Upto: ${data.item.valid_upto}`"
          ></v-list-item-subtitle>
          <v-list-item-subtitle
            v-if="data.item.customer"
            v-html="`Customer: ${data.item.customer}`"
          ></v-list-item-subtitle>
        <v-list-item-subtitle
            v-if="data.item.description"
            v-html="`Description: ${data.item.description}`"
          ></v-list-item-subtitle>
          <v-list-item-subtitle
            v-if="data.item.max_use"
            v-html="` Max Use : ${data.item.max_use} | Used : ${data.item.used}`"
          ></v-list-item-subtitle>
        </v-list-item-content>
      </template>
    </v-autocomplete>

    <div class="mb-8">
      <Member></Member>
    </div>
  </div>
</template>


<script>
import { evntBus } from '../../bus';
import UpdateCustomer from './UpdateCustomer.vue';
import Member from './Member.vue';

export default {
  data: () => ({
    pos_profile: '',
    customers: [],
    customer: '',
    membershipcards: [],
    membershipcard: '',
    readonly: false,
    customer_info: {},
  }),

  components: {
    UpdateCustomer,
    Member
  },

  methods: {
    get_customer_names() {
      const vm = this;
      if (this.customers.length > 0) {
        return;
      }
      if (vm.pos_profile.posa_local_storage && localStorage.customer_storage) {
        vm.customers = JSON.parse(localStorage.getItem('customer_storage'));
      }
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_customer_names',
        args: {
          pos_profile: this.pos_profile.pos_profile,
        },
        callback: function (r) {
          if (r.message) {
            vm.customers = r.message;
            if (vm.pos_profile.posa_local_storage) {
              localStorage.setItem('customer_storage', JSON.stringify(r.message));
            }
          }
        },
      });
    },
    get_membership_card_names() {
      const vm = this;
      if(this.customer == ""){
        if (vm.pos_profile.posa_local_storage && localStorage.membership_storage) {
          vm.membershipcards = JSON.parse(localStorage.getItem('membership_storage'));
        }
      }
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_membership_card_names',
        args: {
          pos_profile: vm.pos_profile.pos_profile,
          customer: vm.customer
        },
        callback: function (r) {
          if (r.message) {
            vm.membershipcards = r.message;
            if (vm.pos_profile.posa_local_storage && this.customer == "") {
              localStorage.setItem('membership_storage', JSON.stringify(r.message));
            }
          }
        },
      });
    },
    new_customer() {
      evntBus.$emit('open_update_customer', null);
    },
    new_membership() {
      evntBus.$emit('open_member', null);
    },


    edit_customer() {
      evntBus.$emit('open_update_customer', this.customer_info);
    },
    customFilter(item, queryText, itemText) {
      const textOne = item.customer_name ? item.customer_name.toLowerCase() : '';
      const textTwo = item.tax_id ? item.tax_id.toLowerCase() : '';
      const textThree = item.email_id ? item.email_id.toLowerCase() : '';
      const textFour = item.mobile_no ? item.mobile_no.toLowerCase() : '';
      const textFifth = item.name.toLowerCase();
      const searchText = queryText.toLowerCase();

      return (
        textOne.indexOf(searchText) > -1 ||
        textTwo.indexOf(searchText) > -1 ||
        textThree.indexOf(searchText) > -1 ||
        textFour.indexOf(searchText) > -1 ||
        textFifth.indexOf(searchText) > -1
      );
    },
    customMemberFilter(item, queryText, itemText) {
      const textOne = item.customer ? item.customer.toLowerCase() : '';
      const textTwo = item.unique_code ? item.unique_code.toLowerCase() : '';
      const searchText = queryText.toLowerCase();

      return (
        textOne.indexOf(searchText) > -1 ||
        textTwo.indexOf(searchText) > -1
      );
    },
  },

  created() {
    this.$nextTick(function () {
      evntBus.$on('register_pos_profile', (pos_profile) => {
        this.pos_profile = pos_profile;
        this.get_customer_names();
        this.get_membership_card_names();
      });
      evntBus.$on('payments_register_pos_profile', (pos_profile) => {
        this.pos_profile = pos_profile;
        this.get_customer_names();
        this.get_membership_card_names();
      });
     evntBus.$on('set_customer', (customer) => {
        this.customer = customer;
      });
      evntBus.$on('set_membershipcard', (membershipcard) => {
        this.membershipcard = membershipcard;
      });
      evntBus.$on('add_customer_to_list', (customer) => {
        this.customers.push(customer);
      });
      evntBus.$on('add_member_to_list', (membershipcard) => {
        this.membershipcards.push(membershipcard);
      });
      evntBus.$on('add_customer_to_list_membership', (customer) => {
        this.customers.push(customer);
      });
      evntBus.$on('set_customer_readonly', (value) => {
        this.readonly = value;
      });
      evntBus.$on('set_customer_info_to_edit', (data) => {
        this.customer_info = data;
      });
      evntBus.$on('fetch_customer_details', () => {
        this.get_customer_names();
        this.get_membership_card_names();
      });
    });
  },

  watch: {
    customer() {
      evntBus.$emit('update_customer', this.customer);
      this.get_membership_card_names()
    },
     membershipcard() {
      evntBus.$emit('update_membershipcard', this.membershipcard);
    },
  },
  
};
</script>
