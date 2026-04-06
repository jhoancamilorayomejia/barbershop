<script setup>
import { ref, computed, watch } from 'vue'
import { jsPDF } from 'jspdf'
import { useRouter } from 'vue-router'

const router = useRouter()

const nombre   = ref('')
const whatsapp = ref('')
const fecha    = ref('')
const hora     = ref('')
const notas    = ref('')
const enviado  = ref(false)
const mostrarModal = ref(false)
const errorMsg = ref('')
const loading  = ref(false)
const mostrarLogin = ref(false)

const loginEmail    = ref('')
const loginPassword = ref('')
const loginError    = ref('')
const loginLoading  = ref(false)

const loginAdmin = async () => {
  loginLoading.value = true
  loginError.value = ''
  try {
    const res = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username: loginEmail.value, password: loginPassword.value })
    })
    const data = await res.json()
    if (!res.ok) throw new Error(data.error || 'Usuario o contraseña incorrectos')
    localStorage.setItem('token', data.token)
    localStorage.setItem('rol', data.rol)
    localStorage.setItem('username', data.user)
    mostrarLogin.value = false
    if (data.rol === 'admin' || data.rol === 'proveedor') {
      router.push('/api/proyectos')
    } else {
      router.push('/api/login')
    }
  } catch (err) {
    loginError.value = err.message
  } finally {
    loginLoading.value = false
  }
}

const formValido = computed(() =>
  nombre.value && whatsapp.value.length === 12 && fecha.value && hora.value
)

const formatearWhatsApp = (value) => {
  const numeros = value.replace(/\D/g, '').slice(0, 10)
  const partes = []
  if (numeros.length > 0) partes.push(numeros.slice(0, 3))
  if (numeros.length >= 4) partes.push(numeros.slice(3, 6))
  if (numeros.length >= 7) partes.push(numeros.slice(6, 10))
  return partes.join(' ')
}

watch(whatsapp, (newVal) => {
  const formateado = formatearWhatsApp(newVal)
  if (formateado !== newVal) whatsapp.value = formateado
})

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
  d.toLocaleDateString('es-CO', { weekday: 'long', day: 'numeric', month: 'long' })
    .replace(/^\w/, c => c.toUpperCase())

const DURACION_MINUTOS = 60

const horasDisponibles = computed(() => {
  const horas = []

  for (let h = 6; h <= 20; h++) {
    for (let m = 0; m < 60; m += DURACION_MINUTOS) {
      if (h === 20 && m > 0) continue
      horas.push(`${String(h).padStart(2,'0')}:${String(m).padStart(2,'0')}`)
    }
  }

  return horas
})

watch(fecha, () => { hora.value = '' })

const generarPDF = () => {
  const doc = new jsPDF()
  const fechaFormateada = new Date(`${fecha.value}T${hora.value}`).toLocaleString('es-CO')
  doc.setFont('Helvetica', 'bold')
  doc.setFontSize(16)
  doc.text('Confirmación de Reserva', 20, 20)
  doc.setFontSize(12)
  doc.setFont('Helvetica', 'normal')
  doc.text(`Nombre: ${nombre.value}`, 20, 40)
  doc.text(`Teléfono: ${whatsapp.value}`, 20, 50)
  doc.text(`Fecha y hora: ${fechaFormateada}`, 20, 60)
  if (notas.value) doc.text(`Notas: ${notas.value}`, 20, 70)
  doc.text('Gracias por tu reserva 🙌', 20, 90)
  doc.save(`reserva-${nombre.value}.pdf`)
}

const enviar = () => {
  if (!formValido.value) { errorMsg.value = 'Completa todos los campos'; return }
  errorMsg.value = ''
  mostrarModal.value = true
}

const confirmarReserva = async () => {
  loading.value = true
  errorMsg.value = ''
  try {
    const payload = {
      name: nombre.value,
      phone: whatsapp.value,
      reservation_date: `${fecha.value} ${hora.value}`,
      note: notas.value
    }
    const token = localStorage.getItem('token')
    const res = await fetch('/api/reservations', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
      body: JSON.stringify(payload)
    })
    if (res.status === 401) { localStorage.removeItem('token'); router.push('/api/login'); return }
    const text = await res.text()
    let data
    try { data = JSON.parse(text) } catch { throw new Error('Respuesta inválida: ' + text) }
    if (!res.ok) throw new Error(data.error || 'Error desconocido')
    generarPDF()
    mostrarModal.value = false
    enviado.value = true
    setTimeout(() => router.push('/dashboard'), 2000)
  } catch (error) {
    errorMsg.value = error.message
  } finally {
    loading.value = false
  }
}

const nuevaReserva = () => {
  nombre.value = ''
  whatsapp.value = ''
  fecha.value = ''
  hora.value = ''
  notas.value = ''
  enviado.value = false
  errorMsg.value = ''
}

const scrollToForm = () => {
  document.getElementById('reservas')?.scrollIntoView({ behavior: 'smooth' })
}
</script>

<template>
  <div class="page">
    <div class="bg-grid"></div>
    <div class="glow left"></div>
    <div class="glow right"></div>

    <!-- ── HERO ── -->
    <section class="hero">
      <div class="hero-inner">
        <span class="eyebrow">Barbería &amp; Estilo</span>
        <h1 class="hero-title">
          Barber Charly<br>
          <em>Tu estilo empieza donde termina lo común</em>
        </h1>
        <p class="hero-sub">Agendá tu turno en segundos.</p>
        <div class="hero-actions">
          <button class="btn-hero" @click="scrollToForm">✂ Agendar turno</button>
          <button class="btn-ghost" @click="mostrarLogin = true">Administrador</button>
        </div>
      </div>
    </section>

    <!-- ── FORMULARIO ── -->
    <section class="section-form" id="reservas">
      <div class="section-header">
        <span class="eyebrow">Reservá online</span>
        <h2 class="section-title">Agendar Turno</h2>
        <div class="divider"><span></span><span class="diamond">◆</span><span></span></div>
      </div>

      <transition name="fade" mode="out-in">
        <div v-if="enviado" class="success-panel" key="ok">
          <div class="success-icon">✦</div>
          <h3>¡Turno solicitado!</h3>
          <p>Te contactaremos por WhatsApp para confirmar.</p>
          <button class="btn-app btn-app-outline" @click="nuevaReserva">Hacer otra reserva</button>
        </div>

        <form v-else key="form" class="form" @submit.prevent="enviar">

          <div class="field">
            <label>Nombre completo <span class="req">*</span></label>
            <input v-model="nombre" type="text" required placeholder="Ej: Jhoan Camilo..." autocomplete="name" />
          </div>

          <div class="field">
            <label>WhatsApp <span class="req">*</span></label>
            <input v-model="whatsapp" type="tel" required placeholder="323 517 9341" maxlength="12" autocomplete="tel" />
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
              >{{ formatearFecha(d) }}</option>
            </select>
          </div>

          <div v-if="fecha" class="field">
            <label>Horario <span class="req">*</span></label>
            <div class="horas-grid">
              <button
                v-for="h in horasDisponibles"
                :key="h"
                type="button"
                class="hora-btn"
                :class="{ active: hora === h }"
                @click="hora = h"
              >{{ h }}</button>
            </div>
          </div>

          <div class="field">
            <label>Notas adicionales</label>
            <textarea v-model="notas" rows="3" placeholder="Alguna preferencia o consulta especial..."></textarea>
          </div>

          <p v-if="errorMsg" class="error-msg">{{ errorMsg }}</p>

          <button type="submit" class="btn-app btn-app-gold">
            Confirmar Turno
          </button>
        </form>
      </transition>
    </section>

    <!-- ── FOOTER ── -->
    <footer class="footer">
      <p class="footer-text">© 2026 Barber Charly · Todos los derechos reservados.</p>
    </footer>

    <!-- ── MODAL CONFIRMACIÓN ── -->
    <div v-if="mostrarModal" class="fullscreen-modal">
      <div class="fs-inner">
        <div class="fs-icon">✦</div>
        <h2 class="fs-title">Confirmar reserva</h2>
        <p class="fs-sub">¿Confirmás tu turno?</p>
        <div class="fs-actions">
          <button class="btn-app btn-app-gold" @click="confirmarReserva" :disabled="loading">
            {{ loading ? 'Guardando...' : 'Sí, confirmar' }}
          </button>
          <button class="btn-app btn-app-outline" @click="mostrarModal = false">Cancelar</button>
        </div>
        <p v-if="errorMsg" class="error-msg">{{ errorMsg }}</p>
      </div>
    </div>

    <!-- ── MODAL LOGIN PANTALLA COMPLETA ── -->
    <div v-if="mostrarLogin" class="fullscreen-modal">
      <div class="fs-inner">
        <button class="fs-close" @click="mostrarLogin = false">✕</button>
        <div class="fs-icon">🔐</div>
        <h2 class="fs-title">Administrador</h2>
        <p class="fs-sub">Ingresá tus credenciales</p>

        <div class="fs-form">
          <div class="field">
            <label>Usuario</label>
            <input
              type="text"
              v-model="loginEmail"
              placeholder="Tu usuario"
              autocomplete="username"
            />
          </div>
          <div class="field">
            <label>Contraseña</label>
            <input
              type="password"
              v-model="loginPassword"
              placeholder="••••••••"
              autocomplete="current-password"
            />
          </div>
          <p v-if="loginError" class="error-msg">{{ loginError }}</p>
          <button class="btn-app btn-app-gold" @click="loginAdmin" :disabled="loginLoading">
            {{ loginLoading ? 'Ingresando...' : 'Ingresar' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,400;0,600;1,400;1,600&family=Montserrat:wght@300;400;500;600&display=swap');

/* ── RESET Y META ── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

/* Previene zoom en iOS al hacer focus en inputs */
input, select, textarea {
  font-size: 16px !important;
}

/* ── BASE ── */
.page {
  min-height: 50vh;
  background: #0d0d0d;
  font-family: 'Montserrat', sans-serif;
  position: relative;
  overflow-x: hidden;
  color: #f0e6d0;
}

.bg-grid {
  position: fixed; inset: 0;
  background-image:
    linear-gradient(rgba(180,145,80,.025) 1px, transparent 1px),
    linear-gradient(90deg, rgba(180,145,80,.025) 1px, transparent 1px);
  background-size: 60px 60px;
  pointer-events: none; z-index: 0;
}

.glow {
  position: fixed; border-radius: 50%;
  filter: blur(140px); pointer-events: none; z-index: 0;
  width: 500px; height: 500px;
}
.glow.left  { background: radial-gradient(circle, rgba(180,145,80,.12), transparent 70%); top: -100px; left: -150px; }
.glow.right { background: radial-gradient(circle, rgba(180,145,80,.08), transparent 70%); bottom: -100px; right: -150px; }

/* ── HERO ── */
.hero {
  position: relative; z-index: 1;
  min-height: 55vh;
  display: flex; align-items: center; justify-content: center;
  text-align: center;
  padding: 30px 24px 20px;
  border-bottom: 1px solid rgba(180,145,80,.1);
}

.hero-inner { max-width: 700px; width: 100%; }

.eyebrow {
  display: inline-block;
  font-size: .78rem; letter-spacing: .4em;
  text-transform: uppercase; color: #b49150;
  margin-bottom: 20px;
}

.hero-title {
  font-family: 'Cormorant Garamond', serif;
  font-size: clamp(2.8rem, 10vw, 6.5rem);
  font-weight: 600; line-height: 1.1;
  color: #f0e6d0; margin-bottom: 16px;
}
.hero-title em { font-style: italic; color: #b49150; font-size: .82em; display: block; margin-top: 6px; }

.hero-sub {
  font-size: 1rem; font-weight: 300;
  color: #6a5c44; letter-spacing: .04em;
  margin-bottom: 36px;
}

.hero-actions {
  display: flex; flex-direction: column;
  align-items: center; gap: 14px;
}

/* ── BOTONES TIPO APP ── */
.btn-app {
  width: 100%;
  max-width: 420px;
  padding: 22px 24px;
  font-family: 'Montserrat', sans-serif;
  font-size: 1.05rem;
  font-weight: 600;
  letter-spacing: .15em;
  text-transform: uppercase;
  cursor: pointer;
  border: none;
  transition: all .25s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.btn-app-gold {
  background: #b49150;
  color: #0d0d0d;
}
.btn-app-gold:hover { background: #c9a862; }
.btn-app-gold:active { background: #9e7e42; transform: scale(.98); }

.btn-app-outline {
  background: transparent;
  color: #b49150;
  border: 1px solid rgba(180,145,80,.4) !important;
}
.btn-app-outline:hover { border-color: #b49150 !important; background: rgba(180,145,80,.07); }

/* Botones hero tienen borde para el ghost */
.btn-hero {
  width: 100%; max-width: 420px;
  padding: 22px 24px;
  background: #b49150; color: #0d0d0d;
  border: none;
  font-family: 'Montserrat', sans-serif;
  font-size: 1.05rem; font-weight: 600;
  letter-spacing: .15em; text-transform: uppercase;
  cursor: pointer; transition: all .25s;
}
.btn-hero:active { transform: scale(.98); }

.btn-ghost {
  width: 100%; max-width: 420px;
  padding: 22px 24px;
  background: transparent; color: #8a7455;
  border: 1px solid rgba(180,145,80,.25);
  font-family: 'Montserrat', sans-serif;
  font-size: 1rem; font-weight: 400;
  letter-spacing: .12em; text-transform: uppercase;
  cursor: pointer; transition: all .25s;
}
.btn-ghost:hover { color: #b49150; border-color: rgba(180,145,80,.5); }

/* ── SECCIÓN FORM ── */
.section-form {
  position: relative; z-index: 1;
  max-width: 640px; margin: 0 auto;
  padding: 52px 24px 60px;
}

.section-header { text-align: center; margin-bottom: 40px; }

.section-title {
  font-family: 'Cormorant Garamond', serif;
  font-size: clamp(2.4rem, 8vw, 4rem);
  font-weight: 600; color: #f0e6d0;
  letter-spacing: .06em; margin: 10px 0 20px;
}

.divider {
  display: flex; align-items: center;
  justify-content: center; gap: 12px;
}
.divider span:not(.diamond) {
  width: 70px; height: 1px;
  background: linear-gradient(90deg, transparent, #b49150);
}
.divider span:last-child { background: linear-gradient(90deg, #b49150, transparent); }
.diamond { font-size: .4rem; color: #b49150; }

/* ── FORM ── */
.form { display: flex; flex-direction: column; gap: 22px; }

.field { display: flex; flex-direction: column; gap: 10px; }

.field label {
  font-size: .82rem; letter-spacing: .18em;
  text-transform: uppercase; color: #8a7455; font-weight: 500;
}
.req { color: #b49150; }
.hint { font-size: .8rem; color: #4a3f30; }

/* inputs grandes tipo app — font-size 16px forzado arriba para evitar zoom iOS */
.field input,
.field select,
.field textarea {
  background: #1a1a1a;
  border: 1px solid rgba(180,145,80,.2);
  border-radius: 0;
  color: #e8dcc8;
  font-family: 'Montserrat', sans-serif;
  font-weight: 300;
  padding: 20px 18px;
  outline: none;
  transition: border-color .2s, background .2s;
  resize: none; width: 100%;
  -webkit-appearance: none;
}
.field input::placeholder,
.field textarea::placeholder { color: #3a3028; }
.field input:focus,
.field select:focus,
.field textarea:focus {
  border-color: #b49150;
  background: #181510;
}

.field select {
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='14' height='9' viewBox='0 0 14 9'%3E%3Cpath d='M1 1l6 6 6-6' stroke='%23b49150' stroke-width='1.8' fill='none'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 18px center;
  background-color: #1a1a1a;
  padding-right: 46px; cursor: pointer;
}
.field select option { background: #141414; color: #e8dcc8; }

/* ── HORAS ── */
.horas-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 10px;
}

.hora-btn {
  background: #1a1a1a;
  border: 1px solid rgba(180,145,80,.18);
  color: #6a5c44;
  padding: 18px 4px;
  font-size: .9rem;
  cursor: pointer; transition: all .15s;
  font-family: 'Montserrat', sans-serif;
  -webkit-tap-highlight-color: transparent;
}
.hora-btn:active { transform: scale(.95); }
.hora-btn.active {
  background: #b49150; color: #0d0d0d;
  border-color: #b49150; font-weight: 600;
}

.error-msg {
  color: #e05555; font-size: .88rem;
  text-align: center; padding: 4px 0;
}

/* ── ÉXITO ── */
.success-panel {
  display: flex; flex-direction: column;
  align-items: center; text-align: center;
  padding: 40px 0; gap: 20px;
}
.success-icon { font-size: 3rem; color: #b49150; animation: pulse 2s ease-in-out infinite; }
@keyframes pulse { 0%,100%{opacity:1} 50%{opacity:.3} }
.success-panel h3 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 2.6rem; font-weight: 600; color: #f0e6d0;
}
.success-panel p { color: #7a6a50; font-size: 1rem; font-weight: 300; }

/* ── FOOTER ── */
.footer {
  text-align: center; padding: 32px 24px;
  border-top: 1px solid rgba(180,145,80,.08);
  position: relative; z-index: 1;
}
.footer-text { font-size: .7rem; letter-spacing: .15em; text-transform: uppercase; color: #2e2820; }

/* ── MODALES PANTALLA COMPLETA ── */
.fullscreen-modal {
  position: fixed; inset: 0;
  background: #0d0d0d;
  z-index: 2000;
  display: flex; align-items: center; justify-content: center;
  padding: 32px 24px;
  overflow-y: auto;
}

.fs-inner {
  width: 100%; max-width: 480px;
  display: flex; flex-direction: column;
  align-items: center; text-align: center;
  gap: 0;
  position: relative;
}

.fs-close {
  position: absolute; top: -8px; right: 0;
  background: transparent; border: none;
  color: #6a5c44; font-size: 1.4rem;
  cursor: pointer; padding: 8px;
  line-height: 1;
  transition: color .2s;
}
.fs-close:hover { color: #b49150; }

.fs-icon {
  font-size: 2.8rem; color: #b49150;
  margin-bottom: 20px;
}

.fs-title {
  font-family: 'Cormorant Garamond', serif;
  font-size: clamp(2.4rem, 8vw, 3.6rem);
  font-weight: 600; color: #f0e6d0;
  letter-spacing: .04em;
  margin-bottom: 10px;
}

.fs-sub {
  font-size: .9rem; font-weight: 300;
  color: #6a5c44; letter-spacing: .04em;
  margin-bottom: 36px;
}

.fs-actions {
  width: 100%;
  display: flex; flex-direction: column;
  gap: 14px; align-items: center;
}

.fs-form {
  width: 100%;
  display: flex; flex-direction: column;
  gap: 20px;
}

.fs-form .field { text-align: left; }

/* ── TRANSICIÓN ── */
.fade-enter-active, .fade-leave-active { transition: opacity .3s, transform .3s; }
.fade-enter-from { opacity: 0; transform: translateY(-8px); }
.fade-leave-to   { opacity: 0; transform: translateY(8px); }

/* ── DESKTOP ── */
@media (min-width: 601px) {
  .hero-actions { flex-direction: row; justify-content: center; }
  .btn-hero, .btn-ghost { width: auto; min-width: 240px; }
  .horas-grid { grid-template-columns: repeat(5, 1fr); }
}
</style>