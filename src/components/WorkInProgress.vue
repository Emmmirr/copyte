<script setup>
import { ref, computed, watch, onMounted } from "vue";

// Emitir evento para regresar al menú inicial
defineEmits(["back"]);

// Pestaña activa: 'porcentajes' o 'cobranza'
const activeTab = ref("porcentajes");

// Toast global de copiado
const toast = ref({ show: false, title: "" });
let toastTimer = null;

const copyValue = async (val, label) => {
  try {
    const formatted = formatCurrency(Math.abs(val));
    const prefix = val < 0 ? "-$" : "$";
    const fullText = `${prefix}${formatted}`;
    await navigator.clipboard.writeText(fullText);

    if (toastTimer) clearTimeout(toastTimer);
    toast.value = { show: true, title: `Copiado: ${fullText} (${label})` };
    toastTimer = setTimeout(() => {
      toast.value.show = false;
    }, 2500);
  } catch (err) {
    alert("Error al copiar al portapapeles");
  }
};

const formatCurrency = (val) => {
  const num = parseFloat(val) || 0;
  return num.toLocaleString("en-US", {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  });
};

// ================= ESTADO: PESTAÑA 1 (PORCENTAJES) =================
const baseAmount = ref(10000);
const baseAmountDisplay = ref("10,000.00");

const percentages = ref([
  { id: 1, value: 10 },
  { id: 2, value: 20 },
  { id: 3, value: 30 },
  { id: 4, value: 40 },
  { id: 5, value: 50 },
]);

const handleBaseInput = (event) => {
  let val = event.target.value.replace(/[^0-9.]/g, "");
  const parts = val.split(".");
  if (parts.length > 2) val = parts[0] + "." + parts.slice(1).join("");
  if (parts[1] && parts[1].length > 2)
    val = parts[0] + "." + parts[1].slice(0, 2);

  const intFormatted = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, ",");
  baseAmountDisplay.value =
    parts.length > 1 ? `${intFormatted}.${parts[1]}` : intFormatted;
  baseAmount.value = parseFloat(val) || 0;
};

const handleBaseBlur = () => {
  if (!baseAmount.value && baseAmount.value !== 0) return;
  baseAmountDisplay.value = baseAmount.value.toLocaleString("en-US", {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  });
};

const getExactPercent = (percent) => {
  const p = parseFloat(percent) || 0;
  return (baseAmount.value * p) / 100;
};

const getRound50 = (val) => (!val || val <= 0 ? 0 : Math.ceil(val / 50) * 50);
const getRound100 = (val) =>
  !val || val <= 0 ? 0 : Math.ceil(val / 100) * 100;
const getSaldoConvenio = (discountAmount) =>
  Math.max(0, baseAmount.value - discountAmount);

const addPercentage = () => {
  const lastVal =
    percentages.value.length > 0
      ? percentages.value[percentages.value.length - 1].value
      : 0;
  percentages.value.push({ id: Date.now(), value: lastVal + 10 });
  savePercentagesStorage();
};

const removePercentage = (index) => {
  if (percentages.value.length <= 1) return;
  percentages.value.splice(index, 1);
  savePercentagesStorage();
};

const savePercentagesStorage = () => {
  localStorage.setItem(
    "porcentajes_convenios_v2",
    JSON.stringify(percentages.value),
  );
};

watch(percentages, () => savePercentagesStorage(), { deep: true });

// ================= ESTADO: PESTAÑA 2 (GASTO DE COBRANZA + DEUDA + PAGO MÍNIMO) =================
// 1. Cantidad de la Deuda
const debtAmount = ref(10000);
const debtAmountDisplay = ref("10,000.00");

const handleDebtInput = (event) => {
  let val = event.target.value.replace(/[^0-9.]/g, "");
  const parts = val.split(".");
  if (parts.length > 2) val = parts[0] + "." + parts.slice(1).join("");
  if (parts[1] && parts[1].length > 2)
    val = parts[0] + "." + parts[1].slice(0, 2);

  const intFormatted = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, ",");
  debtAmountDisplay.value =
    parts.length > 1 ? `${intFormatted}.${parts[1]}` : intFormatted;
  debtAmount.value = parseFloat(val) || 0;
};

const handleDebtBlur = () => {
  if (!debtAmount.value && debtAmount.value !== 0) return;
  debtAmountDisplay.value = debtAmount.value.toLocaleString("en-US", {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  });
};

// 2. Pago mínimo o pago para no generar intereses
const minPayment = ref(1500);
const minPaymentDisplay = ref("1,500.00");

const handleMinPaymentInput = (event) => {
  let val = event.target.value.replace(/[^0-9.]/g, "");
  const parts = val.split(".");
  if (parts.length > 2) val = parts[0] + "." + parts.slice(1).join("");
  if (parts[1] && parts[1].length > 2)
    val = parts[0] + "." + parts[1].slice(0, 2);

  const intFormatted = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, ",");
  minPaymentDisplay.value =
    parts.length > 1 ? `${intFormatted}.${parts[1]}` : intFormatted;
  minPayment.value = parseFloat(val) || 0;
};

const handleMinPaymentBlur = () => {
  if (!minPayment.value && minPayment.value !== 0) return;
  minPaymentDisplay.value = minPayment.value.toLocaleString("en-US", {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  });
};

// 3. Gastos de Cobranza (Cantidades)
const collectionItems = ref([{ id: 1, raw: 1000, display: "1,000.00" }]);

const handleCollectionInput = (index, event) => {
  let val = event.target.value.replace(/[^0-9.]/g, "");
  const parts = val.split(".");
  if (parts.length > 2) val = parts[0] + "." + parts.slice(1).join("");
  if (parts[1] && parts[1].length > 2)
    val = parts[0] + "." + parts[1].slice(0, 2);

  const intFormatted = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, ",");
  collectionItems.value[index].display =
    parts.length > 1 ? `${intFormatted}.${parts[1]}` : intFormatted;
  collectionItems.value[index].raw = parseFloat(val) || 0;
};

const handleCollectionBlur = (index) => {
  const item = collectionItems.value[index];
  if (!item) return;
  item.display = (item.raw || 0).toLocaleString("en-US", {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  });
};

const addCollectionItem = () => {
  collectionItems.value.push({
    id: Date.now(),
    raw: 0,
    display: "0.00",
  });
};

const removeCollectionItem = (index) => {
  if (collectionItems.value.length <= 1) return;
  collectionItems.value.splice(index, 1);
};

// CÁLCULOS
const totalCantidades = computed(() => {
  return collectionItems.value.reduce(
    (acc, item) => acc + (parseFloat(item.raw) || 0),
    0,
  );
});

const totalIva = computed(() => {
  return totalCantidades.value * 0.16;
});

const totalConIva = computed(() => {
  return totalCantidades.value + totalIva.value;
});

// 1. Resta: Deuda − (Cantidades + IVA)
const debtMinusExpenses = computed(() => {
  return debtAmount.value - totalConIva.value;
});

// 2. Resta secundaria: [Deuda − (Cantidades + IVA)] − Pago Mínimo
const finalRemainingWithMinPayment = computed(() => {
  return debtMinusExpenses.value - minPayment.value;
});

// CARGAR AL INICIAR
onMounted(() => {
  const saved = localStorage.getItem("porcentajes_convenios_v2");
  if (saved) {
    try {
      percentages.value = JSON.parse(saved);
    } catch (e) {}
  }
  handleBaseBlur();
  handleDebtBlur();
  handleMinPaymentBlur();
});
</script>

<template>
  <div
    class="w-full h-full flex flex-col bg-[#f8fafc] text-slate-800 overflow-hidden font-sans select-none"
  >
    <!-- ================= CABECERA DEL MÓDULO ================= -->
    <header
      class="h-16 px-6 lg:px-8 flex items-center justify-between border-b border-slate-200/70 bg-white flex-shrink-0"
    >
      <div class="flex items-center gap-4">
        <h2 class="text-base font-bold text-slate-900 tracking-tight">
          {{
            activeTab === "porcentajes"
              ? "Calculadora de Porcentajes"
              : "Gasto de Cobranza"
          }}
        </h2>

        <!-- Selector de Pestañas -->
        <div
          class="bg-slate-100 p-1 rounded-xl flex items-center gap-1 border border-slate-200/60 shadow-inner"
        >
          <button
            @click="activeTab = 'porcentajes'"
            class="px-3.5 py-1.5 rounded-lg text-xs font-bold transition-all cursor-pointer flex items-center gap-1.5"
            :class="
              activeTab === 'porcentajes'
                ? 'bg-white text-slate-900 shadow-sm'
                : 'text-slate-500 hover:text-slate-800'
            "
          >
            Porcentajes
          </button>

          <button
            @click="activeTab = 'cobranza'"
            class="px-3.5 py-1.5 rounded-lg text-xs font-bold transition-all cursor-pointer flex items-center gap-1.5"
            :class="
              activeTab === 'cobranza'
                ? 'bg-white text-slate-900 shadow-sm'
                : 'text-slate-500 hover:text-slate-800'
            "
          >
            Gasto de Cobranza
          </button>
        </div>
      </div>

      <!-- Botón para regresar al selector de módulos -->
      <button
        @click="$emit('back')"
        class="px-3.5 py-1.5 rounded-xl border border-slate-200 bg-white hover:bg-slate-50 text-slate-600 hover:text-slate-900 text-xs font-semibold flex items-center gap-1.5 shadow-xs transition-all cursor-pointer active:scale-95"
        title="Volver a la selección de aplicaciones"
      >
        <svg
          class="w-3.5 h-3.5"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M4 6a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2H6a2 2 0 01-2-2V6zM14 6a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2V6zM4 16a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2H6a2 2 0 01-2-2v-2zM14 16a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2v-2z"
          />
        </svg>
        <span>Módulos</span>
      </button>
    </header>

    <!-- ================= CONTENIDO SCROLLABLE ================= -->
    <main class="flex-1 overflow-y-auto p-6 lg:p-8">
      <!-- ================= PESTAÑA 1: PORCENTAJES ================= -->
      <div
        v-if="activeTab === 'porcentajes'"
        class="max-w-6xl mx-auto space-y-6"
      >
        <!-- CANTIDAD BASE -->
        <div
          class="bg-white rounded-2xl p-6 lg:p-8 shadow-sm border border-slate-200/80"
        >
          <div class="max-w-md">
            <label
              class="block text-xs font-bold text-slate-400 uppercase tracking-wider mb-2"
            >
              Cantidad Base (Monto total o adeudo)
            </label>

            <div class="relative flex items-center">
              <span
                class="absolute left-4 text-lg font-bold text-slate-400 select-none"
                >$</span
              >
              <input
                :value="baseAmountDisplay"
                @input="handleBaseInput"
                @blur="handleBaseBlur"
                type="text"
                inputmode="decimal"
                class="w-full bg-slate-50 border border-slate-200 rounded-xl pl-9 pr-4 py-3 text-2xl font-bold text-slate-900 outline-none focus:bg-white focus:ring-2 focus:ring-slate-300 focus:border-slate-400 transition-all font-mono"
                placeholder="0.00"
              />
            </div>
          </div>
        </div>

        <!-- GRID DE PORCENTAJES -->
        <div
          class="bg-white rounded-2xl p-6 lg:p-8 shadow-sm border border-slate-200/80"
        >
          <div
            class="flex items-center justify-between mb-6 pb-4 border-b border-slate-100"
          >
            <div>
              <h3 class="text-sm font-bold text-slate-900">
                Porcentajes y Escenarios
              </h3>
              <p class="text-xs text-slate-400 mt-0.5">
                Compara el cálculo exacto frente a los redondeos al primer 50 y
                primer 100.
              </p>
            </div>

            <button
              @click="addPercentage"
              class="px-3.5 py-2 rounded-xl bg-[#1a1b26] text-white text-xs font-semibold hover:bg-slate-800 transition-all flex items-center gap-1.5 cursor-pointer shadow-xs active:scale-95"
            >
              <svg
                class="w-3.5 h-3.5"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2.5"
                  d="M12 4v16m8-8H4"
                ></path>
              </svg>
              <span>Agregar porcentaje</span>
            </button>
          </div>

          <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-6">
            <div
              v-for="(item, idx) in percentages"
              :key="item.id"
              class="bg-slate-50/60 border border-slate-200/90 rounded-2xl p-5 flex flex-col justify-between hover:border-slate-300 transition-all shadow-xs"
            >
              <!-- Cabecera -->
              <div
                class="flex items-center justify-between pb-3.5 border-b border-slate-200/70 mb-4"
              >
                <div class="flex items-center gap-2">
                  <span
                    class="text-xs font-bold text-slate-500 uppercase tracking-wider"
                    >Porcentaje:</span
                  >
                  <div
                    class="flex items-center bg-white border border-slate-200 rounded-lg px-2.5 py-1 shadow-xs focus-within:border-slate-400 focus-within:ring-1 focus-within:ring-slate-300 transition-all"
                  >
                    <input
                      v-model="item.value"
                      type="number"
                      step="any"
                      min="0"
                      class="w-12 bg-transparent border-none outline-none text-xs font-bold text-slate-900 font-mono text-center"
                    />
                    <span class="text-xs font-bold text-slate-400 select-none"
                      >%</span
                    >
                  </div>
                </div>

                <button
                  v-if="percentages.length > 1"
                  @click="removePercentage(idx)"
                  class="w-6 h-6 rounded-lg text-slate-300 hover:text-red-500 hover:bg-red-50 flex items-center justify-center transition-colors cursor-pointer text-sm font-bold"
                  title="Eliminar este porcentaje"
                >
                  ×
                </button>
              </div>

              <!-- Las 3 Posibilidades -->
              <div class="space-y-3">
                <!-- 1. Exacto -->
                <div
                  class="bg-white border border-slate-200/80 rounded-xl p-3 shadow-2xs"
                >
                  <span
                    class="text-[10px] font-bold text-slate-400 uppercase tracking-wider block mb-2"
                    >1. Cálculo Exacto</span
                  >

                  <div class="grid grid-cols-2 gap-2 text-xs">
                    <div
                      class="bg-slate-50/80 border border-slate-100 rounded-lg p-2 flex items-center justify-between"
                    >
                      <div class="overflow-hidden pr-1">
                        <span
                          class="text-[10px] text-slate-400 block font-medium"
                          >Porcentaje</span
                        >
                        <span
                          class="font-bold font-mono text-slate-800 truncate block"
                          >${{
                            formatCurrency(getExactPercent(item.value))
                          }}</span
                        >
                      </div>
                      <button
                        @click="
                          copyValue(
                            getExactPercent(item.value),
                            'Porcentaje exacto',
                          )
                        "
                        class="p-1 rounded-md text-slate-400 hover:text-slate-800 hover:bg-slate-200/60 transition-colors cursor-pointer"
                        title="Copiar porcentaje exacto"
                      >
                        <svg
                          class="w-3.5 h-3.5"
                          fill="none"
                          stroke="currentColor"
                          viewBox="0 0 24 24"
                        >
                          <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                          />
                        </svg>
                      </button>
                    </div>

                    <div
                      class="bg-slate-50/80 border border-slate-100 rounded-lg p-2 flex items-center justify-between"
                    >
                      <div class="overflow-hidden pr-1">
                        <span
                          class="text-[10px] text-slate-400 block font-medium"
                          >Saldo a convenio</span
                        >
                        <span
                          class="font-bold font-mono text-slate-900 truncate block"
                          >${{
                            formatCurrency(
                              getSaldoConvenio(getExactPercent(item.value)),
                            )
                          }}</span
                        >
                      </div>
                      <button
                        @click="
                          copyValue(
                            getSaldoConvenio(getExactPercent(item.value)),
                            'Saldo exacto',
                          )
                        "
                        class="p-1 rounded-md text-slate-400 hover:text-slate-800 hover:bg-slate-200/60 transition-colors cursor-pointer"
                        title="Copiar saldo a convenio"
                      >
                        <svg
                          class="w-3.5 h-3.5"
                          fill="none"
                          stroke="currentColor"
                          viewBox="0 0 24 24"
                        >
                          <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                          />
                        </svg>
                      </button>
                    </div>
                  </div>
                </div>

                <!-- 2. Al Primer 50 -->
                <div
                  class="bg-white border border-slate-200/80 rounded-xl p-3 shadow-2xs"
                >
                  <div class="flex items-center justify-between mb-2">
                    <span
                      class="text-[10px] font-bold text-slate-500 uppercase tracking-wider"
                      >2. Al Primer 50</span
                    >
                    <span class="text-[10px] text-slate-400 font-mono"
                      >+$50</span
                    >
                  </div>

                  <div class="grid grid-cols-2 gap-2 text-xs">
                    <div
                      class="bg-slate-50/80 border border-slate-100 rounded-lg p-2 flex items-center justify-between"
                    >
                      <div class="overflow-hidden pr-1">
                        <span
                          class="text-[10px] text-slate-400 block font-medium"
                          >Porcentaje</span
                        >
                        <span
                          class="font-bold font-mono text-slate-800 truncate block"
                          >${{
                            formatCurrency(
                              getRound50(getExactPercent(item.value)),
                            )
                          }}</span
                        >
                      </div>
                      <button
                        @click="
                          copyValue(
                            getRound50(getExactPercent(item.value)),
                            'Porcentaje a 50',
                          )
                        "
                        class="p-1 rounded-md text-slate-400 hover:text-slate-800 hover:bg-slate-200/60 transition-colors cursor-pointer"
                        title="Copiar porcentaje redondeado a 50"
                      >
                        <svg
                          class="w-3.5 h-3.5"
                          fill="none"
                          stroke="currentColor"
                          viewBox="0 0 24 24"
                        >
                          <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                          />
                        </svg>
                      </button>
                    </div>

                    <div
                      class="bg-slate-50/80 border border-slate-100 rounded-lg p-2 flex items-center justify-between"
                    >
                      <div class="overflow-hidden pr-1">
                        <span
                          class="text-[10px] text-slate-400 block font-medium"
                          >Saldo a convenio</span
                        >
                        <span
                          class="font-bold font-mono text-slate-900 truncate block"
                          >${{
                            formatCurrency(
                              getSaldoConvenio(
                                getRound50(getExactPercent(item.value)),
                              ),
                            )
                          }}</span
                        >
                      </div>
                      <button
                        @click="
                          copyValue(
                            getSaldoConvenio(
                              getRound50(getExactPercent(item.value)),
                            ),
                            'Saldo con redondeo a 50',
                          )
                        "
                        class="p-1 rounded-md text-slate-400 hover:text-slate-800 hover:bg-slate-200/60 transition-colors cursor-pointer"
                        title="Copiar saldo con redondeo a 50"
                      >
                        <svg
                          class="w-3.5 h-3.5"
                          fill="none"
                          stroke="currentColor"
                          viewBox="0 0 24 24"
                        >
                          <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                          />
                        </svg>
                      </button>
                    </div>
                  </div>
                </div>

                <!-- 3. Al Primer 100 -->
                <div
                  class="bg-white border border-slate-200/80 rounded-xl p-3 shadow-2xs"
                >
                  <div class="flex items-center justify-between mb-2">
                    <span
                      class="text-[10px] font-bold text-slate-500 uppercase tracking-wider"
                      >3. Al Primer 100</span
                    >
                    <span class="text-[10px] text-slate-400 font-mono"
                      >+$100</span
                    >
                  </div>

                  <div class="grid grid-cols-2 gap-2 text-xs">
                    <div
                      class="bg-slate-50/80 border border-slate-100 rounded-lg p-2 flex items-center justify-between"
                    >
                      <div class="overflow-hidden pr-1">
                        <span
                          class="text-[10px] text-slate-400 block font-medium"
                          >Porcentaje</span
                        >
                        <span
                          class="font-bold font-mono text-slate-800 truncate block"
                          >${{
                            formatCurrency(
                              getRound100(getExactPercent(item.value)),
                            )
                          }}</span
                        >
                      </div>
                      <button
                        @click="
                          copyValue(
                            getRound100(getExactPercent(item.value)),
                            'Porcentaje a 100',
                          )
                        "
                        class="p-1 rounded-md text-slate-400 hover:text-slate-800 hover:bg-slate-200/60 transition-colors cursor-pointer"
                        title="Copiar porcentaje redondeado a 100"
                      >
                        <svg
                          class="w-3.5 h-3.5"
                          fill="none"
                          stroke="currentColor"
                          viewBox="0 0 24 24"
                        >
                          <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                          />
                        </svg>
                      </button>
                    </div>

                    <div
                      class="bg-slate-50/80 border border-slate-100 rounded-lg p-2 flex items-center justify-between"
                    >
                      <div class="overflow-hidden pr-1">
                        <span
                          class="text-[10px] text-slate-400 block font-medium"
                          >Saldo a convenio</span
                        >
                        <span
                          class="font-bold font-mono text-slate-900 truncate block"
                          >${{
                            formatCurrency(
                              getSaldoConvenio(
                                getRound100(getExactPercent(item.value)),
                              ),
                            )
                          }}</span
                        >
                      </div>
                      <button
                        @click="
                          copyValue(
                            getSaldoConvenio(
                              getRound100(getExactPercent(item.value)),
                            ),
                            'Saldo con redondeo a 100',
                          )
                        "
                        class="p-1 rounded-md text-slate-400 hover:text-slate-800 hover:bg-slate-200/60 transition-colors cursor-pointer"
                        title="Copiar saldo con redondeo a 100"
                      >
                        <svg
                          class="w-3.5 h-3.5"
                          fill="none"
                          stroke="currentColor"
                          viewBox="0 0 24 24"
                        >
                          <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                          />
                        </svg>
                      </button>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- ================= PESTAÑA 2: GASTO DE COBRANZA ================= -->
      <div v-else class="max-w-4xl mx-auto space-y-6">
        <!-- ENTRADA DE LA DEUDA, PAGO MÍNIMO Y GASTOS -->
        <div
          class="bg-white rounded-2xl p-6 lg:p-8 shadow-sm border border-slate-200/80 space-y-6"
        >
          <!-- 1. CANTIDAD DE LA DEUDA Y PAGO MÍNIMO (LADO A LADO) -->
          <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
            <!-- Deuda Total -->
            <div>
              <div class="mb-2 pb-1 border-b border-slate-100">
                <label
                  class="block text-xs font-bold text-slate-800 uppercase tracking-wider"
                >
                  Cantidad de la Deuda
                </label>
                <p class="text-[11px] text-slate-400 mt-0.5">
                  Monto total adeudado
                </p>
              </div>

              <div class="relative flex items-center">
                <span
                  class="absolute left-3.5 text-base font-bold text-slate-400 select-none"
                  >$</span
                >
                <input
                  :value="debtAmountDisplay"
                  @input="handleDebtInput"
                  @blur="handleDebtBlur"
                  type="text"
                  inputmode="decimal"
                  class="w-full bg-slate-50 border border-slate-200 rounded-xl pl-8 pr-4 py-2.5 text-lg font-bold text-slate-900 outline-none focus:bg-white focus:ring-2 focus:ring-slate-300 focus:border-slate-400 transition-all font-mono"
                  placeholder="0.00"
                />
              </div>
            </div>

            <!-- Pago mínimo o para no generar intereses -->
            <div>
              <div class="mb-2 pb-1 border-b border-slate-100">
                <label
                  class="block text-xs font-bold text-slate-800 uppercase tracking-wider truncate"
                >
                  Pago mínimo o sin intereses
                </label>
                <p class="text-[11px] text-slate-400 mt-0.5">
                  Monto mínimo o para no generar intereses
                </p>
              </div>

              <div class="relative flex items-center">
                <span
                  class="absolute left-3.5 text-base font-bold text-slate-400 select-none"
                  >$</span
                >
                <input
                  :value="minPaymentDisplay"
                  @input="handleMinPaymentInput"
                  @blur="handleMinPaymentBlur"
                  type="text"
                  inputmode="decimal"
                  class="w-full bg-slate-50 border border-slate-200 rounded-xl pl-8 pr-4 py-2.5 text-lg font-bold text-slate-900 outline-none focus:bg-white focus:ring-2 focus:ring-slate-300 focus:border-slate-400 transition-all font-mono"
                  placeholder="0.00"
                />
              </div>
            </div>
          </div>

          <!-- 2. GASTOS DE COBRANZA (CAMPOS COMPACTOS Y ESTABLES) -->
          <div class="pt-4 border-t border-slate-100">
            <div
              class="flex items-center justify-between mb-3 pb-2 border-b border-slate-100"
            >
              <div>
                <label
                  class="block text-xs font-bold text-slate-800 uppercase tracking-wider"
                >
                  Gastos de Cobranza (Cantidades)
                </label>
                <p class="text-xs text-slate-400 mt-0.5">
                  Cada cantidad calcula y muestra su respectivo IVA del 16%.
                </p>
              </div>

              <!-- Botón para agregar más cantidades -->
              <button
                @click="addCollectionItem"
                class="px-3 py-1.5 rounded-xl border border-slate-200 bg-white hover:bg-slate-50 text-slate-700 text-xs font-semibold flex items-center gap-1.5 transition-colors cursor-pointer shadow-xs active:scale-95"
              >
                <svg
                  class="w-3.5 h-3.5"
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2.5"
                    d="M12 4v16m8-8H4"
                  ></path>
                </svg>
                <span>Agregar otra cantidad</span>
              </button>
            </div>

            <!-- Lista de cantidades con inputs compactos de tamaño fijo -->
            <div class="space-y-2.5">
              <div
                v-for="(item, idx) in collectionItems"
                :key="item.id"
                class="flex flex-wrap sm:flex-nowrap items-center justify-between gap-3 p-2.5 bg-slate-50/70 border border-slate-200/80 rounded-xl"
              >
                <!-- Entrada de Monto compacta y fija (w-36 / w-40 fija, sin deformaciones) -->
                <div class="flex items-center gap-2">
                  <span
                    class="text-xs font-bold text-slate-400 w-5 text-center select-none"
                    >{{ idx + 1 }}.</span
                  >
                  <div class="relative flex items-center w-36 sm:w-40">
                    <span
                      class="absolute left-3 text-xs font-bold text-slate-400 select-none"
                      >$</span
                    >
                    <input
                      :value="item.display"
                      @input="handleCollectionInput(idx, $event)"
                      @blur="handleCollectionBlur(idx)"
                      type="text"
                      inputmode="decimal"
                      class="w-full bg-white border border-slate-200 rounded-lg pl-7 pr-2.5 py-1.5 text-xs font-bold text-slate-900 outline-none focus:ring-2 focus:ring-slate-300 focus:border-slate-400 transition-all font-mono"
                      placeholder="0.00"
                    />
                  </div>
                </div>

                <!-- IVA individual + Total del renglón + Acciones -->
                <div class="flex items-center gap-2 ml-auto">
                  <!-- Tarjeta IVA Individual (16%) -->
                  <div
                    class="bg-white border border-slate-200/80 rounded-lg px-2.5 py-1 flex items-center gap-1.5 shadow-2xs"
                  >
                    <span
                      class="text-[10px] text-slate-400 uppercase font-semibold"
                      >IVA 16%:</span
                    >
                    <span class="text-xs font-bold font-mono text-slate-800"
                      >${{ formatCurrency((item.raw || 0) * 0.16) }}</span
                    >
                    <button
                      @click="
                        copyValue(
                          (item.raw || 0) * 0.16,
                          `IVA de cantidad #${idx + 1}`,
                        )
                      "
                      class="p-0.5 rounded text-slate-400 hover:text-slate-800 hover:bg-slate-100 transition-colors cursor-pointer ml-0.5"
                      title="Copiar IVA"
                    >
                      <svg
                        class="w-3.5 h-3.5"
                        fill="none"
                        stroke="currentColor"
                        viewBox="0 0 24 24"
                      >
                        <path
                          stroke-linecap="round"
                          stroke-linejoin="round"
                          stroke-width="2"
                          d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                        />
                      </svg>
                    </button>
                  </div>

                  <!-- Tarjeta Total con IVA de este renglón -->
                  <div
                    class="bg-white border border-slate-200/80 rounded-lg px-2.5 py-1 flex items-center gap-1.5 shadow-2xs"
                  >
                    <span
                      class="text-[10px] text-slate-400 uppercase font-semibold"
                      >+ IVA:</span
                    >
                    <span class="text-xs font-bold font-mono text-slate-900"
                      >${{ formatCurrency((item.raw || 0) * 1.16) }}</span
                    >
                    <button
                      @click="
                        copyValue(
                          (item.raw || 0) * 1.16,
                          `Total con IVA #${idx + 1}`,
                        )
                      "
                      class="p-0.5 rounded text-slate-400 hover:text-slate-800 hover:bg-slate-100 transition-colors cursor-pointer ml-0.5"
                      title="Copiar total con IVA"
                    >
                      <svg
                        class="w-3.5 h-3.5"
                        fill="none"
                        stroke="currentColor"
                        viewBox="0 0 24 24"
                      >
                        <path
                          stroke-linecap="round"
                          stroke-linejoin="round"
                          stroke-width="2"
                          d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                        />
                      </svg>
                    </button>
                  </div>

                  <!-- Eliminar cantidad -->
                  <button
                    v-if="collectionItems.length > 1"
                    @click="removeCollectionItem(idx)"
                    class="w-6 h-6 rounded-lg text-slate-300 hover:text-red-500 hover:bg-red-50 flex items-center justify-center transition-colors cursor-pointer text-xs font-bold"
                    title="Eliminar esta cantidad"
                  >
                    ×
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- TARJETA DE RESULTADOS Y DESGLOSE COMPLETO -->
        <div
          class="bg-white rounded-2xl p-6 lg:p-8 shadow-sm border border-slate-200/80 space-y-6"
        >
          <div class="border-b border-slate-100 pb-3">
            <h3 class="text-sm font-bold text-slate-900">
              Desglose y Saldos Resultantes
            </h3>
            <p class="text-xs text-slate-400 mt-0.5">
              Totales de gastos y cálculos de saldo restante tras gastos y pago
              mínimo.
            </p>
          </div>

          <!-- FILA 1: GASTOS E IVA -->
          <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
            <!-- Total de Cantidades -->
            <div
              class="bg-slate-50/70 border border-slate-200/80 rounded-2xl p-4 flex flex-col justify-between"
            >
              <div>
                <span
                  class="text-[10px] font-bold text-slate-400 uppercase tracking-wider block mb-1"
                >
                  Total Cantidades
                </span>
                <p
                  class="text-lg font-bold text-slate-900 font-mono tracking-tight truncate"
                >
                  ${{ formatCurrency(totalCantidades) }}
                </p>
              </div>

              <button
                @click="copyValue(totalCantidades, 'Total cantidades')"
                class="w-full mt-3 py-1.5 rounded-lg border border-slate-200 bg-white hover:bg-slate-100 text-slate-700 text-xs font-semibold flex items-center justify-center gap-1.5 transition-colors cursor-pointer active:scale-95 shadow-xs"
              >
                <svg
                  class="w-3.5 h-3.5 text-slate-400"
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                  />
                </svg>
                <span>Copiar</span>
              </button>
            </div>

            <!-- Total IVA (16%) -->
            <div
              class="bg-slate-50/70 border border-slate-200/80 rounded-2xl p-4 flex flex-col justify-between"
            >
              <div>
                <span
                  class="text-[10px] font-bold text-slate-400 uppercase tracking-wider block mb-1"
                >
                  Total IVA (16%)
                </span>
                <p
                  class="text-lg font-bold text-slate-900 font-mono tracking-tight truncate"
                >
                  ${{ formatCurrency(totalIva) }}
                </p>
              </div>

              <button
                @click="copyValue(totalIva, 'Total IVA 16%')"
                class="w-full mt-3 py-1.5 rounded-lg border border-slate-200 bg-white hover:bg-slate-100 text-slate-700 text-xs font-semibold flex items-center justify-center gap-1.5 transition-colors cursor-pointer active:scale-95 shadow-xs"
              >
                <svg
                  class="w-3.5 h-3.5 text-slate-400"
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                  />
                </svg>
                <span>Copiar</span>
              </button>
            </div>

            <!-- Gastos + IVA -->
            <div
              class="bg-slate-900 text-white rounded-2xl p-4 flex flex-col justify-between shadow-sm"
            >
              <div>
                <span
                  class="text-[10px] font-bold text-slate-300 uppercase tracking-wider block mb-1"
                >
                  Gastos + IVA
                </span>
                <p
                  class="text-lg font-bold text-white font-mono tracking-tight truncate"
                >
                  ${{ formatCurrency(totalConIva) }}
                </p>
              </div>

              <button
                @click="copyValue(totalConIva, 'Total gastos con IVA')"
                class="w-full mt-3 py-1.5 rounded-lg bg-white/10 hover:bg-white/20 text-white text-xs font-semibold flex items-center justify-center gap-1.5 transition-colors cursor-pointer active:scale-95 border border-white/20"
              >
                <svg
                  class="w-3.5 h-3.5 text-slate-200"
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                  />
                </svg>
                <span>Copiar</span>
              </button>
            </div>
          </div>

          <!-- FILA 2: LAS DOS RESTAS RESULTANTES (DESTACADAS) -->
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4 pt-2">
            <!-- 1. DEUDA − (GASTOS + IVA) -->
            <div
              class="bg-slate-50 border border-slate-200/90 rounded-2xl p-5 flex flex-col justify-between"
            >
              <div>
                <span
                  class="text-[11px] font-bold text-slate-500 uppercase tracking-wider block mb-1"
                >
                  1. Deuda − (Gastos + IVA)
                </span>
                <p
                  class="text-2xl font-bold font-mono tracking-tight truncate"
                  :class="
                    debtMinusExpenses < 0 ? 'text-rose-600' : 'text-slate-900'
                  "
                >
                  {{ debtMinusExpenses < 0 ? "-$" : "$"
                  }}{{ formatCurrency(Math.abs(debtMinusExpenses)) }}
                </p>
                <p class="text-[11px] text-slate-400 mt-1">
                  Saldo restante antes de considerar pago mínimo
                </p>
              </div>

              <button
                @click="
                  copyValue(debtMinusExpenses, 'Deuda menos gastos con IVA')
                "
                class="w-full mt-4 py-2 rounded-xl border border-slate-200 bg-white hover:bg-slate-100 text-slate-800 text-xs font-semibold flex items-center justify-center gap-1.5 transition-colors cursor-pointer active:scale-95 shadow-xs"
              >
                <svg
                  class="w-3.5 h-3.5 text-slate-500"
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                  />
                </svg>
                <span>Copiar Saldo</span>
              </button>
            </div>

            <!-- 2. [DEUDA − (GASTOS + IVA)] − PAGO MÍNIMO -->
            <div
              class="bg-emerald-50/70 border border-emerald-200 rounded-2xl p-5 flex flex-col justify-between shadow-xs"
            >
              <div>
                <span
                  class="text-[11px] font-bold text-emerald-800 uppercase tracking-wider block mb-1"
                >
                  2. Resta final tras Pago Mínimo
                </span>
                <p
                  class="text-2xl font-bold font-mono tracking-tight truncate"
                  :class="
                    finalRemainingWithMinPayment < 0
                      ? 'text-rose-600'
                      : 'text-emerald-950'
                  "
                >
                  {{ finalRemainingWithMinPayment < 0 ? "-$" : "$"
                  }}{{ formatCurrency(Math.abs(finalRemainingWithMinPayment)) }}
                </p>
                <p class="text-[11px] text-emerald-700/80 mt-1">
                  [Deuda − Gastos con IVA] − Pago mínimo o sin intereses
                </p>
              </div>

              <button
                @click="
                  copyValue(
                    finalRemainingWithMinPayment,
                    'Resta final tras pago mínimo',
                  )
                "
                class="w-full mt-4 py-2 rounded-xl border border-emerald-300 bg-white hover:bg-emerald-100/60 text-emerald-900 text-xs font-semibold flex items-center justify-center gap-1.5 transition-colors cursor-pointer active:scale-95 shadow-xs"
              >
                <svg
                  class="w-3.5 h-3.5 text-emerald-600"
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    stroke-width="2"
                    d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"
                  />
                </svg>
                <span>Copiar Resta Final</span>
              </button>
            </div>
          </div>
        </div>
      </div>
    </main>

    <!-- Toast de Notificación -->
    <div
      class="fixed bottom-8 right-8 z-50 transition-all duration-300 transform"
      :class="
        toast.show
          ? 'translate-y-0 opacity-100'
          : 'translate-y-8 opacity-0 pointer-events-none'
      "
    >
      <div
        class="bg-white border border-slate-200 rounded-2xl shadow-xl px-4 py-3 flex items-center gap-2.5"
      >
        <div
          class="w-5 h-5 rounded-full bg-emerald-100 text-emerald-600 flex items-center justify-center"
        >
          <svg
            class="w-3.5 h-3.5 stroke-[3]"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M5 13l4 4L19 7"
            ></path>
          </svg>
        </div>
        <span class="text-xs font-bold text-slate-800">{{ toast.title }}</span>
      </div>
    </div>
  </div>
</template>
