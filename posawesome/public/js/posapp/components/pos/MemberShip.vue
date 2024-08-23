<template>
  <div>
    <v-card
      class="selection mx-auto grey lighten-5"
      style="max-height: 80vh; height: 80vh"
    >
      <v-card-title>
        <v-row no-gutters align="center" justify="center">
          <v-col cols="6">
            <v-text-field
              dense
              outlined
              color="primary"
              :label="frappe._('Membership ID')"
              background-color="white"
              hide-details
              v-model="new_membership"
              class="mr-4"
            ></v-text-field>
          </v-col>
          <v-col cols="2">
            <v-btn
              class="pa-1"
              color="info"
              dark
              @click="get_member_details(new_membership)"
              >{{ __("Details") }}</v-btn
            >
          </v-col>
          <v-col cols="2">
            <v-btn
              class="pa-1"
              color="success"
              dark
              @click="add_member(new_membership)"
              >{{ __("Apply") }}</v-btn
            >
          </v-col>
        </v-row>
      </v-card-title>
      <div class="my-0 py-0 overflow-y-auto" style="max-height: 75vh">
        <span v-if="member_cus_name">
          Customer Name: {{ member_cus_name }} <br />
          <!-- Total Discount Amount: {{ total_dis_amt }} <br />
          Available Discount Amount: {{ available_dis_amt }} <br /> -->
          Used Discount Amount: {{ used_discount_amount }} <br />
          Max Use: {{ max_use }} <br />
          Valid From: {{ start_date }} <br />
          Valid Upto: {{ end_date }}</span
        >
      </div>
      <div class="my-0 py-0 overflow-y-auto" style="max-height: 75vh">
        <template @mouseover="style = 'cursor: pointer'">
          <v-data-table
            :headers="items_headers"
            :items="member_list"
            :single-expand="singleExpand"
            :expanded.sync="expanded"
            item-key="coupon"
            class="elevation-1"
            :items-per-page="itemsPerPage"
            hide-default-footer
          >
            <template v-slot:item.add="{ item }">
              <v-icon class="me-2" size="large" @click="addMemberID(item)">
                mdi-plus-circle-outline
              </v-icon>
            </template>
          </v-data-table>
        </template>
      </div>
    </v-card>

    <v-card
      flat
      style="max-height: 11vh; height: 11vh"
      class="cards mb-0 mt-3 py-0"
    >
      <v-row align="start" no-gutters>
        <v-col cols="12">
          <v-btn
            block
            class="pa-1"
            large
            color="warning"
            dark
            @click="back_to_invoice"
            >{{ __("Back") }}</v-btn
          >
        </v-col>
      </v-row>
    </v-card>
  </div>
</template>

<script>
import { evntBus } from "../../bus";
export default {
  data: () => ({
    loading: false,
    pos_profile: "",
    customer: "",
    new_membership: "",
    member_cus_name: "",
    total_dis_amt: "",
    available_dis_amt: "",
    start_date: "",
    end_date: "",
    used_discount_amount: "",
    max_use: "",
    member_list: [],
    itemsPerPage: 1000,
    items_headers: [
      { text: __("ID"), value: "name", align: "start" },
      { text: __("Discount Amount"), value: "discount_amount", align: "start" },
      { text: __("Valid Upto"), value: "valid_upto", align: "start" },
      { text: __("Add"), value: "add", align: "start" },
    ],
  }),

  computed: {},

  methods: {
    addMemberID(item) {
      this.member_cus_name = "";
      this.new_membership = item.name;
    },

    get_member_details(new_membership) {
      var vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_member_details",
        args: {
          membership: new_membership,
        },
        callback: function (r) {
          if (r.message) {
            (vm.member_cus_name = r.message.customer),
              (vm.total_dis_amt = r.message.max_discount_amount),
              (vm.used_discount_amount = r.message.used_discount_amount),
              (vm.max_use = r.message.max_use),
              (vm.start_date = r.message.valid_from),
              (vm.end_date = r.message.valid_upto),
              (vm.available_dis_amt =
                r.message.max_discount_amount - r.message.used_discount_amount);
          } else {
            evntBus.$emit("show_mesage", {
              text: __("Membership ID not found."),
              color: "error",
            });
          }
        },
      });
    },
    back_to_invoice() {
      evntBus.$emit("show_membership_card", "false");
    },

    add_member(new_membership) {
      var vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_member_details",
        args: {
          membership: new_membership,
        },
        callback: function (r) {
          if (r.message) {
            vm.available_dis_amt =
              r.message.max_discount_amount - r.message.used_discount_amount;
            evntBus.$emit(
              "set_membership_card_discount",
              vm.available_dis_amt,
              r.message,

            );
          } else {
            evntBus.$emit("show_mesage", {
              text: __("Membership ID not found."),
              color: "error",
            });
          }
        },
      });
    },

    get_member_ship_details(customer) {
      var vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_member_list_details",
        args: {
          customer: customer,
        },
        callback: function (r) {
          if (r.message) {
            vm.member_list = r.message;
          }
        },
      });
    },
  },

  watch: {},

  created: function () {},
  mounted: function () {
    this.$nextTick(function () {
      evntBus.$on("mem_send_cust_name", (customer) => {
        this.new_membership = "";
        this.member_cus_name = "";
        this.get_member_ship_details(customer);
      });
    });
  },
};
</script>
