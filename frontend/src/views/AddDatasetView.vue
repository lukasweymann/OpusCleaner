<script setup>
import { ref, reactive, computed, watch, onMounted } from "vue";
import { formatSize } from "../format.js";
import VueSelect from "vue-select";
import { DownloadIcon } from "vue3-feather";
import DownloadPopup from "../components/DownloadPopup.vue";
import { fetchJSON } from "../store/fetch.js";
import {
  download,
  isDownloading,
  fetchDownloadableDatasets,
  fetchSourceLanguages,
  fetchTargetLanguages,
} from "../store/downloads.js";

import "vue-select/dist/vue-select.css";

function nonEmpty(x) {
  return x != "";
}

const Preprocessing = {
  MONOLINGUAL: "monolingual",
  BILINGUAL: "bilingual",
};

const SORT_ORDER_OPTIONS = [
  {
    label: "Corpus name",
    compare: (a, b) => a.corpus.localeCompare(b.corpus),
  },
  {
    label: "Download size",
    compare: (a, b) => b.size - a.size,
  },
  {
    label: "Sentence pairs",
    compare: (a, b) => b.pairs - a.pairs,
  },
];

const sortOrder = ref(SORT_ORDER_OPTIONS[0]);

// Per language all target languages
const languages = new Map();

// Datasets by language
const cache = new Map();

const nameFilter = ref("");

const latestOnly = ref(true);

const preprocessing = ref(Preprocessing.BILINGUAL);

const srcLang = ref();

const trgLang = ref();

const srcLangs = ref();

const trgLangs = computed(() => {
  if (!srcLang.value) return [];

  if (!languages.has(srcLang.value)) {
    const list = ref([]);
    languages.set(srcLang.value, list);
    fetchTargetLanguages(srcLang.value).then((langs) => {
      list.value = langs;
    });
  }

  return languages.get(srcLang.value).value; // reactive, so will update once fetch() finishes
});

const srcLangOptions = computed(() => {
  const intl = new Intl.DisplayNames([], {
    type: "language",
    languageDisplay: "standard",
  });
  return (srcLangs.value || []).map((lang) => {
    try {
      return { lang, label: `${intl.of(lang)} (${lang})` };
    } catch (RangeError) {
      return { lang, label: lang };
    }
  });
});

const trgLangOptions = computed(() => {
  const intl = new Intl.DisplayNames([], {
    type: "language",
    languageDisplay: "standard",
  });
  return (trgLangs.value || []).map((lang) => {
    try {
      return { lang, label: `${intl.of(lang)} (${lang})` };
    } catch (RangeError) {
      return { lang, label: lang };
    }
  });
});

const datasets = computed(() => {
  let key;

  switch (preprocessing.value) {
    case Preprocessing.BILINGUAL:
      if (!srcLang.value || !trgLang.value) return [];
      key = `${srcLang.value}-${trgLang.value}`;
      break;
    case Preprocessing.MONOLINGUAL:
      if (!srcLang.value) return [];
      key = `${srcLang.value}`;
      break;
    default:
      throw new Error("Unknown preprocessing type");
  }

  if (!cache.has(key)) {
    const list = ref([]);
    cache.set(key, list);
    // Fetches actual list async, but the cache entry is available immediately.
    fetchDownloadableDatasets(key).then((datasets) => (list.value = datasets));
  }

  // cache contains refs, so this computed() is called again once the data
  // is actually fetched.
  let datasets = cache.get(key).value;

  if (nameFilter.value.length > 0)
    datasets = datasets.filter(({ corpus, group }) => {
      return (
        corpus.toLowerCase().indexOf(nameFilter.value.toLowerCase()) !== -1
      );
    });

  datasets = datasets.filter((dataset) => {
    switch (preprocessing.value) {
      case Preprocessing.BILINGUAL:
        return dataset.langs.filter(nonEmpty).length > 1;
      case Preprocessing.MONOLINGUAL:
        return dataset.langs.filter(nonEmpty).length == 1;
      default:
        return false;
    }
  });

  if (latestOnly.value) {
    datasets = Array.from(
      datasets
        .reduce((latest, dataset) => {
          if (
            !latest.has(dataset.corpus) ||
            latest.get(dataset.corpus).version < dataset.version
          )
            latest.set(dataset.corpus, dataset);

          return latest;
        }, new Map())
        .values()
    );
  }

  datasets.sort(sortOrder.value.compare);

  return datasets;
});

onMounted(async () => {
  fetchSourceLanguages().then((languages) => {
    srcLangs.value = languages.filter((num) => !/\d/.test(num));
  });
});

function assignList(current, update, key = "id") {
  const updates = Object.fromEntries(
    update.map((entry) => [entry[key], entry])
  );
  for (let i = 0; i < current.length; ++i)
    if (current[i][key] in updates)
      Object.assign(current[i], updates[current[i][key]]);
  return current;
}

const countFormat = new Intl.NumberFormat();
</script>

<template>
  <div class="downloader">
    <h1 class="datasets-catalogue-title">
      Datasets catalogue
      <small class="small"><em>TODO</em> datasets</small>
    </h1>
    <div class="search-inputs">
      <label>
        <input
          type="search"
          placeholder="Search dataset…"
          v-model="nameFilter"
        />
      </label>
      <label
        class="search-button"
        :class="{ checked: preprocessing == Preprocessing.MONOLINGUAL }"
      >
        <input
          type="radio"
          name="preprocessing"
          v-model="preprocessing"
          :value="Preprocessing.MONOLINGUAL"
        />
        Monolingual
      </label>
      <label
        class="search-button"
        :class="{ checked: preprocessing == Preprocessing.BILINGUAL }"
      >
        <input
          type="radio"
          name="preprocessing"
          v-model="preprocessing"
          :value="Preprocessing.BILINGUAL"
        />
        Bilingual
      </label>
      <label class="search-button" :class="{ checked: latestOnly }">
        <input type="checkbox" v-model="latestOnly" />
        Latest only
      </label>
      <label>
        <VueSelect
          class="origin-lang-selector"
          id="originSelector"
          v-model="srcLang"
          :options="srcLangOptions"
          :reduce="({ lang }) => lang"
          placeholder="Origin language"
        />
      </label>
      <label v-show="preprocessing == Preprocessing.BILINGUAL">
        <VueSelect
          class="origin-lang-selector"
          v-model="trgLang"
          :options="trgLangOptions"
          :reduce="({ lang }) => lang"
          placeholder="Target language"
        />
      </label>
      <label>
        <VueSelect
          v-model="sortOrder"
          :options="SORT_ORDER_OPTIONS"
          placeholder="Sort order"
        />
      </label>
    </div>
    <div class="dataset-list">
      <div
        class="dataset"
        v-for="dataset in datasets"
        :key="dataset.id"
        :id="`did-${dataset.id}`"
      >
        <div class="dataset-name">
          <h3 class="dataset-title">
            <a
              :href="`https://opus.nlpl.eu/${dataset.corpus}-${dataset.version}.php`"
              target="_blank"
              >{{ dataset.corpus }}</a
            >
          </h3>
          <button
            class="download-dataset-button"
            @click="download(dataset)"
            :disabled="isDownloading(dataset) || 'paths' in dataset"
          >
            <DownloadIcon
              class="download-icon"
              color="white"
              stroke-width="1"
            />
          </button>
        </div>
        <dl class="metadata-dataset">
          <dt>Version</dt>
          <dd title="Version">{{ dataset.version }}</dd>
          <dt>Languages</dt>
          <dd title="Languages">
            {{ dataset.langs.filter(nonEmpty).join("→") }}
          </dd>
          <dt>Pairs</dt>
          <dd title="Sentence pairs">
            {{ dataset.pairs ? countFormat.format(dataset.pairs) : "" }}
          </dd>
          <dt>Size</dt>
          <dd title="Download size">
            {{ dataset.size ? formatSize(dataset.size) : "" }}
          </dd>
        </dl>
      </div>
    </div>
    <Teleport to=".navbar">
      <DownloadPopup />
    </Teleport>
  </div>
</template>

<style scoped>
.downloader {
  min-height: 60vh;
}
.small {
  color: rgb(119, 130, 149);
}
.datasets-catalogue-title {
  font-size: 20px;
  text-transform: uppercase;
  display: flex;
  align-items: baseline;
}

.datasets-catalogue-title small {
  display: inline-block;
  border-left: 1px solid currentColor;
  margin-left: 10px;
  padding-left: 10px;
}

.datasets-catalogue-title span {
  font-size: 16px;
  font-weight: lighter;
}

.search-inputs {
  margin: 5px 0 20px 0;
  display: flex;
  align-items: center;
}

.search-button {
  color: #182231;
  border: none;
  border-radius: 2px;
  line-height: 28px;
  height: 32px;
  padding: 0 10px;
  font-size: 14px;
  display: flex;
  align-items: center;
  margin: 5px 2px 0 2px;
  cursor: pointer;
  background-color: #dbe5e6;
}

.search-button.checked {
  background-color: #17223d;
  color: #efefef;
}

.search-button input {
  display: none;
}

.search-inputs input {
  height: 32px;
  border-radius: 3px;
  box-shadow: none;
  padding-left: 7px;
  border-style: solid;
  border-color: var(--input-border);
  font-size: 14px;
  margin-top: 5px;
}

.search-inputs .v-select {
  display: inline-block;
  width: 230px;
  height: 28px;
  border-radius: 3px;
  font-size: 14px;
  margin-right: 5px;
  padding: 0;
  border-color: var(--input-border);
}

.dataset-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(400px, 1fr));
  row-gap: 20px;
  column-gap: 20px;
}

.dataset {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  background-color: var(--secondary-background);
  border-radius: 5px;
  padding: 15px;
  height: 120px;
  color: var(--main-copy);
}

.dataset-name {
  display: flex;
  justify-content: space-between;
}
.dataset-title {
  font-size: 22px;
}
.dataset-title a {
  text-decoration: none;
  color: var(--main-copy);
}
.dataset-title a:hover {
  text-decoration: none;
  color: var(--main-copy);
  color: var(--copy-hover);
}

.metadata-dataset dt {
  display: none;
}

.metadata-dataset dd {
  display: inline;
  margin-right: 20px;
  color: var(--secondary-copy);
}

.download-dataset-button {
  width: 34px;
  height: 30px;
  border: none;
  border-radius: 2px;
  padding: 2px;
  background-color: #647089;
  cursor: pointer;
  margin-left: 10px;
}
.download-dataset-button:hover {
  cursor: pointer;
}
.download-dataset-button:disabled {
  cursor: default;
}
</style>
