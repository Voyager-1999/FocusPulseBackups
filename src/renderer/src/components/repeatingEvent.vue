<template>
  <div
    id="reDropDown"
    class="header-menu-icons dropdown-toggle"  
    type="button"
    data-bs-toggle="dropdown"
    data-bs-auto-close="outside"
    aria-expanded="false"
    title="重复任务"
    @click="handleClick"
    ref="dropdownRef"  
  >
    <i id="btnRepeatingEvent" :class="{ 'bi-arrow-clockwise': !repeatingEvent, 'bi-arrow-repeat': repeatingEvent }"></i>
  </div>

  <ul class="dropdown-menu repeating-event-dropdown" aria-labelledby="btnRepeatingEvent">
    <div class="mx-3 d-flex flex-column drop-container">
      <select
        class="form-select re-input"
        aria-label="Default select example"
        v-model="repeatingType"
        :disabled="repeatingEvent"
      >
        <option value="">不重复</option>
        <option value="3">每天</option>
        <option value="2">每周</option>
        <option value="4">工作日</option>
        <option value="5">自定义工作日</option>
        <option value="1">每月</option>
        <option value="6">每月指定日期</option>
        <option value="0">每年</option>
      </select>

      <div v-if="repeatingType" class="d-flex flex-column">
        <div v-show="repeatingType == '5' || (repeatingEvent && repeatingType == '2')" class="weekDays-selector mt-3 mb-1">
          <input type="checkbox" :disabled="repeatingEvent" id="weekday-mon" class="weekday" v-model="weekdays.mon" />
          <label for="weekday-mon">{{ getWeekdayLabel(1) }}</label>
          <input type="checkbox" :disabled="repeatingEvent" id="weekday-tue" class="weekday" v-model="weekdays.tue" />
          <label for="weekday-tue">{{ getWeekdayLabel(2) }}</label>
          <input type="checkbox" :disabled="repeatingEvent" id="weekday-wed" class="weekday" v-model="weekdays.wed" />
          <label for="weekday-wed">{{ getWeekdayLabel(3) }}</label>
          <input type="checkbox" :disabled="repeatingEvent" id="weekday-thu" class="weekday" v-model="weekdays.thu" />
          <label for="weekday-thu">{{ getWeekdayLabel(4) }}</label>
          <input type="checkbox" :disabled="repeatingEvent" id="weekday-fri" class="weekday" v-model="weekdays.fri" />
          <label for="weekday-fri">{{ getWeekdayLabel(5) }}</label>
          <input type="checkbox" :disabled="repeatingEvent" id="weekday-sat" class="weekday" v-model="weekdays.sat" />
          <label for="weekday-sat">{{ getWeekdayLabel(6) }}</label>
          <input type="checkbox" :disabled="repeatingEvent" id="weekday-sun" class="weekday" v-model="weekdays.sun" />
          <label for="weekday-sun">{{ getWeekdayLabel(7) }}</label>
        </div>

        <div v-show="repeatingType == '6'" class="mt-3 mb-1">
          <input
            v-model="daysOfMonth"
            type="text"
            class="form-control day-of-month"
            :disabled="repeatingEvent"
            placeholder="1,2,15"
          />
        </div>

        <div class="d-flex align-items-center justify-content-between re-input">
          <label for="interval-input" class="opacity-50">间隔</label>
          <input id="interval-input" type="number" class="form-control lex-shrink-1 counter" v-model="interval" :disabled="repeatingEvent" />
        </div>

        <select
          class="form-select re-input"
          aria-label="Default select example"
          v-model="ocurrencesType"
          :disabled="repeatingEvent"
        >
          <option value="">无限重复</option>
          <option value="ocurrences">重复次数</option>
          <option value="untilDate">结束日期</option>
        </select>

        <input
          v-if="ocurrencesType == 'ocurrences'"
          type="number"
          class="form-control re-input last-input"
          v-model="ocurrences"
          :disabled="repeatingEvent"
        />
        <input
          v-if="ocurrencesType == 'untilDate'"
          type="date"
          class="form-control re-input last-input"
          v-model="untilDate"
          :disabled="repeatingEvent"
        />
      </div>

      <div class="d-flex flex-row re-form-row re-input">
        <button v-if="!repeatingEvent" type="button" class="btn flex-fill" @click="done">
          完成
        </button>
        <button v-if="repeatingEvent" type="button" class="btn flex-fill" @click="split">
          拆分
        </button>
      </div>
    </div>
  </ul>
</template>

<script setup>
import { ref, reactive, computed, watch, onMounted, onUnmounted } from 'vue'
import { RRule, rrulestr } from 'rrule'
import dayjs from 'dayjs'
import utc from 'dayjs/plugin/utc'
import localeData from 'dayjs/plugin/localeData'
import { Dropdown } from 'bootstrap'
import repeatingEventRepository from '../repositories/repeatingEventRepository'
import { useRepeatingEventStore } from '../store/repeatingEvent.store'


// 初始化 dayjs 插件
dayjs.extend(utc)
dayjs.extend(localeData)

const props = defineProps({
  repeatingEvent: { required: true, type: [String, null] },
  todo: { required: true, type: [Object, null] },
  rule: { required: true, type: [Object, null] }
})

const emit = defineEmits(['repeatingEventSelected'])

// State
const repeatingType = ref('')
const interval = ref(1)
const ocurrencesType = ref('')
const ocurrences = ref(1)
const untilDate = ref(null)
const daysOfMonth = ref('')
const weekdays = reactive({
  mon: false,
  tue: false,
  wed: false,
  thu: false,
  fri: false,
  sat: false,
  sun: false
})

// Computed
const language = computed(() => repeatingEventStore.config?.language || 'en')

const dropdownRef = ref(null)

const repeatingEventStore = useRepeatingEventStore()

// Methods
function getWeekdayLabel(day) {
  return dayjs().locale(language.value).day(day).format('dd')[0]
}

function done() {
  const rule = repeatingEventRule()
  let repeatingEventId = props.repeatingEvent ? props.repeatingEvent : dayjs().valueOf().toString()

  if (dropdownInstance) {
    dropdownInstance.hide()
  }
  emit('repeatingEventSelected', repeatingEventId, rule, repeatingType.value, ocurrencesType.value)
}

function split() {
  repeatingEventRepository.remove(props.repeatingEvent)
  // 修改这里：使用 ref 代替 getElementById
  if (dropdownInstance) {
    dropdownInstance.hide()
  }
  emit('repeatingEventSelected', null, null)
}

function repeatingEventRule() {
  if (!repeatingType.value) return null

  let ruleOptions = {
    freq: repeatingType.value,
    interval: interval.value,
    dtstart: dayjs.utc(props.todo.listId, 'YYYYMMDD').toDate()
  }

  if (repeatingType.value == 4) {
    ruleOptions.byweekday = [RRule.MO, RRule.TU, RRule.WE, RRule.TH, RRule.FR]
    ruleOptions.freq = 2
  }

  if (repeatingType.value == 5) {
    ruleOptions.byweekday = []
    weekdays.mon && ruleOptions.byweekday.push(RRule.MO)
    weekdays.tue && ruleOptions.byweekday.push(RRule.TU)
    weekdays.wed && ruleOptions.byweekday.push(RRule.WE)
    weekdays.thu && ruleOptions.byweekday.push(RRule.TH)
    weekdays.fri && ruleOptions.byweekday.push(RRule.FR)
    weekdays.sat && ruleOptions.byweekday.push(RRule.SA)
    weekdays.sun && ruleOptions.byweekday.push(RRule.SU)
    ruleOptions.freq = 2
  }

  if (repeatingType.value == 6) {
    const daysOfMonthArray = daysOfMonth.value.split(',').map(num => parseInt(num))
    ruleOptions.bymonthday = daysOfMonthArray
    ruleOptions.freq = 1
  }

  if (ocurrencesType.value == 'ocurrences') {
    ruleOptions.count = ocurrences.value
  } else if (ocurrencesType.value == 'untilDate') {
    ruleOptions.until = dayjs(untilDate.value).toDate()
  }

  return new RRule(ruleOptions)
}

function handleWeekdays(rule) {
  if (!rule.options.byweekday) return false;
  
  rule.options.byweekday.forEach(day => {
    switch(day) {
      case 0: weekdays.mon = true; break;
      case 1: weekdays.tue = true; break;
      case 2: weekdays.wed = true; break;
      case 3: weekdays.thu = true; break;
      case 4: weekdays.fri = true; break;
      case 5: weekdays.sat = true; break;
      case 6: weekdays.sun = true; break;
    }
  });
  
  return isWorkdayRule(rule.options.byweekday);
}

function isWorkdayRule(byweekday) {
  return byweekday.length === 5 && 
    byweekday.includes(0) && 
    byweekday.includes(1) && 
    byweekday.includes(2) && 
    byweekday.includes(3) && 
    byweekday.includes(4);
}

function handleOccurrences(rule) {
  if (rule.options.count) {
    ocurrencesType.value = 'ocurrences';
    ocurrences.value = rule.options.count;
  } else if (rule.options.until) {
    ocurrencesType.value = 'untilDate';
    untilDate.value = dayjs(rule.options.until).format('YYYY-MM-DD');
  } else {
    ocurrencesType.value = '';
    ocurrences.value = 1;
    untilDate.value = '';
  }
}

watch(
  [() => props.repeatingEvent, () => props.rule],
  ([newRepeatingEvent, newRule]) => {
    let re = newRule ? { repeating_rule: newRule.toString() } : repeatingEventStore.repeatingEventList[newRepeatingEvent];
    weekdays.mon = weekdays.tue = weekdays.wed = weekdays.thu = weekdays.fri = weekdays.sat = weekdays.sun = false;
    
    if (re) {
      const rule = rrulestr(re.repeating_rule);
      repeatingType.value = rule.options.freq;
      interval.value = rule.options.interval || 1;
      
      handleOccurrences(rule);
      
      if (handleWeekdays(rule)) {
        repeatingType.value = 4; // 工作日
      } else if (rule.options.byweekday) {
        repeatingType.value = 5; // 自定义工作日
      }

      if (rule.options.bymonthday?.length > 0) {
        repeatingType.value = 6;
        daysOfMonth.value = rule.options.bymonthday.join(',');
      }
    } else {
      resetValues();
    }
  },
  { immediate: true }
)

function resetValues() {
  repeatingType.value = '';
  ocurrencesType.value = '';
  daysOfMonth.value = '';
  interval.value = 1;
  untilDate.value = '';
  ocurrences.value = 1;
}

let dropdownInstance = null

onMounted(() => {
  // 使用 ref 引用来初始化 dropdown
  if (dropdownRef.value) {
    dropdownInstance = new Dropdown(dropdownRef.value, {
      autoClose: 'outside'
    })
  }
})

function handleClick(event) {
  if (dropdownInstance) {
    dropdownInstance.toggle()
  }
}

onUnmounted(() => {
  if (dropdownInstance) {
    dropdownInstance.dispose()
  }
})
</script>

<style scoped lang="scss">
@use "/src/assets/style/globalVars.scss" as *;

.header-menu-icons {
  margin-left: 6px;
  @include btn-icon;
}

.repeating-event-dropdown {
  margin-left: -80px !important;
  width: 200px;
}

.re-input {
  margin-top: 10px;
  font-size: 15px;
}

.counter {
  width: 60px;
  border: none;
  border-bottom: 1px solid #ced4da;
  border-radius: unset;
  padding: unset;
}

.last-input {
  border: none;
  border-bottom: 1px solid #ced4da;
  border-radius: unset;
  padding: 5px;
}

.form-control:focus,
.form-select:focus {
  box-shadow: unset;
}

[type="date"]::-webkit-inner-spin-button {
  display: none;
}

[type="date"]::-webkit-calendar-picker-indicator {
  display: none;
}

.dark-theme input {
  background-color: #21262d;
  color: #c9d1d9;
  border-bottom: 1px solid #464647;
}

.dark-theme .day-of-month {
  background-color: #15161e;
  border: 1px solid #464647;
}

.day-of-month::placeholder {
  color: #9b9b9b;
}

.dark-theme .day-of-month::placeholder {
  color: #838383;
}

.weekDays-selector {
  display: flex;
  justify-content: center;
}

.weekDays-selector input {
  display: none !important;
}

.weekDays-selector input[type="checkbox"] + label {
  display: inline-block;
  border-radius: 3px;
  background: #eaecef;
  height: 20px;
  width: 20px;
  margin-right: 3px;
  line-height: 20px;
  text-align: center;
  cursor: pointer;
  font-size: 12px;
  text-transform: uppercase;
}

.dark-theme .weekDays-selector input[type="checkbox"] + label {
  background: #15161e;
  color: #ffffff;
}

.weekDays-selector input[type="checkbox"]:disabled + label {
  cursor: unset;
  opacity: 0.6;
}

.weekDays-selector input[type="checkbox"]:checked + label {
  background: #5c5c5c;
  color: #ffffff;
}

.dropdown-toggle::after {
  display: none; // 隐藏默认的下拉箭头
}

.dropdown-menu {
  margin-top: 0.5rem;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.drop-container {
  padding: 0.5rem;
}
</style>
