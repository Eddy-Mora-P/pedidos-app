<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Tab 3</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content :fullscreen="true">
      <form class="ion-padding" @submit.prevent="guardarPost">
        <ion-item>
          <ion-label position="stacked">Título</ion-label>
          <ion-input v-model="nuevoPost.title" name="title" required :maxlength="100" />
        </ion-item>

        <ion-item>
          <ion-label position="stacked">Body</ion-label>
          <ion-textarea v-model="nuevoPost.body" name="body" required :auto-grow="true" />
        </ion-item>

        <ion-item>
          <ion-label position="stacked">Usuario</ion-label>
          <ion-select
            v-model="nuevoPost.userId"
            name="userId"
            placeholder="Selecciona un usuario"
            :disabled="cargandoUsuarios || usuarios.length === 0"
            required
          >
            <ion-select-option v-for="usuario in usuarios" :key="usuario.id" :value="usuario.id">
              {{ usuario.name }}
            </ion-select-option>
          </ion-select>
        </ion-item>

        <ion-text v-if="error" color="danger" role="alert">
          <p>{{ error }}</p>
        </ion-text>
        <ion-text v-if="mensaje" color="success" role="status">
          <p>{{ mensaje }}</p>
        </ion-text>

        <ion-button
          class="ion-margin-top"
          expand="block"
          type="submit"
          :disabled="guardando || cargandoUsuarios || usuarios.length === 0"
        >
          {{ guardando ? 'Guardando...' : 'Guardar post' }}
        </ion-button>
        <ion-button
          class="ion-margin-top"
          expand="block"
          fill="outline"
          type="button"
          :disabled="guardando"
          @click="escanearQr"
        >
          Escanear QR
        </ion-button>
      </form>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  CapacitorBarcodeScanner,
  CapacitorBarcodeScannerCameraDirection,
  CapacitorBarcodeScannerScanOrientation,
  CapacitorBarcodeScannerTypeHint,
} from '@capacitor/barcode-scanner';
import {
  IonButton,
  IonContent,
  IonHeader,
  IonInput,
  IonItem,
  IonLabel,
  IonPage,
  IonSelect,
  IonSelectOption,
  IonTextarea,
  IonText,
  IonTitle,
  IonToolbar,
} from '@ionic/vue';
import { Posts } from '@/interfaces/posts';
import { Usuarios } from '@/interfaces/usuarios';
import { onMounted, reactive, ref } from 'vue';

interface FormularioPost {
  title: string;
  body: string;
  userId: number | null;
}

const usuarios = ref<Usuarios[]>([]);
const nuevoPost = reactive<FormularioPost>({
  title: '',
  body: '',
  userId: null,
});
const cargandoUsuarios = ref(false);
const guardando = ref(false);
const error = ref('');
const mensaje = ref('');

async function cargarUsuario() {
  cargandoUsuarios.value = true;
  error.value = '';

  try {
    const response = await fetch('https://jsonplaceholder.typicode.com/users');
    if (!response.ok) {
      throw new Error('No se pudo cargar la lista de usuarios.');
    }

    usuarios.value = (await response.json()) as Usuarios[];
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Ocurrió un error al cargar usuarios.';
  } finally {
    cargandoUsuarios.value = false;
  }
}

async function guardarPost() {
  if (!nuevoPost.title.trim() || !nuevoPost.body.trim() || nuevoPost.userId === null) {
    error.value = 'Completa todos los campos antes de guardar.';
    mensaje.value = '';
    return;
  }

  guardando.value = true;
  error.value = '';
  mensaje.value = '';

  try {
    const response = await fetch('https://jsonplaceholder.typicode.com/posts', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        title: nuevoPost.title.trim(),
        body: nuevoPost.body.trim(),
        userId: nuevoPost.userId,
      }),
    });

    if (!response.ok) {
      throw new Error('No se pudo guardar el post.');
    }

    const postGuardado = (await response.json()) as Posts;
    mensaje.value = `Post guardado correctamente (ID: ${postGuardado.id}).`;
    nuevoPost.title = '';
    nuevoPost.body = '';
    nuevoPost.userId = null;
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Ocurrió un error al guardar el post.';
  } finally {
    guardando.value = false;
  }
}

async function escanearQr() {
  error.value = '';
  mensaje.value = '';

  try {
    const result = await CapacitorBarcodeScanner.scanBarcode({
      hint: CapacitorBarcodeScannerTypeHint.QR_CODE,
      cameraDirection: CapacitorBarcodeScannerCameraDirection.BACK,
      scanOrientation: CapacitorBarcodeScannerScanOrientation.ADAPTIVE,
      scanInstructions: 'Escanea un QR con JSON: title y body',
    });
    const scannedData: unknown = JSON.parse(result.ScanResult);

    if (typeof scannedData !== 'object' || scannedData === null || Array.isArray(scannedData)) {
      throw new Error('El QR debe contener un objeto JSON con título y contenido.');
    }

    const data = scannedData as Record<string, unknown>;
    if (typeof data.title !== 'string' || typeof data.body !== 'string') {
      throw new Error('El QR debe incluir los campos title y body.');
    }

    const title = data.title.trim();
    const body = data.body.trim();
    if (!title || !body) {
      throw new Error('El QR debe incluir un título y un contenido no vacíos.');
    }

    nuevoPost.title = title;
    nuevoPost.body = body;
    mensaje.value = 'Título y contenido cargados desde el QR.';
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'No se pudo leer el código QR.';
  }
}

onMounted(cargarUsuario);
</script>
