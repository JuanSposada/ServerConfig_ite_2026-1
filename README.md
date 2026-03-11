# Gestión de Seguridad y Límites para el Usuarios

## Instalacion de Ubuntu Server


## Descripción
Este documento detalla las configuraciones aplicadas al usuario **pako** para garantizar la estabilidad del sistema y limitar el uso de recursos, manteniendo un control sobre sus privilegios de superusuario.

---

## 1. Límites de Recursos (`limits.conf`)
Se han establecido restricciones en `/etc/security/limits.conf` para evitar el consumo excesivo de recursos:

| Parámetro | Límite (Hard) | Descripción |
| :--- | :--- | :--- |
| `maxlogins` | 2 | Máximo de sesiones concurrentes. |
| `fsize` | 500,000 KB | Tamaño máximo por archivo individual. |
| `nproc` | 50 | Límite máximo de procesos activos. |

---

## 2. Cuotas de Almacenamiento (Alternativa 1)
El primer aproximamiento para limitar el espacio en disco se utilizo el Siguiente: 
Se ha limitado el espacio en disco en la partición raíz (`/`) utilizando la herramienta `quota`:

* **Límite Blando:** 2 GB
* **Límite Duro:** 2.5 GB
* **Comando de gestión:** `setquota -u pako 2G 2.5G 0 0 /`

### ⚠️Problemas con Quota y Workaround Realizado
La implementacion de quotas es el primer acercamiento, en nuestro caso esta implementacion fallo pues el sistema alerta de sistema de archivos deprecado y sugiere mover a uno nuevo
por ello es que optamos por la particion del disco, siguiendo el siguiente proceso:

Generar ´particioin para limitar espacio en disco

#### 1. Respalda los datos actuales de pako
sudo cp -r /home/pako /home/pako_backup

#### 2. Crea un archivo de imagen de 500MB
`sudo dd if=/dev/zero of=/var/pako_disk.img bs=1M count=500`

#### 3. Formatea el archivo como ext4
`sudo mkfs.ext4 /var/pako_disk.img`

#### 4. Crea un directorio temporal para montar
`sudo mkdir -p /mnt/pako_temp`

#### 5. Monta temporalmente la imagen
`sudo mount -o loop /var/pako_disk.img /mnt/pako_temp`

#### 6. Copia los datos actuales de pako a la nueva imagen
`sudo cp -a /home/pako/* /mnt/pako_temp/ 2>/dev/null || echo "Algunos archivos no cupieron"`

#### 7. Desmonta el temporal
`sudo umount /mnt/pako_temp`

#### 8. Renombra el directorio actual de pako
`sudo mv /home/pako /home/pako_old`

#### 9. Crea el nuevo directorio de pako
`sudo mkdir /home/pako`

# 10. Monta la imagen en /home/pako
`sudo mount -o loop /var/pako_disk.img /home/pako`

# 11. Verifica que los archivos estén ahí
`ls -la /home/pako`

# 12. Verifica el espacio disponible (debe mostrar ~500MB)
`df -h /home/pako`

# 13. Asegura permisos correctos
`sudo chown -R pako:pako /home/pako`

# 14. Haz permanente el montaje
`echo "/var/pako_disk.img /home/pako ext4 loop,defaults 0 0" | sudo tee -a /etc/fstab`

# 15. Verifica que el fstab esté correcto
`sudo mount -a`

# 16. Confirma el límite
`df -h /home/pako`


---

## 3. Seguridad y Privilegios
Dado que `pako` cuenta con acceso `sudo`, se han aplicado las siguientes capas de seguridad:

### Restricción de Comandos (`sudoers`)
En lugar de privilegios totales (`ALL`), se ha limitado el uso de `sudo` mediante `visudo` exclusivamente a:
* `/usr/bin/apt` (Actualizaciones y software)
* `/usr/bin/systemctl` (Gestión de servicios del sistema)

### Acceso SSH
* Configuración gestionada en `/etc/ssh/sshd_config`.
* *Recomendación:* Se sugiere implementar un entorno `Chroot` para restringir el acceso a directorios fuera de `/home/pako`.

---

## ⚠️ Advertencia de Seguridad
El usuario posee privilegios administrativos. Es responsabilidad del administrador verificar los logs del sistema (`/var/log/auth.log`) periódicamente para asegurar que no se hayan intentado modificar los archivos de configuración aquí mencionados.

---
*Configuración mantenida por el equipo de administración de sistemas.*
