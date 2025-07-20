<script setup lang="ts">
import {
  DateFormatter,
  type DateValue,
  getLocalTimeZone,
} from '@internationalized/date'
import { CalendarIcon } from 'lucide-vue-next'

import { ref, watch } from 'vue'
import { cn } from '@/lib/utils'
import { Button } from '@/components/ui/button'
import { Calendar } from '@/components/ui/calendar'
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover'

const df = new DateFormatter('en-US', {
  dateStyle: 'long',
})

const props = defineProps<{ modelValue?: DateValue | string }>()
const emit = defineEmits(['update:modelValue'])

const value = ref<DateValue | undefined>()

// Sync prop to local value
watch(
  () => props.modelValue,
  (val) => {
    value.value = val as DateValue
  },
  { immediate: true }
)

// Emit when local value changes
watch(value, (val) => {
  emit('update:modelValue', val)
})
</script>

<template>
  <Popover>
    <PopoverTrigger as-child>
      <Button
        variant="outline"
        :class="cn(
          'w-full max-w-xs justify-start text-left font-normal',
          !value && 'text-muted-foreground',
        )"
      >
        <CalendarIcon class="mr-2 h-4 w-4" />
        {{ value ? df.format(value.toDate(getLocalTimeZone())) : "Pick a date" }}
      </Button>
    </PopoverTrigger>
    <PopoverContent class="w-full max-w-xs min-w-[220px] p-0" :align="'start'">
      <Calendar v-model="value" initial-focus />
    </PopoverContent>
  </Popover>
</template>