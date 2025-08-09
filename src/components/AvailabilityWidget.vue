<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';

type PriceResp = { nightly: number; currency: string };
type AvailResp = { available: boolean; minStay?: number };

const loading = ref(true);
const error = ref<string | null>(null);

const dateFrom = ref('');
const dateTo = ref('');
const guests = ref(2);

const availability = ref<AvailResp | null>(null);
const price = ref<PriceResp | null>(null);

const apiBase = import.meta.env.PUBLIC_API_BASE;
const isDev = import.meta.env.DEV;

async function fetchData() {
    error.value = null;
    loading.value = true;
    try {
        if (isDev) {
            // MOCK en local (ton API n’autorise pas localhost)
            await new Promise(r => setTimeout(r, 400));
            availability.value = { available: true, minStay: 2 };
            price.value = { nightly: 195, currency: 'EUR' };
        } else {
            const qs = new URLSearchParams({
                from: dateFrom.value,
                to: dateTo.value,
                guests: String(guests.value),
            });
            const [availRes, priceRes] = await Promise.all([
                fetch(`${apiBase}/availability?${qs}`),
                fetch(`${apiBase}/price?${qs}`),
            ]);
            if (!availRes.ok || !priceRes.ok) throw new Error('Erreur API');
            availability.value = await availRes.json();
            price.value = await priceRes.json();
        }
    } catch (e: any) {
        error.value = e.message ?? 'Erreur';
    } finally {
        loading.value = false;
    }
}

onMounted(() => {
    const start = new Date(); start.setDate(start.getDate() + 7);
    const end = new Date(start); end.setDate(start.getDate() + 2);
    dateFrom.value = start.toISOString().slice(0,10);
    dateTo.value = end.toISOString().slice(0,10);
    fetchData();
});

const priceText = computed(() => {
    if (!price.value) return '';
    return new Intl.NumberFormat(undefined, { style: 'currency', currency: price.value.currency }).format(price.value.nightly);
});
</script>

<template>
    <div class="p-5 rounded-2xl shadow bg-white space-y-3">
        <div class="grid grid-cols-2 gap-2">
            <label class="text-sm">Arrivée
                <input type="date" v-model="dateFrom" class="w-full border rounded-lg px-2 py-1">
            </label>
            <label class="text-sm">Départ
                <input type="date" v-model="dateTo" class="w-full border rounded-lg px-2 py-1">
            </label>
            <label class="col-span-2 text-sm">Invités
                <input type="number" min="1" v-model="guests" class="w-full border rounded-lg px-2 py-1">
            </label>
        </div>

        <button @click="fetchData" class="w-full py-2 rounded-2xl bg-black text-white">Mettre à jour</button>

        <div v-if="loading" class="animate-pulse h-10 bg-gray-100 rounded"></div>
        <p v-else-if="error" class="text-red-600">{{ error }}</p>
        <div v-else class="space-y-1">
            <p v-if="availability?.available">✅ Disponible</p>
            <p v-else>❌ Indisponible</p>
            <p v-if="price">À partir de <strong>{{ priceText }}</strong> / nuit</p>
            <p v-if="availability?.minStay">Séjour min. : {{ availability.minStay }} nuits</p>
        </div>
    </div>
</template>
