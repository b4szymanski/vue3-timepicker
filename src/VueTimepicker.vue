<script setup>
import { ref, computed, watch, nextTick, onMounted, onBeforeUnmount } from 'vue'

const CONFIG = {
  HOUR_TOKENS: ['HH', 'H', 'hh', 'h', 'kk', 'k'],
  MINUTE_TOKENS: ['mm', 'm'],
  SECOND_TOKENS: ['ss', 's'],
  APM_TOKENS: ['A', 'a'],
  BASIC_TYPES: ['hour', 'minute', 'second', 'apm']
}

const DEFAULT_OPTIONS = {
  format: 'HH:mm',
  minuteInterval: 1,
  secondInterval: 1,
  hourRange: null,
  minuteRange: null,
  secondRange: null,
  hideDisabledHours: false,
  hideDisabledMinutes: false,
  hideDisabledSeconds: false,
  hideDisabledItems: false,
  advancedKeyboard: false,
  hideDropdown: false,
  blurDelay: 300,
  manualInputTimeout: 1000,
  dropOffsetHeight: 160
}

// Props
const props = defineProps({
  modelValue: { type: [Object, String] },
  format: { type: String },
  minuteInterval: { type: [Number, String] },
  secondInterval: { type: [Number, String] },
  hourRange: { type: Array },
  minuteRange: { type: Array },
  secondRange: { type: Array },
  hideDisabledHours: { type: Boolean, default: false },
  hideDisabledMinutes: { type: Boolean, default: false },
  hideDisabledSeconds: { type: Boolean, default: false },
  hideDisabledItems: { type: Boolean, default: false },
  hideClearButton: { type: Boolean, default: false },
  disabled: { type: Boolean, default: false },
  closeOnComplete: { type: Boolean, default: false },
  id: { type: String },
  name: { type: String },
  inputClass: { type: [String, Object, Array] },
  placeholder: { type: String },
  tabindex: { type: [Number, String], default: 0 },
  inputWidth: { type: String },
  autocomplete: { type: String, default: 'off' },
  hourLabel: { type: String },
  minuteLabel: { type: String },
  secondLabel: { type: String },
  apmLabel: { type: String },
  amText: { type: String },
  pmText: { type: String },
  blurDelay: { type: [Number, String] },
  advancedKeyboard: { type: Boolean, default: false },
  lazy: { type: Boolean, default: false },
  autoScroll: { type: Boolean, default: false },
  dropDirection: { type: String, default: 'down' },
  dropOffsetHeight: { type: [Number, String] },
  containerId: { type: String },
  appendToBody: { type: Boolean, default: false },
  manualInput: { type: Boolean, default: false },
  manualInputTimeout: { type: [Number, String] },
  hideDropdown: { type: Boolean, default: false },
  fixedDropdownButton: { type: Boolean, default: false },
  debugMode: { type: Boolean, default: false }
})

// Emits
const emit = defineEmits(['update:modelValue', 'change', 'open', 'close', 'focus', 'blur', 'error'])

// Refs
const input = ref(null)
const dropdown = ref(null)

// State
const timeValue = ref({})
const hours = ref([])
const minutes = ref([])
const seconds = ref([])
const apms = ref([])
const isActive = ref(false)
const showDropdown = ref(false)
const isFocusing = ref(false)
const debounceTimer = ref(undefined)
const hourType = ref('HH')
const minuteType = ref('mm')
const secondType = ref('')
const apmType = ref('')
const hour = ref('')
const minute = ref('')
const second = ref('')
const apm = ref('')
const fullValues = ref(undefined)
const bakDisplayTime = ref(undefined)
const doClearApmChecking = ref(false)
const selectionTimer = ref(undefined)
const kbInputTimer = ref(undefined)
const kbInputLog = ref('')
const bakCurrentPos = ref(undefined)
const forceDropOnTop = ref(false)

// Computed
const opts = computed(() => {
  const options = { ...DEFAULT_OPTIONS }
  
  if (props.format?.length) {
    options.format = String(props.format)
  }

  if (isNumber(props.minuteInterval)) {
    options.minuteInterval = +props.minuteInterval
  }
  if (!options.minuteInterval || options.minuteInterval < 1 || options.minuteInterval > 60) {
    if (props.debugMode) {
      if (options.minuteInterval > 60) {
        debugLog(`"minute-interval" should be less than 60. Current value is ${props.minuteInterval}`)
      } else if (options.minuteInterval === 0 || options.minuteInterval < 1) {
        debugLog(`"minute-interval" should be NO less than 1. Current value is ${props.minuteInterval}`)
      }
    }
    options.minuteInterval = options.minuteInterval === 0 ? 60 : 1
  }

  if (isNumber(props.secondInterval)) {
    options.secondInterval = +props.secondInterval
  }
  if (!options.secondInterval || options.secondInterval < 1 || options.secondInterval > 60) {
    if (props.debugMode) {
      if (options.secondInterval > 60) {
        debugLog(`"second-interval" should be less than 60. Current value is ${props.secondInterval}`)
      } else if (options.secondInterval === 0 || options.secondInterval < 1) {
        debugLog(`"second-interval" should be NO less than 1. Current value is ${props.secondInterval}`)
      }
    }
    options.secondInterval = options.secondInterval === 0 ? 60 : 1
  }

  if (props.hourRange?.length) {
    options.hourRange = JSON.parse(JSON.stringify(props.hourRange))
    if (!props.hourRange.length && props.debugMode) {
      debugLog('The "hour-range" array is empty (length === 0)')
    }
  }

  if (props.minuteRange?.length) {
    options.minuteRange = JSON.parse(JSON.stringify(props.minuteRange))
    if (!props.minuteRange.length && props.debugMode) {
      debugLog('The "minute-range" array is empty (length === 0)')
    }
  }

  if (props.secondRange?.length) {
    options.secondRange = JSON.parse(JSON.stringify(props.secondRange))
    if (!props.secondRange.length && props.debugMode) {
      debugLog('The "second-range" array is empty (length === 0)')
    }
  }

  if (props.hideDisabledItems) {
    options.hideDisabledItems = true
  }

  if (props.hideDisabledHours || props.hideDisabledItems) {
    options.hideDisabledHours = true
  }
  if (props.hideDisabledMinutes || props.hideDisabledItems) {
    options.hideDisabledMinutes = true
  }

  return options
})

const useStringValue = computed(() => typeof props.modelValue === 'string')
const formatString = computed(() => opts.value.format || DEFAULT_OPTIONS.format)

const inUse = computed(() => {
  const typesInUse = CONFIG.BASIC_TYPES.filter(type => getTokenByType(type))
  typesInUse.sort((l, r) => {
    return formatString.value.indexOf(getTokenByType(l) || null) - formatString.value.indexOf(getTokenByType(r) || null)
  })
  const tokensInUse = typesInUse.map(type => getTokenByType(type))
  return {
    hour: !!hourType.value,
    minute: !!minuteType.value,
    second: !!secondType.value,
    apm: !!apmType.value,
    types: typesInUse || [],
    tokens: tokensInUse || []
  }
})

const displayTime = computed(() => {
  let formatStr = String(formatString.value)
  if (hour.value) {
    formatStr = formatStr.replace(new RegExp(hourType.value, 'g'), hour.value)
  }
  if (minute.value) {
    formatStr = formatStr.replace(new RegExp(minuteType.value, 'g'), minute.value)
  }
  if (second.value && secondType.value) {
    formatStr = formatStr.replace(new RegExp(secondType.value, 'g'), second.value)
  }
  if (apm.value && apmType.value) {
    formatStr = formatStr.replace(new RegExp(apmType.value, 'g'), apm.value)
  }
  return formatStr
})

const customDisplayTime = computed(() => {
  if (!props.amText && !props.pmText) {
    return displayTime.value
  }
  return displayTime.value.replace(new RegExp(apm.value, 'g'), apmDisplayText(apm.value))
})

const inputIsEmpty = computed(() => formatString.value === displayTime.value)

const allValueSelected = computed(() => {
  if (
    (inUse.value.hour && !hour.value) ||
    (inUse.value.minute && !minute.value) ||
    (inUse.value.second && !second.value) ||
    (inUse.value.apm && !apm.value)
  ) {
    return false
  }
  return true
})

const columnsSequence = computed(() => inUse.value.types.map(type => type) || [])

const invalidValues = computed(() => {
  if (inputIsEmpty.value) return []
  if (!restrictedHourRange.value && !minuteRangeList.value && !secondRangeList.value && opts.value.minuteInterval === 1 && opts.value.secondInterval === 1) return []

  const result = []
  if (inUse.value.hour && !isEmptyValue(hourType.value, hour.value) && (!isValidValue(hourType.value, hour.value) || isDisabled('hour', hour.value))) {
    result.push('hour')
  }
  if (inUse.value.minute && !isEmptyValue(minuteType.value, minute.value) && (!isValidValue(minuteType.value, minute.value) || isDisabled('minute', minute.value) || notInInterval('minute', minute.value))) {
    result.push('minute')
  }
  if (inUse.value.second && !isEmptyValue(secondType.value, second.value) && (!isValidValue(secondType.value, second.value) || isDisabled('second', second.value) || notInInterval('second', second.value))) {
    result.push('second')
  }
  if (inUse.value.apm && !isEmptyValue(apmType.value, apm.value) && (!isValidValue(apmType.value, apm.value) || isDisabled('apm', apm.value))) {
    result.push('apm')
  }
  return result
})

const hasInvalidInput = computed(() => Boolean(invalidValues.value && invalidValues.value.length))

const autoDirectionEnabled = computed(() => props.dropDirection === 'auto')

const dropdownDirClass = computed(() => {
  if (autoDirectionEnabled.value) {
    return forceDropOnTop.value ? 'drop-up' : 'drop-down'
  }
  return props.dropDirection === 'up' ? 'drop-up' : 'drop-down'
})

// Core methods
const getTokenByType = (type) => {
  if (!type || !CONFIG.BASIC_TYPES.includes(type)) return ''
  const tokens = CONFIG[`${type.toUpperCase()}_TOKENS`]
  if (!tokens) return ''
  return checkAcceptingType(tokens, formatString.value)
}

const checkAcceptingType = (validValues, formatString) => {
  if (!validValues || !formatString || !formatString.length) return ''
  for (let i = 0; i < validValues.length; i++) {
    if (formatString.indexOf(validValues[i]) > -1) {
      return validValues[i]
    }
  }
  return ''
}

const formatValue = (token, i) => {
  if (!isNumber(i)) return ''
  i = +i
  switch (token) {
    case 'H':
    case 'h':
    case 'k':
    case 'm':
    case 's':
      if (['h', 'k'].includes(token) && i === 0) {
        return token === 'k' ? '24' : '12'
      }
      return String(i)
    case 'HH':
    case 'mm':
    case 'ss':
    case 'hh':
    case 'kk':
      if (['hh', 'kk'].includes(token) && i === 0) {
        return token === 'kk' ? '24' : '12'
      }
      return i < 10 ? `0${i}` : String(i)
    default:
      return ''
  }
}

const isNumber = (value) => !isNaN(parseFloat(value)) && isFinite(value)

const debugLog = (msg) => {
  if (props.debugMode) {
    console.warn(`[vue-timepicker] ${msg}`)
  }
}

// Event handlers
const onFocus = () => {
  if (props.disabled) return
  isActive.value = true
}

const onBlur = () => {
  isActive.value = false
}

const onChange = (evt) => {
  if (props.disabled) return
  if (props.manualInput) {
    const inputValue = evt.target.value
    if (!inputValue.length) {
      clearTime()
      return
    }
    readStringValues(inputValue)
  }
}

const onMouseDown = () => {
  if (!props.manualInput) return
  window.clearTimeout(selectionTimer.value)
  selectionTimer.value = window.setTimeout(() => {
    window.clearTimeout(selectionTimer.value)
    if (input.value) {
      const nearestSlot = getNearestChunkByPos(input.value.selectionStart || 0)
      debounceSetInputSelection(nearestSlot)
    }
  }, 50)
}

const keyDownHandler = (evt) => {
  if (evt.isComposing || evt.keyCode === 229) {
    evt.preventDefault()
    evt.stopPropagation()
    return false
  }
  // Numbers
  if ((evt.keyCode >= 48 && evt.keyCode <= 57) || (evt.keyCode >= 96 && evt.keyCode <= 105)) {
    evt.preventDefault()
    keyboardInput(evt.key)
  // A|P|M
  } else if ([65, 80, 77].includes(evt.keyCode)) {
    evt.preventDefault()
    keyboardInput(evt.key, true)
  // Arrow keys
  } else if (evt.keyCode >= 37 && evt.keyCode <= 40) {
    evt.preventDefault()
    clearKbInputLog()
    arrowHandler(evt)
  // Delete|Backspace
  } else if (evt.keyCode === 8 || evt.keyCode === 46) {
    evt.preventDefault()
    clearKbInputLog()
    clearTime()
  // Tab
  } else if (evt.keyCode === 9) {
    clearKbInputLog()
    tabHandler(evt)
  // Prevent any Non-ESC and non-pasting inputs
  } else if (evt.keyCode !== 27 && !(evt.metaKey || evt.ctrlKey)) {
    evt.preventDefault()
  }
}

const onCompostionStart = (evt) => {
  evt.preventDefault()
  evt.stopPropagation()
  bakCurrentPos.value = getCurrentTokenChunk()
  return false
}

const onCompostionEnd = (evt) => {
  evt.preventDefault()
  evt.stopPropagation()
  if (bakCurrentPos.value) {
    debounceSetInputSelection(bakCurrentPos.value)
  }
  return false
}

const select = (type, value) => {
  if (props.disabled || !type || !CONFIG.BASIC_TYPES.includes(type)) return
  if (isDisabled(type, value)) return

  switch (type) {
    case 'hour':
      setManualHour(value)
      break
    case 'minute':
    case 'second':
    case 'apm':
      setSanitizedValueToSection(type, value)
      break
  }

  if (props.closeOnComplete && allValueSelected.value) {
    setDropdownState(false)
  }
}

// Lifecycle hooks
onMounted(() => {
  renderFormat(props.format)
  if (props.value) {
    readValues()
  }
  if (props.autoScroll) {
    window.addEventListener('scroll', checkDropDirection)
    window.addEventListener('resize', checkDropDirection)
  }
})

onBeforeUnmount(() => {
  if (props.autoScroll) {
    window.removeEventListener('scroll', checkDropDirection)
    window.removeEventListener('resize', checkDropDirection)
  }
})

// Watchers
watch(() => props.format, (newValue) => {
  renderFormat(newValue)
})

watch(() => props.minuteInterval, (newInterval) => {
  renderList('minute', newInterval)
})

watch(() => props.secondInterval, (newInterval) => {
  renderList('second', newInterval)
})

watch(() => props.value, {
  deep: true,
  handler() {
    readValues()
  }
})

watch(displayTime, () => {
  fillValues()
})

watch(() => props.disabled, (toDisabled) => {
  if (toDisabled) {
    if (isActive.value) {
      isActive.value = false
    }
    if (showDropdown.value) {
      showDropdown.value = false
    }
  }
})

watch(() => invalidValues.value.length, (newLength, oldLength) => {
  if (newLength && newLength >= 1) {
    emit('error', invalidValues.value)
  } else if (oldLength && oldLength >= 1) {
    emit('error', [])
  }
})

// ... rest of the component ...
</script>

<template>
  <span class="vue__time-picker" :style="inputWidthStyle">
    <input
      type="text"
      class="vue__time-picker-input"
      ref="input"
      :class="[
        inputClass,
        {
          'is-empty': inputIsEmpty,
          'invalid': hasInvalidInput,
          'all-selected': allValueSelected,
          'disabled': disabled,
          'has-custom-icon': $slots?.icon
        }
      ]"
      :style="inputWidthStyle"
      :id="id"
      :name="name"
      :value="inputIsEmpty ? null : customDisplayTime"
      :placeholder="placeholder ? placeholder : formatString"
      :tabindex="disabled ? -1 : tabindex"
      :disabled="booleanAttr(disabled)"
      :readonly="booleanAttr(!manualInput)"
      :autocomplete="autocomplete"
      @focus="onFocus"
      @change="onChange"
      @blur="debounceBlur(); blurEvent()"
      @mousedown="onMouseDown"
      @keydown="keyDownHandler"
      @compositionstart="onCompostionStart"
      @compositionend="onCompostionEnd"
      @paste="pasteHandler"
      @keydown.esc.exact="escBlur"
    />
    <div class="controls" v-if="showClearBtn || showDropdownBtn" tabindex="-1">
      <span
        v-if="!isActive && showClearBtn"
        class="clear-btn"
        tabindex="-1"
        :class="{ 'has-custom-btn': $slots?.clearButton }"
        @click="clearTime"
      >
        <slot name="clearButton"><span class="char">&times;</span></slot>
      </span>
      <span
        v-if="showDropdownBtn"
        class="dropdown-btn"
        tabindex="-1"
        :class="{ 'has-custom-btn': $slots?.dropdownButton }"
        @click="setDropdownState(fixedDropdownButton ? !showDropdown : true, true)"
        @mousedown="keepFocusing"
      >
        <slot name="dropdownButton"><span class="char">&dtrif;</span></slot>
      </span>
    </div>
    <div class="custom-icon" v-if="$slots?.icon">
      <slot name="icon"></slot>
    </div>
    <div
      class="time-picker-overlay"
      v-if="showDropdown"
      @click="toggleActive"
      tabindex="-1"
    ></div>
    <div
      class="dropdown"
      ref="dropdown"
      v-show="showDropdown"
      tabindex="-1"
      :class="[dropdownDirClass]"
      :style="inputWidthStyle"
      @mouseup="keepFocusing"
      @click.stop=""
    >
      <div class="select-list" :style="inputWidthStyle" tabindex="-1">
        <!-- Common Keyboard Support -->
        <template v-if="!advancedKeyboard">
          <template v-for="column in columnsSequence" :key="column">
            <ul v-if="column === 'hour'" class="hours" @scroll="keepFocusing">
              <li class="hint" v-text="hourLabelText"></li>
              <template v-for="(hr, hIndex) in hours" :key="hIndex">
                <li
                  v-if="!opts.hideDisabledHours || (opts.hideDisabledHours && !isDisabled('hour', hr))"
                  :class="{ active: hour === hr }"
                  :disabled="booleanAttr(isDisabled('hour', hr))"
                  :data-key="hr"
                  v-text="hr"
                  @click="select('hour', hr)"
                ></li>
              </template>
            </ul>
            <ul v-if="column === 'minute'" class="minutes" @scroll="keepFocusing">
              <li class="hint" v-text="minuteLabelText"></li>
              <template v-for="(m, mIndex) in minutes" :key="mIndex">
                <li
                  v-if="!opts.hideDisabledMinutes || (opts.hideDisabledMinutes && !isDisabled('minute', m))"
                  :class="{ active: minute === m }"
                  :disabled="booleanAttr(isDisabled('minute', m))"
                  :data-key="m"
                  v-text="m"
                  @click="select('minute', m)"
                ></li>
              </template>
            </ul>
            <ul v-if="column === 'second'" class="seconds" @scroll="keepFocusing">
              <li class="hint" v-text="secondLabelText"></li>
              <template v-for="(s, sIndex) in seconds" :key="sIndex">
                <li
                  v-if="!opts.hideDisabledSeconds || (opts.hideDisabledSeconds && !isDisabled('second', s))"
                  :class="{ active: second === s }"
                  :disabled="booleanAttr(isDisabled('second', s))"
                  :data-key="s"
                  v-text="s"
                  @click="select('second', s)"
                ></li>
              </template>
            </ul>
            <ul v-if="column === 'apm'" class="apms" @scroll="keepFocusing">
              <li class="hint" v-text="apmLabelText"></li>
              <template v-for="(a, aIndex) in apms" :key="aIndex">
                <li
                  v-if="!opts.hideDisabledHours || (opts.hideDisabledHours && !isDisabled('apm', a))"
                  :class="{ active: apm === a }"
                  :disabled="booleanAttr(isDisabled('apm', a))"
                  :data-key="a"
                  v-text="apmDisplayText(a)"
                  @click="select('apm', a)"
                ></li>
              </template>
            </ul>
          </template>
        </template>

        <!-- Advanced Keyboard Support -->
        <template v-if="advancedKeyboard">
          <template v-for="column in columnsSequence" :key="column">
            <ul v-if="column === 'hour'" class="hours" tabindex="-1" @scroll="keepFocusing">
              <li class="hint" v-text="hourLabelText" tabindex="-1"></li>
              <template v-for="(hr, hIndex) in hours" :key="hIndex">
                <li
                  v-if="!opts.hideDisabledHours || (opts.hideDisabledHours && !isDisabled('hour', hr))"
                  :class="{ active: hour === hr }"
                  :tabindex="isDisabled('hour', hr) ? -1 : tabindex"
                  :data-key="hr"
                  :disabled="booleanAttr(isDisabled('hour', hr))"
                  v-text="hr"
                  @click="select('hour', hr)"
                  @keydown.space.prevent="select('hour', hr)"
                  @keydown.enter.prevent="select('hour', hr)"
                  @keydown.up.prevent="prevItem('hour', hr)"
                  @keydown.down.prevent="nextItem('hour', hr)"
                  @keydown.left.prevent="toLeftColumn('hour')"
                  @keydown.right.prevent="toRightColumn('hour')"
                  @keydown.esc.exact="debounceBlur"
                  @blur="debounceBlur"
                  @focus="keepFocusing"
                ></li>
              </template>
            </ul>
            <ul v-if="column === 'minute'" class="minutes" tabindex="-1" @scroll="keepFocusing">
              <li class="hint" v-text="minuteLabelText" tabindex="-1"></li>
              <template v-for="(m, mIndex) in minutes" :key="mIndex">
                <li
                  v-if="!opts.hideDisabledMinutes || (opts.hideDisabledMinutes && !isDisabled('minute', m))"
                  :class="{ active: minute === m }"
                  :tabindex="isDisabled('minute', m) ? -1 : tabindex"
                  :data-key="m"
                  :disabled="booleanAttr(isDisabled('minute', m))"
                  v-text="m"
                  @click="select('minute', m)"
                  @keydown.space.prevent="select('minute', m)"
                  @keydown.enter.prevent="select('minute', m)"
                  @keydown.up.prevent="prevItem('minute', m)"
                  @keydown.down.prevent="nextItem('minute', m)"
                  @keydown.left.prevent="toLeftColumn('minute')"
                  @keydown.right.prevent="toRightColumn('minute')"
                  @keydown.esc.exact="debounceBlur"
                  @blur="debounceBlur"
                  @focus="keepFocusing"
                ></li>
              </template>
            </ul>
            <ul v-if="column === 'second'" class="seconds" tabindex="-1" @scroll="keepFocusing">
              <li class="hint" v-text="secondLabelText" tabindex="-1"></li>
              <template v-for="(s, sIndex) in seconds" :key="sIndex">
                <li
                  v-if="!opts.hideDisabledSeconds || (opts.hideDisabledSeconds && !isDisabled('second', s))"
                  :class="{ active: second === s }"
                  :tabindex="isDisabled('second', s) ? -1 : tabindex"
                  :data-key="s"
                  :disabled="booleanAttr(isDisabled('second', s))"
                  v-text="s"
                  @click="select('second', s)"
                  @keydown.space.prevent="select('second', s)"
                  @keydown.enter.prevent="select('second', s)"
                  @keydown.up.prevent="prevItem('second', s)"
                  @keydown.down.prevent="nextItem('second', s)"
                  @keydown.left.prevent="toLeftColumn('second')"
                  @keydown.right.prevent="toRightColumn('second')"
                  @keydown.esc.exact="debounceBlur"
                  @blur="debounceBlur"
                  @focus="keepFocusing"
                ></li>
              </template>
            </ul>
            <ul v-if="column === 'apm'" class="apms" tabindex="-1" @scroll="keepFocusing">
              <li class="hint" v-text="apmLabelText" tabindex="-1"></li>
              <template v-for="(a, aIndex) in apms" :key="aIndex">
                <li
                  v-if="!opts.hideDisabledHours || (opts.hideDisabledHours && !isDisabled('apm', a))"
                  :class="{ active: apm === a }"
                  :tabindex="isDisabled('apm', a) ? -1 : tabindex"
                  :data-key="a"
                  :disabled="booleanAttr(isDisabled('apm', a))"
                  v-text="apmDisplayText(a)"
                  @click="select('apm', a)"
                  @keydown.space.prevent="select('apm', a)"
                  @keydown.enter.prevent="select('apm', a)"
                  @keydown.up.prevent="prevItem('apm', a)"
                  @keydown.down.prevent="nextItem('apm', a)"
                  @keydown.left.prevent="toLeftColumn('apm')"
                  @keydown.right.prevent="toRightColumn('apm')"
                  @keydown.esc.exact="debounceBlur"
                  @blur="debounceBlur"
                  @focus="keepFocusing"
                ></li>
              </template>
            </ul>
          </template>
        </template>
      </div>
    </div>
  </span>
</template>

<style scoped>
.vue__time-picker {
  display: inline-block;
  position: relative;
  font-size: 1em;
  width: 10em;
  font-family: sans-serif;
  vertical-align: middle;
}

.vue__time-picker * {
  box-sizing: border-box;
}

.vue__time-picker input.vue__time-picker-input {
  border: 1px solid #d2d2d2;
  width: 10em;
  height: 2.2em;
  padding: 0.3em 0.5em;
  font-size: 1em;
  transition: border-color 0.2s ease;
}

.vue__time-picker input.has-custom-icon {
  padding-left: 1.8em;
}

.vue__time-picker input.vue__time-picker-input.invalid:not(.skip-error-style) {
  border-color: #cc0033;
  outline-color: #cc0033;
}

.vue__time-picker input.vue__time-picker-input:disabled,
.vue__time-picker input.vue__time-picker-input.disabled {
  color: #d2d2d2;
  background-color: #f5f5f5;
}

.vue__time-picker .controls {
  position: absolute;
  top: 0;
  bottom: 0;
  right: 0;
  z-index: 3;
  display: flex;
  flex-flow: row nowrap;
  justify-content: flex-end;
  align-items: stretch;
  pointer-events: none;
}

.vue__time-picker .controls > * {
  cursor: pointer;
  width: auto;
  display: flex;
  flex-flow: column nowrap;
  justify-content: center;
  align-items: center;
  padding: 0 0.35em;
  color: #d2d2d2;
  line-height: 100%;
  font-style: normal;
  pointer-events: initial;
  transition: color 0.2s, opacity 0.2s;
}

.vue__time-picker .controls > *:hover {
  color: #797979;
}

.vue__time-picker .controls > *:focus,
.vue__time-picker .controls > *:active {
  outline: 0;
}

.vue__time-picker .controls .char {
  font-size: 1.1em;
  line-height: 100%;
  margin-block-start: -0.15em;
}

.vue__time-picker .custom-icon {
  z-index: 2;
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 1.8em;
  display: flex;
  flex-flow: column nowrap;
  justify-content: center;
  align-items: center;
  pointer-events: none;
}

.vue__time-picker .custom-icon img,
.vue__time-picker .custom-icon svg,
.vue__time-picker .controls img,
.vue__time-picker .controls svg {
  display: inline-block;
  vertical-align: middle;
  margin: 0;
  border: 0;
  outline: 0;
  max-width: 1em;
  height: auto;
}

.vue__time-picker .time-picker-overlay {
  z-index: 4;
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(2px);
}

.vue__time-picker .dropdown {
  position: absolute;
  z-index: 5;
  top: calc(2.2em + 2px);
  left: 0;
  background: #fff;
  box-shadow: 0 1px 6px rgba(0, 0, 0, 0.15);
  width: 10em;
  height: auto;
  margin: 0;
  padding: 0;
  border-radius: 4px;
  overflow: hidden;
}

.vue__time-picker .dropdown.drop-up {
  top: auto;
  bottom: calc(2.2em + 1px);
}

.vue__time-picker .select-list {
  width: 10em;
  height: auto;
  overflow: hidden;
  display: flex;
  flex-flow: row nowrap;
  align-items: stretch;
  justify-content: space-between;
}

.vue__time-picker .select-list ul {
  padding: 0;
  margin: 0;
  list-style: none;
  flex: 1;
  overflow-y: auto;
  height: 10em;
}

.vue__time-picker .select-list ul.apms {
  flex: 0.8;
}

.vue__time-picker .select-list ul li {
  text-align: center;
  padding: 0.3em 0;
  color: #161616;
  cursor: pointer;
  user-select: none;
  transition: background 0.2s ease;
}

.vue__time-picker .select-list ul li.hint {
  color: #a5a5a5;
  cursor: default;
  font-size: 0.8em;
  padding: 0.85em 0;
}

.vue__time-picker .select-list ul li.active {
  background: #41b883;
  color: #fff;
}

.vue__time-picker .select-list ul li:not(.hint):not(:disabled):hover {
  background: #f5f5f5;
}

.vue__time-picker .select-list ul li:not(.hint):not(:disabled):active {
  background: #e8e8e8;
}

.vue__time-picker .select-list ul li:not(.hint):disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
</style>
