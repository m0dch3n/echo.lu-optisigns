<script setup lang="ts">
import { computedAsync } from '@vueuse/core';
import QRCode from 'qrcode';
import { format, parseISO, setDay, addWeeks, isBefore } from 'date-fns';
import { fr, de, enUS } from 'date-fns/locale';
import type { LocationQuery } from 'vue-router';

type Event = {
  id: string;
  url: string;
  title: string;
  description: string;
  date: Date;
  dateString: string;
  address: string;
  imageUrl: string;
  absoluteImageUrl: string;
  initialDescription: string;
  contactName?: string;
  phone?: string;
  email?: string;
  website?: string;
  record: any; // The raw record data from the API
  upcomingEvents?: string;
  organizerInformation?: string;
};

const route = useRoute();
const query = route.query;

// Shared function to fetch data from echo.lu API
async function fetchEchoData(query: LocationQuery): Promise<Event[]> {
  let url = 'https://api.echo.lu/v1/allExperiences';

  const apiKey = query['api-key'] as string;

  if (!apiKey) {
    return [];
  }

  const params = new URLSearchParams();

  // Handle timeRange parameter, translate it to from/to dates
  const timeRange = query.timeRange as string;
  if (timeRange) {
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

  // Add all other query parameters except api-key, timeRange, interval, currentIndex
  // they serve us for internal usage
  const toIgnore = ['api-key', 'timeRange', 'interval', 'currentIndex', 'raw', 'lang'];
  Object.entries(query).forEach(([key, value]) => {
    if (!toIgnore.includes(key)) {
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

  // construct the url with the query params
  if (params.toString()) {
    url += '?' + params.toString();
  }

  // pass the api key from the query params
  const options = {
    method: 'GET',
    headers: { 'api-key': apiKey, Accept: 'application/json' }
  };

  // fetch the data with the query params
  const response = await fetch(url, options);
  const data = await response.json();

  const lang = query.lang as string || 'fr';
  
  const locales = {
    fr: fr,
    de: de,
    en: enUS
  };

  // map the data records to the events
  return data.records.map((record: any) => {
    const now = new Date();

    // get date information from the record
    let dateFrom, dateTo, openingHours;

    for (const dateObject of record.dates) {
      dateFrom = new Date(dateObject.from);
      dateTo = new Date(dateObject.to);
      openingHours = dateObject.openingHours;
      if (dateFrom >= now || record.dates.length === 1) {
        break;
      }
    } 

    if (!dateFrom || !dateTo) {
      dateFrom = now;
      dateTo = now;
      openingHours = null;
    }

    let dateString;
    let date;

    // handle recurring events with opening hours
    if (openingHours) {
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

      // if we have a valid next date and time, format the date string
      if (nextDate && nextTime) {
        const dayFormatted = format(nextDate, 'EEEE', { locale: locales[lang as keyof typeof locales] });
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
      // Date formatting for non-recurring events
      const timeDiff = dateTo.getTime() - dateFrom.getTime();
      const moreThan24Hours = timeDiff > 24 * 60 * 60 * 1000;

      const day = format(dateFrom, 'EEEE', { locale: locales[lang as keyof typeof locales] });
      const startDate = format(dateFrom, 'dd/MM/yyyy');
      const endDate = format(dateTo, 'dd/MM/yyyy');
      const startTime = format(dateFrom, 'HH:mm');
      const endTime = format(dateTo, 'HH:mm');

      // if the event is more than 24 hours, show the start and end date
      // otherwise show the start date and time
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
    const number = addr?.number ?? '';
    const street = addr?.street ?? '';
    const postcode = addr?.postcode ?? '';
    const town = addr?.town ?? '';
    const address = number + ' ' + street + ', ' + postcode + ' ' + town;
    const imageUrl = record.pictures[0]?.previews?.media?.url || '';
    const title = record.title[lang] || record.title.fr || record.title.de || record.title.en || 'Événement';

    const upcomingEventsTranslation = {
      fr: 'Événements à venir',
      de: 'Kommende Ereignisse',
      en: 'Upcoming Events',
    };

    const organizerTranslation = {
      fr: 'Informations sur l\'organisateur',
      de: 'Infos zum Organisator',
      en: 'Organizer Information',
    };

    const upcomingEvents = upcomingEventsTranslation[lang as keyof typeof upcomingEventsTranslation];
    const organizerInformation = organizerTranslation[lang as keyof typeof organizerTranslation];
    
    const salesContact = record.salesContact;
    const contactName = salesContact?.name;
    const phone = salesContact?.phone;
    const email = salesContact?.email;
    const website = salesContact?.website;

    // prefer the english description
    const initialDescription = record.description[lang] || record.description.fr || record.description.de || record.description.en || '';

    let rawDescription = initialDescription;

    // remove the style tags
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
      website,
      record,
      upcomingEvents,
      organizerInformation
    }
    // sort the events by date ascending
  }).sort((a: any, b: any) => {
    return a.date.getTime() - b.date.getTime();
  }).filter((event: any) => {
    // filter out events without an image
    return event.absoluteImageUrl !== '';
  });
}

const events = computedAsync(
  () => fetchEchoData(query),
  [], // initial state
)

const currentIndex = ref(query.currentIndex ? parseInt(query.currentIndex as string) : 0);

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
  // if the interval is set from the query params, use it, otherwise use 15 seconds
  setInterval(nextEvent, query.interval ? parseInt(query.interval as string) * 1000 : 15000);
});

// handle left and right arrow key events
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

// generate the QR code url
const qrCodeUrl = computedAsync(
  () => QRCode.toDataURL(currentEvent.value?.url ?? ''),
  undefined,
);

const showRaw = ref(query.raw === 'true');

</script>
<template>
  <div
    class="min-h-screen w-full flex items-center justify-center bg-black text-white relative overflow-hidden overscroll-none">
    <template v-if="currentEvent">
      <template v-if="showRaw">
        <pre class="w-full h-full overflow-scroll">{{ currentIndex }}/{{ events.length - 1 }}
{{ currentEvent.record }}
        </pre>
      </template>
      <template v-else>
        <div class="max-w-[1080px] aspect-9/16 flex flex-col overflow-hidden relative">
          <div class="absolute bottom-2 right-2 flex items-center gap-2">
            <div>Source:</div>
            <img src="/echolu.png" alt="Echo.lu Website" class="w-10 h-10" />
          </div>
          <div class="mx-auto relative">
            <img src="/header.png" alt="Mir sin uewen" />
            <img v-if="qrCodeUrl" :src="qrCodeUrl" alt="Event QR Code" class="absolute top-8 right-4 w-64 h-64" />
            <div class="absolute top-8 right-4 w-64 h-64" >{{ currentEvent.upcomingEvents }}</div>
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
            <div class="text-7xl mt-10">{{ currentEvent.organizerInformation}}</div>
            <div class="text-4xl mt-10" v-if="currentEvent.address">{{ currentEvent.address }}</div>
            <div class="text-4xl mt-10" v-if="currentEvent.contactName">{{ currentEvent.contactName }}</div>
            <div class="text-2xl" v-if="currentEvent.phone">{{ currentEvent.phone }}</div>
            <div class="text-2xl" v-if="currentEvent.email">{{ currentEvent.email }}</div>
            <div class="text-2xl" v-if="currentEvent.website">{{ currentEvent.website }}</div>
          </div>
        </div>
      </template>
    </template>
  </div>
</template>