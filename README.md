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
