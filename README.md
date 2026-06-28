# NefroKids Web — Unity WebGL + Firebase Hosting

🌐 **Demo en vivo:** [https://nefrokids-web-1d52f.web.app/](https://nefrokids-web-1d52f.web.app/)
🎮 **Proyecto Unity (fuente):** [NefroKids_Niv1](https://github.com/Julieta-8/NefroKids_Niv1)

Pipeline para exportar un build de Unity (WebGL) y desplegarlo en Firebase Hosting.

---

## Requisitos

- Unity con soporte WebGL instalado
- Node.js (LTS recomendado)
- Firebase CLI:
  ```bash
  npm install -g firebase-tools
  ```
- Cuenta de Firebase activa

---

## 1. Exportar desde Unity (WebGL)

El proyecto Unity que se compila para este deploy vive en:
👉 [https://github.com/Julieta-8/NefroKids_Niv1](https://github.com/Julieta-8/NefroKids_Niv1)

```bash
git clone https://github.com/Julieta-8/NefroKids_Niv1.git
```

Abrirlo desde Unity Hub antes de continuar.

### 1.1 Configurar plataforma

1. Ir a **File → Build Settings**
2. Seleccionar **WebGL**
3. Click en **Switch Platform**

### 1.2 Configuración recomendada (Player Settings)

- **Compression Format:** Disabled o Gzip (según hosting)
- **Publishing Settings:** dejar default en primera iteración
- **WebGL Template:** Default o custom si ya existe

### 1.3 Generar build

1. Click en **Build**
2. Seleccionar carpeta de salida
3. Unity genera:
   ```
   Build/
   TemplateData/
   index.html
   ```

---

## 2. Estructura del proyecto

```
NefroKids-web/
  public/
    index.html
    Build/
    TemplateData/
  firebase.json
  .firebaserc
```

---

## 3. Configuración de Firebase

### 3.1 Login

```bash
firebase login
```

Este comando abre el flujo de autenticación en el navegador y conecta la CLI con la cuenta de Firebase.

> ⚠️ **Problemas de red / certificados (entornos corporativos)**
>
> En redes institucionales o PCs administradas, el login puede fallar por inspección de tráfico HTTPS o certificados intermedios no confiables.
>
> **Errores típicos:**
> - `self-signed certificate in certificate chain`
> - `SELF_SIGNED_CERT_IN_CHAIN`
> - `Failed to list Firebase projects`
>
> Esto ocurre porque Node.js no confía en el certificado insertado por un proxy o antivirus corporativo, bloqueando las llamadas HTTPS a Firebase. No es un problema del proyecto ni de Firebase.
>
> **Solución temporal (solo para diagnóstico)** — en CMD de Windows:
> ```bash
> set NODE_TLS_REJECT_UNAUTHORIZED=0
> ```
> Luego ejecutar nuevamente `firebase login`, `firebase init` o `firebase projects:list`.
>
> ⚠️ Este bypass es solo para diagnóstico o entornos controlados. Si funciona con esta variable activa, confirma que el problema es la red/proxy del sistema. Si el problema persiste, usar otra red o hotspot.

### 3.2 Inicializar hosting

Dentro del repo:

```bash
firebase init hosting
```

Opciones recomendadas:

- **Use existing project** → seleccionar proyecto Firebase
- **Public directory** → `public`
- **SPA?** → No (Unity no lo requiere en esta etapa)
- **GitHub deploy?** → No (por ahora)

---

## 4. Preparar build para deploy

Cada vez que Unity genera un build:

1. Copiar contenido del build a:
   ```
   public/
     index.html
     Build/
     TemplateData/
   ```
2. Asegurarse de que `index.html` es el generado por Unity, **no** el del template de Firebase.

---

## 5. Comunicación React Native → Unity WebGL

### ¿Por qué es necesario?

Cada nuevo build WebGL genera un `index.html` que inicializa el juego mediante `createUnityInstance()`. La instancia resultante (`unityInstance`) queda encapsulada dentro de ese archivo y React Native no puede acceder a ella directamente.

Para que la app pueda enviar mensajes al juego (iniciar un nivel, pasar datos del paciente, etc.) es necesario exponer una función global de JavaScript que actúe como puente. Esa función es `window.receiveFromReact`, que recibe un JSON desde el `WebView` vía `injectJavaScript()` y lo reenvía a Unity usando `unityInstance.SendMessage()`.

### Código a agregar en `index.html`

Dentro del `index.html` generado por Unity, localizar el bloque `script.onload` y reemplazarlo por el siguiente (el fragmento nuevo está marcado con el comentario `React Native -> Unity`):

```html
script.onload = () => {
  createUnityInstance(canvas, config, (progress) => {
    progressBarFull.style.width = 100 * progress + "%";
  })
    .then((unityInstance) => {
      loadingBar.style.display = "none";

      fullscreenButton.onclick = () => {
        unityInstance.SetFullscreen(1);
      };

      // React Native -> Unity
      console.log("Unity creada");
      console.log("ReactNativeWebView:", window.ReactNativeWebView);
      window.receiveFromReact = function (json) {
        unityInstance.SendMessage(
          "ReactConnection", // Nombre del GameObject en la jerarquía
          "Receive",         // Método del script ReactConnection
          json
        );
      };
    })
    .catch((message) => {
      alert(message);
    });
};
```

### Flujo completo

```
React Native
      │
injectJavaScript()
      │
      ▼
window.receiveFromReact(json)
      │
      ▼
unityInstance.SendMessage(...)
      │
      ▼
ReactConnection.Receive()
      │
      ▼
OnMessageReceived
      │
      ▼
LevelManager / otros managers
```

### Notas importantes

- El primer parámetro de `SendMessage` (`"ReactConnection"`) es el **nombre del GameObject** en la jerarquía de Unity, no el nombre del script. Si el GameObject cambia de nombre, este valor debe actualizarse.
- `createUnityInstance()` solo devuelve la instancia cuando el juego terminó de cargar completamente; el registro de `window.receiveFromReact` ocurre dentro del `.then()` por esa razón.

### Mantenimiento

> ⚠️ Este bloque **no forma parte del código generado automáticamente por Unity**. Cada nuevo build WebGL sobrescribe el `index.html`, por lo que debe volver a agregarse manualmente.
>
> **Mejora futura:** usar un WebGL Template personalizado para incorporar esta lógica al template y que todos los builds la incluyan de forma automática.

---

## 6. Deploy a Firebase Hosting

```bash
firebase deploy --only hosting
```

Firebase devuelve una URL como:

```
https://<project-id>.web.app
```

---

## 7. Flujo de trabajo recomendado

```
Unity build WebGL
        │
        ▼
Copiar a public/
(index.html + Build/ + TemplateData/)
        │
        ▼
Agregar puente JS en index.html
(window.receiveFromReact dentro del .then())
        │
        ▼
firebase deploy --only hosting
        │
        ▼
Test en navegador
        │
        ▼
Repetir
```

> ⚠️ El paso de agregar el puente JS debe repetirse en cada build nuevo, ya que Unity sobrescribe el `index.html` automáticamente.

---

## 8. Troubleshooting

| Síntoma | Causa probable |
|---|---|
| Pantalla negra en WebGL | Build mal copiado, falta carpeta `Build/`, problemas con `.wasm` |
| 404 en archivos Build | Estructura incorrecta en `public/`, assets fuera del directorio correcto |
| WebGL no carga en móvil | Compression incompatible, problemas de memoria, CORS o headers incorrectos |
| Firebase CLI no funciona | Error de certificados (`SELF_SIGNED_CERT_IN_CHAIN`), proxy corporativo o inspección HTTPS |

---

## 9. Objetivo de esta configuración

Este pipeline se usa para:

- Validar Unity WebGL en entorno real
- Probar integración con React (si aplica)
- Testear performance en navegador móvil
- Preparar futura integración React ↔ Unity