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

        <!-- ✅ TABLA -->
        <div class="table-container" v-if="reservas.length">
          <table>
            <thead>
              <tr>
                <th>Nombre</th>
                <th>Whasapt</th>
                <th>Fecha</th>
                <th>Nota</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="r in reservas" :key="r.id">
                <td>{{ r.name }}</td>
                <td>{{ r.phone }}</td>
                <td>{{ r.reservation_date }}</td>
                <td>{{ r.note || '-' }}</td>
              </tr>
            </tbody>
          </table>
        </div>

        <p v-else class="empty">No hay reservas</p>

        <!-- 🔘 BOTONES -->
        <button class="btn-primary center-btn" @click="volver">
          <span>Volver</span>
        </button>

        <button class="btn-primary center-btn" @click="cerrarSesion">
          <span>Cerrar sesión</span>
        </button>

      </div>

    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const reservas = ref([])

// 🔍 Validar expiración del token
const tokenExpirado = (token) => {
  try {
    const payload = JSON.parse(atob(token.split('.')[1]))
    return payload.exp * 1000 < Date.now()
  } catch {
    return true
  }
}

// 🔴 LOGOUT
const cerrarSesion = (mensaje = 'Sesión cerrada') => {
  localStorage.removeItem('token')
  localStorage.removeItem('rol')
  localStorage.removeItem('username')

  alert(mensaje)
  router.push('/api/login')
}

// 🔐 VALIDAR SESIÓN
const validarSesion = () => {
  const token = localStorage.getItem('token')

  if (!token || tokenExpirado(token)) {
    cerrarSesion('Tu sesión expiró, inicia sesión nuevamente')
    return null
  }

  return token
}

// 🔙 VOLVER
const volver = () => {
  router.push('/dashboard')
}

// 📡 Obtener reservas
const obtenerReservas = async () => {
  try {
    const token = validarSesion()
    if (!token) return

    const res = await fetch('/api/proyectos', {
      headers: {
        Authorization: `Bearer ${token}`
      }
    })

    if (res.status === 401) {
      cerrarSesion('Sesión inválida')
      return
    }

    reservas.value = await res.json()

  } catch (error) {
    console.error('Error cargando reservas:', error)
  }
}

// 🚀 AL MONTAR
onMounted(() => {
  obtenerReservas()
})
</script>

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

.bg-grid,
.glow {
  pointer-events: none;
}

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

.card {
  position: relative;
  z-index: 10;
  width: 100%;
  max-width: 700px;
  background: #141414;
  border: 1px solid rgba(180,145,80,.2);
  box-shadow: 0 32px 80px rgba(0,0,0,.7);
  padding: 60px 40px;
  text-align: center;
}

.center-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.eyebrow {
  font-size: .65rem;
  letter-spacing: .3em;
  text-transform: uppercase;
  color: #b49150;
}

h1 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 2.3rem;
  color: #f0e6d0;
  letter-spacing: .08em;
}

.divider {
  display: flex;
  align-items: center;
  gap: 10px;
}

.divider span:not(.diamond) {
  width: 60px;
  height: 1px;
  background: linear-gradient(90deg, transparent, #b49150);
}

.divider span:last-child {
  background: linear-gradient(90deg, #b49150, transparent);
}

.diamond {
  font-size: .4rem;
  color: #b49150;
}

.table-container {
  width: 100%;
  margin-top: 30px;
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  color: #f0e6d0;
  font-size: 0.85rem;
}

thead {
  background: rgba(180,145,80,0.1);
}

th, td {
  padding: 10px;
  border-bottom: 1px solid rgba(180,145,80,0.2);
  text-align: center;
}

th {
  color: #b49150;
  font-weight: 500;
  letter-spacing: .1em;
}

tbody tr:hover {
  background: rgba(180,145,80,0.08);
}

.empty {
  margin-top: 20px;
  color: #888;
  font-size: 0.85rem;
}

.btn-primary {
  margin-top: 20px;
  padding: 14px 40px;
  border: 1px solid #b49150;
  background: transparent;
  color: #b49150;
  letter-spacing: .3em;
  text-transform: uppercase;
  cursor: pointer;
  transition: all .3s;
}

.btn-primary:hover {
  background: #b49150;
  color: #0d0d0d;
}
</style>