<script setup>
import { ref } from 'vue';
import { useRouter } from 'vue-router';
import DataTable from '../components/DataTable.vue';
import { exhibitions, exhibitionArtworks, search } from '../api/resources';
import { useAuthStore } from '../stores/auth';
import { normalizeError } from '../api/http';

const auth = useAuthStore();
const router = useRouter();

const table = ref(null);
const query = ref('');
const actionError = ref(null);

const columns = [
  { key: 'id', label: 'ID', sortable: true },
  { key: 'title', label: 'Title', sortable: true },
  { key: 'exhibitorName', label: 'Exhibitor', sortable: false },
  { key: 'startDate', label: 'Start', sortable: true },
  { key: 'endDate', label: 'End', sortable: true },
  { key: 'artworkCount', label: 'Artworks', sortable: false },
];

function normalizePage(payload) {
  if (payload?.data?.content) {
    return payload.data;
  }

  if (payload?.content) {
    return payload;
  }

  return {
    content: Array.isArray(payload) ? payload : [],
    page: 0,
    size: Array.isArray(payload) ? payload.length : 0,
    totalElements: Array.isArray(payload) ? payload.length : 0,
    totalPages: 1,
    first: true,
    last: true,
    numberOfElements: Array.isArray(payload) ? payload.length : 0,
  };
}

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

async function fetcher(params) {
  const rawPage = query.value.trim()
    ? await search.exhibitions(query.value.trim(), params)
    : await exhibitions.list(params);

  const page = normalizePage(rawPage);

  const content = await Promise.all(
    (page.content || []).map(async (exhibition) => {
      try {
        const linkedArtworks = asArray(await exhibitionArtworks.list(exhibition.id));

        return {
          ...exhibition,
          artworkCount: linkedArtworks.length,
        };
      } catch {
        return {
          ...exhibition,
          artworkCount: 0,
        };
      }
    }),
  );

  return {
    ...page,
    content,
  };
}

function reload() {
  table.value?.reload();
}

function clearSearch() {
  query.value = '';
  reload();
}

async function remove(item) {
  const confirmed = confirm(`Delete exhibition "${item.title}"?`);

  if (!confirmed) {
    return;
  }

  actionError.value = null;

  try {
    await exhibitions.remove(item.id);
    reload();
  } catch (error) {
    actionError.value = normalizeError(error).message;
  }
}
</script>

<template>
  <div class="page-header">
    <div>
      <h1>Exhibitions</h1>
      <p class="muted">
        View exhibitions and manage assigned artworks.
      </p>
    </div>

    <router-link
      v-if="auth.isAdmin"
      class="btn btn-primary"
      :to="{ name: 'exhibition-new' }"
    >
      + New Exhibition
    </router-link>
  </div>

  <div v-if="actionError" class="alert alert-error">
    {{ actionError }}
  </div>

  <div class="card">
    <DataTable
      ref="table"
      :columns="columns"
      :fetcher="fetcher"
      default-sort="startDate,desc"
    >
      <template #toolbar>
        <div class="table-toolbar">
          <input
            v-model="query"
            class="table-search-input"
            placeholder="Search exhibitions..."
            @keyup.enter="reload"
          />

          <button class="btn" type="button" @click="reload">
            Search
          </button>

          <button
            v-if="query"
            class="btn"
            type="button"
            @click="clearSearch"
          >
            Clear
          </button>
        </div>
      </template>

      <template #row="{ item }">
        <td>{{ item.id }}</td>

        <td>
          <strong class="table-title">{{ item.title }}</strong>
        </td>

        <td>{{ item.exhibitorName || '—' }}</td>
        <td>{{ item.startDate || '—' }}</td>
        <td>{{ item.endDate || '—' }}</td>

        <td>
          <span class="count-pill">
            {{ item.artworkCount }}
          </span>
        </td>
      </template>

      <template #actions="{ item }">
        <button
          class="table-action table-action-neutral"
          type="button"
          @click="router.push({ name: 'exhibition-details', params: { id: item.id } })"
        >
          View
        </button>

        <template v-if="auth.isAdmin">
          <button
            class="table-action table-action-edit"
            type="button"
            @click="router.push({ name: 'exhibition-edit', params: { id: item.id } })"
          >
            Edit
          </button>

          <button
            class="table-action table-action-delete"
            type="button"
            @click="remove(item)"
          >
            Delete
          </button>
        </template>
      </template>
    </DataTable>
  </div>
</template>