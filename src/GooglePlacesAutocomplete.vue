<template>
    <div>
        <slot
            name="input"
            :context="context"
            :actions="{ selectItemFromList, shiftResultsSelection, unshiftResultsSelection }"
            :events="{ inputHasReceivedFocus, inputHasChanged }"
        >
            <input
                type="search"
                v-model="context.input"
                @focus="inputHasReceivedFocus"
                @input="inputHasChanged"
                @keydown.enter.prevent="selectItemFromList"
                @keydown.down.prevent="shiftResultsSelection"
                @keydown.up.prevent="unshiftResultsSelection"
                class="vbga-input"
            >
        </slot>
        <ul v-if="hasResults" class="vbga-results">
            <li
                v-for="(result, index) in autocomplete.results"
                :class="{ highlighted: index === autocomplete.resultsHighlight }"
                :key="result.id"
                @click="resultHasBeenSelected(result)"
            >
                <slot name="item" :place="result" v-if="index !== autocomplete.resultsHighlight">
                    {{ result.description }}
                </slot>
                <slot name="activeItem" :place="result" v-if="index === autocomplete.resultsHighlight">
                    {{ result.description }}
                </slot>
            </li>
        </ul>
    </div>
</template>

<script>
export default {

    name: "GooglePlacesAutocomplete",

    data() {
        return {
            autocomplete: {
                service: null,
                sessionToken: null,
                results: [],
                resultsHighlight: 0,
                status: null,
                selected: this.place,
            },
            context: {
                input: this.value,
                disableSearch: false,
            },
        }
    },

    props: {

        bounds: {
            type: Object,
            required: false,
            default: null,
        },

        fields: {
            type: Array,
            required: false,
            default: () => ([]),
        },

        value: {
            type: String,
            required: false,
            default: '',
        },

        place: {
            type: Object,
            required: false,
            default: () => ({}),
        },

    },

    computed: {

        hasResults() {
            return this.autocomplete.results.length > 0
        },

        searchValue() {
            return this.context.input
        },

        resultField() {
            return [
                'formatted_address',
                'geometry',
                ...this.fields,
            ]
        }

    },

    watch: {

        value: {
            handler(value) {
                if (!value) return

                this.$set(this.context, 'input', value)
            },
            immediate: true,
        },

        place: {
            handler(value) {
                if (!value) return

                this.$set(this.autocomplete, 'selected', value)
            },
            immediate: true,
        },

        searchValue(newValue, oldValue) {
            if (newValue || !oldValue) return

            this.$emit('resultCleared')
        },

    },

    methods: {

        initGoogleAutoCompleteService() {
            this.$set(this.autocomplete, 'sessionToken', new window.google.maps.places.AutocompleteSessionToken())
            this.$set(this.autocomplete, 'service', new window.google.maps.places.AutocompleteService())
        },

        selectItemFromList() {
            const { results, resultsHighlight, selected } = this.autocomplete
            const { input } = this.context

            /**
             * Bail if there is nothing to work with
             */
            if (!input && !results.length) {
                return
            }

            /**
             * Return the last result if things haven't changed
             */
            if (input === this.value && Object.keys(selected).length) {
                return this.returnLastSelection()
            }

            /**
             * Show the search results again
             */
            if (input && !results.length) {
                return this.inputHasChanged()
            }

            /**
             * The expected standard user journey. The user selected a result from the list.
             */
            this.resultHasBeenSelected(results[resultsHighlight] || place)
        },

        shiftResultsSelection() {
            const { results, resultsHighlight } = this.autocomplete
            let newIndex = Math.min(results.length, resultsHighlight) + 1

            if (newIndex >= results.length) newIndex = 0

            this.$set(this.autocomplete, 'resultsHighlight', newIndex)
        },

        unshiftResultsSelection() {
            const { results, resultsHighlight } = this.autocomplete
            let newIndex = Math.min(results.length, resultsHighlight) - 1

            if (newIndex < 0) newIndex = results.length - 1

            this.$set(this.autocomplete, 'resultsHighlight', newIndex)
        },

        inputHasReceivedFocus() {
            if (this.autocomplete.service) return

            this.initGoogleAutoCompleteService()
        },

        inputHasChanged() {
            const { service, sessionToken } = this.autocomplete
            const { input } = this.context
            const { bounds } = this

            this.$set(this.autocomplete, 'resultsHighlight', 0)

            if (!input) {
                this.$set(this.autocomplete, 'selected', {})
                this.$set(this.autocomplete, 'results', [])
                return
            }

            service.getPlacePredictions({
                input,
                sessionToken,
                bounds,
            }, (predictions, status) => {
                this.$set(this.autocomplete, 'status', status)

                if (status !== window.google.maps.places.PlacesServiceStatus.OK) return

                this.$set(this.autocomplete, 'results', predictions)
            })
        },

        resultHasBeenSelected({ place_id: placeId, description }) {
            const placeService = new window.google.maps.places.PlacesService(document.createElement('div'))

            placeService.getDetails({
                placeId,
                fields: this.resultFields,
            }, (place) => {
                this.$set(this.autocomplete, 'selected', place)
                this.$set(this.context, 'input', description)
                this.$set(this.autocomplete, 'results', [])
                this.$emit('resultChanged', place)
            })
        },

        returnLastSelection() {
            const { selected: place } = this.autocomplete

            if (!place) return

            this.$emit('resultChanged', place)
        }

    },

}
</script>
