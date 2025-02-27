<template>
  <div
    :class="[{ 'dark': dark }, size]"
    class="vue-phone-number-input flex"
  >
    <div class="select-country-container">
      <CountrySelector
        :id="`${id}_country_selector`"
        ref="CountrySelector"
        v-model="countryCode"
        :items="codesCountries"
        :color="color"
        :valid-color="validColor"
        :error="shouldChooseCountry"
        :hint="shouldChooseCountry ? t.countrySelectorError : null"
        :dark="dark"
        :disabled="disabled"
        :valid="isValid && !noValidatorState"
        :preferred-countries="preferredCountries"
        :only-countries="onlyCountries"
        :ignored-countries="ignoredCountries"
        :label="t.countrySelectorLabel"
        :no-flags="noFlags"
        :size="size"
        class="input-country-selector"
      />
    </div>
    <div class="flex-1">
      <VueInputUI
        :id="`${id}_phone_number`"
        ref="PhoneNumberInput"
        v-model="phoneNumber"
        :label="t.phoneNumberLabel"
        :hint="hintValue"
        :color="color"
        :valid-color="validColor"
        :dark="dark"
        :disabled="disabled"
        :size="size"
        :error="error"
        :valid="isValid && !noValidatorState"
        :required="required"
        type="tel"
        v-bind="$attrs"
        class="input-phone-number"
        @focus="$emit('phone-number-focused')"
        @blur="$emit('phone-number-blur')"
      />
    </div>
  </div>
</template>
<script>
  /* eslint-disable */
  import { countries, countriesIso } from './assets/js/phoneCodeCountries.js'
  import examples from 'libphonenumber-js/examples.mobile.json'
  import { parsePhoneNumberFromString, AsYouType, getExampleNumber } from 'libphonenumber-js'
  import VueInputUI from 'vue-input-ui'
  import 'vue-input-ui/dist/vue-input-ui.css'
  import CountrySelector from './_subs/CountrySelector'
  import locales from './assets/locales'

  const browserLocale = () => {
    if (typeof window === 'undefined') return null
    const browserLocale = window.navigator.userLanguage || window.navigator.language
    let locale = browserLocale ? browserLocale.substr(3, 4).toUpperCase() : null
    if (locale === '') locale = browserLocale.substr(0, 2).toUpperCase()
    return locale
  }

  const isCountryAvailable = (locale) => {
    return countriesIso.includes(locale)
  }

  export default {
    name: 'VuePhoneNumberInput',
    components: {
      VueInputUI,
      CountrySelector
    },
    props: {
      value: { type: String, default: null },
      id: { type: String, default: 'VuePhoneNumberInput' },
      color: { type: String, default: 'dodgerblue' },
      validColor: { type: String, default: 'yellowgreen' },
      dark: { type: Boolean, default: Boolean },
      disabled: { type: Boolean, default: Boolean },
      defaultCountryCode: { type: String, default: null },
      size: { type: String, default: String },
      preferredCountries: { type: Array, default: null },
      onlyCountries: { type: Array, default: null },
      ignoredCountries: { type: Array, default: Array },
      translations: { type: Object, default: Object },
      noValidatorState: { type: Boolean, default: false },
      noUseBrowserLocale: { type: Boolean, default: false },
      noFlags: { type: Boolean, default: false },
      error: { type: Boolean, default: false },
      noExample: { type: Boolean, default: false },
      required: { type: Boolean, default: false }
    },
    data () {
      return {
        results: {},
        focusInput: false
      }
    },
    computed: {
      t () {
        return {
          ...locales,
          ...this.translations
        }
      },
      codesCountries () {
        return countries
      },
      locale () {
        const locale = this.defaultCountryCode || (!this.noUseBrowserLocale ? browserLocale() : null)
        const countryAvailable = isCountryAvailable(locale)
        
        if (countryAvailable && locale) {
          this.countryCode = locale
        } else if (!countryAvailable && this.defaultCountryCode) {
          // If default country code is not available
          console.warn(`The locale ${locale} is not available`)
        }
        return countryAvailable ? locale : null
      },
      countryCode: {
        get () {
          return this.results.countryCode || this.locale
        },
        set (newCountry) { 
          this.emitValues({countryCode: newCountry, phoneNumber: this.phoneNumber})
          if (this.focusInput) {
            this.$refs.PhoneNumberInput.$el.querySelector('input').focus()
          }
          this.focusInput = true
        }
      },
      phoneNumber: {
        get () {
          return this.value
        },
        set (newPhone) {
          this.emitValues({countryCode: this.countryCode, phoneNumber: newPhone})
        }
      },
      shouldChooseCountry () {
        return !this.countryCode && !!this.phoneNumber
      },
      phoneFormatted () {
        return this.results.formatInternational
      },
      isValid () {
        return this.results.isValid
      },
      phoneNumberExample () {
        const phoneNumber = this.countryCode ? getExampleNumber(this.countryCode, examples) : null
        return phoneNumber ? phoneNumber.formatNational() : null
      },
      hasEmptyPhone () {
        return this.phoneNumber === '' || this.phoneNumber === null
      },
      hintValue () {
        return  this.noExample || !this.phoneNumberExample
          ? null
          : this.hasEmptyPhone || this.isValid ? null : `${this.t.example} ${this.phoneNumberExample}`
      }
    },
    methods: {
      getAsYouTypeFormat (payload) {
        const asYouType = new AsYouType(payload.countryCode)
        return asYouType.input(payload.phoneNumber)
      },
      getParsePhoneNumberFromString ({ phoneNumber, countryCode }) {
        const parsing = phoneNumber && countryCode ? parsePhoneNumberFromString(phoneNumber, countryCode) : null
        return {
          phoneNumber: phoneNumber ? phoneNumber : null,
          countryCode: countryCode,
          isValid: false,
          ...( parsing
            ? { 
              formattedNumber: parsing.number,
              nationalNumber: parsing.nationalNumber,
              isValid: parsing.isValid(),
              type: parsing.getType(),
              formatInternational: parsing.formatInternational(),
              formatNational: parsing.formatNational(),
              uri: parsing.getURI(),
              e164: parsing.format('E.164')
            }
            : null
          )
        }
      },
      emitValues (payload) {
        const asYouType = this.getAsYouTypeFormat(payload)
        this.$emit('input', asYouType)
        this.results = this.getParsePhoneNumberFromString(payload)
        this.$emit('update', this.results)
      }
    }
  }
</script>
<style lang="scss">
  @import "./assets/scss/flexbox-helper.scss";
  @import "./assets/iti-flags/flags.css";
  .vue-phone-number-input {
    *, *::before, *::after {
      box-sizing: border-box;
    }
    font-family: Roboto, -apple-system, BlinkMacSystemFont, Segoe UI, Oxygen,
        Ubuntu, Cantarell, Fira Sans, Droid Sans, Helvetica Neue, sans-serif;
    .select-country-container {
      flex: 0 0 120px;
      width: 120px;
      min-width: 120px;
      max-width: 120px;
    }
    &.sm .select-country-container {
      flex: 0 0 110px;
      width: 110px;
      min-width: 110px;
      max-width: 110px;
    }
    &.lg .select-country-container {
      flex: 0 0 130px;
      width: 130px;
      min-width: 130px;
      max-width: 130px;
    }
    .country-selector {
      cursor: pointer;
    }
    .select-country-container {
      .input-country-selector input {
        border-top-right-radius: 0 !important; 
        border-bottom-right-radius: 0 !important;
      }
    }
    .input-phone-number input {
      margin-left: -3px !important;
      border-top-left-radius: 0 !important;
      border-bottom-left-radius: 0 !important;
    }
    .input-phone-number:not(.is-dark):not(.is-disabled) {
      input {
        background-color: transparent !important;
      }
    }
  }
</style>
