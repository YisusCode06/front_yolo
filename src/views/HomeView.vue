<template>
  <div id="app" class="container">
    <h1 class="text-center my-2">Detección de Objetos con YOLOv8</h1>

    <!-- Selección de cámaras -->
    <div class="row justify-content-center mb-4">
      <div class="col-md-6">
        <div class="input-group">
          <select class="form-select m-2" v-model="selectedCamera" @change="changeCamera">
            <option v-for="device in videoDevices" :key="device.deviceId" :value="device.deviceId">
              {{ device.label || `Cámara ${device.deviceId}` }}
            </option>
          </select>
          <button class="btn btn-primary m-2" @click="startProcessing" :disabled="isDetecting">Iniciar Detección</button>
          <button class="btn btn-danger m-2" @click="stopProcessing" :disabled="!isDetecting">Detener Detección</button>
        </div>
      </div>
    </div>

    <!-- Video y canvas -->
    <div class="row">
      <div class="col-md-6">
        <h2>Entrada:</h2>
        <video ref="video" autoplay playsinline class="border w-100 mb-3"></video>
      </div>
      <div class="col-md-6">
        <h2>Resultado Procesado:</h2>
        <canvas ref="canvas" class="border w-100 mb-3" style="display: none;"></canvas>
        <div v-if="processedImage">
          <img :src="processedImage" alt="Processed Image" class="border w-100 mb-3" />
        </div>
      </div>
      <div>
        <h3>Conteo de Plátanos:</h3>
        <ul>
          <li v-for="(count, className) in bananaCounts" :key="className">
            {{ className }}: {{ count }}
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client'; // Importar socket.io-client

export default {
  data() {
    return {
      processedImage: null, // Guardar la imagen procesada devuelta por el backend
      intervalId: null, // ID del intervalo para enviar fotogramas al backend
      isDetecting: false, // Estado para saber si se está detectando
      videoDevices: [], // Lista de dispositivos de video (cámaras)
      selectedCamera: null, // Cámara seleccionada por el usuario
      bananaCounts: { // Contador de plátanos
        'Mal estado': 0,
        'Apto para chifles': 0,
        'No apto para chifles': 0
      },
      socket: null // Conexión de WebSocket
    };
  },
  methods: {
    // Iniciar la detección
    startProcessing() {
      if (!this.isDetecting) {
        this.setupCamera();
        this.isDetecting = true;
      }
    },
    // Detener la detección
    stopProcessing() {
      if (this.isDetecting) {
        clearInterval(this.intervalId); // Detener el envío de fotogramas
        this.isDetecting = false;
      }
    },
    // Configurar la cámara para capturar el video
    async setupCamera() {
      const video = this.$refs.video;

      try {
        const stream = await navigator.mediaDevices.getUserMedia({
          video: { deviceId: this.selectedCamera ? { exact: this.selectedCamera } : undefined },
        });
        video.srcObject = stream;

        // Procesar un fotograma cada 1 segundo
        this.intervalId = setInterval(this.captureFrame, 1000);
      } catch (error) {
        console.error("Error accediendo a la cámara:", error);
      }
    },
    // Capturar un fotograma del video y enviarlo al backend
    captureFrame() {
      const canvas = this.$refs.canvas;
      const video = this.$refs.video;
      const context = canvas.getContext("2d");

      // Establecer el tamaño del canvas
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;

      // Dibujar el fotograma actual en el canvas
      context.drawImage(video, 0, 0, canvas.width, canvas.height);

      // Convertir el fotograma a base64
      canvas.toBlob((blob) => {
        const reader = new FileReader();
        reader.onloadend = () => {
          const base64Image = reader.result.split(',')[1]; // Obtener solo la parte de la imagen base64
          this.sendFrame(base64Image); // Enviar el fotograma como base64
        };
        reader.readAsDataURL(blob);
      }, "image/jpeg");
    },
    // Enviar el fotograma al backend mediante WebSockets
    sendFrame(base64Image) {
      this.socket.emit('process_image', { image: base64Image });
    },
    // Obtener la lista de cámaras disponibles
    async getCameras() {
      try {
        const devices = await navigator.mediaDevices.enumerateDevices();
        this.videoDevices = devices.filter((device) => device.kind === "videoinput");
        if (this.videoDevices.length > 0 && !this.selectedCamera) {
          this.selectedCamera = this.videoDevices[0].deviceId; // Seleccionar la primera cámara por defecto
        }
      } catch (error) {
        console.error("Error obteniendo cámaras:", error);
      }
    },
    // Cambiar de cámara
    changeCamera() {
      this.stopProcessing();
      this.startProcessing();
    },
  },
  mounted() {
    this.getCameras(); // Cargar las cámaras al montar el componente

    // Conectar al servidor de WebSockets
    this.socket = io('http://localhost:5000'); // Asegúrate de que la URL apunte a tu backend

    // Escuchar el evento de imagen procesada desde el backend
    this.socket.on('image_processed', (data) => {
      if (data.image) {
        this.processedImage = `data:image/jpeg;base64,${data.image}`;
      }

      if (data.banana_counts) {
        this.bananaCounts = data.banana_counts;
      }
    });

    // Manejar errores del servidor
    this.socket.on('error', (error) => {
      console.error("Error desde el servidor:", error);
    });
  },
  beforeDestroy() {
    // Limpiar el intervalo si el componente es destruido o si se detiene la detección
    this.stopProcessing();

    // Desconectar el socket cuando el componente se destruye
    if (this.socket) {
      this.socket.disconnect();
    }
  },
};
</script>



<style>
video,
canvas {
  border: 2px solid #000;
  border-radius: 8px;
  width: 100%;
  height: auto;
}

button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  background-color: #cccccc44;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
}

h1 {
  font-weight: 700;
  color: #34495e;
  font-size: 2.5rem;
  margin-bottom: 40px;
}

h2 {
  font-size: 1.8rem;
  color: #34495e;
  margin-bottom: 20px;
}

select {
  background-color: #ecf0f1;
  color: #2c3e50;
}

button {
  font-weight: 500;
  transition: background-color 0.3s ease;
}

button.btn-primary {
  background-color: #3498db;
  color: #fff;
}

button.btn-danger {
  background-color: #e74c3c;
  color: #fff;
}

button:hover {
  background-color: #2980b9;
  color: #fff;
}

button.btn-danger:hover {
  background-color: #c0392b;
}
</style>
