<script setup>
import { ref, computed } from 'vue'

const nombre   = ref('')
const whatsapp = ref('')
const fecha    = ref('')
const notas    = ref('')
const enviado  = ref(false)

// Próximos 30 días hábiles (lun–sáb, sin domingos)
const fechasDisponibles = computed(() => {
  const dias = []
  const cursor = new Date()
  cursor.setHours(0, 0, 0, 0)
  while (dias.length < 30) {
    cursor.setDate(cursor.getDate() + 1)
    if (cursor.getDay() !== 0) dias.push(new Date(cursor))
  }
  return dias
})

const formatearFecha = (d) =>
  d.toLocaleDateString('es-AR', { weekday: 'long', day: 'numeric', month: 'long' })
    .replace(/^\w/, c => c.toUpperCase())

const enviar = () => { enviado.value = true }

const nuevaReserva = () => {
  nombre.value = ''; whatsapp.value = ''; fecha.value = ''; notas.value = ''
  enviado.value = false
}
</script>

<template>
  <div class="screen">
    <div class="bg-grid"></div>
    <div class="glow left"></div>
    <div class="glow right"></div>

    <div class="card">
      <header class="card-header">
        <span class="eyebrow">Reservá online</span>
        <h1>Agendar Turno</h1>
        <p class="subtitle">Completá el formulario y confirmamos tu turno al instante.</p>
        <div class="divider"><span></span><span class="diamond">◆</span><span></span></div>
      </header>

      <transition name="fade" mode="out-in">

        <!-- ── ÉXITO ── -->
        <div v-if="enviado" class="success-panel" key="ok">
          <div class="success-icon">✦</div>
          <h2>¡Turno solicitado!</h2>
          <p>Te contactaremos por WhatsApp para confirmar tu reserva.</p>
          <button class="btn-outline" @click="nuevaReserva">Hacer otra reserva</button>
        </div>

        <!-- ── FORMULARIO ── -->
        <form v-else key="form" class="form" @submit.prevent="enviar">

          <div class="field">
            <label>Tu nombre <span class="req">*</span></label>
            <input v-model="nombre" type="text" required placeholder="Ej: Martín García" />
          </div>

          <div class="field">
            <label>WhatsApp <span class="req">*</span></label>
            <input v-model="whatsapp" type="tel" required placeholder="+54 9 351 000-0000" />
            <span class="hint">Para confirmar o consultar tu turno</span>
          </div>

          <div class="field">
            <label>Fecha <span class="req">*</span></label>
            <select v-model="fecha" required>
              <option value="" disabled>Seleccioná una fecha</option>
              <option
                v-for="d in fechasDisponibles"
                :key="d.toISOString()"
                :value="d.toISOString().split('T')[0]"
              >
                {{ formatearFecha(d) }}
              </option>
            </select>
          </div>

          <div class="field">
            <label>Notas adicionales</label>
            <textarea v-model="notas" rows="3" placeholder="Alguna preferencia o consulta especial..."></textarea>
          </div>

          <button type="submit" class="btn-primary">
            <span>Confirmar Turno</span>
          </button>

        </form>
      </transition>
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@400;600;700&family=Montserrat:wght@300;400;500;600&display=swap');

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

.screen {
  min-height: 100vh;
  background: #0d0d0d;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 60px 16px;
  font-family: 'Montserrat', sans-serif;
  position: relative;
  overflow: hidden;
}

.bg-grid {
  position: fixed; inset: 0;
  background-image:
    linear-gradient(rgba(180,145,80,.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(180,145,80,.03) 1px, transparent 1px);
  background-size: 60px 60px;
  pointer-events: none;
}

.glow {
  position: fixed;
  width: 500px; height: 500px;
  border-radius: 50%;
  filter: blur(120px);
  pointer-events: none;
  opacity: .3;
}
.glow.left  { background: radial-gradient(circle, #b4915025, transparent 70%); top: -100px; left: -150px; }
.glow.right { background: radial-gradient(circle, #b4915015, transparent 70%); bottom: -100px; right: -150px; }

/* ── Tarjeta ── */
.card {
  position: relative;
  width: 100%; max-width: 520px;
  background: #141414;
  border: 1px solid rgba(180,145,80,.2);
  box-shadow: 0 32px 80px rgba(0,0,0,.7), inset 0 1px 0 rgba(180,145,80,.08);
  padding: 48px 44px 44px;
  animation: cardIn .65s cubic-bezier(.16,1,.3,1) both;
}
.card::before, .card::after {
  content: ''; position: absolute;
  width: 20px; height: 20px;
  border-color: #b49150; border-style: solid;
}
.card::before { top: 12px; left: 12px; border-width: 1px 0 0 1px; }
.card::after  { bottom: 12px; right: 12px; border-width: 0 1px 1px 0; }

@keyframes cardIn {
  from { opacity: 0; transform: translateY(22px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* ── Header ── */
.card-header { text-align: center; margin-bottom: 36px; }

.eyebrow {
  display: block;
  font-size: .63rem; letter-spacing: .3em;
  text-transform: uppercase; color: #b49150; margin-bottom: 10px;
}
.card-header h1 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 2.6rem; font-weight: 600;
  color: #f0e6d0; letter-spacing: .08em;
  text-transform: uppercase; line-height: 1; margin-bottom: 10px;
}
.subtitle { color: #6a5c44; font-size: .77rem; font-weight: 300; letter-spacing: .04em; margin-bottom: 20px; }

.divider { display: flex; align-items: center; justify-content: center; gap: 10px; color: #b49150; }
.divider span:not(.diamond) { display: block; width: 60px; height: 1px; background: linear-gradient(90deg, transparent, #b49150); }
.divider span:last-child    { background: linear-gradient(90deg, #b49150, transparent); }
.diamond { font-size: .4rem; }

/* ── Campos ── */
.form { display: flex; flex-direction: column; gap: 22px; }

.field { display: flex; flex-direction: column; gap: 8px; }

.field label {
  font-size: .63rem; letter-spacing: .2em;
  text-transform: uppercase; color: #8a7455; font-weight: 500;
}
.req { color: #b49150; }
.hint { font-size: .63rem; color: #4a3f30; letter-spacing: .03em; }

.field input,
.field select,
.field textarea {
  background: #1c1c1c;
  border: 1px solid #2a2218;
  color: #e8dcc8;
  font-family: 'Montserrat', sans-serif;
  font-size: .85rem; font-weight: 300;
  padding: 13px 14px;
  outline: none;
  transition: border-color .25s, background .25s;
  resize: vertical;
}
.field input::placeholder,
.field textarea::placeholder { color: #3a3028; }
.field input:focus,
.field select:focus,
.field textarea:focus { border-color: #b49150; background: #181510; }

.field select {
  appearance: none;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8' viewBox='0 0 12 8'%3E%3Cpath d='M1 1l5 5 5-5' stroke='%23b49150' stroke-width='1.5' fill='none'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 14px center;
  background-color: #1c1c1c;
  padding-right: 38px;
  cursor: pointer;
}
.field select option { background: #1c1c1c; color: #e8dcc8; }

/* ── Botón principal ── */
.btn-primary {
  width: 100%; padding: 15px;
  background: transparent; border: 1px solid #b49150;
  color: #b49150;
  font-family: 'Montserrat', sans-serif;
  font-size: .7rem; font-weight: 600;
  letter-spacing: .3em; text-transform: uppercase;
  cursor: pointer;
  display: flex; align-items: center; justify-content: center;
  position: relative; overflow: hidden;
  transition: color .3s; margin-top: 4px;
}
.btn-primary::before {
  content: ''; position: absolute; inset: 0;
  background: #b49150;
  transform: scaleX(0); transform-origin: left;
  transition: transform .35s cubic-bezier(.4,0,.2,1);
}
.btn-primary:hover::before { transform: scaleX(1); }
.btn-primary:hover { color: #0d0d0d; }
.btn-primary span { position: relative; z-index: 1; }

/* ── Éxito ── */
.success-panel {
  display: flex; flex-direction: column;
  align-items: center; text-align: center;
  padding: 20px 0 10px; gap: 12px;
}
.success-icon { font-size: 2.5rem; color: #b49150; animation: pulse 2s ease-in-out infinite; }
@keyframes pulse { 0%,100% { opacity: 1; } 50% { opacity: .35; } }
.success-panel h2 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 2rem; font-weight: 600; color: #f0e6d0; letter-spacing: .06em;
}
.success-panel p { color: #7a6a50; font-size: .82rem; font-weight: 300; }

.btn-outline {
  padding: 11px 28px; background: transparent;
  border: 1px solid #2e2820; color: #8a7455;
  font-family: 'Montserrat', sans-serif;
  font-size: .68rem; font-weight: 500;
  letter-spacing: .2em; text-transform: uppercase;
  cursor: pointer; margin-top: 8px;
  transition: border-color .2s, color .2s;
}
.btn-outline:hover { border-color: #b49150; color: #b49150; }

/* ── Transición ── */
.fade-enter-active, .fade-leave-active { transition: opacity .35s, transform .35s; }
.fade-enter-from { opacity: 0; transform: translateY(-8px); }
.fade-leave-to   { opacity: 0; transform: translateY(8px); }

/* ── Responsive ── */
@media (max-width: 560px) {
  .card { padding: 36px 22px 32px; }
  .card-header h1 { font-size: 2rem; }
}
</style>