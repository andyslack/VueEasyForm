<template>
    <div class="vue-country-select">
        <div
            class="dropdown"
            @click="toggleDropdown"
            v-click-outside="clickedOutside"
            :class="{ open: open }"
            @keydown="keyboardNav"
            tabindex="0"
            @keydown.esc="reset"
        >
      <span class="current">
        <div
            v-if="enabledFlags"
            class="iti-flag"
            :class="activeCountry.iso2.toLowerCase()"
        ></div>
        <span v-if="enabledCountryCode" class="country-code"
        >+{{ activeCountry.dialCode }}</span
        >
        <span class="dropdown-arrow">{{ open ? "▲" : "▼" }}</span>
      </span>
            <ul v-show="open" ref="list" class="dropdown-list">

                <li
                    v-if="enableSearchField"
                    class="search-field"
                    @click.stop
                >
                    <input
                        v-model="searchText"
                        :placeholder="searchPlaceholderText"
                        type="text"
                    >
                </li>

                <li
                    class="dropdown-item"
                    v-for="(pb, index) in sortedCountries"
                    :key="pb.iso2 + (pb.preferred ? '-preferred' : '')"
                    @click="choose(pb)"
                    :class="getItemClass(index, pb.iso2)"
                    @mousemove="selectedIndex = index"
                >
                    <div
                        class="iti-flag"
                        v-if="enabledFlags"
                        :class="pb.iso2.toLowerCase()"
                    ></div>
                    <strong>{{ pb.name }}</strong>
                    <span v-if="dropdownOptions && !dropdownOptions.disabledDialCode"
                    >+{{ pb.dialCode }}</span
                    >
                </li>
            </ul>
        </div>
    </div>
</template>

<script>
import allCountries from "./utils/allCountries";

export default {
    name: "country_dropdown",
    props: {
        searchPlaceholderText: {
            type: String,
            default: 'Search country'
        },
        enableSearchField: {
            type: Boolean,
            default: false
        },
        disabledFormatting: {
            type: Boolean,
            default: false
        },
        defaultCountry: {
            // Default country code, ie: 'AU'
            // Will override the current country of user
            type: String,
            default: ""
        },
        enabledCountryCode: {
            type: Boolean,
            default: false
        },
        enabledFlags: {
            type: Boolean,
            default: true
        },
        preferredCountries: {
            type: Array,
            default: () => []
        },
        onlyCountries: {
            type: Array,
            default: () => []
        },
        ignoredCountries: {
            type: Array,
            default: () => []
        },
        dropdownOptions: {
            type: Object,
            default: () => ({})
        },
        selectedCountryCode: {
            type: Boolean,
            default: false
        }
    },
    mounted() {
        this.initializeCountry();
        this.$emit("onSelect", this.activeCountry);
    },
    data() {
        return {
            activeCountry: { iso2: "" },
            open: false,
            selectedIndex: null,
            typeToFindInput: "",
            typeToFindTimer: null,
            searchText: ''
        };
    },
    computed: {
        filteredCountries() {
            // List countries after filtered

            if(this.searchText.length){
                return allCountries.filter(country => {
                    return country.name.toLowerCase().includes(this.searchText.toLowerCase())
                });
            }

            if (this.onlyCountries.length) {
                return this.getCountries(this.onlyCountries);
            }

            if (this.ignoredCountries.length) {
                return allCountries.filter(
                    ({ iso2 }) =>
                        !this.ignoredCountries.includes(iso2.toUpperCase()) &&
                        !this.ignoredCountries.includes(iso2.toLowerCase())
                );
            }

            return allCountries;
        },
        sortedCountries() {
            // Sort the list countries: from preferred countries to all countries
            const preferredCountries = this.getCountries(
                this.preferredCountries
            ).map(country => ({ ...country, preferred: true }));

            return [...preferredCountries, ...this.filteredCountries];
        }
    },
    watch: {
        defaultCountry: function(){
            this.initializeCountry();
        }
    },
    methods: {
        initializeCountry() {
            /**
             * 1. Use default country if passed from parent
             */
            if (this.defaultCountry) {
                const defaultCountry = this.findCountry(this.defaultCountry);
                if (defaultCountry) {
                    this.activeCountry = defaultCountry;
                    return;
                }
            }
            /**
             * 2. Use the first country from preferred list (if available) or all countries list
             */
            this.activeCountry =
                this.findCountry(this.preferredCountries[0]) ||
                this.filteredCountries[0];
        },
        /**
         * Get the list of countries from the list of iso2 code
         */
        getCountries(list = []) {
            return list
                .map(countryCode => this.findCountry(countryCode))
                .filter(Boolean);
        },
        findCountry(iso = "") {
            return allCountries.find(country => country.iso2 === iso.toUpperCase());
        },
        getItemClass(index, iso2) {
            const highlighted = this.selectedIndex === index;
            const lastPreferred = index === this.preferredCountries.length - 1;
            const preferred = !!~this.preferredCountries
                .map(c => c.toUpperCase())
                .indexOf(iso2);
            return {
                highlighted,
                "last-preferred": lastPreferred,
                preferred
            };
        },
        choose(country) {
            this.activeCountry = country;
            this.$emit("onSelect", this.activeCountry);
        },
        toggleDropdown() {
            if (this.disabled) {
                return;
            }
            this.open = !this.open;
        },
        keyboardNav(e) {
            if (e.keyCode === 40) {
                // down arrow
                this.open = true;
                if (this.selectedIndex === null) {
                    this.selectedIndex = 0;
                } else {
                    this.selectedIndex = Math.min(
                        this.sortedCountries.length - 1,
                        this.selectedIndex + 1
                    );
                }
                let selEle = this.$refs.list.children[this.selectedIndex];
                if (
                    selEle.offsetTop + selEle.clientHeight >
                    this.$refs.list.scrollTop + this.$refs.list.clientHeight
                )
                    this.$refs.list.scrollTop =
                        selEle.offsetTop -
                        this.$refs.list.clientHeight +
                        selEle.clientHeight;
            } else if (e.keyCode === 38) {
                // up arrow
                this.open = true;
                if (this.selectedIndex === null) {
                    this.selectedIndex = this.sortedCountries.length - 1;
                } else {
                    this.selectedIndex = Math.max(0, this.selectedIndex - 1);
                }
                let selEle = this.$refs.list.children[this.selectedIndex];
                if (selEle.offsetTop < this.$refs.list.scrollTop)
                    this.$refs.list.scrollTop = selEle.offsetTop;
            } else if (e.keyCode === 13) {
                // enter key
                if (this.selectedIndex !== null) {
                    this.choose(this.sortedCountries[this.selectedIndex]);
                }
                this.open = !this.open;
            } else {
                // typing a country's name
                this.typeToFindInput += e.key;
                clearTimeout(this.typeToFindTimer);
                this.typeToFindTimer = setTimeout(() => {
                    this.typeToFindInput = "";
                }, 700);
                // don't include preferred countries so we jump to the right place in the alphabet
                let typedCountryI = this.sortedCountries
                    .slice(this.preferredCountries.length)
                    .findIndex(c =>
                        c.name.toLowerCase().startsWith(this.typeToFindInput)
                    );
                if (~typedCountryI) {
                    this.selectedIndex = this.preferredCountries.length + typedCountryI;
                    let selEle = this.$refs.list.children[this.selectedIndex];
                    if (
                        selEle.offsetTop < this.$refs.list.scrollTop ||
                        selEle.offsetTop + selEle.clientHeight >
                        this.$refs.list.scrollTop + this.$refs.list.clientHeight
                    ) {
                        this.$refs.list.scrollTop =
                            selEle.offsetTop - this.$refs.list.clientHeight / 2;
                    }
                }
            }
        },
        reset() {
            this.selectedIndex = this.sortedCountries
                .map(c => c.iso2)
                .indexOf(this.activeCountry.iso2);
            this.open = false;
        }
    },
};
</script>

<style src="./assets/sprite.css"></style>
<style>
/* TODO: Find the right way to resolve alias in style block */
/* @import url("~@/assets/sprite.css"); */
.vue-country-select {
    border-radius: 3px;
    display: inline-block;
    border: 1px solid #bbb;
    text-align: left;
}

.vue-country-select .search-field{
    padding: 10px;
}

.vue-country-select .search-field input{
    width: 100%;
    border: 1px solid black;
    border-radius: 4px;
    padding-left: 8px;
}

.vue-country-select:focus-within {
    box-shadow: inset 0 1px 1px rgba(0, 0, 0, 0.075),
    0 0 8px rgba(102, 175, 233, 0.6);
    border-color: #66afe9;
}
.vue-country-select .dropdown {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100%;
    padding: 0.5em;
    position: relative;
    cursor: pointer;
}
.vue-country-select .dropdown.open {
    background-color: #f3f3f3;
}
.vue-country-select .dropdown:hover {
    background-color: #f3f3f3;
}
.vue-country-select .dropdown-list {
    z-index: 1;
    padding: 0;
    margin: 0;
    text-align: left;
    list-style: none;
    max-height: 200px;
    overflow-y: scroll;
    position: absolute;
    top: 100%;
    left: -1px;
    background-color: #fff;
    border: 1px solid #ccc;
    width: 390px;
}
.vue-country-select .dropdown-item {
    cursor: pointer;
    padding: 4px 15px;
}
.vue-country-select .dropdown-item .iti-flag {
    display: inline-block;
    margin-right: 5px;
    margin-left: 5px;
}
.vue-country-select .dropdown-item.highlighted {
    background-color: #f3f3f3;
}
.vue-country-select .dropdown-item.last-preferred {
    border-bottom: 1px solid #cacaca;
}
.vue-country-select .dropdown-arrow {
    transform: scaleY(0.5);
    display: inline-block;
    color: #666;
}
.vue-country-select .current {
    font-size: 0.8em;
    display: flex;
    align-items: center;
}
.vue-country-select .country-code {
    color: #666;
}
.vue-country-select.disabled .current,
.vue-country-select.disabled .dropdown {
    cursor: no-drop;
}
</style>
