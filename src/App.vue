<script setup>
import { ref, onMounted, nextTick } from "vue";
import axios from "axios";
import {
  Search,
  Heart,
  Image,
  Music,
  Smile,
  Calendar,
  MessageCircle,
  Sparkles
} from "lucide-vue-next";

const API_URL = import.meta.env.VITE_API_URL;

const query = ref("");
const results = ref([]);
const contextMessages = ref([]);
const stats = ref(null);

const loading = ref(false);
const loadingMore = ref(false);
const hasMore = ref(true);
const offset = ref(0);
const limit = 50;

const selectedMessageId = ref(null);
const selectedContextIds = ref([]);

const audioPlayer = ref(null);
const reading = ref(false);

async function loadStats() {
  const { data } = await axios.get(`${API_URL}/api/stats`);
  stats.value = data;
}

function highlightText(text) {
  if (!text || !query.value) return text;

  const escaped = query.value.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
  const regex = new RegExp(`(${escaped})`, "gi");

  return text.replace(
    regex,
    `<span class="bg-yellow-300/90 text-black px-1 rounded font-semibold">$1</span>`
  );
}

async function searchMessages(reset = true) {
  const q = query.value.trim();

  if (!q) {
    results.value = [];
    contextMessages.value = [];
    selectedMessageId.value = null;
    selectedContextIds.value = [];
    offset.value = 0;
    hasMore.value = true;
    return;
  }

  if (reset) {
    results.value = [];
    contextMessages.value = [];
    selectedMessageId.value = null;
    selectedContextIds.value = [];
    offset.value = 0;
    hasMore.value = true;
  }

  loading.value = reset;
  loadingMore.value = !reset;

  try {
    const { data } = await axios.get(`${API_URL}/api/search`, {
      params: {
        q,
        offset: offset.value,
        limit
      }
    });

    results.value.push(...data);
    offset.value += data.length;

    if (data.length < limit) {
      hasMore.value = false;
    }
  } finally {
    loading.value = false;
    loadingMore.value = false;
  }
}

async function handleResultsScroll(event) {
  const el = event.target;
  const nearBottom = el.scrollTop + el.clientHeight >= el.scrollHeight - 200;

  if (nearBottom && hasMore.value && !loadingMore.value && !loading.value) {
    await searchMessages(false);
  }
}

async function openContext(messageId) {
  selectedMessageId.value = messageId;
  selectedContextIds.value = [];

  const { data } = await axios.get(`${API_URL}/api/context/${messageId}`);
  contextMessages.value = data;

  await nextTick();

  const selectedElement = document.getElementById(
    `context-message-${messageId}`
  );

  if (selectedElement) {
    selectedElement.scrollIntoView({
      behavior: "smooth",
      block: "start"
    });
  }
}

function toggleContextSelection(messageId) {
  if (selectedContextIds.value.includes(messageId)) {
    selectedContextIds.value = selectedContextIds.value.filter(
      id => id !== messageId
    );
  } else {
    selectedContextIds.value.push(messageId);
  }
}

function clearSelection() {
  selectedContextIds.value = [];
}

function buildSelectedText() {
  return contextMessages.value
    .filter(m => selectedContextIds.value.includes(m.id))
    .filter(m => m.content)
    .map(m => m.content)
    .join(". ");
}

async function readSelectedContext() {
  const text = buildSelectedText();

  if (!text) return;

  stopReading();
  reading.value = true;

  try {
    const response = await axios.post(
      `${API_URL}/api/tts`,
      { text },
      { responseType: "blob" }
    );

    const audioUrl = URL.createObjectURL(response.data);
    const audio = new Audio(audioUrl);

    audioPlayer.value = audio;

    audio.onended = () => {
      reading.value = false;
      URL.revokeObjectURL(audioUrl);
      audioPlayer.value = null;
    };

    await audio.play();
  } catch (error) {
    console.error(error);
    reading.value = false;
  }
}

function stopReading() {
  if (audioPlayer.value) {
    audioPlayer.value.pause();
    audioPlayer.value.currentTime = 0;
    audioPlayer.value = null;
  }

  reading.value = false;
}

function formatDate(date) {
  return new Date(date).toLocaleString("es-PE", {
    dateStyle: "medium",
    timeStyle: "short"
  });
}

function isMine(sender) {
  return sender?.toLowerCase().includes("alejandro");
}

const categories = [
  { name: "Timeline", icon: Calendar },
  { name: "Cariño", icon: Heart },
  { name: "Fotos", icon: Image },
  { name: "Audios", icon: Music },
  { name: "Stickers", icon: Smile },
  { name: "Importantes", icon: Sparkles }
];

onMounted(() => {
  loadStats();
});
</script>

<template>
  <div class="min-h-screen bg-[#070a12] text-slate-100">
    <div class="mx-auto max-w-7xl p-6">
      <header class="mb-6 rounded-3xl border border-white/10 bg-white/[0.04] p-6 shadow-2xl backdrop-blur">
        <div class="flex flex-col gap-4 md:flex-row md:items-end md:justify-between">
          <div>
            <p class="mb-2 text-sm uppercase tracking-[0.3em] text-cyan-300/80">
              Messenger Archive
            </p>

            <h1 class="text-3xl font-bold md:text-5xl">
              Chat Memory Explorer
            </h1>

            <p class="mt-3 max-w-2xl text-slate-400">
              Busca, categoriza y revisa tu historial completo de mensajes.
            </p>
          </div>

          <div v-if="stats" class="grid grid-cols-2 gap-3 text-right md:grid-cols-1">
            <div class="rounded-2xl bg-black/30 px-4 py-3">
              <p class="text-xs text-slate-400">Mensajes</p>
              <p class="text-2xl font-bold">{{ stats.totalMessages }}</p>
            </div>
          </div>
        </div>

        <div class="mt-6 flex gap-3">
          <div class="relative flex-1">
            <Search class="absolute left-4 top-1/2 h-5 w-5 -translate-y-1/2 text-slate-500" />

            <input
              v-model="query"
              @keyup.enter="searchMessages(true)"
              class="w-full rounded-2xl border border-white/10 bg-black/40 py-4 pl-12 pr-4 text-slate-100 outline-none transition placeholder:text-slate-500 focus:border-cyan-400/60 focus:ring-4 focus:ring-cyan-400/10"
              placeholder="Buscar en toda la conversación..."
            />
          </div>

          <button
            @click="searchMessages(true)"
            class="rounded-2xl bg-cyan-400 px-6 font-semibold text-slate-950 transition hover:bg-cyan-300 disabled:cursor-not-allowed disabled:opacity-60"
            :disabled="loading"
          >
            {{ loading ? "Buscando..." : "Buscar" }}
          </button>
        </div>
      </header>

      <main class="grid gap-6 lg:grid-cols-[260px_1fr_420px]">
        <aside class="rounded-3xl border border-white/10 bg-white/[0.04] p-4">
          <h2 class="mb-4 text-sm font-semibold uppercase tracking-[0.2em] text-slate-400">
            Categorías
          </h2>

          <div class="space-y-2">
            <button
              v-for="category in categories"
              :key="category.name"
              class="flex w-full items-center gap-3 rounded-2xl px-4 py-3 text-left text-slate-300 transition hover:bg-white/10 hover:text-white"
            >
              <component :is="category.icon" class="h-5 w-5 text-cyan-300" />
              <span>{{ category.name }}</span>
            </button>
          </div>

          <div v-if="stats" class="mt-6">
            <h3 class="mb-3 text-sm font-semibold uppercase tracking-[0.2em] text-slate-400">
              Años
            </h3>

            <div class="space-y-2">
              <div
                v-for="year in stats.years"
                :key="year.year"
                class="flex justify-between rounded-xl bg-black/30 px-3 py-2 text-sm text-slate-300"
              >
                <span>{{ year.year }}</span>
                <span>{{ year.total }}</span>
              </div>
            </div>
          </div>
        </aside>

        <section class="rounded-3xl border border-white/10 bg-white/[0.04] p-4">
          <div class="mb-4 flex items-center justify-between">
            <h2 class="text-xl font-bold">Resultados</h2>

            <span class="text-sm text-slate-400">
              {{ results.length }} cargados
            </span>
          </div>

          <div v-if="loading" class="rounded-2xl bg-black/30 p-6 text-slate-400">
            Buscando...
          </div>

          <div v-else-if="!results.length" class="rounded-2xl bg-black/30 p-6 text-slate-400">
            Escribe una palabra o frase para buscar en todos los mensajes.
          </div>

          <div
            v-else
            class="max-h-[720px] space-y-3 overflow-y-auto pr-2"
            @scroll="handleResultsScroll"
          >
            <article
              v-for="message in results"
              :key="message.id"
              @click="openContext(message.id)"
              class="cursor-pointer rounded-2xl border border-white/10 bg-black/30 p-4 transition hover:border-cyan-400/40 hover:bg-cyan-400/5"
              :class="selectedMessageId === message.id ? 'border-cyan-400/60 bg-cyan-400/10' : ''"
            >
              <div class="mb-2 flex items-center justify-between gap-3">
                <p class="font-semibold text-cyan-200">
                  {{ message.sender_name }}
                </p>

                <p class="text-xs text-slate-500">
                  {{ formatDate(message.date) }}
                </p>
              </div>

              <p
                class="line-clamp-3 text-slate-300"
                v-html="highlightText(message.content || `[${message.type}]`)"
              ></p>
            </article>

            <div
              v-if="loadingMore"
              class="rounded-2xl bg-black/30 p-4 text-center text-sm text-slate-400"
            >
              Cargando más resultados...
            </div>

            <div
              v-else-if="!hasMore"
              class="rounded-2xl bg-black/20 p-4 text-center text-sm text-slate-500"
            >
              No hay más resultados.
            </div>
          </div>
        </section>

        <section class="rounded-3xl border border-white/10 bg-white/[0.04] p-4">
          <div class="mb-4 flex flex-col gap-3">
            <div class="flex items-center gap-2">
              <MessageCircle class="h-5 w-5 text-cyan-300" />
              <h2 class="text-xl font-bold">Contexto</h2>
            </div>

            <div class="flex items-center justify-between gap-2 rounded-2xl bg-black/30 p-3">
              <span class="text-sm text-slate-400">
                {{ selectedContextIds.length }} seleccionados
              </span>

              <div class="flex gap-2">
                <button
                  @click="readSelectedContext"
                  :disabled="!selectedContextIds.length || reading"
                  class="rounded-xl bg-cyan-400 px-3 py-2 text-xs font-semibold text-slate-950 transition hover:bg-cyan-300 disabled:cursor-not-allowed disabled:opacity-50"
                >
                  {{ reading ? "Preparando..." : "▶ Leer" }}
                </button>

                <button
                  @click="stopReading"
                  :disabled="!reading"
                  class="rounded-xl bg-white/10 px-3 py-2 text-xs font-semibold text-slate-200 transition hover:bg-white/20 disabled:cursor-not-allowed disabled:opacity-50"
                >
                  ■ Detener
                </button>

                <button
                  @click="clearSelection"
                  :disabled="!selectedContextIds.length"
                  class="rounded-xl bg-white/10 px-3 py-2 text-xs font-semibold text-slate-200 transition hover:bg-white/20 disabled:cursor-not-allowed disabled:opacity-50"
                >
                  Limpiar
                </button>
              </div>
            </div>
          </div>

          <div v-if="!contextMessages.length" class="rounded-2xl bg-black/30 p-6 text-slate-400">
            Haz clic en un resultado para ver los mensajes cercanos.
          </div>

          <div v-else class="max-h-[720px] space-y-4 overflow-y-auto pr-2">
            <div
              v-for="message in contextMessages"
              :key="message.id"
              :id="`context-message-${message.id}`"
              class="flex scroll-mt-4"
              :class="isMine(message.sender_name) ? 'justify-end' : 'justify-start'"
            >
              <div
                class="max-w-[85%] rounded-2xl border px-4 py-3 transition-all"
                :class="[
                  isMine(message.sender_name)
                    ? 'bg-cyan-400 text-slate-950'
                    : 'bg-white/10 text-slate-100',

                  message.id === selectedMessageId
                    ? 'border-yellow-300 shadow-[0_0_25px_rgba(253,224,71,0.35)] scale-[1.01]'
                    : 'border-transparent',

                  selectedContextIds.includes(message.id)
                    ? 'ring-2 ring-yellow-300/60'
                    : ''
                ]"
              >
                <div class="mb-2 flex items-start justify-between gap-3">
                  <p class="text-xs font-semibold opacity-70">
                    {{ message.sender_name }}
                  </p>

                  <input
                    type="checkbox"
                    class="mt-1 h-4 w-4 cursor-pointer accent-yellow-300"
                    :checked="selectedContextIds.includes(message.id)"
                    @change="toggleContextSelection(message.id)"
                  />
                </div>

                <p
                  class="whitespace-pre-wrap text-sm leading-relaxed"
                  v-html="highlightText(message.content || `[${message.type}]`)"
                ></p>

                <p class="mt-2 text-[11px] opacity-60">
                  {{ formatDate(message.date) }}
                </p>
              </div>
            </div>
          </div>
        </section>
      </main>
    </div>
  </div>
</template>