<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue';

type DayStatus = 'free' | 'booked' | 'arrival' | 'departure';

type DayCell = {
    date: string;
    day: number;
    inMonth: boolean;
    isPast: boolean;
    status: DayStatus;
};

type Range = { from: string; to: string };

const apiBase = import.meta.env.PUBLIC_API_BASE;
const isDev = import.meta.env.DEV;

const cursor = ref(new Date());
const loading = ref(false);
const error = ref<string|null>(null);
const bookedRanges = ref<Range[]>([]);

function ymd(d: Date) { return d.toISOString().slice(0,10); }
function startOfMonth(d: Date) { return new Date(d.getFullYear(), d.getMonth(), 1); }
function endOfMonth(d: Date) { return new Date(d.getFullYear(), d.getMonth() + 1, 0); }
function addMonths(d: Date, n: number) { return new Date(d.getFullYear(), d.getMonth() + n, 1); }
function weekdayMondayFirst(d: Date) { const w = d.getDay(); return (w + 6) % 7; }

function buildBookedSet(ranges: Range[]): Set<string> {
    const s = new Set<string>();
    for (const r of ranges) {
        const a = new Date(r.from);
        const b = new Date(r.to);
        for (let d = new Date(a); d <= b; d.setDate(d.getDate() + 1)) {
            s.add(ymd(d));
        }
    }
    return s;
}

async function fetchMonth() {
    error.value = null;
    loading.value = true;
    bookedRanges.value = [];
    try {
        const first = startOfMonth(cursor.value);
        const last  = endOfMonth(cursor.value);
        const from = ymd(first);
        const to   = ymd(last);

        if (isDev) {
            await new Promise(r => setTimeout(r, 150));
            const y = first.getFullYear(), m = first.getMonth();
            const mk = (d1: number, d2: number): Range => ({ from: ymd(new Date(y, m, d1)), to: ymd(new Date(y, m, d2)) });
            bookedRanges.value = [ mk(6,7), mk(16,17), mk(26,27), mk(5,6) ];
        } else {
            const url = `${apiBase}/availability/range?from=${from}&to=${to}`;
            const res = await fetch(url);
            if (!res.ok) throw new Error('Erreur API calendrier');
            const data = await res.json();

            if (Array.isArray(data) && data.length && "from" in data[0]) {
                bookedRanges.value = data as Range[];
            } else if (Array.isArray(data) && data.length && "date" in data[0]) {
                const busy = (data as any[]).filter(d => d.available === false).map(d => d.date).sort();
                const out: Range[] = [];
                let s: string | null = null, prev: string | null = null;
                for (const dt of busy) {
                    if (!s) { s = dt; prev = dt; continue; }
                    const nxt = new Date(prev!); nxt.setDate(nxt.getDate() + 1);
                    if (ymd(nxt) === dt) prev = dt;
                    else { out.push({ from: s, to: prev! }); s = dt; prev = dt; }
                }
                if (s) out.push({ from: s, to: prev! });
                bookedRanges.value = out;
            } else {
                bookedRanges.value = [];
            }
        }
    } catch (e:any) {
        error.value = e.message ?? 'Erreur inconnue';
    } finally {
        loading.value = false;
    }
}

const monthLabel = computed(() =>
        cursor.value.toLocaleDateString('fr-BE', { month: 'long', year: 'numeric' })
);

const weeks = computed<DayCell[][]>(() => {
    const first = startOfMonth(cursor.value);
    const last  = endOfMonth(cursor.value);
    const today = new Date(); today.setHours(0,0,0,0);
    const before = weekdayMondayFirst(first);

    const busy = buildBookedSet(bookedRanges.value);

    function statusFor(dateISO: string): DayStatus {
        if (busy.has(dateISO)) return 'booked';
        const d = new Date(dateISO);
        const prev = new Date(d); prev.setDate(prev.getDate() - 1);
        const next = new Date(d); next.setDate(next.getDate() + 1);
        const prevBusy = busy.has(ymd(prev));
        const nextBusy = busy.has(ymd(next));

        if (prevBusy && nextBusy) return 'booked';
        if (prevBusy) return 'arrival';
        if (nextBusy) return 'departure';
        return 'free';
    }

    const cells: DayCell[] = [];
    for (let i = before; i > 0; i--) {
        const d = new Date(first); d.setDate(first.getDate() - i);
        const iso = ymd(d);
        cells.push({ date: iso, day: d.getDate(), inMonth: false, isPast: d < today, status: statusFor(iso) });
    }
    for (let day = 1; day <= last.getDate(); day++) {
        const d = new Date(first); d.setDate(day);
        const iso = ymd(d);
        cells.push({ date: iso, day, inMonth: true, isPast: d < today, status: statusFor(iso) });
    }
    while (cells.length < 42) {
        const d = new Date(last); d.setDate(last.getDate() + (cells.length - (before + last.getDate())) + 1);
        const iso = ymd(d);
        cells.push({ date: iso, day: d.getDate(), inMonth: false, isPast: d < today, status: statusFor(iso) });
    }
    const out: DayCell[][] = [];
    for (let i = 0; i < 42; i += 7) out.push(cells.slice(i, i+7));
    return out;
});

function prevMonth() { cursor.value = addMonths(cursor.value, -1); }
function nextMonth() { cursor.value = addMonths(cursor.value, +1); }

onMounted(fetchMonth);
watch(cursor, fetchMonth);
</script>

<template>
    <div class="rounded-2xl border border-gray-200 bg-white p-5">
        <div class="flex items-center justify-between mb-3">
            <h3 class="text-lg font-semibold">Calendrier des disponibilités</h3>
            <div class="flex items-center gap-2">
                <button @click="prevMonth" class="px-2 py-1 rounded-lg ring-1 ring-gray-300 hover:bg-gray-50">‹</button>
                <div class="min-w-[10ch] text-center font-medium capitalize">{{ monthLabel }}</div>
                <button @click="nextMonth" class="px-2 py-1 rounded-lg ring-1 ring-gray-300 hover:bg-gray-50">›</button>
            </div>
        </div>

        <div v-if="error" class="text-red-600 text-sm mb-2">{{ error }}</div>

        <div class="grid grid-cols-7 text-xs text-gray-500 mb-1">
            <div class="text-center py-1">L</div><div class="text-center py-1">M</div><div class="text-center py-1">M</div>
            <div class="text-center py-1">J</div><div class="text-center py-1">V</div><div class="text-center py-1">S</div><div class="text-center py-1">D</div>
        </div>

        <div v-if="loading" class="h-64 grid place-items-center text-gray-500">Chargement…</div>

        <div v-else class="grid grid-cols-7 gap-1">
            <template v-for="(week, wi) in weeks" :key="wi">
                <template v-for="day in week" :key="day.date">
                    <div
                            class="aspect-square rounded-xl border text-sm grid place-items-center relative"
                            :class="[ day.inMonth ? 'bg-white' : 'bg-gray-50 text-gray-400' ]"
                            :style="day.status === 'booked'
              ? 'background: rgba(16,185,129,.85); border-color: rgba(16,185,129,1); color:#fff;'
              : day.status === 'arrival'
              ? 'background: linear-gradient(to right, rgba(16,185,129,.85) 0%, rgba(16,185,129,.85) 65%, #ffffff 65%); border-color: rgba(16,185,129,1); color:#093c31;'
              : day.status === 'departure'
              ? 'background: linear-gradient(to left, rgba(16,185,129,.85) 0%, rgba(16,185,129,.85) 65%, #ffffff 65%); border-color: rgba(16,185,129,1); color:#093c31;'
              : ''"
                            :title="day.date + ' — ' + (
              day.status==='booked' ? 'occupé' :
              day.status==='arrival' ? 'arrivée possible' :
              day.status==='departure' ? 'départ possible' :
              'libre'
            )"
                    >
                        {{ day.day }}
                    </div>
                </template>
            </template>
        </div>

        <div class="mt-4 flex flex-wrap items-center gap-4 text-xs text-gray-600">
            <div class="flex items-center gap-1">
                <span class="inline-block w-3 h-3 rounded border border-gray-300 bg-white"></span> Libre
            </div>
            <div class="flex items-center gap-1">
                <span class="inline-block w-3 h-3 rounded border border-emerald-700" style="background: rgba(16,185,129,.85)"></span> Occupé
            </div>
            <div class="flex items-center gap-1">
        <span class="inline-block w-3 h-3 rounded border border-emerald-700"
              style="background: linear-gradient(to right, rgba(16,185,129,.85) 0%, rgba(16,185,129,.85) 65%, #ffffff 65%);"></span>
                Arrivée
            </div>
            <div class="flex items-center gap-1">
        <span class="inline-block w-3 h-3 rounded border border-emerald-700"
              style="background: linear-gradient(to left, rgba(16,185,129,.85) 0%, rgba(16,185,129,.85) 65%, #ffffff 65%);"></span>
                Départ
            </div>
        </div>
    </div>
</template>
