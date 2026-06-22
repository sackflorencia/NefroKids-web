# NefroKids Web — Unity WebGL + Firebase Hosting

🌐 **Demo en vivo:** [https://nefrokids-web-1d52f.web.app/](https://nefrokids-web-1d52f.web.app/)

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

## 5. Deploy a Firebase Hosting

```bash
firebase deploy --only hosting
```

Firebase devuelve una URL como:

```
https://<project-id>.web.app
```

---

## 6. Flujo de trabajo recomendado

```
Unity build WebGL → copiar a public/ → firebase deploy → test en navegador → repetir
```

---

## 7. Troubleshooting

| Síntoma | Causa probable |
|---|---|
| Pantalla negra en WebGL | Build mal copiado, falta carpeta `Build/`, problemas con `.wasm` |
| 404 en archivos Build | Estructura incorrecta en `public/`, assets fuera del directorio correcto |
| WebGL no carga en móvil | Compression incompatible, problemas de memoria, CORS o headers incorrectos |
| Firebase CLI no funciona | Error de certificados (`SELF_SIGNED_CERT_IN_CHAIN`), proxy corporativo o inspección HTTPS |

---

## 8. Objetivo de esta configuración

Este pipeline se usa para:

- Validar Unity WebGL en entorno real
- Probar integración con React (si aplica)
- Testear performance en navegador móvil
- Preparar futura integración React ↔ Unity