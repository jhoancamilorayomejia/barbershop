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

const horasDisponibles = computed(() => {
  const horas = []
  for (let h = 6; h <= 20; h++) {
    for (let m of [0, 30]) {
      if (h === 20 && m === 30) continue
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
    setTimeout(() => router.push('/api/proyectos'), 2000)
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
        <p class="hero-sub">Tu estilo, nuestra pasión. Agendá tu turno en segundos.</p>
        <div class="hero-actions">
          <button class="btn-hero" @click="scrollToForm">Agendar mi turno</button>
          <button class="btn-ghost" @click="mostrarLogin = true">Administrador</button>
        </div>
      </div>
      <div class="hero-deco">
        <div class="deco-line"></div>
        <span class="deco-sym">✦</span>
        <div class="deco-line"></div>
      </div>
    </section>

    <!-- ── FORMULARIO ── -->
    <section class="section-form" id="reservas">
      <div class="section-header">
        <span class="eyebrow">Reservá online</span>
        <h2 class="section-title">Agendar Turno</h2>
        <p class="section-sub">Completá el formulario y confirmamos tu turno al instante.</p>
        <div class="divider"><span></span><span class="diamond">◆</span><span></span></div>
      </div>

      <transition name="fade" mode="out-in">
        <div v-if="enviado" class="success-panel" key="ok">
          <div class="success-icon">✦</div>
          <h3>¡Turno solicitado!</h3>
          <p>Te contactaremos por WhatsApp para confirmar tu reserva.</p>
          <button class="btn-outline" @click="nuevaReserva">Hacer otra reserva</button>
        </div>

        <form v-else key="form" class="form" @submit.prevent="enviar">
          <div class="form-row">
            <div class="field">
              <label>Nombre completo <span class="req">*</span></label>
              <input v-model="nombre" type="text" required placeholder="Ej: Jhoan Camilo..." />
            </div>
            <div class="field">
              <label>WhatsApp <span class="req">*</span></label>
              <input v-model="whatsapp" type="tel" required placeholder="323 517 9341" maxlength="12" />
              <span class="hint">Para confirmar o consultar tu turno</span>
            </div>
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

          <button type="submit" class="btn-primary">
            <span>Confirmar Turno</span>
          </button>
        </form>
      </transition>
    </section>

    <!-- ── FOOTER ── -->
    <footer class="footer">
      <div class="divider" style="margin-bottom:20px"><span></span><span class="diamond">◆</span><span></span></div>
      <p class="footer-text">© 2026 Barbershop · Todos los derechos reservados.</p>
    </footer>

    <!-- MODAL CONFIRMACIÓN -->
    <div v-if="mostrarModal" class="modal-overlay">
      <div class="modal">
        <h3>Confirmar reserva</h3>
        <p>¿Deseas confirmar tu turno?</p>
        <div class="modal-actions">
          <button class="btn-outline" @click="mostrarModal = false">Cancelar</button>
          <button class="btn-primary" @click="confirmarReserva" :disabled="loading">
            <span>{{ loading ? 'Guardando...' : 'Confirmar' }}</span>
          </button>
        </div>
        <p v-if="errorMsg" class="error-msg">{{ errorMsg }}</p>
      </div>
    </div>

    <!-- MODAL LOGIN -->
    <div v-if="mostrarLogin" class="modal-overlay">
      <div class="modal">
        <h3>Iniciar sesión</h3>
        <div class="field">
          <label>Usuario</label>
          <input type="text" v-model="loginEmail" placeholder="Tu usuario" />
        </div>
        <div class="field">
          <label>Contraseña</label>
          <input type="password" v-model="loginPassword" placeholder="********" />
        </div>
        <div class="modal-actions">
          <button class="btn-outline" @click="mostrarLogin = false">Cancelar</button>
          <button class="btn-primary" @click="loginAdmin" :disabled="loginLoading">
            <span>{{ loginLoading ? 'Ingresando...' : 'Ingresar' }}</span>
          </button>
        </div>
        <p v-if="loginError" class="error-msg">{{ loginError }}</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,400;0,600;0,700;1,400;1,600&family=Montserrat:wght@300;400;500;600&display=swap');

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

.page {
  min-height: 100vh;
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
  position: fixed;
  width: 600px; height: 600px;
  border-radius: 50%;
  filter: blur(140px);
  pointer-events: none; z-index: 0;
}
.glow.left  { background: radial-gradient(circle, rgba(180,145,80,.12), transparent 70%); top: -150px; left: -200px; }
.glow.right { background: radial-gradient(circle, rgba(180,145,80,.08), transparent 70%); bottom: -150px; right: -200px; }

/* HERO */
.hero {
  position: relative; z-index: 1;
  min-height: 100vh;
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  text-align: center;
  padding: 80px 24px 80px;
  border-bottom: 1px solid rgba(180,145,80,.1);
}

.hero-inner { max-width: 720px; }

.eyebrow {
  display: inline-block;
  font-size: .62rem; letter-spacing: .4em;
  text-transform: uppercase; color: #b49150;
  margin-bottom: 28px;
}

.hero-title {
  font-family: 'Cormorant Garamond', serif;
  font-size: clamp(3rem, 9vw, 7rem);
  font-weight: 600; line-height: 1.05;
  letter-spacing: .03em; color: #f0e6d0;
  margin-bottom: 28px;
}
.hero-title em { font-style: italic; color: #b49150; }

.hero-sub {
  font-size: .88rem; font-weight: 300;
  color: #6a5c44; letter-spacing: .06em;
  margin-bottom: 52px;
}

.hero-actions { display: flex; gap: 16px; justify-content: center; flex-wrap: wrap; }

.btn-hero {
  padding: 16px 48px;
  background: #b49150; color: #0d0d0d;
  border: none;
  font-family: 'Montserrat', sans-serif;
  font-size: .72rem; font-weight: 600;
  letter-spacing: .3em; text-transform: uppercase;
  cursor: pointer; transition: all .3s;
}
.btn-hero:hover { background: #c9a862; transform: translateY(-2px); }

.btn-ghost {
  padding: 16px 48px;
  background: transparent; color: #b49150;
  border: 1px solid rgba(180,145,80,.35);
  font-family: 'Montserrat', sans-serif;
  font-size: .72rem; font-weight: 500;
  letter-spacing: .3em; text-transform: uppercase;
  cursor: pointer; transition: all .3s;
}
.btn-ghost:hover { border-color: #b49150; background: rgba(180,145,80,.06); }

.hero-deco {
  position: absolute; bottom: 40px;
  display: flex; align-items: center; gap: 16px;
}
.deco-line { width: 80px; height: 1px; background: rgba(180,145,80,.25); }
.deco-sym { font-size: .5rem; color: #b49150; }

/* SECCIÓN FORM */
.section-form {
  position: relative; z-index: 1;
  max-width: 800px; margin: 0 auto;
  padding: 100px 28px 80px;
}

.section-header { text-align: center; margin-bottom: 64px; }

.section-title {
  font-family: 'Cormorant Garamond', serif;
  font-size: clamp(2.2rem, 6vw, 3.8rem);
  font-weight: 600; color: #f0e6d0;
  letter-spacing: .06em; margin: 12px 0 16px;
}

.section-sub {
  font-size: .8rem; font-weight: 300;
  color: #6a5c44; letter-spacing: .04em; margin-bottom: 28px;
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

/* FORM */
.form { display: flex; flex-direction: column; gap: 28px; }

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 24px;
}

.field { display: flex; flex-direction: column; gap: 10px; }

.field label {
  font-size: .62rem; letter-spacing: .22em;
  text-transform: uppercase; color: #8a7455; font-weight: 500;
}
.req { color: #b49150; }
.hint { font-size: .62rem; color: #4a3f30; letter-spacing: .03em; }

.field input,
.field select,
.field textarea {
  background: #141414;
  border: 1px solid rgba(180,145,80,.15);
  color: #e8dcc8;
  font-family: 'Montserrat', sans-serif;
  font-size: .88rem; font-weight: 300;
  padding: 16px 18px; outline: none;
  transition: border-color .25s, background .25s;
  resize: vertical; width: 100%;
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
  background-position: right 18px center;
  background-color: #141414;
  padding-right: 44px; cursor: pointer;
}
.field select option { background: #141414; color: #e8dcc8; }

.horas-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(82px, 1fr));
  gap: 10px;
}

.hora-btn {
  background: #141414;
  border: 1px solid rgba(180,145,80,.15);
  color: #8a7455; padding: 14px 8px;
  font-size: .78rem; cursor: pointer;
  transition: all .2s;
  font-family: 'Montserrat', sans-serif; font-weight: 400;
}
.hora-btn:hover { border-color: rgba(180,145,80,.5); color: #f0e6d0; }
.hora-btn.active { background: #b49150; color: #0d0d0d; border-color: #b49150; font-weight: 600; }

.btn-primary {
  width: 100%; padding: 18px;
  background: transparent; border: 1px solid #b49150;
  color: #b49150;
  font-family: 'Montserrat', sans-serif;
  font-size: .72rem; font-weight: 600;
  letter-spacing: .35em; text-transform: uppercase;
  cursor: pointer; display: flex;
  align-items: center; justify-content: center;
  position: relative; overflow: hidden;
  transition: color .3s; margin-top: 8px;
}
.btn-primary::before {
  content: ''; position: absolute; inset: 0;
  background: #b49150; transform: scaleX(0);
  transform-origin: left;
  transition: transform .35s cubic-bezier(.4,0,.2,1);
}
.btn-primary:hover::before { transform: scaleX(1); }
.btn-primary:hover { color: #0d0d0d; }
.btn-primary span { position: relative; z-index: 1; }

.btn-outline {
  padding: 14px 32px; background: transparent;
  border: 1px solid rgba(180,145,80,.25); color: #8a7455;
  font-family: 'Montserrat', sans-serif;
  font-size: .68rem; font-weight: 500;
  letter-spacing: .2em; text-transform: uppercase;
  cursor: pointer; transition: border-color .2s, color .2s;
}
.btn-outline:hover { border-color: #b49150; color: #b49150; }

.success-panel {
  display: flex; flex-direction: column;
  align-items: center; text-align: center;
  padding: 60px 0; gap: 16px;
}
.success-icon { font-size: 3rem; color: #b49150; animation: pulse 2s ease-in-out infinite; }
@keyframes pulse { 0%,100%{opacity:1} 50%{opacity:.3} }
.success-panel h3 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 2.4rem; font-weight: 600;
  color: #f0e6d0; letter-spacing: .06em;
}
.success-panel p { color: #7a6a50; font-size: .85rem; font-weight: 300; }

.footer {
  position: relative; z-index: 1;
  text-align: center; padding: 40px 24px;
  border-top: 1px solid rgba(180,145,80,.08);
}
.footer-text {
  font-size: .62rem; letter-spacing: .2em;
  text-transform: uppercase; color: #3a3028;
}

.modal-overlay {
  position: fixed; inset: 0;
  background: rgba(0,0,0,.82);
  display: flex; align-items: center; justify-content: center;
  z-index: 1000; padding: 24px;
}

.modal {
  background: #141414;
  border: 1px solid rgba(180,145,80,.2);
  padding: 36px; width: 100%; max-width: 440px;
  text-align: center;
  box-shadow: 0 40px 100px rgba(0,0,0,.9);
}

.modal h3 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 1.8rem; color: #f0e6d0;
  letter-spacing: .06em; margin-bottom: 12px;
}
.modal p { color: #8a7455; margin-bottom: 24px; font-size: .82rem; }
.modal .field { text-align: left; margin-bottom: 16px; }

.modal-actions { display: flex; gap: 12px; justify-content: center; flex-wrap: wrap; }

.error-msg { color: #c0392b; font-size: .74rem; margin-top: 12px; margin-bottom: 0 !important; }

.fade-enter-active, .fade-leave-active { transition: opacity .35s, transform .35s; }
.fade-enter-from { opacity: 0; transform: translateY(-10px); }
.fade-leave-to   { opacity: 0; transform: translateY(10px); }

@media (max-width: 600px) {
  .form-row { grid-template-columns: 1fr; }
  .horas-grid { grid-template-columns: repeat(4, 1fr); }
  .section-form { padding: 60px 20px 60px; }
}
</style>