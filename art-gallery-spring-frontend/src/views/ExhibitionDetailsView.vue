<script setup>
import { onMounted, ref } from 'vue';
import { useRouter } from 'vue-router';
import { exhibitions, exhibitionArtworks } from '../api/resources';
import { normalizeError } from '../api/http';

const props = defineProps({
  id: {
    type: String,
    required: true,
  },
});

const router = useRouter();

const exhibition = ref(null);
const linkedArtworks = ref([]);
const loading = ref(true);
const error = ref(null);

function asArray(payload) {
  if (Array.isArray(payload)) {
    return payload;
  }

  if (Array.isArray(payload?.data)) {
    return payload.data;
  }

  if (Array.isArray(payload?.content)) {
    return payload.content;
  }

  if (Array.isArray(payload?.data?.content)) {
    return payload.data.content;
  }

  return [];
}

function asEntity(payload) {
  if (payload?.data) {
    return payload.data;
  }

  return payload || {};
}

function money(value) {
  return value == null
    ? '—'
    : new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: 'USD',
      }).format(value);
}

async function loadData() {
  loading.value = true;
  error.value = null;

  try {
    const [loadedExhibition, loadedArtworks] = await Promise.all([
      exhibitions.get(props.id),
      exhibitionArtworks.list(props.id),
    ]);

    exhibition.value = asEntity(loadedExhibition);
    linkedArtworks.value = asArray(loadedArtworks);
  } catch (err) {
    error.value = normalizeError(err).message;
  } finally {
    loading.value = false;
  }
}

onMounted(() => {
  loadData();
});
</script>

<template>
  <div class="page-header">
    <div>
      <h1>{{ exhibition?.title || 'Exhibition Details' }}</h1>
      <p class="muted">
        View exhibition details and assigned artworks.
      </p>
    </div>

    <button class="btn" type="button" @click="router.push({ name: 'exhibitions' })">
      ← Back
    </button>
  </div>

  <div v-if="error" class="alert alert-error">
    {{ error }}
  </div>

  <div v-if="loading" class="card">
    <div class="empty">
      Loading exhibition...
    </div>
  </div>

  <template v-else>
    <section class="exhibition-detail-hero">
      <div>
        <p class="eyebrow">Exhibition</p>
        <h2>{{ exhibition.title || '—' }}</h2>
        <p>{{ exhibition.description || 'No description available.' }}</p>
      </div>

      <div class="exhibition-detail-side">
        <span>Artworks</span>
        <strong>{{ linkedArtworks.length }}</strong>
      </div>
    </section>

    <section class="details-grid detail-grid-spaced">
      <div>
        <span>Exhibitor</span>
        <strong>{{ exhibition.exhibitorName || '—' }}</strong>
      </div>

      <div>
        <span>Start date</span>
        <strong>{{ exhibition.startDate || '—' }}</strong>
      </div>

      <div>
        <span>End date</span>
        <strong>{{ exhibition.endDate || '—' }}</strong>
      </div>
    </section>

    <div class="card">
      <div class="section-heading">
        <h2>Assigned artworks</h2>
        <p class="muted">
          {{ linkedArtworks.length }} artworks assigned to this exhibition.
        </p>
      </div>

      <div v-if="linkedArtworks.length" class="detail-artwork-grid">
        <article
          v-for="artwork in linkedArtworks"
          :key="artwork.id"
          class="detail-artwork-card"
        >
          <div>
            <h3>{{ artwork.title }}</h3>
            <p>{{ artwork.artistName || 'Unknown artist' }}</p>
          </div>

          <dl>
            <div>
              <dt>Year</dt>
              <dd>{{ artwork.yearCreated || '—' }}</dd>
            </div>

            <div>
              <dt>Medium</dt>
              <dd>{{ artwork.medium || '—' }}</dd>
            </div>

            <div>
              <dt>Estimated value</dt>
              <dd>{{ money(artwork.estimatedValue) }}</dd>
            </div>
          </dl>
        </article>
      </div>

      <div v-else class="empty">
        No artworks assigned to this exhibition.
      </div>
    </div>
  </template>
</template>