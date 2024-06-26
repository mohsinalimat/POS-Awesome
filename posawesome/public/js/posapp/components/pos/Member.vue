<template>
  <v-row justify="center">
    <v-dialog
      v-model="membershipcardDialog"
      max-width="600px"
      @click:outside="clear_customer"
    >
      <v-card>
        <v-card-title>
          <span v-if="membershipcard_id" class="headline primary--text">{{
            __('Update Customer')
          }}</span>
          <span v-else class="headline primary--text">{{
            __('Create Membership Card')
          }}</span>
        </v-card-title>
        <v-card-text class="pa-0">
          <v-container>
            <v-row>
              <v-col cols="6">
                <v-menu
                  ref="valid_from_menu"
                  v-model="valid_from_menu"
                  :close-on-content-click="false"
                  transition="scale-transition"
                  dense
                >
                  <template v-slot:activator="{ on, attrs }">
                    <v-text-field
                      v-model="valid_from"
                      :label="frappe._('Valid From') + ' *'"
                      readonly
                      dense
                      clearable
                      hide-details
                      v-bind="attrs"
                      v-on="on"
                      color="primary"
                    ></v-text-field>
                  </template>
                  <v-date-picker
                    v-model="valid_from"
                    color="primary"
                    no-title
                    scrollable
                    @input="valid_from_menu = false"
                  ></v-date-picker>
                </v-menu>
              </v-col>

              <v-col cols="12">
                <v-text-field
                  dense
                  color="primary"
                  :label="frappe._('Valid Upto') + ' *'"
                  background-color="white"
                  hide-details
                  v-model="valid_upto"
                  readonly
                ></v-text-field>
              </v-col>

              <v-col cols="12">
                <v-autocomplete
                  id="customer-autocomplete"
                  dense
                  clearable
                  auto-select-first
                  :label="frappe._('Customer') + ' *'"
                  v-model="customer"
                  :items="customers"
                  item-text="customer_name"
                  item-value="name"
                  background-color="white"
                  :no-data-text="__('Customer not found')"
                  hide-details
                  :disabled="readonly"
                ></v-autocomplete>
              </v-col>
              <v-col cols="12">
                <v-text-field
                  dense
                  color="primary"
                  :label="frappe._('Description') + ' *'"
                  background-color="white"
                  hide-details
                  v-model="description"
                  :disabled="readonly"
                ></v-text-field>
              </v-col>
            </v-row>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="error" dark @click="close_dialog">{{
            __('Close')
          }}</v-btn>
          <v-btn color="success" dark @click="submit_dialog">{{
            __('Submit')
          }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import { evntBus } from '../../bus';

export default {
  data() {
    return {
      membershipcardDialog: false,
      pos_profile: '',
      membershipcard_id: '',
      valid_from: '',
      valid_upto: '',
      description:'',
      customer: '',
      customers: [],
      readonly: false,
      items: [], // Define and initialize the items array
      rate: 0,
    };
  },
  watch: {
    valid_from(newVal) {
      if (newVal) {
        const validFromDate = new Date(newVal);
        const validUptoDate = new Date(validFromDate.setFullYear(validFromDate.getFullYear() + 2));
        this.valid_upto = validUptoDate.toISOString().substr(0, 10);
      } else {
        this.valid_upto = '';
      }
    },
  },
  methods: {
        calc_item_price(item) {
      if (!item.posa_offer_applied) {
        if (item.price_list_rate) {
          item.rate = item.price_list_rate;
        }
      }
      if (item.discount_percentage) {
        item.rate =
          flt(item.price_list_rate) -
          (flt(item.price_list_rate) * flt(item.discount_percentage)) / 100;
        item.discount_amount = this.flt(
          flt(item.price_list_rate) - flt(item.rate),
          this.currency_precision
        );
      } else if (item.discount_amount) {
        item.rate = this.flt(
          flt(item.price_list_rate) - flt(item.discount_amount),
          this.currency_precision
        );
      }
    },
     get_price_list() {
      let price_list = this.pos_profile.selling_price_list;
      if (this.customer_info && this.pos_profile) {
        const { customer_price_list, customer_group_price_list } =
          this.customer_info;
        const pos_price_list = this.pos_profile.selling_price_list;
        if (customer_price_list && customer_price_list != pos_price_list) {
          price_list = customer_price_list;
        } else if (
          customer_group_price_list &&
          customer_group_price_list != pos_price_list
        ) {
          price_list = customer_group_price_list;
        }
      }
      return price_list;
    },

    get_payments() {
      const payments = [];
      this.pos_profile.payments.forEach((payment) => {
        payments.push({
          amount: 0,
          mode_of_payment: payment.mode_of_payment,
          default: payment.default,
          account: "",
        });
      });
      return payments;
    },
    get_invoice_doc() {
      // Define and return your invoice doc structure here
      let doc = {
        doctype: "Sales Invoice",
        is_pos: 1,
        membership_card: this.membershipcard,
        ignore_pricing_rule: 1,
        company: this.pos_profile.company || '',
        pos_profile: this.pos_profile.name || '',
        campaign: this.pos_profile.campaign || '',
        currency: this.pos_profile.currency || '',
        naming_series: this.pos_profile.naming_series || '',
        customer: this.customer || '',
        items: this.items || [], // Assuming this is an array in your data
        total: this.subtotal || 0,
        discount_amount: this.discount_amount || 0,
        additional_discount_percentage: this.additional_discount_percentage || 0,
        payments: this.get_payments() || [], // Assuming this is a function returning an array
        taxes: [],
        posting_date: this.posting_date || '',
        warehouse: this.pos_profile.warehouse || '',
      };

      return doc;
    },
    fetchCustomers() {
      if (!this.pos_profile) {
        return;
      }

      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_customer_names_list',
        args: {
          pos_profile: JSON.stringify(this.pos_profile)
        },
        callback: (r) => {
          if (!r.exc) {
            this.customers = r.message;
          }
        },
      });
    },
    close_dialog() {
      this.membershipcardDialog = false;
    },
    clear_customer() {
      this.valid_from = '';
      this.valid_upto = '';
      this.customer = '';
    },
    submit_dialog() {
      if (!this.valid_from) {
        evntBus.$emit('show_mesage', {
          text: __('Valid from required.'),
          color: 'error',
        });
        return;
      }
      if (!this.customer) {
        evntBus.$emit('show_mesage', {
          text: __('Customer required.'),
          color: 'error',
        });
        return;
      }
      const args = {
        valid_from: this.valid_from,
        customer: this.customer,
        valid_upto: this.valid_upto,
        pos_profile_doc: this.pos_profile,
        description:this.description
      };
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.create_membership_card',
        args: args,
        callback: (r) => {
          if (!r.exc && r.message.name) {
            let text = __('Membership Card created successfully.');
            evntBus.$emit('show_mesage', {
              text: text,
              color: 'success',
            });
            args.name = r.message.name;
            frappe.utils.play_sound('submit');
            evntBus.$emit('add_member_to_list', args);
            evntBus.$emit('add_customer_to_list_membership', args); // Ensure this event is emitted
            evntBus.$emit('set_membershipcard', r.message.name);

            // Call add_item with static item details
           frappe.call({
              method: "posawesome.posawesome.api.posapp.get_item_price_list",
              args: { item_code: "MEMBERSHIP" },
              callback: function (r) {
                   if (r.message && Array.isArray(r.message) && r.message.length > 0) {
                      let item_details = r.message[0]; // Access the first item in the list
                     
                      item = { item_name : "MEMBERSHIP",rate:item_details.price_list_rate,uom: item_details.uom};
                      if (item.has_variants) {
                        evntBus.$emit("open_variants_model", item, this.items);
                      } else {
                        if (!item.qty || item.qty === 1) {
                          item.qty = Math.abs(this.qty);
                          item.item_code="MEMBERSHIP"
                        }
                        evntBus.$emit("add_item", item);
                        this.qty = 1;
                        this.item_code = "MEMBERSHIP";
                      }
                  } 
              }
            });

            this.close_dialog();
          } else {
            frappe.utils.play_sound('error');
            evntBus.$emit('show_mesage', {
              text: __('Membership Card creation failed.'),
              color: 'error',
            });
          }
        },
      });
      this.membershipcardDialog = false;
    },
   add_item(item_name, rate, uom) {
    const item = {
      item_name: "MEMBERSHIP",
      rate: rate,
      uom: uom,
      item_code: "MEMBERSHIP",
      qty: 1,
      stock_uom: uom,
      has_serial_no: false,
      has_batch_no: false,
      posa_is_offer: false,
      posa_is_replace: false,
    };

    // Check if items array is defined and initialized
    if (!this.items) {
      this.items = [];
    }

    let index = -1;
    if (!this.new_line) {
      index = this.items.findIndex(
        (el) =>
          el.item_code === item.item_code &&
          el.uom === item.uom &&
          !el.posa_is_offer &&
          !el.posa_is_replace
      );
    }

    if (index === -1 || this.new_line) {
     
      if (item.has_serial_no && item.to_set_serial_no) {
         item.serial_no_selected = [];
         item.serial_no_selected.push(item.to_set_serial_no);
         item.to_set_serial_no = null;
       }
       if (item.has_batch_no && item.to_set_batch_no) {
         item.batch_no = item.to_set_batch_no;
         item.to_set_batch_no = null;
         item.batch_no = null;
         this.set_batch_qty(item, item.batch_no, false);
       }
      this.items.unshift(item);
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_item_detail",
        args: {
          warehouse: this.pos_profile.warehouse,
          doc: this.get_invoice_doc(),
          price_list: this.pos_profile.price_list,
          item: {
            item_code: "MEMBERSHIP",
            customer: this.customer,
            
            doctype: "Sales Invoice",
            membership_card: this.membershipcard,
            name: "New Sales Invoice 1",
            company: this.pos_profile.company,
            conversion_rate: 1,
            qty: item.qty,
            income_account:"Service - BV",
            price_list_rate: item.price_list_rate,
            child_docname: "New Sales Invoice Item 1",
            cost_center: this.pos_profile.cost_center,
            currency: this.pos_profile.currency,
            // plc_conversion_rate: 1,
            pos_profile: this.pos_profile.name,
            uom: item.uom,
            tax_category: "",
            transaction_type: "selling",
            update_stock: this.pos_profile.update_stock,
            price_list: "Standard Selling",
            has_batch_no: item.has_batch_no,
            serial_no: item.serial_no,
            batch_no: item.batch_no,
            is_stock_item: item.is_stock_item,
          },
        },
        callback: function (r) {
          if (r.message) {
            const data = r.message;
            if (data.batch_no_data) {
              item.batch_no_data = data.batch_no_data;
            }
            if (
              item.has_batch_no &&
              vm.pos_profile.posa_auto_set_batch &&
              !item.batch_no &&
              data.batch_no_data
            ) {
              item.batch_no_data = data.batch_no_data;
              vm.set_batch_qty(item, item.batch_no, false);
            }
            if (data.has_pricing_rule) {
            } else if (
              vm.pos_profile.posa_apply_customer_discount &&
              vm.customer_info.posa_discount > 0 &&
              vm.customer_info.posa_discount <= 100
            ) {
              if (
                item.posa_is_offer == 0 &&
                !item.posa_is_replace &&
                item.posa_offer_applied == 0
              ) {
                if (item.max_discount > 0) {
                  item.discount_percentage =
                    item.max_discount < vm.customer_info.posa_discount
                      ? item.max_discount
                      : vm.customer_info.posa_discount;
                } else {
                  item.discount_percentage = vm.customer_info.posa_discount;
                }
              }
            }
            if (!item.batch_price) {
              if (
                !item.is_free_item &&
                !item.posa_is_offer &&
                !item.posa_is_replace
              ) {
                item.price_list_rate = data.price_list_rate;
              }
            }
            item.last_purchase_rate = data.last_purchase_rate;
            item.projected_qty = data.projected_qty;
            item.reserved_qty = data.reserved_qty;
            item.conversion_factor = data.conversion_factor;
            item.stock_qty = data.stock_qty;
            item.actual_qty = data.actual_qty;
            item.stock_uom = data.stock_uom;
            item.item_code = "MEMBERSHIP";
            
            vm.calc_item_price(item);
          }
          
          evntBus.$emit("add_item", item);
          this.qty = 1;
        },
      });
    } 
    this.$forceUpdate();
  },
  },
 
  created() {
    this.fetchCustomers();
    evntBus.$on('open_member', (data) => {
      this.membershipcardDialog = true;
      if (data) {
        this.valid_from = data.valid_from;
        this.customer = data.customer;
        this.description = data.description;
        if (this.valid_from) {
          this.$watch('valid_from')(this.valid_from);
        }
      }
    });
    evntBus.$on('register_pos_profile', (data) => {
      this.pos_profile = data.pos_profile;
      this.fetchCustomers(); // Fetch customers when pos_profile is registered
    });
    evntBus.$on('payments_register_pos_profile', (data) => {
      this.pos_profile = data.pos_profile;
    });
    evntBus.$on('add_customer_to_list_membership', (customer) => {
      this.customers.push(customer);
    });

    // Add the event listener for adding items
   
  },
};
</script>

