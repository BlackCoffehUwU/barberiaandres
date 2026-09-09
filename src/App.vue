<template>

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

    <div v-if="obtenerAbonados().length > 0" class="seccion-abonados">
      <h2 class="titulo-seccion">💰 Clientes abonados</h2>
      <div class="grilla-abonados">
        <div class="abonado-card" v-for="servicio in obtenerAbonados()" :key="servicio.id">
          <div class="abonado-cliente">{{ servicio.cliente }}</div>
          <div class="abonado-fila">
            <div class="abonado-dato">
              <div class="abonado-valor abonado-color">{{ formatearPrecio(servicio.cantidadAbonada) }}</div>
              <div class="abonado-etiqueta">Abonó</div>
            </div>
            <div class="abonado-dato">
              <div class="abonado-valor falta-color">{{ formatearPrecio((servicio.precio || 0) - (servicio.cantidadAbonada || 0)) }}</div>
              <div class="abonado-etiqueta">Falta</div>
            </div>
            <div class="abonado-dato">
              <div class="abonado-valor">{{ formatearPrecio(servicio.precio) }}</div>
              <div class="abonado-etiqueta">Total</div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <h2 class="titulo-seccion">🧾 Registros</h2>

    <div v-if="servicios.length === 0" class="vacio">
      Todavía no hay servicios registrados. Toca el botón + para agregar el primero.
    </div>

    <div class="grilla-servicios">
      <div
        class="servicio-card"
        v-for="servicio in servicios"
        :key="servicio.id"
        :class="{ pendiente: servicio.estadoPago === 'pendiente', abonado: servicio.estadoPago === 'abonado' }"
      >
        <div class="fila-superior">
          <div>
            <div class="cliente">{{ servicio.cliente }}</div>
            <span class="badge" :class="servicio.estadoPago">{{ servicio.estadoPago }}</span>
          </div>
          <div class="precio">{{ formatearPrecio(servicio.precio) }}</div>
        </div>

        <div class="chips-servicios">
          <span class="chip" v-for="tipo in servicio.servicios" :key="tipo">{{ tipo }}</span>
        </div>

        <div class="fila-info">
          <span class="info-item">✂️ {{ servicio.barbero }}</span>
          <span class="info-item">🗓️ {{ servicio.fecha }}</span>
          <span class="info-item">🕐 {{ servicio.hora }}</span>
        </div>

        <div class="fila-info">
          <span class="info-item" v-if="servicio.metodoPago === 'efectivo'">💵 Efectivo</span>
          <span class="info-item" v-else-if="servicio.metodoPago === 'transferencia'">🏦 Transferencia</span>
          <span class="info-item" v-else-if="servicio.metodoPago === 'tarjeta'">💳 Tarjeta</span>
        </div>

        <div class="fila-info" v-if="servicio.estadoPago === 'abonado'">
          <span class="info-item">💰 Abonado: {{ formatearPrecio(servicio.cantidadAbonada) }}</span>
          <span class="info-item">⏳ Falta: {{ formatearPrecio((servicio.precio || 0) - (servicio.cantidadAbonada || 0)) }}</span>
        </div>

        <div class="zona-calificacion">
          <div v-if="servicio.calificacion > 0" class="fila-calificacion">
            <div class="estrellas" :class="{ baja: servicio.calificacion <= 2 }">
              <span v-for="n in 5" :key="n" :class="{ llena: n <= servicio.calificacion }">★</span>
            </div>
            <button class="btn-link" @click="abrirModalCalificacion(servicio.id)">Editar calificación</button>
          </div>
          <button v-else class="btn-calificar" @click="abrirModalCalificacion(servicio.id)">
            ⭐ Califica tu corte aquí
          </button>
        </div>

        <div class="observaciones" v-if="servicio.observaciones">
          📝 {{ servicio.observaciones }}
        </div>

        <div class="acciones-card">
          <button class="btn-editar" @click="abrirModalEditar(servicio)">✏️ Editar</button>
          <button class="btn-eliminar" @click="pedirConfirmacionEliminar(servicio.id)">🗑️ Eliminar</button>
        </div>
      </div>
    </div>

    <button class="btn-flotante" @click="abrirModalNuevo">+</button>

    <div class="fondo-modal" v-if="mostrarModal">
      <div class="caja-modal">
        <h2>{{ modoEdicion ? 'Editar servicio' : 'Nuevo servicio' }}</h2>

        <div class="campo">
          <label>Nombre del cliente</label>
          <input type="text" v-model="formulario.cliente" placeholder="Ej: Carlos Pérez">
          <div class="error" v-if="errores.cliente">{{ errores.cliente }}</div>
        </div>

        <div class="campo">
          <label>Servicios solicitados</label>
          <div class="dropdown-servicios">
            <div class="dropdown-cabecera" @click="mostrarListaServicios = !mostrarListaServicios">
              <span v-if="formulario.servicios.length === 0" class="placeholder">Selecciona uno o varios...</span>
              <span v-else>{{ formulario.servicios.join(', ') }}</span>
              <span class="flecha">{{ mostrarListaServicios ? '▲' : '▼' }}</span>
            </div>
            <div class="dropdown-opciones" v-show="mostrarListaServicios">
              <label class="check-servicio" v-for="tipo in catalogoServicios" :key="tipo.nombre">
                <input
                  type="checkbox"
                  :value="tipo.nombre"
                  :checked="formulario.servicios.includes(tipo.nombre)"
                  @change="seleccionarServicio(tipo.nombre)"
                >
                {{ tipo.nombre }} (${{ formatearNumero(tipo.precio) }})
              </label>
            </div>
          </div>
          <div class="error" v-if="errores.servicios">{{ errores.servicios }}</div>
        </div>

        <div class="campo">
          <label>Barbero que atendió</label>
          <select v-model="formulario.barbero">
            <option value="">Selecciona...</option>
            <option v-for="b in barberos" :key="b" :value="b">{{ b }}</option>
          </select>
          <div class="error" v-if="errores.barbero">{{ errores.barbero }}</div>
        </div>

        <div class="campo-doble">
          <div class="campo">
            <label>Fecha</label>
            <input type="date" v-model="formulario.fecha" :min="fechaHoy()">
            <div class="error" v-if="errores.fecha">{{ errores.fecha }}</div>
          </div>
          <div class="campo">
            <label>Hora</label>
            <input type="time" v-model="formulario.hora" :min="horaMinimaCampo()">
            <div class="error" v-if="errores.hora">{{ errores.hora }}</div>
          </div>
        </div>

        <div class="campo">
          <label>Precio cobrado</label>
          <input
            type="text"
            inputmode="numeric"
            :value="formatearNumero(formulario.precio)"
            @input="actualizarCampoMoneda($event, 'precio')"
            placeholder="Se llena solo al elegir los servicios"
          >
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
            <option value="abonado">Abonado</option>
            <option value="pendiente">Pendiente</option>
          </select>
          <div class="error" v-if="errores.estadoPago">{{ errores.estadoPago }}</div>
        </div>

        <div class="campo" v-if="formulario.estadoPago === 'abonado'">
          <label>Cantidad abonada</label>
          <input
            type="text"
            inputmode="numeric"
            :value="formatearNumero(formulario.cantidadAbonada)"
            @input="actualizarCampoMoneda($event, 'cantidadAbonada')"
            placeholder="Ej: 10.000"
          >
          <div class="error" v-if="errores.cantidadAbonada">{{ errores.cantidadAbonada }}</div>
        </div>

        <div class="botones-modal">
          <button class="btn-cancelar" @click="cerrarModal">Cancelar</button>
          <button class="btn-guardar" @click="guardarServicio">Guardar</button>
        </div>
      </div>
    </div>

    <div class="fondo-modal" v-if="guardando">
      <div class="caja-confirmacion">
        <div class="spinner"></div>
        <p>Guardando el corte...</p>
      </div>
    </div>

    <div class="fondo-modal" v-if="mostrarModalCalificacion">
      <div class="caja-modal">
        <h2>¿Cómo estuvo tu corte?</h2>

        <div class="campo">
          <label>Tu calificación</label>
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
          <label>Cuéntanos más (opcional)</label>
          <textarea v-model="formularioCalificacion.observaciones" rows="2" placeholder="Ej: me encantó el corte, volveré pronto"></textarea>
        </div>

        <div class="botones-modal">
          <button class="btn-cancelar" @click="cerrarModalCalificacion">Cancelar</button>
          <button class="btn-guardar" @click="guardarCalificacion">Guardar calificación</button>
        </div>
      </div>
    </div>

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

    const catalogoServicios = [
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
    const mostrarListaServicios = ref(false)

    function formularioVacio() {
      return {
        cliente: '',
        servicios: [],
        barbero: '',
        fecha: '',
        hora: '',
        precio: null,
        metodoPago: '',
        estadoPago: '',
        cantidadAbonada: null
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

    function fechaHoy() {
      const hoy = new Date()
      const anio = hoy.getFullYear()
      const mes = String(hoy.getMonth() + 1).padStart(2, '0')
      const dia = String(hoy.getDate()).padStart(2, '0')
      return `${anio}-${mes}-${dia}`
    }

    function actualizarPrecio() {
      let total = 0
      for (const nombre of formulario.value.servicios) {
        const encontrado = catalogoServicios.find(t => t.nombre === nombre)
        if (encontrado) {
          total += encontrado.precio
        }
      }
      formulario.value.precio = total
    }

    const tiposDeCorte = ['Corte clásico', 'Corte moderno', 'Corte + Barba']

    function seleccionarServicio(tipo) {
      const yaEstaba = formulario.value.servicios.includes(tipo)

      if (yaEstaba) {
        formulario.value.servicios = formulario.value.servicios.filter(s => s !== tipo)
      } else {
        let nuevaSeleccion = [...formulario.value.servicios, tipo]

        if (tiposDeCorte.includes(tipo)) {
          nuevaSeleccion = nuevaSeleccion.filter(s => !tiposDeCorte.includes(s) || s === tipo)
        }

        if (tipo === 'Corte + Barba') {
          nuevaSeleccion = nuevaSeleccion.filter(s => s !== 'Barba')
        }

        if (tipo === 'Barba') {
          nuevaSeleccion = nuevaSeleccion.filter(s => s !== 'Corte + Barba')
        }

        formulario.value.servicios = nuevaSeleccion
      }

      actualizarPrecio()
    }

    function actualizarCampoMoneda(evento, campo) {
      const soloDigitos = evento.target.value.replace(/\D/g, '')
      formulario.value[campo] = soloDigitos ? Number(soloDigitos) : null
    }

    function abrirModalNuevo() {
      modoEdicion.value = false
      idEditando.value = null
      formulario.value = formularioVacio()
      errores.value = {}
      mostrarListaServicios.value = false
      mostrarModal.value = true
    }

    function abrirModalEditar(servicio) {
      modoEdicion.value = true
      idEditando.value = servicio.id
      formulario.value = {
        ...formularioVacio(),
        ...servicio,
        servicios: [...servicio.servicios]
      }
      errores.value = {}
      mostrarListaServicios.value = false
      mostrarModal.value = true
    }

    function cerrarModal() {
      mostrarModal.value = false
      mostrarListaServicios.value = false
    }

    function horaValida(hora) {
      return (hora >= '08:00' && hora <= '12:00') || (hora >= '14:00' && hora <= '18:00')
    }

    function horaMinimaCampo() {
      if (formulario.value.fecha !== fechaHoy()) return ''
      const ahora = new Date()
      ahora.setHours(ahora.getHours() + 1)
      const horas = String(ahora.getHours()).padStart(2, '0')
      const minutos = String(ahora.getMinutes()).padStart(2, '0')
      return `${horas}:${minutos}`
    }

    function minutosDesdeMedianoche(horaTexto) {
      const partes = horaTexto.split(':')
      const horas = Number(partes[0])
      const minutos = Number(partes[1])
      return (horas * 60) + minutos
    }

    function minutosAhora() {
      const ahora = new Date()
      return (ahora.getHours() * 60) + ahora.getMinutes()
    }

    function validarFormulario() {
      const nuevosErrores = {}

      if (!formulario.value.cliente.trim()) {
        nuevosErrores.cliente = 'El nombre del cliente es obligatorio.'
      }
      if (formulario.value.servicios.length === 0) {
        nuevosErrores.servicios = 'Selecciona al menos un servicio.'
      }
      if (!formulario.value.barbero) {
        nuevosErrores.barbero = 'Selecciona quién atendió.'
      }
      if (!formulario.value.fecha) {
        nuevosErrores.fecha = 'Selecciona la fecha.'
      } else if (formulario.value.fecha < fechaHoy()) {
        nuevosErrores.fecha = 'La fecha no puede ser anterior a hoy.'
      }
      if (!formulario.value.hora) {
        nuevosErrores.hora = 'Selecciona la hora.'
      } else if (!horaValida(formulario.value.hora)) {
        nuevosErrores.hora = 'Horario de atención: 8:00 AM - 12:00 PM y 2:00 PM - 6:00 PM.'
      } else if (formulario.value.fecha === fechaHoy()) {
        const minutosElegidos = minutosDesdeMedianoche(formulario.value.hora)
        const minutosActuales = minutosAhora()

        if (minutosElegidos < minutosActuales) {
          nuevosErrores.hora = 'Hora pasada, elija una hora válida.'
        } else if (minutosElegidos < minutosActuales + 60) {
          nuevosErrores.hora = 'La cita debe hacerse mínimo una hora después de la actual.'
        }
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
      if (formulario.value.estadoPago === 'abonado') {
        if (!formulario.value.cantidadAbonada || formulario.value.cantidadAbonada <= 0) {
          nuevosErrores.cantidadAbonada = 'Ingresa la cantidad abonada.'
        } else if (formulario.value.cantidadAbonada >= formulario.value.precio) {
          nuevosErrores.cantidadAbonada = 'El abono debe ser menor al precio total.'
        }
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

      const datosAGuardar = { ...formulario.value }
      if (datosAGuardar.estadoPago !== 'abonado') {
        datosAGuardar.cantidadAbonada = null
      }

      if (modoEdicion.value) {
        const index = servicios.value.findIndex(s => s.id === idEditando.value)
        if (index !== -1) {
          servicios.value[index] = { ...servicios.value[index], ...datosAGuardar }
        }
      } else {
        servicios.value.push({
          ...datosAGuardar,
          id: Date.now(),
          calificacion: 0,
          observaciones: ''
        })
      }

      setTimeout(() => {
        guardando.value = false
      }, 2000)
    }

    function abrirModalCalificacion(id) {
      const servicio = servicios.value.find(s => s.id === id)
      idParaCalificar.value = id
      formularioCalificacion.value = {
        calificacion: servicio && servicio.calificacion > 0 ? servicio.calificacion : 5,
        observaciones: servicio ? servicio.observaciones : ''
      }
      mostrarModalCalificacion.value = true
    }

    function cerrarModalCalificacion() {
      mostrarModalCalificacion.value = false
      idParaCalificar.value = null
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

    function formatearNumero(valor) {
      if (!valor) return ''
      const numero = Math.round(Number(valor))
      return numero.toString().replace(/\B(?=(\d{3})+(?!\d))/g, '.')
    }

    function formatearPrecio(valor) {
      if (!valor) return '$0'
      return '$' + Math.round(Number(valor)).toString().replace(/\B(?=(\d{3})+(?!\d))/g, '.')
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
        for (const tipo of (s.servicios || [])) {
          conteo[tipo] = (conteo[tipo] || 0) + 1
        }
      }

      let masPedido = '—'
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
      return servicios.value.filter(s => s.estadoPago === 'pendiente' || s.estadoPago === 'abonado').length
    }

    function obtenerAbonados() {
      return servicios.value.filter(s => s.estadoPago === 'abonado')
    }

    return {
      cargando,
      catalogoServicios,
      barberos,
      servicios,
      mostrarModal,
      modoEdicion,
      mostrarListaServicios,
      formulario,
      errores,
      guardando,
      mostrarModalCalificacion,
      formularioCalificacion,
      mostrarConfirmacion,
      fechaHoy,
      horaMinimaCampo,
      formatearNumero,
      actualizarPrecio,
      seleccionarServicio,
      actualizarCampoMoneda,
      abrirModalNuevo,
      abrirModalEditar,
      cerrarModal,
      guardarServicio,
      abrirModalCalificacion,
      cerrarModalCalificacion,
      guardarCalificacion,
      pedirConfirmacionEliminar,
      cancelarEliminar,
      confirmarEliminar,
      formatearPrecio,
      totalSemana,
      servicioMasPedido,
      contarPendientes,
      obtenerAbonados
    }
  }
}
</script>

<style>
* { box-sizing: border-box; }

body {
  margin: 0;
  font-family: Georgia, "Times New Roman", serif;
  background: #f3e7d3;
  color: #3b2412;
  padding: 16px;
  padding-bottom: 90px;
  font-size: 16px;
}

.pantalla-carga {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 14px;
  color: #8a6a4a;
  font-size: 16px;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #e3d2b4;
  border-top: 4px solid #7a4a26;
  border-radius: 50%;
  animation: girar 0.8s linear infinite;
  margin: 0 auto 10px;
}

@keyframes girar {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

header {
  background: #3b2412;
  color: #f3e7d3;
  padding: 20px 16px;
  text-align: left;
  border-bottom: 4px solid #a97449;
  margin: -16px -16px 20px -16px;
}

header h1 {
  margin: 0;
  font-size: 22px;
  font-weight: 700;
  color: #f3e7d3;
  letter-spacing: 0.5px;
}

.resumen {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  margin-bottom: 20px;
}

.resumen .tarjeta-resumen {
  flex: 1;
  min-width: 130px;
  background: #ece0c8;
  border: 1px solid #d9c4a0;
  border-radius: 12px;
  padding: 14px 10px;
  text-align: center;
}

.resumen .valor {
  font-size: 18px;
  font-weight: 700;
  color: #7a4a26;
}

.resumen .etiqueta {
  font-size: 12.5px;
  color: #8a6a4a;
  margin-top: 3px;
}

.seccion-abonados {
  margin-bottom: 22px;
}

.titulo-seccion {
  font-size: 16px;
  font-weight: 700;
  color: #3b2412;
  margin: 0 0 10px;
}

.grilla-abonados {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
}

.abonado-card {
  background: #ecdcbf;
  border: 1px solid #c9a15a;
  border-radius: 14px;
  padding: 14px 16px;
  width: 260px;
}

.abonado-cliente {
  font-weight: 700;
  font-size: 15.5px;
  margin-bottom: 10px;
  color: #3b2412;
}

.abonado-fila {
  display: flex;
  justify-content: space-between;
  gap: 8px;
  text-align: center;
}

.abonado-dato {
  flex: 1;
}

.abonado-valor {
  font-size: 15.5px;
  font-weight: 700;
  color: #3b2412;
}

.abonado-valor.abonado-color { color: #a05a1c; }
.abonado-valor.falta-color { color: #9a3324; }

.abonado-etiqueta {
  font-size: 12px;
  color: #8a6a4a;
  margin-top: 2px;
}

.grilla-servicios {
  display: flex;
  flex-wrap: wrap;
  gap: 14px;
}

.servicio-card {
  background: #fbf4e6;
  border: 1px solid #d9c4a0;
  border-radius: 14px;
  padding: 16px;
  box-shadow: 0 4px 14px rgba(59,36,18,0.15);
  width: 300px;
  min-height: 300px;
}

.servicio-card.pendiente {
  border-color: #c9a15a;
  background: #fdf3e0;
}

.servicio-card.abonado {
  border-color: #a97449;
  background: #f6ead4;
}

.fila-superior {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 10px;
}

.servicio-card .cliente {
  font-weight: 700;
  font-size: 18px;
  color: #3b2412;
}

.servicio-card .precio {
  font-weight: 700;
  font-size: 18px;
  color: #7a4a26;
}

.badge {
  display: inline-block;
  font-size: 12px;
  padding: 3px 10px;
  border-radius: 20px;
  margin-top: 5px;
  font-weight: 600;
}

.badge.pagado { background: #dbe7c9; color: #4d6b2e; }
.badge.pendiente { background: #f3ddb0; color: #8a5a15; }
.badge.abonado { background: #e6d1a8; color: #7a4a1e; }

.chips-servicios {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 10px;
}

.chip {
  background: #ecdcbf;
  color: #6b4423;
  font-size: 13px;
  font-weight: 600;
  padding: 4px 10px;
  border-radius: 20px;
}

.fila-info {
  display: flex;
  flex-wrap: wrap;
  gap: 14px;
  margin-bottom: 6px;
}

.info-item {
  font-size: 14.5px;
  color: #6b4f37;
}

.zona-calificacion {
  margin-top: 10px;
}

.fila-calificacion {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.estrellas {
  color: #e3d2b4;
  font-size: 17px;
}

.estrellas .llena {
  color: #b8860b;
}

.estrellas.baja .llena {
  color: #9a3324;
}

.btn-link {
  background: none;
  border: none;
  color: #7a4a26;
  font-size: 13px;
  text-decoration: underline;
  cursor: pointer;
  padding: 0;
}

.btn-calificar {
  background: #ecdcbf;
  color: #6b4423;
  border: none;
  border-radius: 8px;
  padding: 8px 12px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.observaciones {
  font-size: 14px;
  color: #6b4f37;
  font-style: italic;
  margin-top: 10px;
  background: #f3e7d3;
  padding: 8px 10px;
  border-radius: 8px;
}

.acciones-card {
  display: flex;
  gap: 10px;
  margin-top: 14px;
}

.acciones-card button {
  flex: 1;
  padding: 10px;
  border: none;
  border-radius: 8px;
  font-size: 14.5px;
  font-weight: 600;
  cursor: pointer;
}

.btn-editar { background: #ece0c8; color: #3b2412; }
.btn-eliminar { background: #f3ddd3; color: #9a3324; }

.vacio {
  text-align: center;
  color: #a4876a;
  padding: 30px 10px;
  font-size: 15px;
}

.btn-flotante {
  position: fixed;
  bottom: 20px;
  right: 20px;
  width: 58px;
  height: 58px;
  border-radius: 50%;
  background: #3b2412;
  color: #f3e7d3;
  font-size: 28px;
  border: none;
  box-shadow: 0 4px 14px rgba(59,36,18,0.4);
  cursor: pointer;
}

.fondo-modal {
  position: fixed;
  inset: 0;
  background: rgba(59,36,18,0.55);
  display: flex;
  align-items: flex-end;
  justify-content: center;
  z-index: 10;
}

.caja-modal {
  background: #fbf4e6;
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
  font-size: 19px;
  font-weight: 700;
  color: #3b2412;
}

.campo {
  margin-bottom: 14px;
  flex: 1;
}

.campo-doble {
  display: flex;
  gap: 10px;
}

.campo label {
  display: block;
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 5px;
  color: #5a3d24;
}

.campo input,
.campo select,
.campo textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #d9c4a0;
  border-radius: 8px;
  font-size: 15px;
  background: #fffaf0;
  color: #3b2412;
}

.campo input:focus,
.campo select:focus,
.campo textarea:focus {
  outline: none;
  border-color: #7a4a26;
}

.dropdown-servicios {
  border: 1px solid #d9c4a0;
  border-radius: 8px;
  overflow: hidden;
}

.dropdown-cabecera {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  font-size: 15px;
  cursor: pointer;
  background: #fffaf0;
}

.dropdown-cabecera .placeholder {
  color: #a4876a;
}

.dropdown-cabecera .flecha {
  color: #8a6a4a;
  font-size: 12px;
  margin-left: 8px;
}

.dropdown-opciones {
  border-top: 1px solid #e3d2b4;
  padding: 10px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  background: #f3e7d3;
}

.check-servicio {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14.5px;
  font-weight: normal;
  color: #3b2412;
}

.check-servicio input {
  width: 18px;
  height: 18px;
  accent-color: #7a4a26;
}

.error {
  color: #9a3324;
  font-size: 12.5px;
  margin-top: 4px;
}

.selector-estrellas {
  font-size: 28px;
  cursor: pointer;
}

.selector-estrellas span {
  color: #e3d2b4;
  margin-right: 5px;
}

.selector-estrellas span.activa {
  color: #b8860b;
}

.botones-modal {
  display: flex;
  gap: 10px;
  margin-top: 16px;
}

.botones-modal button {
  flex: 1;
  padding: 13px;
  border: none;
  border-radius: 8px;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
}

.btn-cancelar { background: #ece0c8; color: #3b2412; }
.btn-guardar { background: #3b2412; color: #f3e7d3; }

.caja-confirmacion {
  background: #fbf4e6;
  border-radius: 12px;
  padding: 24px;
  max-width: 320px;
  text-align: center;
  font-size: 15px;
}

.caja-confirmacion p {
  margin-bottom: 20px;
  color: #3b2412;
}

</style>