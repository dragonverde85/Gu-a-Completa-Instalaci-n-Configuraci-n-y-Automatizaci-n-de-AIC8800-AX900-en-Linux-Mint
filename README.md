Markdown

# Instalación y Configuración del Adaptador AIC8800 (AX900) en Linux Mint

Guía paso a paso para la instalación, activación y automatización del adaptador USB combo **Wi-Fi 6 + Bluetooth (AIC8800 / AX900)** con IDs de hardware `1111:1111` y `368b:8d81`.

---

## 📋 Tabla de Contenidos
- [Requisitos e Instalación de Dependencias](#1-requisitos-e-instalación-de-dependencias)
- [Instalación del Controlador Base (Wi-Fi)](#2-instalación-del-controlador-base-wi-fi)
- [Conmutación USB y Verificación](#3-conmutación-usb-y-verificación)
- [Activación de Bluetooth (hci0)](#4-activación-de-bluetooth-hci0)
- [Automatización al Iniciar el Sistema](#5-automatización-al-iniciar-el-sistema)
- [Solución de Problemas Frecuentes](#6-solución-de-problemas-frecuentes)

---

## 1. Requisitos e Instalación de Dependencias

Antes de comenzar, instala las herramientas necesarias para compilar módulos del kernel y gestionar las interfaces inalámbricas:

```
sudo apt update
sudo apt install -y dkms build-essential git usb-modeswitch rfkill bluez bluetooth

2. Instalación del Controlador Base (Wi-Fi)

    Importante: Es indispensable utilizar la rama principal (main). No uses la rama legacy-mcu1, ya que desactiva la interfaz Wi-Fi para este chipset.

    Clona el repositorio oficial (o entra a la carpeta si ya lo habías clonado):



cd ~
[ -d "aic8800d80" ] && cd aic8800d80 || git clone [https://github.com/shenmintao/aic8800d80.git](https://github.com/shenmintao/aic8800d80.git) && cd aic8800d80

    Asegúrate de estar en la rama main e inicia el instalador:



git checkout main
sudo ./install.sh

    Reinicia el sistema para registrar el módulo en DKMS y aplicar las reglas iniciales de udev:



sudo reboot

3. Conmutación USB y Verificación

El adaptador inicia por defecto en modo almacenamiento masivo (1111:1111). Si al encender el equipo el Wi-Fi no se activa de inmediato, fuerza la conmutación de modo y desbloquea la radio:


sudo usb_modeswitch -c /etc/usb_modeswitch.d/1111:1111
sudo rfkill unblock all

Para comprobar si la interfaz de red ya está detectada:


ip link

4. Activación de Bluetooth (hci0)

Para que el adaptador levante la interfaz Bluetooth correctamente sobre el controlador btusb, debes cargar la inyección de firmware y el parche de compatibilidad USB:

    Carga los módulos requeridos en el kernel:



sudo modprobe aic_load_fw
sudo modprobe aic_zlp_quirk

    Reinicia el servicio de Bluetooth y levanta la interfaz:



sudo systemctl restart bluetooth
sudo hciconfig hci0 up

    Verifica el estado del controlador:



hciconfig -a

(El parámetro BD Address debe mostrar una dirección MAC válida y el estado indicará UP RUNNING).
5. Automatización al Iniciar el Sistema

Para evitar introducir comandos manualmente en cada reinicio, configura la carga automática de los controladores:

    Agrega los módulos de Bluetooth al archivo de inicio del kernel:



echo "aic_load_fw" | sudo tee -a /etc/modules
echo "aic_zlp_quirk" | sudo tee -a /etc/modules

    (Opcional) Si experimentas desconexiones esporádicas, desactiva la suspensión de energía USB editando GRUB:



sudo nano /etc/default/grub

Añade usbcore.autosuspend=-1 dentro de las comillas en GRUB_CMDLINE_LINUX_DEFAULT, guarda los cambios y actualiza:


sudo update-grub

6. Solución de Problemas Frecuentes
❌ Error Unknown symbol in module al cargar aic8800_fdrv

Ocurre cuando existen residuos de compilaciones manuales previas en el sistema.

Solución: Limpia el registro de DKMS, elimina carpetas antiguas y reinstala desde main:


sudo dkms remove aic8800/1.0.0 --all
sudo rm -rf /lib/modules/$(uname -r)/kernel/drivers/net/wireless/aic8800
sudo depmod -a
cd ~/aic8800d80
git checkout main
sudo ./install.sh
sudo reboot

❌ Error Can't get device info: No such device o Wi-Fi no detectado

Causado por haber compilado con la rama legacy-mcu1.

Solución: Vuelve a la rama estable ejecutando:


cd ~/aic8800d80
git checkout main
sudo ./install.sh
sudo reboot

❌ Error Connection timed out (110) en Bluetooth

Indica un conflicto de inicialización con el driver genérico de Linux.

Solución: Reinicia la secuencia de módulos:


sudo modprobe -r btusb
sudo modprobe aic_load_fw
sudo modprobe aic_zlp_quirk
sudo modprobe btusb
sudo systemctl restart bluetooth
sudo hciconfig hci0 up
