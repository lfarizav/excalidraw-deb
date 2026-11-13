# Cómo empaquetar e instalar Excalidraw (.deb) en Ubuntu 24.04 LTS

Excalidraw no distribuye un paquete `.deb` oficial, ya que está diseñado como una Progressive Web App (PWA) con soporte *offline*. Si deseas empaquetarlo como una aplicación de escritorio nativa en **Ubuntu 24.04 (Noble Numbat)** usando Electron, es necesario resolver tres particularidades de esta versión del sistema:

1. **Metadatos estrictos en `electron-builder`:** Exige definir `homepage`, `author` y `maintainer` para construir paquetes Debian.
2. **Resolución del ícono:** Linux requiere que el ícono tenga al menos `256x256` píxeles (el ícono web por defecto es `180x180`).
3. **Restricción de Sandbox / AppArmor en Ubuntu 24.04:** Ubuntu 24.04 bloquea los espacios de nombres de usuario sin privilegios, lo que provoca un cierre inesperado (`Trace/breakpoint trap`) si el binario `chrome-sandbox` no tiene permisos SUID (`4755`) pertenecientes a `root`.

---

## Requisitos previos

Asegúrate de tener instalados **Node.js** (v18 o v20+), **npm**, **curl** y **Python 3**:

```bash
node -v && npm -v && python3 --version
```

*(Si no tienes Node.js instalado, puedes instalarlo desde los repositorios oficiales o mediante NodeSource).*

---

## Paso 1: Crear el directorio del proyecto y preparar el ícono (512×512)

Crea una carpeta temporal de trabajo, descarga el ícono oficial de Excalidraw y redimensiónalo a `512x512` píxeles para cumplir con las especificaciones de paquetes `.deb`:

```bash
mkdir -p ~/excalidraw-deb/build
cd ~/excalidraw-deb

curl -fsSL https://excalidraw.com/apple-touch-icon.png -o build/icon.png

python3 -c '
try:
    from PIL import Image
    Image.open("build/icon.png").resize((512, 512)).save("build/icon.png")
except Exception:
    import struct, zlib
    w = h = 512
    raw = b"".join(b"\x00" + b"\x69\x65\xdb" * w for _ in range(h))
    chunk = lambda t, d: struct.pack("!I", len(d)) + t + d + struct.pack("!I", zlib.crc32(t + d) & 0xffffffff)
    open("build/icon.png", "wb").write(b"\x89PNG\r\n\x1a\n" + chunk(b"IHDR", struct.pack("!IIBBBBB", w, h, 8, 2, 0, 0, 0)) + chunk(b"IDAT", zlib.compress(raw)) + chunk(b"IEND", b""))
'
```

---

## Paso 2: Configurar la aplicación Electron (`main.js` y `package.json`)

Crea el archivo principal `main.js` (que abre enlaces externos en tu navegador predeterminado y mantiene el lienzo de Excalidraw en una ventana limpia sin barra de menú):

```bash
cat << 'EOF' > main.js
const { app, BrowserWindow, shell } = require('electron');

function createWindow() {
  const win = new BrowserWindow({
    width: 1280,
    height: 800,
    title: 'Excalidraw',
    autoHideMenuBar: true,
    webPreferences: {
      contextIsolation: true,
      nodeIntegration: false
    }
  });

  win.loadURL('https://excalidraw.com');

  win.webContents.setWindowOpenHandler(({ url }) => {
    if (!url.startsWith('https://excalidraw.com')) {
      shell.openExternal(url);
      return { action: 'deny' };
    }
    return { action: 'allow' };
  });
}

app.whenReady().then(createWindow);
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});
EOF
```

Ahora crea el archivo `package.json` con todos los metadatos obligatorios para generar el `.deb` en Ubuntu 24.04:

```bash
cat << 'EOF' > package.json
{
  "name": "excalidraw-desktop",
  "productName": "Excalidraw",
  "version": "1.0.0",
  "description": "Excalidraw Desktop App",
  "homepage": "https://excalidraw.com",
  "author": "Excalidraw Desktop <desktop@excalidraw.local>",
  "main": "main.js",
  "scripts": {
    "build": "electron-builder --linux deb"
  },
  "build": {
    "appId": "com.excalidraw.desktop",
    "linux": {
      "target": "deb",
      "category": "Graphics",
      "icon": "build/icon.png",
      "maintainer": "Excalidraw Desktop <desktop@excalidraw.local>"
    }
  },
  "devDependencies": {
    "electron": "^33.0.0",
    "electron-builder": "^25.1.8"
  }
}
EOF
```

---

## Paso 3: Compilar el paquete `.deb`

Instala las dependencias de compilación y genera el instalador:

```bash
npm install
npm run build
```

Al finalizar, encontrarás el paquete generado en `dist/excalidraw-desktop_1.0.0_amd64.deb`.

---

## Paso 4: Instalar el `.deb` y configurar `chrome-sandbox`

Instala el paquete mediante `apt` (evita el instalador gráfico de Ubuntu App Center, ya que suele fallar con paquetes `.deb` locales) y asigna los permisos SUID necesarios para Ubuntu 24.04:

```bash
sudo apt install ./dist/excalidraw-desktop_1.0.0_amd64.deb

sudo chown root:root /opt/Excalidraw/chrome-sandbox
sudo chmod 4755 /opt/Excalidraw/chrome-sandbox
```

> **¿Por qué son necesarios los comandos `chown` y `chmod`?**  
> Sin ellos, al iniciar la aplicación verás el error: `FATAL:setuid_sandbox_host.cc: The SUID sandbox helper binary was found, but is not configured correctly`.

---

## Paso 5: Ejecutar y limpiar archivos temporales

Ya puedes abrir **Excalidraw** desde el menú de aplicaciones de Ubuntu (tecla `Super` -> escribir `Excalidraw`) o desde la terminal:

```bash
excalidraw-desktop &
```

*(Nota: Tras la primera apertura, la aplicación guarda sus recursos en caché y funciona completamente sin conexión a internet).*

Si deseas conservar el instalador `.deb` para otros equipos con Ubuntu 24.04, muévelo antes de borrar la carpeta temporal de compilación:

```bash
mv dist/excalidraw-desktop_1.0.0_amd64.deb ~/Descargas/
rm -rf ~/excalidraw-deb
```

---

## Alternativa rápida (sin compilar)

Si prefieres no instalar Electron ni Node.js, puedes instalar Excalidraw como PWA directamente desde **Google Chrome**, **Chromium** o **Brave**:

1. Entra a [https://excalidraw.com](https://excalidraw.com).
2. Haz clic en el ícono de **Instalar Excalidraw** en el extremo derecho de la barra de direcciones (o menú `⋮` → *Guardar y compartir* → *Instalar página como aplicación...*).
3. Tendrás un acceso directo en el menú de Ubuntu y soporte completo *offline*.
