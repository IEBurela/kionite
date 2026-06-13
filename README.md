# IEBurela Kinoite CAD OS

Esta es una imagen personalizada (basada en Fedora Kinoite y Universal Blue) optimizada como Workstation Inmutable para flujos de trabajo de CAD (BricsCAD, FreeCAD) y tareas de diseño.

## Instalación (Método Estándar)

Si confías en la red y quieres instalar la imagen de manera rápida (el atajo de la comunidad), puedes hacer un rebase inicial no verificado, y luego saltar al verificado:

```bash
# 1. Rebase inicial (trae las llaves de seguridad al sistema)
sudo rpm-ostree rebase ostree-unverified-registry:ghcr.io/IEBurela/ieburela-kinoite-amd:latest

# 2. Reiniciar
sudo systemctl reboot

# 3. Anclar a la imagen firmada y segura
sudo rpm-ostree rebase ostree-image-signed:docker://ghcr.io/IEBurela/ieburela-kinoite-amd:latest
```

*(Nota: Cambia `-amd` a `-nvidia` si el equipo tiene una GPU Nvidia dedicada).*

---

## 🔒 Instalación Estricta "Zero-Trust" (Recomendado para Producción)

Si eres un purista de la seguridad (Paranoia Level: Expert) y no quieres usar `ostree-unverified-registry` ni por un segundo, puedes inyectar las políticas y llaves de seguridad a mano antes del primer rebase.

**1. Descarga la Llave Pública**
Descarga la llave `cosign.pub` de este repositorio a la máquina destino:
```bash
wget https://raw.githubusercontent.com/IEBurela/kionite/main/cosign.pub -O ~/cosign.pub
```

**2. Instala la llave en el sistema de contenedores**
```bash
sudo mkdir -p /etc/pki/containers/
sudo cp ~/cosign.pub /etc/pki/containers/ieburela.pub
```

**3. Configura la Política de Seguridad (Strict Reject)**
Cierra el candado por completo exigiendo firma para tus imágenes:
```bash
sudo tee /etc/containers/policy.json > /dev/null << 'EOF'
{
  "default": [
    {
      "type": "reject"
    }
  ],
  "transports": {
    "docker": {
      "ghcr.io/IEBurela": [
        {
          "type": "sigstoreSigned",
          "keyPath": "/etc/pki/containers/ieburela.pub",
          "signedIdentity": {
            "type": "matchRepository"
          }
        }
      ]
    }
  }
}
EOF
```

**4. Configura el Registro para buscar Attachments de Sigstore**
Dile a `rpm-ostree` que busque las firmas directamente en GitHub:
```bash
sudo mkdir -p /etc/containers/registries.d
sudo tee /etc/containers/registries.d/ieburela.yaml > /dev/null << 'EOF'
docker:
  ghcr.io/IEBurela:
    use-sigstore-attachments: true
EOF
```

**5. Ejecuta el Rebase Seguro Directo**
Ahora puedes hacer el rebase directamente a la versión firmada sin pasar por la versión insegura:
```bash
sudo rpm-ostree rebase ostree-image-signed:docker://ghcr.io/IEBurela/ieburela-kinoite-amd:latest
```

**6. Reiniciar**
```bash
sudo systemctl reboot
```

## Arquitectura

- **Base AMD:** `kinoite-main` (Soporte nativo del kernel).
  - **Base Nvidia:** `kinoite-nvidia` (Drivers propietarios precompilados).
  - **Actualizaciones:** Automatizadas y gestionadas por `ublue-update` (descarga en segundo plano, se aplica al reiniciar).

## 🛠️ Post-Instalación (Nuevos Equipos)

Una vez que la máquina haya reiniciado en la nueva imagen personalizada (`IEBurela Fedora 44`), debes ejecutar estos comandos **una sola vez** para integrarla a la red corporativa.

### 1. Establecer el Nombre de Host (Hostname)
Evita colisiones de DNS asignando un nombre único antes de unir la máquina a la red.
```bash
sudo hostnamectl set-hostname nombre-unico.ieburela.com
```

### 2. Activar la VPN (Tailscale)
Conecta la máquina a la red privada (requerirá autenticación vía navegador).
```bash
sudo tailscale up
```

### 3. Unir al Dominio (FreeIPA)
Une la máquina al directorio activo para autenticación centralizada.
*(Si el DNS interno no resuelve automáticamente por estar detrás de un proxy como Cloudflare, usa `--fixed-primary` y especifica el servidor).*
```bash
sudo mkdir -p /var/lib/ipa-client/sysrestore /var/lib/ipa-client/pki
sudo ipa-client-install --server=ipa.ieburela.com --domain=ieburela.com --fixed-primary
```

### 4. Desbloqueo Automático de Disco (TPM)
Si el disco está cifrado con LUKS, vincula la partición al chip TPM de la tarjeta madre para que se desbloquee automáticamente al encender.
*(Asegúrate de cambiar `/dev/sda3` por el nombre real de tu partición cifrada usando `lsblk`).*
```bash
sudo systemd-cryptenroll --tpm2-device=auto --wipe-slot=tpm2 /dev/sda3
```

### 5. Montar NFS con Soporte de Caché (FS-Cache)
El servicio `cachefilesd` ya viene habilitado de fábrica en esta imagen. Para evitar que el sistema se cuelgue si el NAS se apaga, utilizaremos el **automontaje bajo demanda** de systemd, junto con la bandera de caché local (`fsc`).

Primero crea la carpeta donde aparecerán los archivos:
```bash
sudo mkdir -p /mnt/ruta_local
```

Edita tu archivo `/etc/fstab`:
```bash
sudo nano /etc/fstab
```
Y añade la línea de montaje con las opciones a prueba de fallos:
```text
tu-servidor-nas.com:/ruta/en/el/nas  /mnt/ruta_local  nfs  noauto,x-systemd.automount,x-systemd.idle-timeout=10min,fsc  0  0
```

Para que Linux detecte el cambio en el archivo, recarga el gestor de servicios:
```bash
sudo systemctl daemon-reload
```

Finalmente, inicia el servicio de automontaje:
```bash
sudo systemctl start mnt-ruta_local.automount
```
*(A partir de ahora, el NAS se conectará automáticamente en el momento exacto en el que abras la carpeta `/mnt/ruta_local`).*
