<template>
  <div class="date-range-container">
    <DropDown
        id="periodDropdown"
        label="Período:"
        v-model="selectedPeriod"
        :options="periodOptions"
        @change="updateDateRange"
    />

    <div class="date-range">
      <input
          type="date"
          v-model="localStartDate"
          @change="onStartDateChange"
          :max="maxStartDate"
          class="filter-input date-input"
      />
      <span class="date-range-separator"></span>
      <input
          type="date"
          v-model="localEndDate"
          @change="onEndDateChange"
          :min="minEndDate"
          :max="maxEndDate"
          class="filter-input date-input"
      />
    </div>
  </div>
</template>

<script setup>
import {ref, computed, watch} from 'vue';
import DropDown from "@/components/filter/DropDown.vue";
import {defineEmits, defineProps} from 'vue';

const props = defineProps(['startDate', 'endDate']);
const emit = defineEmits(['update:startDate', 'update:endDate']);
const selectedPeriod = ref('');
const localStartDate = ref(props.startDate);
const localEndDate = ref(props.endDate);

watch(
    () => props.startDate,
    (newVal) => {
      localStartDate.value = newVal;
    }
);

watch(
    () => props.endDate,
    (newVal) => {
      localEndDate.value = newVal;
    }
);

const periodOptions = [
  {label: 'Hoje', value: 'today'},
  {label: 'Última semana', value: 'lastWeek'},
  {label: 'Último mês', value: 'lastMonth'}
];

const minEndDate = computed(() => {
  return localStartDate.value ? localStartDate.value : null;
});

const maxEndDate = computed(() => {
  return new Date().toISOString().split('T')[0];
});

const maxStartDate = computed(() => {
  return new Date().toISOString().split('T')[0];
});

function updateDateRange() {
  const today = new Date().toISOString().split('T')[0];

  if (selectedPeriod.value === 'today') {
    emit('update:startDate', today);
    emit('update:endDate', today);
  }
  if (selectedPeriod.value === 'lastWeek') {
    const lastWeekStart = new Date();
    lastWeekStart.setDate(lastWeekStart.getDate() - 7 - lastWeekStart.getDay());
    const lastWeekEnd = new Date();
    lastWeekEnd.setDate(lastWeekEnd.getDate() - lastWeekEnd.getDay());
    emit('update:startDate', lastWeekStart.toISOString().split('T')[0]);
    emit('update:endDate', lastWeekEnd.toISOString().split('T')[0]);
  }
  if (selectedPeriod.value === 'lastMonth') {
    const lastMonthStart = new Date();
    lastMonthStart.setDate(1);
    lastMonthStart.setMonth(lastMonthStart.getMonth() - 1);
    const lastMonthEnd = new Date();
    lastMonthEnd.setDate(0);
    emit('update:startDate', lastMonthStart.toISOString().split('T')[0]);
    emit('update:endDate', lastMonthEnd.toISOString().split('T')[0]);
  }
}

function onStartDateChange() {
  selectedPeriod.value = '';
  emit('update:startDate', localStartDate.value);
}

function onEndDateChange() {
  selectedPeriod.value = '';
  emit('update:endDate', localEndDate.value);
}
</script>

<style scoped>
.date-range-container {
  margin-bottom: 16px;
}

.date-range {
  display: flex;
  align-items: center;
}

.date-input {
  width: calc(50% - 8px);
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background-color: #444444;
  color: #fff;
}

.date-range-separator {
  margin: 0 8px;
  color: #000;
}

.date-input::-webkit-calendar-picker-indicator {
  filter: invert(1);
}
</style>
