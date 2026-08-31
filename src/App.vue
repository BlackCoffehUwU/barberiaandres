<template>

  <!-- Spinner de carga inicial: se muestra 1 segundo al abrir la app -->
  <template v-if="cargando">
    <div class="pantalla-carga">
      <div class="spinner"></div>
      <p>Cargando...</p>
    </div>
  </template>

  <template v-else>
    <header>
      <h1>💈 Barbería Don Ramiro</h1>
    </header>

    <!-- Resumen: funciones normales (no computed) llamadas desde el template -->
    <div class="resumen">
      <div class="tarjeta-resumen">
        <div class="valor">{{ formatearPrecio(totalSemana()) }}</div>
        <div class="etiqueta">Vendido esta semana</div>
      </div>
      <div class="tarjeta-resumen">
        <div class="valor">{{ servicioMasPedido() }}</div>
        <div class="etiqueta">Servicio más pedido</div>
      </div>
      <div class="tarjeta-resumen">
        <div class="valor">{{ contarPendientes() }}</div>
        <div class="etiqueta">Pagos pendientes</div>
      </div>
    </div>

    <!-- Lista de servicios registrados -->
    <div v-if="servicios.length === 0" class="vacio">
      Todavía no hay servicios registrados. Toca el botón + para agregar el primero.
    </div>

    <div
      class="servicio-card"
      v-for="servicio in servicios"
      :key="servicio.id"
      :class="{ pendiente: servicio.estadoPago === 'pendiente' }"
    >
      <div class="fila-superior">
        <div>
          <div class="cliente">{{ servicio.cliente }}</div>
          <span class="badge" :class="servicio.estadoPago">{{ servicio.estadoPago }}</span>
        </div>
        <div class="precio">${{ servicio.precio }}</div>
      </div>

      <div class="detalle">
        {{ servicio.tipoServicio }} — atendido por {{ servicio.barbero }}
      </div>
      <div class="detalle">
        🗓️ {{ servicio.fecha }} — 🕐 {{ servicio.hora }}
      </div>
      <div class="detalle">
        <span v-if="servicio.metodoPago === 'efectivo'">💵 Efectivo</span>
        <span v-else-if="servicio.metodoPago === 'transferencia'">🏦 Transferencia</span>
        <span v-else-if="servicio.metodoPago === 'tarjeta'">💳 Tarjeta</span>
      </div>

      <div class="estrellas" :class="{ baja: servicio.calificacion <= 2 }">
        <span v-for="n in 5" :key="n" :class="{ llena: n <= servicio.calificacion }">★</span>
      </div>

      <div class="detalle" v-if="servicio.observaciones">
        📝 {{ servicio.observaciones }}
      </div>

      <div class="acciones-card">
        <button class="btn-editar" @click="abrirModalEditar(servicio)">Editar</button>
        <button class="btn-eliminar" @click="pedirConfirmacionEliminar(servicio.id)">Eliminar</button>
      </div>
    </div>

    <!-- Botón flotante para abrir el modal de nuevo servicio -->
    <button class="btn-flotante" @click="abrirModalNuevo">+</button>

    <!-- Paso 1: modal con los datos básicos del corte -->
    <div class="fondo-modal" v-if="mostrarModal">
      <div class="caja-modal">
        <h2>{{ modoEdicion ? 'Editar servicio' : 'Nuevo servicio' }}</h2>

        <div class="campo">
          <label>Nombre del cliente</label>
          <input type="text" v-model="formulario.cliente" placeholder="Ej: Carlos Pérez">
          <div class="error" v-if="errores.cliente">{{ errores.cliente }}</div>
        </div>

        <div class="campo">
          <label>Tipo de servicio</label>
          <select v-model="formulario.tipoServicio" @change="actualizarPrecio">
            <option value="">Selecciona...</option>
            <option v-for="tipo in tiposServicio" :key="tipo.nombre" :value="tipo.nombre">
              {{ tipo.nombre }} (${{ tipo.precio }})
            </option>
          </select>
          <div class="error" v-if="errores.tipoServicio">{{ errores.tipoServicio }}</div>
        </div>

        <div class="campo">
          <label>Barbero que atendió</label>
          <select v-model="formulario.barbero">
            <option value="">Selecciona...</option>
            <option v-for="b in barberos" :key="b" :value="b">{{ b }}</option>
          </select>
          <div class="error" v-if="errores.barbero">{{ errores.barbero }}</div>
        </div>

        <!-- Fecha y hora en cuadros separados -->
        <div class="campo-doble">
          <div class="campo">
            <label>Fecha</label>
            <input type="date" v-model="formulario.fecha">
            <div class="error" v-if="errores.fecha">{{ errores.fecha }}</div>
          </div>
          <div class="campo">
            <label>Hora</label>
            <input type="time" v-model="formulario.hora">
            <div class="error" v-if="errores.hora">{{ errores.hora }}</div>
          </div>
        </div>

        <div class="campo">
          <label>Precio cobrado</label>
          <input type="number" v-model.number="formulario.precio" min="0" placeholder="Se llena solo al elegir el servicio">
          <div class="error" v-if="errores.precio">{{ errores.precio }}</div>
        </div>

        <div class="campo">
          <label>Método de pago</label>
          <select v-model="formulario.metodoPago">
            <option value="">Selecciona...</option>
            <option value="efectivo">Efectivo</option>
            <option value="transferencia">Transferencia</option>
            <option value="tarjeta">Tarjeta</option>
          </select>
          <div class="error" v-if="errores.metodoPago">{{ errores.metodoPago }}</div>
        </div>

        <div class="campo">
          <label>Estado del pago</label>
          <select v-model="formulario.estadoPago">
            <option value="">Selecciona...</option>
            <option value="pagado">Pagado</option>
            <option value="pendiente">Pendiente</option>
          </select>
          <div class="error" v-if="errores.estadoPago">{{ errores.estadoPago }}</div>
        </div>

        <div class="botones-modal">
          <button class="btn-cancelar" @click="cerrarModal">Cancelar</button>
          <button class="btn-guardar" @click="guardarServicio">Guardar</button>
        </div>
      </div>
    </div>

    <!-- Spinner de "guardando" que aparece 2 segundos después de dar Guardar -->
    <div class="fondo-modal" v-if="guardando">
      <div class="caja-confirmacion">
        <div class="spinner"></div>
        <p>Guardando el corte...</p>
      </div>
    </div>

    <!-- Paso 2: calificación y observaciones, después de guardar el corte -->
    <div class="fondo-modal" v-if="mostrarModalCalificacion">
      <div class="caja-modal">
        <h2>¿Cómo te fue con el cliente?</h2>

        <div class="campo">
          <label>Calificación del cliente</label>
          <div class="selector-estrellas">
            <span
              v-for="n in 5"
              :key="n"
              :class="{ activa: n <= formularioCalificacion.calificacion }"
              @click="formularioCalificacion.calificacion = n"
            >★</span>
          </div>
        </div>

        <div class="campo">
          <label>Observaciones (opcional)</label>
          <textarea v-model="formularioCalificacion.observaciones" rows="2" placeholder="Ej: pidió cita para el próximo mes"></textarea>
        </div>

        <div class="botones-modal">
          <button class="btn-guardar" @click="guardarCalificacion">Finalizar</button>
        </div>
      </div>
    </div>

    <!-- Modal de confirmación de borrado (reemplaza al confirm() nativo de JS) -->
    <div class="fondo-modal" v-if="mostrarConfirmacion">
      <div class="caja-confirmacion">
        <p>¿Seguro que quieres eliminar este servicio? Esta acción no se puede deshacer.</p>
        <div class="botones-modal">
          <button class="btn-cancelar" @click="cancelarEliminar">Cancelar</button>
          <button class="btn-eliminar" @click="confirmarEliminar">Sí, eliminar</button>
        </div>
      </div>
    </div>
  </template>

</template>

<script>
import { ref, onMounted } from 'vue'
import { useLocalStorage } from '@vueuse/core'

export default {
  name: 'App',
  setup() {

    const cargando = ref(true)
    onMounted(() => {
      setTimeout(() => {
        cargando.value = false
      }, 1000)
    })

    const tiposServicio = [
      { nombre: 'Corte clásico', precio: 15000 },
      { nombre: 'Corte moderno', precio: 20000 },
      { nombre: 'Barba', precio: 10000 },
      { nombre: 'Corte + Barba', precio: 25000 },
      { nombre: 'Cejas', precio: 8000 },
      { nombre: 'Tinte', precio: 35000 }
    ]

    const barberos = ['Don Ramiro', 'Empleado 1', 'Empleado 2']

    const servicios = useLocalStorage('barberia-servicios', [])

    const mostrarModal = ref(false)
    const modoEdicion = ref(false)
    const idEditando = ref(null)

    function formularioVacio() {
      return {
        cliente: '',
        tipoServicio: '',
        barbero: '',
        fecha: '',
        hora: '',
        precio: null,
        metodoPago: '',
        estadoPago: ''
      }
    }

    const formulario = ref(formularioVacio())
    const errores = ref({})

    const guardando = ref(false)

    const mostrarModalCalificacion = ref(false)
    const idParaCalificar = ref(null)
    const formularioCalificacion = ref({ calificacion: 5, observaciones: '' })

    const mostrarConfirmacion = ref(false)
    const idAEliminar = ref(null)

    function actualizarPrecio() {
      const encontrado = tiposServicio.find(t => t.nombre === formulario.value.tipoServicio)
      if (encontrado) {
        formulario.value.precio = encontrado.precio
      }
    }

    function abrirModalNuevo() {
      modoEdicion.value = false
      idEditando.value = null
      formulario.value = formularioVacio()
      errores.value = {}
      mostrarModal.value = true
    }

    function abrirModalEditar(servicio) {
      modoEdicion.value = true
      idEditando.value = servicio.id
      formulario.value = { ...servicio }
      errores.value = {}
      mostrarModal.value = true
    }

    function cerrarModal() {
      mostrarModal.value = false
    }

    function validarFormulario() {
      const nuevosErrores = {}

      if (!formulario.value.cliente.trim()) {
        nuevosErrores.cliente = 'El nombre del cliente es obligatorio.'
      }
      if (!formulario.value.tipoServicio) {
        nuevosErrores.tipoServicio = 'Selecciona un tipo de servicio.'
      }
      if (!formulario.value.barbero) {
        nuevosErrores.barbero = 'Selecciona quién atendió.'
      }
      if (!formulario.value.fecha) {
        nuevosErrores.fecha = 'Selecciona la fecha.'
      }
      if (!formulario.value.hora) {
        nuevosErrores.hora = 'Selecciona la hora.'
      }
      if (!formulario.value.precio || formulario.value.precio <= 0) {
        nuevosErrores.precio = 'El precio debe ser mayor a 0.'
      }
      if (!formulario.value.metodoPago) {
        nuevosErrores.metodoPago = 'Selecciona el método de pago.'
      }
      if (!formulario.value.estadoPago) {
        nuevosErrores.estadoPago = 'Selecciona el estado del pago.'
      }

      errores.value = nuevosErrores
      return Object.keys(nuevosErrores).length === 0
    }

    function guardarServicio() {
      if (!validarFormulario()) {
        return
      }

      mostrarModal.value = false
      guardando.value = true

      let idGuardado

      if (modoEdicion.value) {
        idGuardado = idEditando.value
        const index = servicios.value.findIndex(s => s.id === idGuardado)
        if (index !== -1) {
          servicios.value[index] = { ...servicios.value[index], ...formulario.value }
        }
      } else {
        idGuardado = Date.now()
        servicios.value.push({
          ...formulario.value,
          id: idGuardado,
          calificacion: 5,
          observaciones: ''
        })
      }

      setTimeout(() => {
        guardando.value = false
        abrirModalCalificacion(idGuardado)
      }, 2000)
    }

    function abrirModalCalificacion(id) {
      const servicio = servicios.value.find(s => s.id === id)
      idParaCalificar.value = id
      formularioCalificacion.value = {
        calificacion: servicio ? servicio.calificacion : 5,
        observaciones: servicio ? servicio.observaciones : ''
      }
      mostrarModalCalificacion.value = true
    }

    function guardarCalificacion() {
      const index = servicios.value.findIndex(s => s.id === idParaCalificar.value)
      if (index !== -1) {
        servicios.value[index].calificacion = formularioCalificacion.value.calificacion
        servicios.value[index].observaciones = formularioCalificacion.value.observaciones
      }
      mostrarModalCalificacion.value = false
      idParaCalificar.value = null
    }

    function pedirConfirmacionEliminar(id) {
      idAEliminar.value = id
      mostrarConfirmacion.value = true
    }

    function cancelarEliminar() {
      mostrarConfirmacion.value = false
      idAEliminar.value = null
    }

    function confirmarEliminar() {
      servicios.value = servicios.value.filter(s => s.id !== idAEliminar.value)
      mostrarConfirmacion.value = false
      idAEliminar.value = null
    }

    function formatearPrecio(valor) {
      if (!valor) return '$0'
      return '$' + Number(valor).toLocaleString('es-CO')
    }

    function totalSemana() {
      const haceUnaSemana = new Date()
      haceUnaSemana.setDate(haceUnaSemana.getDate() - 7)

      let total = 0
      for (const s of servicios.value) {
        const fechaCompleta = new Date(s.fecha + 'T' + (s.hora || '00:00'))
        if (fechaCompleta >= haceUnaSemana) {
          total += Number(s.precio) || 0
        }
      }
      return total
    }

    function servicioMasPedido() {
      if (servicios.value.length === 0) return '—'

      const conteo = {}
      for (const s of servicios.value) {
        conteo[s.tipoServicio] = (conteo[s.tipoServicio] || 0) + 1
      }

      let masPedido = ''
      let max = 0
      for (const tipo in conteo) {
        if (conteo[tipo] > max) {
          max = conteo[tipo]
          masPedido = tipo
        }
      }
      return masPedido
    }

    function contarPendientes() {
      return servicios.value.filter(s => s.estadoPago === 'pendiente').length
    }

    return {
      cargando,
      tiposServicio,
      barberos,
      servicios,
      mostrarModal,
      modoEdicion,
      formulario,
      errores,
      guardando,
      mostrarModalCalificacion,
      formularioCalificacion,
      mostrarConfirmacion,
      actualizarPrecio,
      abrirModalNuevo,
      abrirModalEditar,
      cerrarModal,
      guardarServicio,
      guardarCalificacion,
      pedirConfirmacionEliminar,
      cancelarEliminar,
      confirmarEliminar,
      formatearPrecio,
      totalSemana,
      servicioMasPedido,
      contarPendientes
    }
  }
}
</script>

<style>
* { box-sizing: border-box; }

body {
  margin: 0;
  font-family: Arial, Helvetica, sans-serif;
  background: #f4f1ee;
  color: #2b2b2b;
  padding: 16px;
  padding-bottom: 90px;
}

.pantalla-carga {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 14px;
  color: #555;
}

.spinner {
  width: 44px;
  height: 44px;
  border: 5px solid #e2ddd5;
  border-top: 5px solid #1f1f1f;
  border-radius: 50%;
  animation: girar 0.8s linear infinite;
  margin: 0 auto 10px;
}

@keyframes girar {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

header {
  background: #1f1f1f;
  color: #fff;
  padding: 16px;
  text-align: center;
  border-radius: 8px;
  margin-bottom: 16px;
}

header h1 {
  margin: 0;
  font-size: 20px;
  color: #f4f1ee;
}

.resumen {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  margin-bottom: 16px;
}

.resumen .tarjeta-resumen {
  flex: 1;
  min-width: 120px;
  background: #fff;
  border-radius: 10px;
  padding: 10px;
  text-align: center;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.resumen .valor {
  font-size: 17px;
  font-weight: bold;
}

.resumen .etiqueta {
  font-size: 11px;
  color: #777;
}

.servicio-card {
  background: #fff;
  border-radius: 8px;
  padding: 14px;
  margin-bottom: 10px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  border-left: 5px solid #c9a267;
}

.servicio-card.pendiente {
  border-left-color: #e0a800;
  background: #fffaf0;
}

.fila-superior {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.servicio-card .cliente {
  font-weight: bold;
  font-size: 15px;
}

.servicio-card .precio {
  font-weight: bold;
  color: #1f7a4d;
}

.badge {
  display: inline-block;
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 20px;
  margin-top: 4px;
  color: #fff;
}

.badge.pagado { background: #2e8b57; }
.badge.pendiente { background: #e0a800; }

.servicio-card .detalle {
  font-size: 13px;
  color: #555;
  margin-top: 6px;
}

.estrellas {
  color: #ccc;
  font-size: 14px;
  margin-top: 6px;
}

.estrellas .llena {
  color: #f0ad4e;
}

.estrellas.baja .llena {
  color: #d9534f;
}

.acciones-card {
  display: flex;
  gap: 8px;
  margin-top: 10px;
}

.acciones-card button {
  flex: 1;
  padding: 8px;
  border: none;
  border-radius: 6px;
  font-size: 13px;
  cursor: pointer;
}

.btn-editar { background: #e9e4dc; }
.btn-eliminar { background: #f8d7d5; color: #a12922; }

.vacio {
  text-align: center;
  color: #888;
  padding: 30px 10px;
}

.btn-flotante {
  position: fixed;
  bottom: 20px;
  right: 20px;
  width: 58px;
  height: 58px;
  border-radius: 50%;
  background: #1f1f1f;
  color: #fff;
  font-size: 28px;
  border: none;
  box-shadow: 0 3px 8px rgba(0,0,0,0.3);
  cursor: pointer;
}

.fondo-modal {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: flex-end;
  justify-content: center;
  z-index: 10;
}

.caja-modal {
  background: #fff;
  width: 100%;
  max-width: 480px;
  max-height: 90vh;
  overflow-y: auto;
  border-radius: 16px 16px 0 0;
  padding: 20px;
}

@media (min-width: 520px) {
  .fondo-modal { align-items: center; }
  .caja-modal { border-radius: 16px; }
}

.caja-modal h2 {
  margin-top: 0;
}

.campo {
  margin-bottom: 12px;
  flex: 1;
}

.campo-doble {
  display: flex;
  gap: 10px;
}

.campo label {
  display: block;
  font-size: 13px;
  font-weight: bold;
  margin-bottom: 4px;
}

.campo input,
.campo select,
.campo textarea {
  width: 100%;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 14px;
}

.error {
  color: #d9534f;
  font-size: 12px;
  margin-top: 3px;
}

.selector-estrellas {
  font-size: 24px;
  cursor: pointer;
}

.selector-estrellas span {
  color: #ccc;
  margin-right: 4px;
}

.selector-estrellas span.activa {
  color: #f0ad4e;
}

.botones-modal {
  display: flex;
  gap: 10px;
  margin-top: 16px;
}

.botones-modal button {
  flex: 1;
  padding: 12px;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  cursor: pointer;
}

.btn-cancelar { background: #eee; }
.btn-guardar { background: #1f1f1f; color: #fff; }

.caja-confirmacion {
  background: #fff;
  border-radius: 12px;
  padding: 24px;
  max-width: 320px;
  text-align: center;
}
</style>