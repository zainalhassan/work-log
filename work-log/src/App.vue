<script setup lang="ts">
import { ref } from 'vue'
import DatePicker from './components/ui/DatePicker.vue'
import { Input } from './components/ui/input'
import { Button } from './components/ui/button'
import { Card, CardHeader, CardContent, CardFooter, CardTitle, CardDescription } from './components/ui/card'
import { getLocalTimeZone, DateFormatter } from '@internationalized/date'

interface PersonEntry {
  name: string
  duration: string
  miles: string
}

interface DateEntry {
  date: string
  people: PersonEntry[]
}

const entries = ref<DateEntry[]>([])
const currentDate = ref<any>(undefined)
const people = ref<PersonEntry[]>([])
const currentPerson = ref<PersonEntry>({ name: '', duration: '', miles: '' })
const showSummary = ref(false)
const df = new DateFormatter('en-US', { dateStyle: 'long' })
const errors = ref<string[]>([])

function parseDuration(duration: string): number {
  // Returns total minutes, supports '1 Hour', '30 Mins', '1 Hour 30 Mins', '2 Hours', '45 Min', etc.
  let total = 0
  if (!duration) return 0
  // Match hours (e.g., 1 Hour, 2 Hours)
  const hourMatch = duration.match(/(\d+)\s*hour(s)?/i)
  if (hourMatch) total += parseInt(hourMatch[1], 10) * 60
  // Match mins (e.g., 30 Min, 45 Mins)
  const minMatch = duration.match(/(\d+)\s*min(s)?/i)
  if (minMatch) total += parseInt(minMatch[1], 10)
  // If only a number is provided, treat as minutes
  if (!hourMatch && !minMatch) {
    const justNumber = duration.match(/^(\d+)$/)
    if (justNumber) total += parseInt(justNumber[1], 10)
  }
  return total
}

function addPerson() {
  errors.value = []
  if (!currentPerson.value.name) errors.value.push('Person name is required.')
  if (!currentPerson.value.duration) errors.value.push('Duration is required.')
  if (currentPerson.value.miles && isNaN(Number(currentPerson.value.miles))) errors.value.push('Miles must be a number.')
  if (errors.value.length === 0) {
    people.value.push({ ...currentPerson.value })
    currentPerson.value = { name: '', duration: '', miles: '' }
  }
}

function addDate() {
  errors.value = []
  if (!currentDate.value) errors.value.push('Date is required.')
  if (people.value.length === 0) errors.value.push('At least one person is required for this date.')
  if (errors.value.length > 0) return
  if (currentDate.value && people.value.length) {
    let dateVal = currentDate.value
    if (dateVal && typeof dateVal.toDate === 'function') {
      const dateStr = df.format(dateVal.toDate(getLocalTimeZone()))
      entries.value.push({
        date: dateStr,
        people: [...people.value],
      })
    }
    currentDate.value = undefined
    people.value = []
    currentPerson.value = { name: '', duration: '', miles: '' }
  }
}

function submit() {
  errors.value = []
  if (currentPerson.value.name || currentPerson.value.duration || currentPerson.value.miles) {
    addPerson()
    if (errors.value.length > 0) return
  }
  addDate()
  if (errors.value.length > 0) return
  showSummary.value = true
}

function reset() {
  entries.value = []
  currentDate.value = undefined
  people.value = []
  currentPerson.value = { name: '', duration: '', miles: '' }
  showSummary.value = false
  errors.value = []
}

function download() {
  const data = entries.value.map(e => {
    const peopleStr = e.people.map(p => `${p.name} - ${p.duration}${p.miles ? ` (${p.miles} Miles)` : ''}`).join('\n')
    return `${e.date}\n${peopleStr}`
  }).join('\n\n')
  const totals = getGrandTotals()
  const totalsStr = `\n\nGrand Totals:\nTotal Hours: ${totals.hours} Hours ${totals.mins} Mins\nTotal Miles: ${totals.miles} Miles`;
  const fullText = data + totalsStr;
  const blob = new Blob([fullText], { type: 'text/plain' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'work-log.txt'
  a.click()
  URL.revokeObjectURL(url)
}

function getGrandTotals() {
  let totalMinutes = 0
  let totalMiles = 0
  for (const entry of entries.value) {
    for (const p of entry.people) {
      totalMinutes += parseDuration(p.duration)
      if (p.miles && !isNaN(Number(p.miles))) {
        totalMiles += Number(p.miles)
      }
    }
  }
  const hours = Math.floor(totalMinutes / 60)
  const mins = totalMinutes % 60
  return { hours, mins, miles: totalMiles }
}
</script>

<template>
  <div class="min-h-screen w-full flex items-center justify-center bg-gradient-to-br from-neutral-100 to-neutral-200 dark:from-neutral-900 dark:to-neutral-800">
    <main class="w-full max-w-2xl lg:max-w-3xl mx-auto animate-fade-in px-2 md:px-8 lg:px-16">
      <Card v-if="!showSummary" class="shadow-xl border-0 bg-white/95 dark:bg-zinc-900/95 rounded-2xl p-6 md:p-12 lg:p-16 w-full">
        <CardHeader class="pb-4 border-b border-zinc-200 dark:border-zinc-700 mb-8">
          <CardTitle class="text-3xl font-extrabold text-zinc-900 dark:text-white tracking-tight mb-1">Work Log Entry</CardTitle>
          <CardDescription class="text-lg text-zinc-500 dark:text-zinc-300">Enter your work log details for each date and person.</CardDescription>
        </CardHeader>
        <CardContent class="space-y-6">
          <div v-if="errors.length" class="mb-2 text-red-600 bg-red-50 border border-red-200 rounded-lg p-3">
            <ul class="list-disc pl-5 text-base">
              <li v-for="(err, i) in errors" :key="i">{{ err }}</li>
            </ul>
          </div>
          <div>
            <label class="block mb-2 font-semibold text-zinc-700 dark:text-zinc-200 text-lg">Date</label>
            <DatePicker v-model="currentDate" />
          </div>
          <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
            <div>
              <label class="block mb-2 font-semibold text-zinc-700 dark:text-zinc-200">Person</label>
              <Input v-model="currentPerson.name" placeholder="e.g. SA, JJ, etc." class="h-12 text-lg" />
            </div>
            <div>
              <label class="block mb-2 font-semibold text-zinc-700 dark:text-zinc-200">Duration</label>
              <Input v-model="currentPerson.duration" placeholder="e.g. 1 Hour 30 Mins" class="h-12 text-lg" />
            </div>
            <div>
              <label class="block mb-2 font-semibold text-zinc-700 dark:text-zinc-200">Miles</label>
              <Input v-model="currentPerson.miles" placeholder="Miles (optional)" class="h-12 text-lg" />
            </div>
          </div>
          <br>
          <div class="flex flex-col md:flex-row gap-3 md:gap-6 mt-6">
            <Button class="w-full md:w-auto h-12 text-lg font-semibold" @click="addPerson">Add Person</Button>
            <Button class="w-full md:w-auto h-12 text-lg font-semibold" variant="secondary" @click="addDate">Add Another Date</Button>
            <Button class="w-full md:w-auto h-12 text-lg font-semibold bg-green-600 hover:bg-green-700 text-white border-0" @click="submit">Submit</Button>
          </div>
          <div v-if="people.length" class="mt-6 bg-zinc-50 dark:bg-zinc-800 rounded-xl p-4 border border-zinc-200 dark:border-zinc-700">
            <h4 class="font-bold mb-3 text-zinc-800 dark:text-zinc-100 text-lg tracking-tight">People for this date:</h4>
            <ul class="space-y-2">
              <li v-for="(p, i) in people" :key="i" class="text-zinc-700 dark:text-zinc-200 text-base">{{ p.name }} - {{ p.duration }}<span v-if="p.miles"> ({{ p.miles }} Miles)</span></li>
            </ul>
          </div>
        </CardContent>
      </Card>
      <Card v-else class="shadow-xl border-0 bg-white/95 dark:bg-zinc-900/95 rounded-2xl p-6 md:p-12 lg:p-16 w-full">
        <CardHeader class="pb-4 border-b border-zinc-200 dark:border-zinc-700 mb-8">
          <CardTitle class="text-3xl font-extrabold text-zinc-900 dark:text-white tracking-tight mb-1">Summary</CardTitle>
          <CardDescription class="text-lg text-zinc-500 dark:text-zinc-300">Review your work log entries below.</CardDescription>
        </CardHeader>
        <CardContent class="space-y-6">
          <div v-for="(entry, idx) in entries" :key="idx" class="mb-6 bg-zinc-50 dark:bg-zinc-800 rounded-xl p-4 border border-zinc-200 dark:border-zinc-700">
            <div class="font-bold text-zinc-800 dark:text-zinc-100 text-lg mb-2">{{ entry.date }}</div>
            <ul class="space-y-2">
              <li v-for="(p, i) in entry.people" :key="i" class="text-zinc-700 dark:text-zinc-200 text-base">{{ p.name }} - {{ p.duration }}<span v-if="p.miles"> ({{ p.miles }} Miles)</span></li>
            </ul>
          </div>
          <div class="mt-6 p-4 bg-blue-50 dark:bg-blue-900/40 rounded-xl border border-blue-200 dark:border-blue-700">
            <div class="font-extrabold text-blue-900 dark:text-blue-200 text-lg mb-1">Grand Totals:</div>
            <div class="text-blue-900 dark:text-blue-200 text-base">
              Total Hours: {{ getGrandTotals().hours }} Hours {{ getGrandTotals().mins }} Mins
            </div>
            <div class="text-blue-900 dark:text-blue-200 text-base">
              Total Miles: {{ getGrandTotals().miles }} Miles
            </div>
          </div>
        </CardContent>
        <CardFooter class="flex flex-col md:flex-row gap-3 md:gap-6 mt-4">
          <Button class="w-full md:w-auto h-12 text-lg font-semibold" variant="secondary" @click="reset">Start Over</Button>
          <Button class="w-full md:w-auto h-12 text-lg font-semibold" variant="default" @click="download">Download</Button>
        </CardFooter>
      </Card>
    </main>
  </div>
</template>

<style scoped>
@keyframes fade-in {
  from { opacity: 0; transform: translateY(24px); }
  to { opacity: 1; transform: none; }
}
.animate-fade-in {
  animation: fade-in 0.7s cubic-bezier(.4,0,.2,1);
}
</style>

