<script setup lang="ts">
import { ref, computed } from "vue";
import cronParser from "cron-parser";
import dayjs from "dayjs";

function getRandomInt(min: number, max: number) {
  min = Math.ceil(min);
  max = Math.floor(max);
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

function getRandomCronExpression() {
  const minute = getRandomInt(0, 59);
  const hour = getRandomInt(0, 23);
  const dayOfMonth = getRandomInt(1, 31);
  const month = getRandomInt(1, 12);
  const dayOfWeek = getRandomInt(0, 6);

  return `${minute} ${hour} ${dayOfMonth} ${month} ${dayOfWeek}`;
}

const cronExpression = ref(getRandomCronExpression());

const handleRandomCron = () => {
  cronExpression.value = getRandomCronExpression();
}

const errorMsg = ref('')
const invalidCron = ref(false)

const fields = computed(() => {
  try {
    const interval = cronParser.parseExpression(cronExpression.value);
    invalidCron.value = false
    errorMsg.value = ''
    return JSON.parse(JSON.stringify(interval.fields));
  } catch (err) {
    console.log(err)
    invalidCron.value = true
    errorMsg.value = String(err)
    console.warn(err);
  }
});

const addPreZero = (n: number) => {
  if (n < 10) {
    return `0${n}`;
  }
  return n + "";
};

function getMonthName(monthNumber: number) {
  const date = new Date(2020, monthNumber - 1);
  return date.toLocaleString("en-GB", { month: "long" });
}
function getDayName(dayNumber: number) {
  const date = new Date(2020, 0, dayNumber);
  return date.toLocaleString("en-GB", { weekday: "long" });
}

const humanReadable = computed(() => {
  if (fields.value) {
    const { minute, hour, dayOfWeek, dayOfMonth, month } = fields.value;
    const result = [] as string[];
    if (minute.length === 1 && hour.length === 1) {
      result.push(
        `<span data-seg="hour">${addPreZero(
          hour[0]
        )}</span>:<span data-seg="minute">${addPreZero(minute[0])}</span>`
      );
    } else {
      result.push(
        `<span data-seg="minute">minute ${minute.join(" and ")}</span> past`
      );
      result.push(`<span data-seg="hour">hour ${hour.join(" and ")}</span>`);
    }

    if (dayOfMonth.length < 31) {
      result.push(
        `on <span data-seg="day-of-month">day-of-month ${dayOfMonth.join(
          " and "
        )}</span>`
      );
    }

    result.push(
      `<span data-seg="month"> in ${month
        .map((m) => getMonthName(m))
        .join(" and ")}</span>`
    );

    if (dayOfWeek.length < 7) {
      result.push(
        `and <span data-seg="day-of-week">on ${dayOfWeek
          .map((d) => getDayName(d))
          .join(" and ")}</span>`
      );
    }

    return '"At ' + result.join(" ") + '"';
  }
});
const timeSpan = computed(() => {
  if (invalidCron.value) {
    return undefined;
  }
  try {
    const interval = cronParser.parseExpression(cronExpression.value);
    return Array(5)
      .fill(0)
      .map(() => {
        const obj = interval.next();
        return dayjs(obj.toString()).format("YYYY/MM/DD HH:mm");
      });
  } catch (err) {
    console.error("Invalid expression: " + err);
  }
});

const cursorPosition = ref(-1);
const handleClick = (e: MouseEvent) => {
  const target = e.target as HTMLInputElement;
  cursorPosition.value = target.selectionStart ?? -1;
};
const handleBlur = () => {
  cursorPosition.value = -1;
};

const keys = ["minute", "hour", "day-of-month", "month", "day-of-week"];

const activeField = computed(() => {
  if (cursorPosition.value < 0) {
    return undefined;
  }
  if (cursorPosition.value === 0) {
    return keys[0];
  }
  if (cursorPosition.value >= cronExpression.value.length) {
    return keys[keys.length - 1];
  }
  let pos = 0;
  const posMap = cronExpression.value
    .split(" ")
    .map((field, index) => ({ key: keys[index], field }))
    .reduce((a, c) => {
      a[c.key] = { from: pos, end: pos + c.field.length };
      pos += c.field.length + 1;
      return a;
    }, {} as Record<string, { from: number; end: number }>);

  for (const p of Array.from(Object.entries(posMap))) {
    const [key, { from, end }] = p;
    if (cursorPosition.value >= from && cursorPosition.value <= end) {
      return key;
    }
  }
});
</script>

<template>
  <main class="mt-12 p-4">
    <v-sheet max-width="300" class="mx-auto">
      <p
        v-html="humanReadable ?? '\u2000'"
        :data-focused-seg="activeField"
        class="human-readable text-lg italic bg-slate-800 text-slate-50 p-2 rounded-md"
      ></p>
      <v-divider></v-divider>
      <div class="mt-4 flex flex-col gap-2">
        <v-text-field
          v-model="cronExpression"
          @click="handleClick"
          @blur="handleBlur"
          :error="invalidCron"
          :error-messages="invalidCron ? [errorMsg] : []"
        ></v-text-field
        ><v-btn small @click="handleRandomCron"> random </v-btn>
      </div>
    </v-sheet>

    <v-divider class="mt-4"></v-divider>

    <v-card
      class="mx-auto mt-4"
      max-width="300"
    >
      <v-list lines="one">
        <v-list-item v-for="span in timeSpan" :title="span"> </v-list-item>
      </v-list>
    </v-card>
  </main>
</template>
<style lang="sass">
.human-readable
  --color-highlight: gold
  &[data-focused-seg="minute"]
    span[data-seg="minute"]
      color: var(--color-highlight)
  &[data-focused-seg="hour"]
    span[data-seg="hour"]
      color: var(--color-highlight)
  &[data-focused-seg="day-of-month"]
    span[data-seg="day-of-month"]
      color: var(--color-highlight)
  &[data-focused-seg="month"]
    span[data-seg="month"]
      color: var(--color-highlight)
  &[data-focused-seg="day-of-week"]
    span[data-seg="day-of-week"]
      color: var(--color-highlight)
</style>
