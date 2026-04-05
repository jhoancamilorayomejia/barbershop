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

// Login
const loginEmail    = ref('')
const loginPassword = ref('')
const loginError    = ref('')
const loginLoading  = ref(false)

const loginAdmin = async () => {
  loginLoading.value = true
  loginError.value = ''

  try {
    const payload = {
      username: loginEmail.value,
      password: loginPassword.value
    }

    // ✅ Ruta relativa — funciona en desarrollo y producción
    const res = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    })

    const data = await res.json()

    if (!res.ok) {
      throw new Error(data.error || 'Usuario o contraseña incorrectos')
    }

    // ✅ Guardar sesión
    localStorage.setItem('token', data.token)
    localStorage.setItem('rol', data.rol)
    localStorage.setItem('username', data.user)

    mostrarLogin.value = false

    // ✅ Redirección por rol
    if (data.rol === 'admin') {
      router.push('/api/proyectos')
    } else if (data.rol === 'proveedor') {
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

// ✅ Validación
const formValido = computed(() => {
  return nombre.value && whatsapp.value.length === 12 && fecha.value && hora.value
})

// ✅ Formatear WhatsApp y limitar a 10 dígitos
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
  if (formateado !== newVal) {
    whatsapp.value = formateado
  }
})

// ✅ Fechas disponibles (próximos 30 días excepto domingos)
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
  d.toLocaleDateString('es-AR', {
    weekday: 'long',
    day: 'numeric',
    month: 'long'
  }).replace(/^\w/, c => c.toUpperCase())

// ✅ Horas disponibles
const horasDisponibles = computed(() => {
  const horas = []
  for (let h = 6; h <= 20; h++) {
    for (let m of [0, 30]) {
      if (h === 20 && m === 30) continue
      const hh = String(h).padStart(2, '0')
      const mm = String(m).padStart(2, '0')
      horas.push(`${hh}:${mm}`)
    }
  }
  return horas
})

// Reset hora si cambia fecha
watch(fecha, () => {
  hora.value = ''
})

// ✅ PDF
const generarPDF = () => {
  const doc = new jsPDF()

  const fechaFormateada = new Date(`${fecha.value}T${hora.value}`)
    .toLocaleString('es-AR')

  doc.setFont("Helvetica", "bold")
  doc.setFontSize(16)
  doc.text("Confirmación de Reserva", 20, 20)

  doc.setFontSize(12)
  doc.setFont("Helvetica", "normal")

  doc.text(`Nombre: ${nombre.value}`, 20, 40)
  doc.text(`Teléfono: ${whatsapp.value}`, 20, 50)
  doc.text(`Fecha y hora: ${fechaFormateada}`, 20, 60)

  if (notas.value) {
    doc.text(`Notas: ${notas.value}`, 20, 70)
  }

  doc.text("Gracias por tu reserva 🙌", 20, 90)

  doc.save(`reserva-${nombre.value}.pdf`)
}

// ✅ Abrir modal
const enviar = () => {
  if (!formValido.value) {
    errorMsg.value = 'Completa todos los campos'
    return
  }
  errorMsg.value = ''
  mostrarModal.value = true
}

// ✅ Confirmar reserva
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
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`
      },
      body: JSON.stringify(payload)
    })

    if (res.status === 401) {
      localStorage.removeItem('token')
      router.push('/api/login')
      return
    }

    const text = await res.text()

    let data
    try {
      data = JSON.parse(text)
    } catch {
      throw new Error("Respuesta inválida del servidor: " + text)
    }

    if (!res.ok) {
      throw new Error(data.error || 'Error desconocido')
    }

    generarPDF()

    mostrarModal.value = false
    enviado.value = true

    setTimeout(() => {
      router.push('/api/proyectos')
    }, 2000)

  } catch (error) {
    errorMsg.value = error.message
  } finally {
    loading.value = false
  }
}

// ✅ Reset
const nuevaReserva = () => {
  nombre.value = ''
  whatsapp.value = ''
  fecha.value = ''
  hora.value = ''
  notas.value = ''
  enviado.value = false
  errorMsg.value = ''
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
            <label>Escriba tu nombre completo<span class="req">*</span></label>
            <input v-model="nombre" type="text" required placeholder="Ej: Jhoan Camilo..." />
          </div>

          <div class="field">
            <label>WhatsApp <span class="req">*</span></label>
            <input
              v-model="whatsapp"
              type="tel"
              required
              placeholder="323 517 9341"
              maxlength="12"
            />
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

          <!-- Solo aparece si hay fecha -->
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
              >
                {{ h }}
              </button>
            </div>
          </div>

          <div class="field">
            <label>Notas adicionales</label>
            <textarea v-model="notas" rows="3" placeholder="Alguna preferencia o consulta especial..."></textarea>
          </div>

          <button type="submit" class="btn-primary">
            <span>Confirmar Turno</span>
          </button>

          <!-- Enlace para iniciar sesión -->
          <div class="login-link">
            <a href="#" @click.prevent="mostrarLogin = true">¿Eres Administrador? Iniciar sesión</a>
          </div>

        </form>
      </transition>
    </div>

    <!-- Modal confirmación reserva -->
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
        <p v-if="errorMsg" style="color:#c0392b; font-size:.75rem; text-align:center;">
          {{ errorMsg }}
        </p>
      </div>
    </div>

    <!-- Modal de Login -->
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
        <p v-if="loginError" style="color:#c0392b; font-size:.75rem; text-align:center;">
          {{ loginError }}
        </p>
      </div>
    </div>

  </div>
</template>

<style scoped>
@media (max-width: 560px) {
  .screen {
    padding: 30px 12px;
  }

  .card {
    padding: 28px 18px;
    max-width: 100%;
  }

  .card-header h1 {
    font-size: 1.8rem;
  }

  .subtitle {
    font-size: .8rem;
  }

  .field input,
  .field select,
  .field textarea {
    font-size: 1rem; /* 🔥 más grande para móvil */
    padding: 14px;
  }

  .btn-primary {
    font-size: .8rem;
    padding: 16px;
  }

  .hora-btn {
    padding: 14px;
    font-size: .9rem;
  }

  .horas-grid {
    grid-template-columns: repeat(2, 1fr); /* 🔥 mejor para celular */
  }
}
</style>