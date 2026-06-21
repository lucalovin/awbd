<script setup>
import { computed, onMounted, ref } from 'vue';
import { useAuthStore } from '../stores/auth';

const auth = useAuthStore();

const exhibitions = ref([]);
const loading = ref(false);
const error = ref('');

const apiBaseUrl = import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080/api';

const dashboardTitle = computed(() => {
  return auth.isAdmin ? 'Gallery dashboard' : 'Gallery exhibitions';
});

const dashboardDescription = computed(() => {
  return auth.isAdmin
    ? 'Quick overview of gallery exhibitions and operational status.'
    : 'Browse gallery exhibitions and see what is currently available.';
});

const orderedExhibitions = computed(() => {
  return [...exhibitions.value].sort((a, b) => {
    const statusOrder = {
      live: 1,
      scheduled: 2,
      closed: 3,
    };

    const aStatus = getExhibitionStatus(a).key;
    const bStatus = getExhibitionStatus(b).key;

    if (statusOrder[aStatus] !== statusOrder[bStatus]) {
      return statusOrder[aStatus] - statusOrder[bStatus];
    }

    return new Date(a.startDate || 0) - new Date(b.startDate || 0);
  });
});

function toDate(value) {
  if (!value) {
    return null;
  }

  const date = new Date(value);
  date.setHours(0, 0, 0, 0);

  return date;
}

function getToday() {
  const today = new Date();
  today.setHours(0, 0, 0, 0);

  return today;
}

function getExhibitionStatus(exhibition) {
  const today = getToday();
  const startDate = toDate(exhibition.startDate);
  const endDate = toDate(exhibition.endDate);

  if (startDate && endDate && today >= startDate && today <= endDate) {
    return {
      key: 'live',
      label: 'LIVE',
    };
  }

  if (startDate && today < startDate) {
    return {
      key: 'scheduled',
      label: 'SCHEDULED',
    };
  }

  return {
    key: 'closed',
    label: 'CLOSED',
  };
}

function formatDate(value) {
  if (!value) {
    return 'Not set';
  }

  return new Intl.DateTimeFormat('en-GB', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  }).format(new Date(value));
}

function extractExhibitions(payload) {
  if (Array.isArray(payload)) {
    return payload;
  }

  if (Array.isArray(payload?.content)) {
    return payload.content;
  }

  if (Array.isArray(payload?.data)) {
    return payload.data;
  }

  if (Array.isArray(payload?.data?.content)) {
    return payload.data.content;
  }

  return [];
}

async function loadExhibitions() {
  loading.value = true;
  error.value = '';

  try {
    const response = await fetch(`${apiBaseUrl}/exhibitions?page=0&size=20&sort=startDate,asc`, {
      credentials: 'include',
    });

    if (!response.ok) {
      throw new Error(`Request failed with status ${response.status}`);
    }

    const payload = await response.json();
    exhibitions.value = extractExhibitions(payload);
  } catch (err) {
    error.value = 'Exhibitions could not be loaded from the backend.';
    exhibitions.value = [];
  } finally {
    loading.value = false;
  }
}

onMounted(() => {
  loadExhibitions();
});
</script>

<template>
  <section class="dashboard-hero simple">
    <div>
      <h1>{{ dashboardTitle }}</h1>
      <p class="hero-text">{{ dashboardDescription }}</p>
    </div>
  </section>

  <section class="exhibition-status-board">
    <div class="status-board-header">
      <div>
        <p class="eyebrow">Exhibitions</p>
        <h2>Gallery schedule</h2>
      </div>

      <span class="status-board-count">
        {{ orderedExhibitions.length }} exhibitions
      </span>
    </div>

    <div v-if="loading" class="empty compact">
      Loading exhibitions...
    </div>

    <div v-else-if="error" class="alert alert-error">
      {{ error }}
    </div>

    <div v-else-if="orderedExhibitions.length" class="exhibition-status-list">
      <RouterLink
        v-for="exhibition in orderedExhibitions"
        :key="exhibition.id"
        class="exhibition-status-card"
        :class="`status-${getExhibitionStatus(exhibition).key}`"
        :to="{ name: 'exhibition-details', params: { id: exhibition.id } }"
      >
        <div class="status-card-main">
          <span
            class="status-chip"
            :class="`status-chip-${getExhibitionStatus(exhibition).key}`"
          >
            {{ getExhibitionStatus(exhibition).label }}
          </span>

          <h3>{{ exhibition.title }}</h3>

          <p v-if="exhibition.description">
            {{ exhibition.description }}
          </p>

          <p v-else class="muted">
            No description available.
          </p>
        </div>

        <div class="status-card-meta">
          <div>
            <span>Period</span>
            <strong>
              {{ formatDate(exhibition.startDate) }}
              —
              {{ formatDate(exhibition.endDate) }}
            </strong>
          </div>

          <div>
            <span>Exhibitor</span>
            <strong>{{ exhibition.exhibitorName || '—' }}</strong>
          </div>
        </div>
      </RouterLink>
    </div>

    <div v-else class="empty compact">
      No exhibitions available yet.
    </div>
  </section>
</template>