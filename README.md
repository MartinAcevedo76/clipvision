# ClipVision — Análisis de clips de video con IA

Aplicación web para subir videos, recortar clips y obtener análisis estructurados con Gemini AI.
Desarrollada como proyecto final para TECMI.

> ⚠️ **La IA genera borradores revisables. El juicio final siempre es del evaluador humano.**

---

## Índice

1. [Funcionalidades](#funcionalidades)
2. [Arquitectura](#arquitectura)
3. [Stack tecnológico](#stack-tecnológico)
4. [Requisitos](#requisitos)
5. [Configurar Firebase](#configurar-firebase)
6. [Configurar Gemini API](#configurar-gemini-api)
7. [Instalación y ejecución](#instalación-y-ejecución)
8. [Flujo de uso](#flujo-de-uso)
9. [Variables de entorno](#variables-de-entorno)
10. [Estructura del proyecto](#estructura-del-proyecto)
11. [Riesgos éticos y de privacidad](#riesgos-éticos-y-de-privacidad)
12. [Reglas de seguridad](#reglas-de-seguridad)

---

## Funcionalidades

- Autenticación con email/contraseña y Google (Firebase Auth)
- Carga de videos con validación de formato y tamaño
- Reproductor con controles de tiempo
- Selección de inicio y fin de clip en cualquier momento del video
- Recorte de clips en el navegador (ffmpeg.wasm — sin servidor)
- Almacenamiento de videos y clips en Firebase Storage
- Envío de clips a Gemini 1.5 Flash para análisis estructurado
- Resultados en JSON con resumen, eventos, calificaciones y advertencias
- Revisión humana con notas por análisis
- Exportación de análisis en JSON
- Historial de análisis por clip

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTEND                              │
│   React 18 + Vite + TypeScript + Tailwind CSS               │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ Login    │  │ Videos   │  │  Clips   │  │ Analysis  │  │
│  │ /login   │  │ /videos  │  │ /clips   │  │ /analysis │  │
│  └──────────┘  └──────────┘  └──────────┘  └───────────┘  │
│                                                              │
│  ┌────────────────────────────┐  ┌──────────────────────┐  │
│  │    ffmpeg.wasm             │  │   Gemini API Client  │  │
│  │ (recorte en el navegador)  │  │ (análisis de video)  │  │
│  └────────────────────────────┘  └──────────────────────┘  │
└───────────────────────┬─────────────────────────────────────┘
                        │ HTTPS
         ┌──────────────▼──────────────┐
         │         FIREBASE             │
         │                              │
         │  Auth  ─  Firestore          │
         │            ├─ videos         │
         │            ├─ clips          │
         │            └─ ai_clip_reviews│
         │                              │
         │  Storage                     │
         │    ├─ /videos/{uid}/...      │
         │    └─ /clips/{uid}/...       │
         └──────────────────────────────┘
                        │
         ┌──────────────▼──────────────┐
         │        GEMINI API            │
         │  gemini-1.5-flash            │
         │  Google AI Studio            │
         └──────────────────────────────┘
```

---

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Frontend | React 18 + Vite + TypeScript |
| Estilos | Tailwind CSS |
| Routing | React Router v6 |
| Auth | Firebase Authentication |
| Base de datos | Firestore |
| Archivos | Firebase Storage |
| Recorte de video | ffmpeg.wasm (WebAssembly) |
| IA | Gemini 1.5 Flash (Google AI Studio) |
| Deploy | Firebase Hosting / Vercel |

---

## Requisitos

- Node.js 18 o superior
- npm 9 o superior
- Cuenta de Google (para Firebase y Gemini)
- Navegador moderno con soporte a SharedArrayBuffer (Chrome, Edge, Firefox)

---

## Configurar Firebase

1. Ve a [https://console.firebase.google.com](https://console.firebase.google.com)
2. Crea un nuevo proyecto (o usa uno existente)
3. Activa los siguientes servicios:

### Authentication
- Ve a **Authentication → Sign-in method**
- Activa **Email/Password**
- Activa **Google**

### Firestore
- Ve a **Firestore Database → Create database**
- Selecciona modo de producción
- Copia y pega estas reglas en **Reglas**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /videos/{videoId} {
      allow read, write: if request.auth != null
        && request.auth.uid == resource.data.uploadedBy;
      allow create: if request.auth != null;
    }
    match /clips/{clipId} {
      allow read, write: if request.auth != null
        && request.auth.uid == resource.data.createdBy;
      allow create: if request.auth != null;
    }
    match /ai_clip_reviews/{reviewId} {
      allow read, write: if request.auth != null;
    }
  }
}
```

### Storage
- Ve a **Storage → Get started**
- Copia y pega estas reglas:

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /videos/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    match /clips/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

### Obtener credenciales
- Ve a **Configuración del proyecto → Tus apps → Web**
- Registra la app y copia las credenciales al archivo `.env`

---

## Configurar Gemini API

1. Ve a [https://aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey)
2. Crea una API key
3. Cópiala en `VITE_GEMINI_API_KEY` en tu archivo `.env`

> ⚠️ Revisa los límites gratuitos antes de generar muchos análisis. El análisis de video consume más tokens que texto.

---

## Instalación y ejecución

```bash
# 1. Clona el repositorio
git clone https://github.com/tu-usuario/clipvision.git
cd clipvision

# 2. Instala dependencias
npm install

# 3. Crea el archivo .env
cp .env.example .env
# Edita .env con tus credenciales reales

# 4. Inicia el servidor de desarrollo
npm run dev
```

Abre [http://localhost:5173](http://localhost:5173)

> El servidor necesita los headers `Cross-Origin-Opener-Policy` y `Cross-Origin-Embedder-Policy` para que ffmpeg.wasm funcione. Están configurados en `vite.config.ts`.

### Build para producción

```bash
npm run build
npm run preview
```

---

## Flujo de uso

1. **Registro / Login** — crea una cuenta o inicia sesión con Google
2. **Subir video** — arrastra o selecciona un .mp4, .mov o .webm (máx. 500 MB)
3. **Reproductor** — reproduce el video, pausa en el momento que quieras
4. **Recortar clip** — marca inicio y fin, dale un nombre y guarda
5. **Analizar** — en la pantalla de análisis, haz clic en "Analizar con Gemini"
6. **Revisar** — lee el resultado, añade tus notas y marca como revisado
7. **Exportar** — descarga el análisis en JSON si lo necesitas

---

## Variables de entorno

Crea un archivo `.env` en la raíz del proyecto (nunca lo subas a GitHub):

```env
VITE_FIREBASE_API_KEY=...
VITE_FIREBASE_AUTH_DOMAIN=...
VITE_FIREBASE_PROJECT_ID=...
VITE_FIREBASE_STORAGE_BUCKET=...
VITE_FIREBASE_MESSAGING_SENDER_ID=...
VITE_FIREBASE_APP_ID=...

VITE_GEMINI_API_KEY=...
```

Consulta `.env.example` para ver la estructura sin valores reales.

---

## Estructura del proyecto

```
clipvision/
├── public/
├── src/
│   ├── components/
│   │   └── layout/
│   │       └── Layout.tsx          # Sidebar + navegación
│   ├── config/
│   │   └── firebase.ts             # Inicialización de Firebase
│   ├── hooks/
│   │   ├── useAuth.tsx             # Contexto de autenticación
│   │   └── useFFmpeg.ts            # Recorte de video con ffmpeg.wasm
│   ├── lib/
│   │   ├── firestore.ts            # CRUD para videos, clips y reviews
│   │   └── gemini.ts               # Llamadas a la API de Gemini
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── DashboardPage.tsx
│   │   ├── VideosPage.tsx          # Upload + lista
│   │   ├── VideoDetailPage.tsx     # Reproductor + recorte
│   │   ├── ClipsPage.tsx           # Lista de clips
│   │   └── AnalysisPage.tsx        # Análisis con IA
│   ├── types/
│   │   └── index.ts                # Interfaces TypeScript
│   ├── App.tsx                     # Rutas protegidas
│   ├── main.tsx
│   └── index.css
├── .env.example
├── .gitignore
├── index.html
├── package.json
├── tailwind.config.js
├── tsconfig.json
└── vite.config.ts
```

---

## Riesgos éticos y de privacidad

### ¿Qué riesgos existen?

**1. Privacidad de personas en los videos**
Los videos pueden contener imágenes de personas identificables. Si se usan para entrenar modelos, compartirlos sin consentimiento o almacenarlos sin protección, se vulnera la privacidad.

**Mitigación:** Las reglas de Firebase Storage impiden que un usuario vea los videos de otro. Los videos nunca son públicos.

**2. Sobreinterpretación de los resultados de IA**
Gemini puede producir análisis plausibles pero incorrectos. Si un evaluador los acepta sin revisarlos, puede tomar decisiones equivocadas.

**Mitigación:** Toda salida de IA está marcada como "borrador revisable". La pantalla de análisis requiere que el revisor humano añada notas antes de marcar como revisado.

**3. Uso en contextos clínicos o educativos**
Esta aplicación no está validada para uso clínico. No debe usarse para diagnóstico, clasificación de riesgo, ni evaluación definitiva de personas.

**Mitigación:** El prompt enviado a Gemini prohíbe explícitamente el diagnóstico. La UI repite el aviso en cada análisis.

**4. Filtración de datos sensibles**
Las API keys en el frontend pueden exponerse si el repositorio es público o si las DevTools están abiertas.

**Mitigación:** Las keys de Gemini en el frontend son una limitación de esta arquitectura. Para producción real, el análisis debe hacerse en un backend seguro (Cloud Functions, por ejemplo), nunca exponiendo la key al cliente.

**5. Falta de consentimiento**
Los participantes que aparecen en los videos deben haber dado su consentimiento para ser grabados y analizados.

**Mitigación:** Responsabilidad del equipo que opera la aplicación. El sistema no puede verificarlo técnicamente.

### Recomendaciones para producción

- Mover la llamada a Gemini a Cloud Functions (la API key nunca llega al cliente)
- Agregar campo de consentimiento en el metadata del video
- Implementar expiración automática de videos después de N días
- Agregar logs de auditoría en Firestore (quién accedió a qué y cuándo)

---

## Reglas de seguridad (checklist para el equipo)

- [ ] No subir videos reales con datos sensibles sin consentimiento
- [ ] No usar videos de niños, pacientes o familias reales para pruebas
- [ ] No guardar API keys en el frontend en producción
- [ ] Nunca subir `.env` a GitHub
- [ ] No hacer el repositorio público mientras tenga datos reales
- [ ] No compartir URLs de Firebase Storage públicamente
- [ ] No usar IA para diagnóstico, clasificación de riesgo o evaluación definitiva
- [ ] Siempre mostrar el aviso de "borrador revisable" en los análisis
- [ ] Registrar quién subió, recortó y analizó cada clip (implementado en Firestore)

---

## Equipo

Proyecto desarrollado para TECMI — Materia: Desarrollo Web con IA

---

## Licencia

Uso educativo. No usar en producción sin revisión de seguridad.
