<script setup lang="ts">
import { computedAsync } from '@vueuse/core';
import QRCode from 'qrcode';
import { format, parseISO, setDay, addWeeks, isBefore } from 'date-fns';
import type { LocationQuery } from 'vue-router';


const error = ref("");

const route = useRoute();
const query = route.query;

// Shared function to fetch data from echo.lu API
async function fetchEchoData(query: LocationQuery) {
  let url = 'https://api.echo.lu/v1/allExperiences';

  const apiKey = query['api-key'] as string;

  if (!apiKey) {
    return [];
  }

  // Create new query parameters without timeRange but with from/to dates
  const params = new URLSearchParams();

  // Handle timeRange parameter
  const timeRange = query.timeRange as string;
  if (timeRange) {
    console.log(timeRange)
    const now = new Date();
    let fromDate = new Date();
    let toDate = new Date();

    // Remove brackets if present
    const cleanTimeRange = timeRange.replace(/[\[\]]/g, '');

    switch (cleanTimeRange) {
      case 'today':
        fromDate.setHours(0, 0, 0, 0);
        toDate.setHours(23, 59, 59, 999);
        break;
      case 'tomorrow':
        fromDate.setDate(now.getDate() + 1);
        fromDate.setHours(0, 0, 0, 0);
        toDate.setDate(now.getDate() + 1);
        toDate.setHours(23, 59, 59, 999);
        break;
      case 'weekend':
        // Find next Saturday
        const daysUntilWeekend = (6 - now.getDay() + 7) % 7;
        fromDate.setDate(now.getDate() + daysUntilWeekend);
        fromDate.setHours(0, 0, 0, 0);
        toDate = new Date(fromDate);
        toDate.setDate(fromDate.getDate() + 1); // Include Sunday
        toDate.setHours(23, 59, 59, 999);
        break;
      case 'week':
        fromDate.setHours(0, 0, 0, 0);
        toDate.setDate(now.getDate() + 6);
        toDate.setHours(23, 59, 59, 999);
        break;
      case 'next-week':
        fromDate.setDate(now.getDate() + 7);
        fromDate.setHours(0, 0, 0, 0);
        toDate.setDate(fromDate.getDate() + 6);
        toDate.setHours(23, 59, 59, 999);
        break;
      case 'month':
        fromDate.setHours(0, 0, 0, 0);
        toDate = new Date(now.getFullYear(), now.getMonth() + 1, 0, 23, 59, 59, 999);
        break;
    }

    params.set('date[from]', fromDate.toISOString().split('.')[0] + 'Z');
    params.set('date[to]', toDate.toISOString().split('.')[0] + 'Z');
  }
  // Add all other query parameters except api-key and timeRange
  Object.entries(query).forEach(([key, value]) => {
    if (key !== 'api-key' && key !== 'timeRange' && key !== 'interval') {
      if (Array.isArray(value)) {
        value.forEach(v => params.append(key, v as string));
      } else if (typeof value === 'string' && value.startsWith('[') && value.endsWith(']')) {
        const values = value.slice(1, -1).split(',');
        values.forEach(v => params.append(key, v.trim()));
      } else {
        params.append(key, value as string);
      }
    }
  });

  if (params.toString()) {
    url += '?' + params.toString();
  }


  const options = {
    method: 'GET',
    headers: { 'api-key': apiKey, Accept: 'application/json' }
  };

  const response = await fetch(url, options);
  const data = await response.json();

  return data.records.map((record: any) => {
    const dateFrom = record.dates[0]?.from
      ? new Date(record.dates[0].from)
      : new Date();

    const dateTo = record.dates[0]?.to
      ? new Date(record.dates[0].to)
      : new Date();

    const openingHours = record.dates[0]?.openingHours;

    let dateString;
    let date;

    if (openingHours) {
      // Handle recurring events with opening hours
      const now = new Date();
      const daysOfWeek = ['sunday', 'monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday'];
      let nextDate = null;
      let nextTime = null;

      // Find the next available day
      for (let i = 0; i < 7; i++) {
        const checkDate = addWeeks(now, 0); // Start from current week

        // Try each day of the week
        for (const [index, day] of daysOfWeek.entries()) {
          if (openingHours[day]) {
            let possibleDate = setDay(checkDate, index);
            // If today's date is past today's event time, start from next week
            if (index === now.getDay() &&
              now.getHours() > parseInt(openingHours[day].from.split(':')[0])) {
              possibleDate = addWeeks(possibleDate, 1);
            }
            // If the day is earlier in the week than today, move to next week
            if (index < now.getDay()) {
              possibleDate = addWeeks(possibleDate, 1);
            }

            // Check if this date is within the event's overall range
            if (!isBefore(possibleDate, parseISO(record.dates[0].from)) &&
              !isBefore(parseISO(record.dates[0].to), possibleDate)) {
              if (nextDate === null || isBefore(possibleDate, nextDate)) {
                nextDate = possibleDate;
                nextTime = openingHours[day].from;
              }
            }
          }
        }
      }

      if (nextDate && nextTime) {
        const dayFormatted = format(nextDate, 'EEEE');
        const dateFormatted = format(nextDate, 'dd/MM/yyyy');
        dateString = `${dayFormatted}, ${dateFormatted}, ${nextTime}`;
        date = nextDate;
      } else {
        // Fallback if no valid next date is found
        const startDate = format(dateFrom, 'dd/MM/yyyy');
        const endDate = format(dateTo, 'dd/MM/yyyy');
        dateString = `From ${startDate} to ${endDate}`;
        date = dateFrom;
      }
    } else {
      // Original date formatting logic for non-recurring events
      const timeDiff = dateTo.getTime() - dateFrom.getTime();
      const moreThan24Hours = timeDiff > 24 * 60 * 60 * 1000;

      const day = format(dateFrom, 'EEEE');
      const startDate = format(dateFrom, 'dd/MM/yyyy');
      const endDate = format(dateTo, 'dd/MM/yyyy');
      const startTime = format(dateFrom, 'HH:mm');
      const endTime = format(dateTo, 'HH:mm');

      dateString = moreThan24Hours
        ? `From ${startDate} to ${endDate}`
        : startDate === endDate
          ? `${day}, ${startDate}, ${startTime}`
          : `${day}, ${startDate}, ${startTime}-${endTime}`;
      date = dateFrom;
    }

    const id = record.id;
    const venue = record.venues[0];
    const addr = venue?.location?.address;
    const address = addr?.number + ' ' + addr?.street + ', ' + addr?.postcode + ' ' + addr?.town;
    const imageUrl = record.pictures[0]?.previews?.media?.url || '';
    const title = record.title.en || record.title.fr || record.title.de || 'Event';

    const salesContact = record.salesContact;
    const contactName = salesContact?.name;
    const phone = salesContact?.phone;
    const email = salesContact?.email;
    const website = salesContact?.website;

    const initialDescription = record.description.en || record.description.fr || record.description.de || '';

    let rawDescription = initialDescription;

    rawDescription = rawDescription.replace(/<style[^>]*>[\s\S]*?<\/style>/gi, '');

    // Extract text between <p> tags using regex
    rawDescription = rawDescription.match(/<p>(.*?)<\/p>/g)?.map((match: string) => {
      // Extract content between <p> tags and remove any nested HTML
      return match.replace(/<p>(.*?)<\/p>/, '$1').replace(/<[^>]*>/g, '');
    })?.join('\n\n') || '';

    // Extract only text content, removing all HTML tags
    rawDescription = rawDescription.replace(/<[^>]*>/g, '')
      .trim()
      .replace(/\s+/g, ' '); // Collapse multiple spaces into single space


    // Replace zero-width space characters with <br> tags
    rawDescription = rawDescription.replace(/\u{FEFF}/gu, '<br/><br/>');

    // Replace &nbsp; with regular spaces
    rawDescription = rawDescription.replace(/&nbsp;/g, ' ');

    const description = rawDescription

    // Convert relative image URLs to absolute
    const absoluteImageUrl = imageUrl
      ? (imageUrl.startsWith('http') ? imageUrl : `https://echo.lu${imageUrl}`)
      : '';

    return {
      id,
      url: `https://www.echo.lu/en/experiences/${id}`,
      title,
      description,
      date,
      dateString,
      address,
      imageUrl,
      absoluteImageUrl,
      initialDescription,
      contactName,
      phone,
      email,
      website
    }
  }).sort((a: any, b: any) => {
    return a.date.getTime() - b.date.getTime();
  }).filter((event: any) => {
    return event.absoluteImageUrl !== '';
  });
}

const events = computedAsync(
  () => fetchEchoData(query),
  [], // initial state
)

const currentIndex = ref(0);

const currentEvent = computed(() => events.value?.[currentIndex.value % events.value.length] ?? null);

const afterCurrentEvent = computed(() => events.value?.[(currentIndex.value + 1) % events.value.length] ?? null);

const beforeCurrentEvent = computed(() => events.value?.[(currentIndex.value - 1 + events.value.length) % events.value.length] ?? null);


const nextEvent = () => {
  currentIndex.value = (currentIndex.value + 1) % events.value.length;
  preloadImage(afterCurrentEvent.value?.absoluteImageUrl);
};

const previousEvent = () => {
  currentIndex.value = (currentIndex.value - 1 + events.value.length) % events.value.length;
  preloadImage(beforeCurrentEvent.value?.absoluteImageUrl);
};

const preloadImage = (url: string) => {
  const img = new Image();
  img.src = url;
};

onMounted(() => {
  setInterval(nextEvent, query.interval ? parseInt(query.interval as string) * 1000 : 15000);
});

const onKeyDown = (e: KeyboardEvent) => {
  if (e.key === 'ArrowRight') {
    nextEvent();
  } else if (e.key === 'ArrowLeft') {
    previousEvent();
  }
};

onMounted(() => {
  window.addEventListener('keydown', onKeyDown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', onKeyDown);
});

const qrCodeUrl = ref<string>('');

// Watch for changes in the website URL and generate QR code
watch(
  () => currentEvent.value?.url,
  async (url) => {
    if (url) {
      qrCodeUrl.value = await QRCode.toDataURL(url);
    } else {
      qrCodeUrl.value = '';
    }
  },
  { immediate: true }
);

</script>
<template>
  <div class="min-h-screen w-full flex items-center justify-center bg-black text-white relative">
    <div v-if="currentEvent" class="max-w-[1080px] aspect-9/16 flex flex-col overflow-hidden relative">
      <div class="absolute bottom-2 right-2 flex items-center gap-2">
        <div>Source:</div>
        <img src="/echolu.png" alt="Echo.lu Website" class="w-10 h-10" />
      </div>
      <div class="mx-auto relative">
        <img src="/header.png" alt="Mir sin uewen" />
        <img v-if="qrCodeUrl" :src="qrCodeUrl" alt="Event QR Code" class="absolute top-10 right-50 w-48 h-48" />
      </div>
      <div class="mx-auto mt-10 px-10 h-[500px]">
        <img v-if="currentEvent.absoluteImageUrl" :src="currentEvent.absoluteImageUrl" alt="Event image"
          class="max-h-[500px]" />
        <div v-else class="h-[500px]"></div>
      </div>
      <div class="h-full w-full mx-auto mt-10 flex flex-col overflow-hidden p-10">
        <div class="text-7xl">{{ currentEvent.dateString }}</div>
        <div class="text-5xl mt-10">{{ currentEvent.title }}</div>
        <div class="text-2xl mt-10 h-[290px] line-clamp-9 overflow-hidden">{{ currentEvent.description }}</div>
        <!-- <div class="text-2xl mt-10 h-[290px]">{{ currentEvent.initialDescription }}</div> -->
        <div class="text-7xl mt-10">Organizer Information</div>
        <div class="text-4xl mt-10" v-if="currentEvent.address">{{ currentEvent.address }}</div>
        <div class="text-4xl mt-10" v-if="currentEvent.contactName">{{ currentEvent.contactName }}</div>
        <div class="text-2xl" v-if="currentEvent.phone">{{ currentEvent.phone }}</div>
        <div class="text-2xl" v-if="currentEvent.email">{{ currentEvent.email }}</div>
        <div class="text-2xl" v-if="currentEvent.website">{{ currentEvent.website }}</div>
      </div>
    </div>
    <div v-else class="text-red-500 text-7xl">{{ error }}</div>
  </div>
</template>