<template>
  <div class="screen">
    <div class="bg-grid"></div>
    <div class="glow left"></div>
    <div class="glow right"></div>

    <div class="card center-card">
      <div class="center-content">
        <span class="eyebrow">Iniciaste sesión</span>

        <h1>Ver Listado de Reservas</h1>

        <div class="divider">
          <span></span>
          <span class="diamond">◆</span>
          <span></span>
        </div>

        <!-- 📅 NAVEGACIÓN DE MES -->
        <div class="month-nav">
          <button class="month-arrow" @click="cambiarMes(-1)">&#8249;</button>
          <span class="month-label">{{ mesActual }}</span>
          <button class="month-arrow" @click="cambiarMes(1)">&#8250;</button>
        </div>

        <!-- 🗓️ CALENDARIO -->
        <div class="calendar">
          <div class="cal-header">
            <span v-for="dia in diasSemana" :key="dia">{{ dia }}</span>
          </div>
          <div class="cal-grid">
            <!-- Espacios vacíos antes del primer día -->
            <div
              v-for="n in primerDiaSemana"
              :key="'empty-' + n"
              class="day-cell empty"
            ></div>

            <!-- Días del mes -->
            <button
              v-for="dia in diasEnMes"
              :key="dia"
              class="day-btn"
              :class="{
                'has-data': tieneReservas(dia),
                'selected': esSeleccionado(dia),
                'today': esHoy(dia)
              }"
              @click="seleccionarDia(dia)"
            >
              {{ dia }}
              <span v-if="tieneReservas(dia)" class="dot"></span>
            </button>
          </div>
        </div>

        <!-- ✅ TABLA (visible solo al seleccionar fecha) -->
        <transition name="fade-slide">
          <div v-if="fechaSeleccionada" class="table-section">
            <p class="date-title">
              Reservas del {{ fechaFormateada }}
            </p>

            <div class="table-container" v-if="reservasFiltradas.length">
              <table>
                <thead>
                  <tr>
                    <th>Nombre</th>
                    <th>WhatsApp</th>
                    <th>Hora</th>
                    <th>Nota</th>
                    <th>Acciones</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="r in reservasFiltradas" :key="r.id">
                    <td>{{ r.name }}</td>
                    <td>{{ r.phone }}</td>
                    <td>{{ r.reservation_date.length > 10 ? r.reservation_date.slice(11) : '—' }}</td>
                    <td>{{ r.note || '—' }}</td>
                    <!-- 🔴 BOTÓN ELIMINAR -->
                    <td>
                    <button class="btn-delete" @click="abrirEliminar(r)">
                    Eliminar
                    </button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <p v-else class="empty">No hay reservas para esta fecha.</p>
          </div>
        </transition>

        <!-- 🔘 BOTONES -->
        <div class="btn-row">
          <button class="btn-primary" @click="volver">
            <span>Volver</span>
          </button>
          <button class="btn-primary" @click="cerrarSesion">
            <span>Cerrar sesión</span>
          </button>
        </div>
        <!-- 🔴 MODAL ELIMINAR -->
<div v-if="mostrarEliminar" class="modal-overlay">
  <div class="modal">
    <h3>Eliminar reserva</h3>
    <p>
      ¿Seguro que deseas eliminar la reserva de 
      <strong>{{ reservaAEliminar?.name }}</strong>?
    </p>

    <div class="modal-actions">
      <button class="btn-outline" @click="mostrarEliminar = false">
        Cancelar
      </button>

      <button class="btn-delete" @click="eliminarReserva" :disabled="eliminando">
        {{ eliminando ? 'Eliminando...' : 'Eliminar' }}
      </button>
    </div>
  </div>
</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const reservas = ref([])
const mostrarEliminar = ref(false)
const reservaAEliminar = ref(null)
const eliminando = ref(false)

// Estado del calendario
const viewDate = ref(new Date())
const fechaSeleccionada = ref(null) // 'YYYY-MM-DD'

const MESES = [
  'Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo', 'Junio',
  'Julio', 'Agosto', 'Septiembre', 'Octubre', 'Noviembre', 'Diciembre'
]

const diasSemana = ['Dom', 'Lun', 'Mar', 'Mié', 'Jue', 'Vie', 'Sáb']

// ── Helpers ──────────────────────────────────────────────────────

const fmtISO = (y, m, d) =>
  `${y}-${String(m + 1).padStart(2, '0')}-${String(d).padStart(2, '0')}`

const tokenExpirado = (token) => {
  try {
    const payload = JSON.parse(atob(token.split('.')[1]))
    return payload.exp * 1000 < Date.now()
  } catch {
    return true
  }
}

// ── Computadas del calendario ─────────────────────────────────────

const mesActual = computed(() => {
  const y = viewDate.value.getFullYear()
  const m = viewDate.value.getMonth()
  return `${MESES[m]} ${y}`
})

const primerDiaSemana = computed(() => {
  const d = new Date(viewDate.value.getFullYear(), viewDate.value.getMonth(), 1)
  return d.getDay() // 0 = Dom
})

const diasEnMes = computed(() => {
  const y = viewDate.value.getFullYear()
  const m = viewDate.value.getMonth()
  return new Date(y, m + 1, 0).getDate()
})

// Extrae solo 'YYYY-MM-DD' de un campo que puede venir como '2026-04-07 16:00'
const soloFecha = (str) => str ? str.slice(0, 10) : ''

const fechasConReservas = computed(() =>
  new Set(reservas.value.map(r => soloFecha(r.reservation_date)))
)

const reservasFiltradas = computed(() => {
  if (!fechaSeleccionada.value) return []
  return reservas.value.filter(r => soloFecha(r.reservation_date) === fechaSeleccionada.value)
})

const fechaFormateada = computed(() => {
  if (!fechaSeleccionada.value) return ''
  const [y, m, d] = fechaSeleccionada.value.split('-')
  return `${parseInt(d)} de ${MESES[parseInt(m) - 1]} de ${y}`
})

// ── Métodos del calendario ────────────────────────────────────────

const tieneReservas = (dia) => {
  const y = viewDate.value.getFullYear()
  const m = viewDate.value.getMonth()
  return fechasConReservas.value.has(fmtISO(y, m, dia))
}

const esSeleccionado = (dia) => {
  const y = viewDate.value.getFullYear()
  const m = viewDate.value.getMonth()
  return fechaSeleccionada.value === fmtISO(y, m, dia)
}

const esHoy = (dia) => {
  const hoy = new Date()
  const y = viewDate.value.getFullYear()
  const m = viewDate.value.getMonth()
  return (
    hoy.getFullYear() === y &&
    hoy.getMonth() === m &&
    hoy.getDate() === dia
  )
}

const seleccionarDia = (dia) => {
  const y = viewDate.value.getFullYear()
  const m = viewDate.value.getMonth()
  fechaSeleccionada.value = fmtISO(y, m, dia)
}

const cambiarMes = (delta) => {
  const d = new Date(viewDate.value)
  d.setMonth(d.getMonth() + delta)
  viewDate.value = d
  fechaSeleccionada.value = null
}

// ── Sesión ────────────────────────────────────────────────────────

const cerrarSesion = (mensaje = 'Sesión cerrada') => {
  localStorage.removeItem('token')
  localStorage.removeItem('rol')
  localStorage.removeItem('username')
  alert(mensaje)
  router.push('/api/login')
}

const volver = () => {
  router.push('/dashboard')
}

// ── Fetch de reservas ─────────────────────────────────────────────

const obtenerReservas = async () => {
  try {
    const token = localStorage.getItem('token')

    // Si no hay token o expiró, redirige sin alert ni borrar storage
    if (!token || tokenExpirado(token)) {
      router.push('/api/login')
      return
    }

    const res = await fetch('/api/proyectos', {
      headers: { Authorization: `Bearer ${token}` }
    })

    // Solo limpia sesión si el backend confirma que el token es inválido
    if (res.status === 401) {
      localStorage.removeItem('token')
      localStorage.removeItem('rol')
      localStorage.removeItem('username')
      router.push('/api/login')
      return
    }

    reservas.value = await res.json()
  } catch (error) {
    console.error('Error cargando reservas:', error)
  }
}

const abrirEliminar = (reserva) => {
  reservaAEliminar.value = reserva
  mostrarEliminar.value = true
}

const eliminarReserva = async () => {
  if (!reservaAEliminar.value) return

  eliminando.value = true

  try {
    const token = localStorage.getItem('token')

    const res = await fetch(`/api/reservations/${reservaAEliminar.value.id}`, {
      method: 'DELETE',
      headers: {
        Authorization: `Bearer ${token}`
      }
    })

    const data = await res.json()

    if (!res.ok) throw new Error(data.error)

    // 🔄 Recargar reservas
    await obtenerReservas()

    mostrarEliminar.value = false
    reservaAEliminar.value = null

  } catch (err) {
    alert(err.message)
  } finally {
    eliminando.value = false
  }
}

onMounted(() => {
  obtenerReservas()
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@400;600;700&family=Montserrat:wght@300;400;500;600&display=swap');

/* RESET */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

/* 🔥 FONDO GLOBAL (SOLUCIONA BLANCO) */
:global(html),
:global(body),
:global(#app) {
  background: #0d0d0d;
  min-height: 100%;
  margin: 0;
  font-size: 18px; /* 🔥 hace todo más grande */
}

/* ── LAYOUT ── */
.screen {
  min-height: 100vh;
  background: #0d0d0d;
  display: flex;
  align-items: flex-start; /* 🔥 arriba */
  justify-content: center;
  padding: 20px 10px;
  font-family: 'Montserrat', sans-serif;
  position: relative;
  overflow: hidden;
}

/* FONDO DECORATIVO */
.bg-grid,
.glow { pointer-events: none; }

.bg-grid {
  position: fixed; inset: 0;
  background-image:
    linear-gradient(rgba(180,145,80,.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(180,145,80,.03) 1px, transparent 1px);
  background-size: 60px 60px;
}

.glow {
  position: fixed;
  width: 500px; height: 500px;
  border-radius: 50%;
  filter: blur(120px);
  opacity: .3;
}
.glow.left  { background: radial-gradient(circle, #b4915025, transparent 70%); top: -100px; left: -150px; }
.glow.right { background: radial-gradient(circle, #b4915015, transparent 70%); bottom: -100px; right: -150px; }

/* CARD */
.card {
  position: relative;
  z-index: 10;
  width: 100%;
  max-width: 95vw; /* 🔥 más ancho en móvil */
  background: #141414;
  border: 1px solid rgba(180,145,80,.2);
  box-shadow: 0 32px 80px rgba(0,0,0,.7);
  padding: 40px 20px;
  text-align: center;
}

.center-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px; /* 🔥 más aire */
}

/* TEXTOS */
.eyebrow {
  font-size: .9rem;
  letter-spacing: .3em;
  text-transform: uppercase;
  color: #b49150;
}

h1 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 2.8rem; /* 🔥 más grande */
  color: #f0e6d0;
  letter-spacing: .08em;
}

/* DIVIDER */
.divider {
  display: flex;
  align-items: center;
  gap: 10px;
}
.divider span:not(.diamond) {
  width: 60px; height: 1px;
  background: linear-gradient(90deg, transparent, #b49150);
}
.divider span:last-child {
  background: linear-gradient(90deg, #b49150, transparent);
}
.diamond { font-size: .5rem; color: #b49150; }

/* ── MES ── */
.month-nav {
  display: flex;
  align-items: center;
  gap: 16px;
}

.month-arrow {
  width: 44px; height: 44px; /* 🔥 más grande */
  font-size: 1.4rem;
}

.month-label {
  font-size: 1.1rem;
}

/* ── CALENDARIO ── */
.calendar { width: 100%; }

.cal-header span {
  font-size: .85rem;
}

.day-btn {
  font-size: 1rem;
  min-height: 65px; /* 🔥 táctil cómodo */
}

/* ── TABLA ── */
table {
  font-size: 1rem;
}

th {
  font-size: .8rem;
}

td {
  padding: 14px;
}

/* ── BOTONES ── */
.btn-primary {
  width: 100%;
  min-height: 60px;
  font-size: .9rem;
}

/* MODAL */
.modal {
  max-width: 90%;
}
</style>