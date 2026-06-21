<script setup>
import { computed, onMounted, reactive, ref } from 'vue';
import { useRouter } from 'vue-router';
import {
  artworks,
  exhibitions,
  exhibitionArtworks,
  exhibitors,
} from '../api/resources';
import { normalizeError } from '../api/http';

const props = defineProps({
  id: {
    type: String,
    default: null,
  },
});

const router = useRouter();

const isEdit = computed(() => props.id != null);

const form = reactive({
  title: '',
  startDate: '',
  endDate: '',
  exhibitorId: null,
  description: '',
});

const exhibitorOptions = ref([]);
const artworkOptions = ref([]);
const selectedArtworkIds = ref([]);

const artworkSearch = ref('');

const fieldErrors = ref({});
const formError = ref(null);
const loading = ref(false);
const initialLoading = ref(true);

const selectedArtworks = computed(() => {
  const selectedIds = selectedArtworkIds.value.map(Number);

  return selectedIds
    .map((id) => artworkOptions.value.find((artwork) => Number(artwork.id) === id))
    .filter(Boolean);
});

const availableArtworks = computed(() => {
  const selectedIds = new Set(selectedArtworkIds.value.map(Number));
  const query = artworkSearch.value.trim().toLowerCase();

  return artworkOptions.value
    .filter((artwork) => !selectedIds.has(Number(artwork.id)))
    .filter((artwork) => {
      if (!query) {
        return true;
      }

      const title = artwork.title?.toLowerCase() || '';
      const artist = artwork.artistName?.toLowerCase() || '';
      const year = artwork.yearCreated ? String(artwork.yearCreated) : '';

      return title.includes(query) || artist.includes(query) || year.includes(query);
    });
});

const availableArtworkIds = computed(() => {
  return availableArtworks.value.map((artwork) => Number(artwork.id));
});

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

function extractId(payload) {
  if (payload?.id) {
    return Number(payload.id);
  }

  if (payload?.data?.id) {
    return Number(payload.data.id);
  }

  return null;
}

function normalizeArtworkIds() {
  selectedArtworkIds.value = [...new Set(selectedArtworkIds.value.map(Number))];
}

function addArtwork(artworkId) {
  const id = Number(artworkId);

  if (!id) {
    return;
  }

  if (selectedArtworkIds.value.map(Number).includes(id)) {
    return;
  }

  selectedArtworkIds.value = [...selectedArtworkIds.value.map(Number), id];
  normalizeArtworkIds();
}

function addVisibleArtworks() {
  if (!availableArtworkIds.value.length) {
    return;
  }

  const selectedIds = new Set(selectedArtworkIds.value.map(Number));

  availableArtworkIds.value.forEach((id) => {
    selectedIds.add(id);
  });

  selectedArtworkIds.value = [...selectedIds];
  normalizeArtworkIds();
}

function removeArtwork(artworkId) {
  const id = Number(artworkId);

  selectedArtworkIds.value = selectedArtworkIds.value
    .map(Number)
    .filter((item) => item !== id);
}

async function syncArtworks(exhibitionId) {
  normalizeArtworkIds();

  const desiredIds = new Set(selectedArtworkIds.value.map(Number));

  if (!isEdit.value) {
    await Promise.all(
      [...desiredIds].map((artworkId) => exhibitionArtworks.add(exhibitionId, artworkId)),
    );

    return;
  }

  const currentArtworks = asArray(await exhibitionArtworks.list(exhibitionId));
  const currentIds = new Set(currentArtworks.map((artwork) => Number(artwork.id)));

  const idsToAdd = [...desiredIds].filter((id) => !currentIds.has(id));
  const idsToRemove = [...currentIds].filter((id) => !desiredIds.has(id));

  await Promise.all([
    ...idsToAdd.map((artworkId) => exhibitionArtworks.add(exhibitionId, artworkId)),
    ...idsToRemove.map((artworkId) => exhibitionArtworks.remove(exhibitionId, artworkId)),
  ]);
}

async function submit() {
  fieldErrors.value = {};
  formError.value = null;
  loading.value = true;

  const payload = {
    title: form.title,
    startDate: form.startDate || null,
    endDate: form.endDate || null,
    exhibitorId: form.exhibitorId ? Number(form.exhibitorId) : null,
    description: form.description || null,
  };

  try {
    let exhibitionId;

    if (isEdit.value) {
      await exhibitions.update(props.id, payload);
      exhibitionId = Number(props.id);
    } else {
      const created = await exhibitions.create(payload);
      exhibitionId = extractId(created);
    }

    if (!exhibitionId) {
      throw new Error('Exhibition was saved, but its ID was not returned.');
    }

    await syncArtworks(exhibitionId);

    router.push({ name: 'exhibitions' });
  } catch (error) {
    const normalized = normalizeError(error);

    if (Object.keys(normalized.fieldErrors).length) {
      fieldErrors.value = normalized.fieldErrors;
    } else {
      formError.value = normalized.message;
    }
  } finally {
    loading.value = false;
  }
}

onMounted(async () => {
  try {
    const [loadedExhibitors, loadedArtworks] = await Promise.all([
      exhibitors.all(),
      artworks.all(),
    ]);

    exhibitorOptions.value = asArray(loadedExhibitors);
    artworkOptions.value = asArray(loadedArtworks);

    if (isEdit.value) {
      const [loadedExhibition, loadedLinkedArtworks] = await Promise.all([
        exhibitions.get(props.id),
        exhibitionArtworks.list(props.id),
      ]);

      const exhibition = asEntity(loadedExhibition);
      const linkedArtworks = asArray(loadedLinkedArtworks);

      Object.assign(form, {
        title: exhibition.title || '',
        startDate: exhibition.startDate || '',
        endDate: exhibition.endDate || '',
        exhibitorId: exhibition.exhibitorId || null,
        description: exhibition.description || '',
      });

      selectedArtworkIds.value = linkedArtworks.map((artwork) => Number(artwork.id));
      normalizeArtworkIds();
    }
  } catch (error) {
    formError.value = normalizeError(error).message;
  } finally {
    initialLoading.value = false;
  }
});
</script>

<template>
  <div class="page-header">
    <div>
      <h1>{{ isEdit ? 'Edit Exhibition' : 'New Exhibition' }}</h1>
      <p class="muted">
        {{ isEdit ? 'Update exhibition details and assigned artworks.' : 'Create an exhibition and assign artworks.' }}
      </p>
    </div>

    <router-link class="btn" :to="{ name: 'exhibitions' }">
      ← Back
    </router-link>
  </div>

  <div class="card form-card exhibition-form-card">
    <div v-if="formError" class="alert alert-error">
      {{ formError }}
    </div>

    <div v-if="initialLoading" class="empty">
      Loading form data...
    </div>

    <form v-else @submit.prevent="submit">
      <div class="field">
        <label for="title">Title *</label>

        <input
          id="title"
          v-model="form.title"
          :class="{ invalid: fieldErrors.title }"
        />

        <div v-if="fieldErrors.title" class="field-error">
          {{ fieldErrors.title }}
        </div>
      </div>

      <div class="grid-2">
        <div class="field">
          <label for="startDate">Start Date *</label>

          <input
            id="startDate"
            v-model="form.startDate"
            type="date"
            :class="{ invalid: fieldErrors.startDate }"
          />

          <div v-if="fieldErrors.startDate" class="field-error">
            {{ fieldErrors.startDate }}
          </div>
        </div>

        <div class="field">
          <label for="endDate">End Date *</label>

          <input
            id="endDate"
            v-model="form.endDate"
            type="date"
            :class="{ invalid: fieldErrors.endDate }"
          />

          <div v-if="fieldErrors.endDate" class="field-error">
            {{ fieldErrors.endDate }}
          </div>
        </div>
      </div>

      <div class="field">
        <label for="exhibitorId">Exhibitor *</label>

        <select
          id="exhibitorId"
          v-model="form.exhibitorId"
          :class="{ invalid: fieldErrors.exhibitorId }"
        >
          <option :value="null" disabled>
            — Select exhibitor —
          </option>

          <option
            v-for="exhibitor in exhibitorOptions"
            :key="exhibitor.id"
            :value="exhibitor.id"
          >
            {{ exhibitor.name }}
          </option>
        </select>

        <div v-if="fieldErrors.exhibitorId" class="field-error">
          {{ fieldErrors.exhibitorId }}
        </div>
      </div>

      <div class="field">
        <label for="description">Description</label>

        <textarea
          id="description"
          v-model="form.description"
          rows="3"
        ></textarea>
      </div>

      <section class="artwork-checklist-manager">
        <div class="artwork-manager-header">
          <div>
            <label>Artworks</label>
            <p class="field-help">
              Search the available artwork list and assign artworks to this exhibition.
              Selected artworks are removed from the available list.
            </p>
          </div>

          <span class="selected-count">
            {{ selectedArtworkIds.length }} selected
          </span>
        </div>

        <div class="artwork-checklist-search">
          <input
            v-model="artworkSearch"
            placeholder="Search available artworks..."
          />
        </div>

        <div class="artwork-checklist-box">
          <label class="artwork-check-row select-all-row">
            <input
              type="checkbox"
              :checked="false"
              :disabled="!availableArtworks.length"
              @change="addVisibleArtworks"
            />

            <span>
              Select visible artworks
            </span>
          </label>

          <div class="artwork-check-scroll">
            <label
              v-for="artwork in availableArtworks"
              :key="artwork.id"
              class="artwork-check-row"
            >
              <input
                type="checkbox"
                :checked="false"
                @change="addArtwork(artwork.id)"
              />

              <span>
                <strong>{{ artwork.title }}</strong>
                <small>
                  {{ artwork.artistName || 'Unknown artist' }}
                  <template v-if="artwork.yearCreated">
                    · {{ artwork.yearCreated }}
                  </template>
                </small>
              </span>
            </label>

            <div v-if="!availableArtworks.length" class="empty compact">
              No available artworks match the current search.
            </div>
          </div>
        </div>

        <div class="selected-artworks-panel">
          <div class="available-artworks-header">
            <span>Selected artworks</span>
            <small>{{ selectedArtworks.length }} selected</small>
          </div>

          <div v-if="selectedArtworks.length" class="selected-artworks-list">
            <div
              v-for="artwork in selectedArtworks"
              :key="artwork.id"
              class="selected-artwork-row"
            >
              <div class="selected-artwork-info">
                <strong>{{ artwork.title }}</strong>
                <span>
                  {{ artwork.artistName || 'Unknown artist' }}
                  <template v-if="artwork.yearCreated">
                    · {{ artwork.yearCreated }}
                  </template>
                </span>
              </div>

              <button
                class="table-action table-action-delete selected-artwork-remove"
                type="button"
                @click="removeArtwork(artwork.id)"
              >
                Remove
              </button>
            </div>
          </div>

          <div v-else class="empty compact">
            No artworks selected yet.
          </div>
        </div>
      </section>

      <div class="toolbar form-actions">
        <button class="btn btn-primary" :disabled="loading">
          {{ loading ? 'Saving...' : 'Save Exhibition' }}
        </button>

        <router-link class="btn" :to="{ name: 'exhibitions' }">
          Cancel
        </router-link>
      </div>
    </form>
  </div>
</template>