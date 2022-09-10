<template>
    <div class="r-filter">
        <div class="r-filter__values">
            <span class="r-filter__half">
                <span v-if="specs.hasPrefix"
                      class="r-filter__prefix">от</span>
                <input class="r-filter__input"
                       :class="{'center': specs.hasPrefix || specs.suffix}"
                       @input="onHandleInput($event, 'min')"
                       @change="onHandleChange($event, 'min')"
                       v-model="lowerInputValue">
                <span v-if="specs.suffix"
                      v-html="specs.suffix"></span>
            </span>
            <span class="r-filter__half max">
                <span v-if="specs.hasPrefix"
                      class="r-filter__prefix">до</span>
                <input class="r-filter__input max"
                       :class="{'center': specs.hasPrefix || specs.suffix}"
                       @input="onHandleInput($event, 'max')"
                       @change="onHandleChange($event, 'max')"
                       v-model="upperInputValue">
                <span v-if="specs.suffix"
                      v-html="specs.suffix"></span>
            </span>
        </div>

        <div class="r-filter__body range-slider"
             @mousedown="isDragging = true"
             @mouseup="isDragging = false">
            <vue-slider ref="slider"
                        v-model="rangeValue"
                        :min="Number(specs.min)"
                        :max="Number(specs.max)"
                        :interval="specs.step"
                        :dot-size="16"
                        :height="2"
                        :tooltip="'none'"
                        @change="updateSlider"
                        @drag-end="dragEnd">
            </vue-slider>
        </div>
    </div>
</template>

<script>
import vueSlider from 'vue-slider-component'
import {removeSpace} from "../../../common/utils/removeSpace";
export default {
    name: "RangeFilter",
    components: {
        vueSlider
    },
    props: {
        specs: Object,
        name: String,
        facets: Object,
        values: {
            type: Array,
            default: () => {
                return [];
            }
        }
    },
    data() {
        return {
            filter: {
                options: {}
            },
            rangeValue: [Number(this.specs.min), Number(this.specs.max)],
            upperInputValue: this.rangeValue && this.rangeValue[1] ? this.rangeValue[1] : this.specs.max,
            lowerInputValue: this.rangeValue && this.rangeValue[0] ? this.rangeValue[0] : this.specs.min,
            isDragging: false,
        }
    },
    mounted(){
        this.rangeValue = !this.values.length ? [Number(this.specs.min), Number(this.specs.max)] : [...this.values];
        this.updateSlider(this.rangeValue, false);
    },
    computed: {
        minValue() {
            if (this.specs && this.specs.min) {
                return this.specs.min;
            } else if (this.facets && this.facets.min) {
                return this.facets.min;
            } else {
                return null;
            }
        },
        maxValue() {
            if (this.specs && this.specs.max) {
                return this.specs.max;
            } else if (this.facets && this.facets.max) {
                return this.facets.max;
            } else {
                return null;
            }
        },
    },
    watch: {
        values(val) {
            this.rangeValue = !val.length ? [Number(this.specs.min), Number(this.specs.max)] : [...val];
            this.updateSlider(this.rangeValue, false);
        },
        facets(val) {
            if (!this.values.length) {
                if (val.min !== this.rangeValue[0] && val.min !== 0) {
                    this.rangeValue[0] = this.name === 'price' ? Number(val.min / 1000000).toFixed(1) : Number(val.min);
                }

                if (val.max !== this.rangeValue[1] && val.max !== 0) {
                    this.rangeValue[1] = this.name === 'price' ? Number(val.max / 1000000).toFixed(1) : Number(val.max);
                }

                this.$refs.slider.setValue(this.rangeValue);
                this.updateSlider(this.rangeValue, false);
            }
        }
    },
    methods: {
        onHandleInput(e, extremum = 'min') {
            let value = removeSpace(e.target.value).replace(/[^0-9,.]+/g, '').replace(',', '.');

            if (extremum === 'min') {
                this.lowerInputValue = value;
            } else {
                this.upperInputValue = value;
            }
        },
        onHandleChange(e, extremum = 'min') {
            const input = e.target;
            const value = input.value;
            let number = Number(removeSpace(value));
            if (Number.isNaN(number)) number = 0;
            let newValue = [...this.rangeValue];

            if (extremum === 'min') {
                if (number > this.maxValue) {
                    newValue[0] = this.maxValue;
                } else if (number < this.minValue) {
                    newValue[0] = this.minValue;
                    newValue[1] = newValue[1].toString();
                } else {
                    newValue[0] = number;
                }
            } else {
                if (number < this.minValue) {
                    newValue[1] = this.minValue;
                } else if (number > this.maxValue) {
                    newValue[1] = this.maxValue;
                } else {
                    newValue[1] = number;
                }
            }

            if (Number(newValue[1]) < Number(newValue[0])) {
                let temp = newValue[0];
                newValue[0] = newValue[1];
                newValue[1] = temp;
            }

            this.updateSlider(newValue, false);
            this.$refs.slider.setValue(newValue);
            this.submitValues(newValue)
        },
        updateSlider(newValues, update = true) {
            if (typeof newValues[0] !== 'undefined') this.lowerInputValue = newValues[0];
            if (typeof newValues[1] !== 'undefined') this.upperInputValue = newValues[1];
            if (this.specs.float) {
                this.lowerInputValue = Number(this.lowerInputValue).toFixed(1);
                this.upperInputValue = Number(this.upperInputValue).toFixed(1);
            }
            if (update && !this.isDragging) this.submitValues(newValues);
        },
        dragEnd() {
            this.submitValues([this.lowerInputValue, this.upperInputValue]);
        },
        isFloat(n){
            return Number(n) === n && n % 1 !== 0;
        },
        submitValues(vals) {
            if (this.$mq === 'mobile') {
                this.$root.$emit('preupdate-values', {[this.name]: vals});
            } else {
                this.$root.$emit('update-values', {[this.name]: vals});
            }
        }
    }
}
</script>

<style lang="scss" scoped>
.r-filter {
    position: relative;
    border: 1px solid #e6e6e6;
    border-radius: 4px;

    &__values {
        display: flex;
        padding: 0 16px;
    }

    &__half {
        display: flex;
        align-items: center;
        height: 48px;
        width: 50%;

        &.max {
            text-align: right;
            justify-content: flex-end;
        }
    }

    &__prefix {
        margin-right: 4px;
        color: $fod-light;
    }

    &__input {
        border: none;
        width: 40px;
        padding: 0;
        font-size: 14px;
        line-height: 20px;
        font-weight: 500;
        outline: none;
        transition: opacity .3s ease-in-out;

        &.max {
            text-align: right;
        }

        &.center {
            text-align: center;
        }

        @media (hover: hover) {

            &:hover {
                opacity: .5;
            }
        }
    }

    &__body {
        position: absolute;
        bottom: -9px;
        width: 100%;
        padding: 0 24px;

        /deep/ .vue-slider-dot {
            background-color: $fod-gray;
            border-radius: 100%;
            cursor: pointer;
        }

        /deep/ .vue-slider-process {
            background-color: $fod-gray;
        }

        /deep/ .vue-slider {
            cursor: pointer;
        }
    }
}
</style>