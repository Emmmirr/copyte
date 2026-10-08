<script setup>
import { ref } from "vue";
import TemplateManager from "./components/TemplateManager.vue";
import WorkInProgress from "./components/WorkInProgress.vue";

// Vistas: 'home' (selector tipo Chrome) | 'templates' | 'workspace'
const currentView = ref("home");
</script>

<template>
  <div
    class="w-screen h-screen flex flex-col bg-[#f8fafc] text-slate-800 font-sans overflow-hidden select-none"
  >
    <!-- ================= 1. PANTALLA INICIAL (SELECTOR ESTILO CHROME) ================= -->
    <div
      v-if="currentView === 'home'"
      class="w-full h-full flex flex-col items-center justify-center p-6 relative"
    >
      <!-- Título centrado -->
      <div class="text-center mb-10">
        <h1 class="text-2xl font-bold text-slate-900 tracking-tight">
          ¿Qué herramienta deseas usar?
        </h1>
        <p class="text-sm text-slate-500 mt-1">
          Selecciona un módulo para comenzar a trabajar.
        </p>
      </div>

      <!-- Fila de tarjetas con círculos grandes estilo perfiles de Chrome -->
      <div class="flex items-center justify-center gap-6 flex-wrap max-w-4xl">
        <!-- MÓDULO 1: PLANTILLAS -->
        <div
          @click="currentView = 'templates'"
          class="w-56 h-64 bg-white border border-slate-200/80 rounded-3xl p-6 flex flex-col items-center justify-center cursor-pointer hover:border-slate-400 hover:shadow-lg transition-all duration-200 group"
        >
          <div
            class="w-24 h-24 rounded-full bg-slate-100 border border-slate-200 flex items-center justify-center mb-4 group-hover:scale-105 transition-transform text-slate-700"
          >
            <svg
              class="w-10 h-10"
              fill="none"
              stroke="currentColor"
              viewBox="0 0 24 24"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="1.8"
                d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"
              />
            </svg>
          </div>

          <h2 class="text-sm font-bold text-slate-800">Plantillas</h2>
          <p class="text-xs text-slate-400 mt-1">Generador de mensajes</p>
        </div>

        <!-- MÓDULO 2: COBRANZA -->
        <div
          @click="currentView = 'workspace'"
          class="w-56 h-64 bg-white border border-slate-200/80 rounded-3xl p-6 flex flex-col items-center justify-center cursor-pointer hover:border-slate-400 hover:shadow-lg transition-all duration-200 group"
        >
          <div
            class="w-24 h-24 rounded-full bg-slate-100 border border-slate-200 flex items-center justify-center mb-4 group-hover:scale-105 transition-transform text-slate-700"
          >
            <svg
              class="w-9 h-9"
              fill="none"
              stroke="currentColor"
              viewBox="0 0 24 24"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="1.8"
                d="M9 7h6m0 10v-3m-3 3h.01M9 17h.01M9 14h.01M12 14h.01M15 11h.01M12 11h.01M9 11h.01M7 21h10a2 2 0 002-2V5a2 2 0 00-2-2H7a2 2 0 00-2 2v14a2 2 0 002 2z"
              />
            </svg>
          </div>

          <h2 class="text-sm font-bold text-slate-800">Cobranza</h2>
          <p class="text-xs text-slate-400 mt-1">Convenios y porcentajes</p>
        </div>
      </div>
    </div>

    <!-- ================= 2. VISTAS DE LAS HERRAMIENTAS ================= -->
    <div v-else class="w-full h-full relative">
      <TemplateManager
        v-if="currentView === 'templates'"
        @back="currentView = 'home'"
      />
      <WorkInProgress
        v-else-if="currentView === 'workspace'"
        @back="currentView = 'home'"
      />
    </div>
  </div>
</template>
