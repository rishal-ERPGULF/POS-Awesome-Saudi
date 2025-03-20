<template>
  <div>
    <v-autocomplete
      density="compact"
      clearable
      auto-select-first
      variant="outlined"
      color="primary"
      :label="frappe._('Customer')"
      v-model="customer"
      :items="customers"
      item-title="customer_name"
      item-value="name"
      bg-color="white"
      :no-data-text="__('Customers not found')"
      hide-details
      :customFilter="customFilter"
      :disabled="readonly"
      append-icon="mdi-plus"
      @click:append="new_customer"
      prepend-inner-icon="mdi-account-edit"
      @click:prepend-inner="edit_customer"
    >
      <template v-slot:item="{ props, item }">
        <v-list-item v-bind="props">
          <v-list-item-subtitle v-if="item.raw.customer_name != item.raw.name">
            <div v-html="`ID: ${item.raw.name}`"></div>
          </v-list-item-subtitle>
          <v-list-item-subtitle v-if="item.raw.tax_id">
            <div v-html="`TAX ID: ${item.raw.tax_id}`"></div>
          </v-list-item-subtitle>
          <v-list-item-subtitle v-if="item.raw.email_id">
            <div v-html="`Email: ${item.raw.email_id}`"></div>
          </v-list-item-subtitle>
          <v-list-item-subtitle v-if="item.raw.mobile_no">
            <div v-html="`Mobile No: ${item.raw.mobile_no}`"></div>
          </v-list-item-subtitle>
          <v-list-item-subtitle v-if="item.raw.primary_address">
            <div v-html="`Primary Address: ${item.raw.primary_address}`"></div>
          </v-list-item-subtitle>
        </v-list-item>
      </template>
    </v-autocomplete>

    <p>Selected customer: {{ customer }}</p>
    <p>Number of customers: {{ customers.length }}</p>

    <div class="mb-8">
      <UpdateCustomer />
    </div>
  </div>
</template>

<script>
import { version as vueVersion } from "vue";
import { version as vuetifyVersion } from "vuetify";
import UpdateCustomer from "./UpdateCustomer.vue";

export default {
  data: () => ({
    pos_profile: null, // Fix: Set null initially to prevent errors
    customers: [],
    customer: "",
    readonly: false,
    customer_info: {},
  }),

  components: {
    UpdateCustomer,
  },

  methods: {
    get_customer_names() {
      var vm = this;
      if (this.customers.length > 0) return;

      if (!this.pos_profile || !this.pos_profile.pos_profile) {
        console.warn("POS Profile is not set yet.");
        return; // Fix: Prevent API call if pos_profile is not ready
      }

      // Fix: Load from localStorage first to avoid unnecessary API calls
      if (localStorage.getItem("customer_storage")) {
        vm.customers = JSON.parse(localStorage.getItem("customer_storage"));
        return;
      }

      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_customer_names",
        args: {
          pos_profile: this.pos_profile.pos_profile,
        },
        callback: function (r) {
          if (r.message) {
            vm.customers = r.message;
            vm.eventBus.emit("set_all_customers", vm.customers);

            if (vm.pos_profile.posa_local_storage) {
              localStorage.setItem("customer_storage", JSON.stringify(r.message));
            }
          }
        },
      });
    },

    new_customer() {
      this.eventBus.emit("open_update_customer", null);
    },

    edit_customer() {
      this.eventBus.emit("open_update_customer", this.customer_info);
    },

    customFilter(itemText, queryText, itemRow) {
      const item = itemRow.raw;
      const searchText = queryText.toLowerCase();

      return [item.customer_name, item.tax_id, item.email_id, item.mobile_no, item.name]
        .map((text) => (text ? text.toLowerCase() : ""))
        .some((text) => text.includes(searchText));
    },
  },

  created() {
    this.$nextTick(() => {
      this.get_customer_names(); // Fix: Fetch data on component creation

      // Fix: Ensure event listeners only update pos_profile instead of blocking API calls
      this.eventBus.on("register_pos_profile", (pos_profile) => {
        this.pos_profile = pos_profile;
        this.get_customer_names();
      });

      this.eventBus.on("payments_register_pos_profile", (pos_profile) => {
        this.pos_profile = pos_profile;
        this.get_customer_names();
      });

      this.eventBus.on("set_customer", (customer) => {
        this.customer = customer;
      });

      this.eventBus.on("add_customer_to_list", (customer) => {
        this.customers.push(customer);
      });

      this.eventBus.on("set_customer_readonly", (value) => {
        this.readonly = value;
      });

      this.eventBus.on("set_customer_info_to_edit", (data) => {
        this.customer_info = data;
      });

      this.eventBus.on("fetch_customer_details", () => {
        this.get_customer_names();
      });
    });
  },

  watch: {
    customer() {
      this.eventBus.emit("update_customer", this.customer);
    },
  },
};
</script>
