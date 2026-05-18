# Lab 4 - Ciberseguridad: Hardening y Auditoría con OpenSCAP en Rocky Linux 9.7

**Estudiante:** Chester Ivan Ferrer Hernandez  
**Facultad:** Ciencias y Tecnologías  
**Carrera:** Ciberseguridad  
**Fecha:** 17 de mayo de 2026

---

## Tabla de Contenidos

1. [Descarga e instalación de Rocky Linux 9.7 Minimal](#1-descarga-e-instalación-de-rocky-linux-97-minimal)
2. [Configuración de la máquina virtual](#2-configuración-de-la-máquina-virtual)
3. [Configuraciones durante la instalación](#3-configuraciones-durante-la-instalación)
4. [Configuración post-instalación](#4-configuración-post-instalación)
5. [Instalación de paquetes necesarios](#5-instalación-de-paquetes-necesarios)
6. [Hardening de SSH](#6-hardening-de-ssh)
7. [Configuración del Firewall](#7-configuración-del-firewall)
8. [Configuración de Fail2Ban y Auditd](#8-configuración-de-fail2ban-y-auditd)
9. [Instalación de Ansible](#9-instalación-de-ansible)
10. [Preparación del contenido SCAP para Rocky Linux](#10-preparación-del-contenido-scap-para-rocky-linux)
11. [Escaneo inicial con OpenSCAP (Baseline)](#11-escaneo-inicial-con-openscap-baseline)
12. [Remediación con Ansible](#12-remediación-con-ansible)
13. [Recuperación del sistema](#13-recuperación-del-sistema)
14. [Escaneo post-remediación](#14-escaneo-post-remediación)
15. [Resultados finales](#15-resultados-finales)

---

## 1. Descarga e instalación de Rocky Linux 9.7 Minimal

La ISO fue descargada desde el sitio oficial de Rocky Linux:  
🔗 [https://rockylinux.org/download](https://rockylinux.org/download)

Se seleccionó la versión **v9.7 Minimal ISO** para arquitectura AMD64 (x86_64).

---

## 2. Configuración de la máquina virtual

La VM fue creada en **VMware Workstation** con las siguientes especificaciones:

| Parámetro         | Valor                        |
|-------------------|------------------------------|
| Nombre            | Rocky Linux 64-bit           |
| Ubicación         | F:\Rocky Linux 64-bit        |
| Versión           | Workstation 25H2 or later    |
| Sistema Operativo | Rocky Linux 64-bit           |
| Disco Duro        | 40 GB                        |
| Memoria RAM       | 8192 MB                      |
| Red               | NAT                          |
| Otros             | 8 CPU cores, CD/DVD, USB Controller, Sound Card |

---

## 3. Configuraciones durante la instalación

### Contraseña de root

- Se configuró una contraseña para root (con fines académicos, relativamente simple).
- Se habilitó el acceso SSH con contraseña para root.

### Creación de usuario

- **Nombre completo:** Chester Ivan Ferrer Hernandez
- **Usuario:** `chester-ferrer`
- **Contraseña configurada durante instalación**

### Particionado del disco

Se seleccionó el modo **Custom** en la pantalla de destino de instalación. Luego se optó por la creación automática de particiones, resultando en el esquema LVM estándar:

| # | Acción         | Tipo                       | Dispositivo                     | Punto de montaje |
|---|----------------|----------------------------|---------------------------------|------------------|
| 1 | destroy format | Unknown                    | VMware Virtual NVMe Disk        |                  |
| 2 | create format  | partition table (MSDOS)    | VMware Virtual NVMe Disk        |                  |
| 3 | create device  | partition                  | nvme0n1p1                       |                  |
| 4 | create device  | partition                  | nvme0n1p2                       |                  |
| 5 | create format  | physical volume (LVM)      | nvme0n1p2                       |                  |
| 6 | create device  | lvmvg                      | rlm                             |                  |
| 7 | create device  | lvmlv                      | rlm-root                        |                  |
| 8 | create format  | xfs                        | rlm-root                        | /                |
| 9 | create device  | lvmlv                      | rlm-swap                        |                  |
| 10| create format  | swap                       | rlm-swap                        |                  |
| 11| create format  | xfs                        | nvme0n1p1                       | /boot            |

---

## 4. Configuración post-instalación

### Conexión SSH via PuTTY

Se verificó la IP del servidor y se estableció conexión remota:

```bash
ip a
```

Conexión desde PuTTY: `192.168.40.137` Puerto: `22`

### Agregar usuario al grupo wheel (administrador)

```bash
su -
usermod -aG wheel chester-ferrer
```

### Actualización del sistema

Cerrar sesión SSH y reconectarse, luego:

```bash
sudo dnf update -y
```

---

## 5. Instalación de paquetes necesarios

```bash
sudo dnf install -y epel-release

sudo dnf install -y \
  openscap-scanner \
  scap-security-guide \
  openscap-utils \
  firewalld \
  fail2ban \
  audit \
  curl wget git unzip net-tools \
  vim nano \
  policycoreutils-python-utils \
  sssd
```

### Identificar y guardar la IP en una variable

```bash
IP_REAL=$(ip -4 addr show ens160 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo "Tu IP: $IP_REAL"
MI_RED=$(echo "$IP_REAL" | cut -d. -f1-3).0/24
echo "Tu red: $MI_RED"
```

---

## 6. Hardening de SSH

### Respaldar configuración original y aplicar hardening

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

sudo tee -a /etc/ssh/sshd_config <<EOF
# Hardening SSH
Port 22
Port 2222
AddressFamily inet
PermitRootLogin no
MaxAuthTries 3
MaxSessions 2
PubkeyAuthentication yes
PasswordAuthentication yes
PermitEmptyPasswords no
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
Banner /etc/ssh/banner.txt
EOF

echo "AllowUsers chester-ferrer" | sudo tee -a /etc/ssh/sshd_config
```

### Configurar banner de identificación

```bash
sudo tee /etc/ssh/banner.txt > /dev/null << 'EOF'
Este es un laboratorio de prueba

Solo el personal autorizado puede ingresar y realizar cambios

Nombre del estudiante:
Chester Ferrer

Facultad de Ciencias y Tecnologías
Ciberseguridad
EOF
```

### Agregar puerto 2222 a SELinux y reiniciar SSH

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
sudo semanage port -l | grep ssh_port_t
sudo systemctl restart sshd
sudo systemctl status sshd
sudo ss -tlnp | grep sshd
```

### Prueba local del nuevo puerto

```bash
ssh -p 2222 chester-ferrer@localhost
```

> ✅ El banner aparece correctamente al conectarse vía WinSCP o PuTTY.

---

## 7. Configuración del Firewall

```bash
sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='$MI_RED' port port='22' protocol='tcp' accept"
sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='$MI_RED' port port='2222' protocol='tcp' accept"
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
sudo firewall-cmd --list-rich-rules
```

---

## 8. Configuración de Fail2Ban y Auditd

### Fail2Ban

```bash
sudo systemctl enable --now fail2ban

sudo tee /etc/fail2ban/jail.local <<EOF
[sshd]
enabled = true
port = 2222
maxretry = 3
bantime = 1h
findtime = 10m
logpath = /var/log/secure
EOF

sudo systemctl restart fail2ban
sudo systemctl status fail2ban
```

### Auditd

```bash
sudo systemctl enable --now auditd
```

---

## 9. Instalación de Ansible

```bash
sudo dnf install -y ansible
ansible --version
```

---

## 10. Preparación del contenido SCAP para Rocky Linux

> **Nota importante:** Rocky Linux 9 se identifica como `cpe:/o:rocky:rocky:9`, pero el datastream oficial de RHEL 9 espera `cpe:/o:redhat:enterprise_linux:9`. Sin corrección, todas las reglas devuelven `notapplicable`.

### Copiar y adaptar el datastream

```bash
mkdir -p ~/scap-lab && cd ~/scap-lab
cp /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml ./ssg-rocky9-custom.xml
```

### Corregir el CPE

```bash
sed -i 's|cpe:/o:redhat:enterprise_linux:9|cpe:/o:rocky:rocky:9|g' ./ssg-rocky9-custom.xml
```

### Verificar el cambio

```bash
grep -c "cpe:/o:rocky:rocky:9" ./ssg-rocky9-custom.xml
# Debe mostrar un número mayor que 0
```

### Crear estructura de directorios del lab

```bash
mkdir -p ~/lab-openscap/{resultados,reportes,remediacion}
```

---

## 11. Escaneo inicial con OpenSCAP (Baseline)

```bash
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --fetch-remote-resources \
  --results ~/lab-openscap/resultados/baseline.xml \
  --report ~/lab-openscap/reportes/baseline.html \
  ~/scap-lab/ssg-rocky9-custom.xml
```

El reporte `baseline.html` fue transferido a Windows vía WinSCP para su visualización.

### Resultados del escaneo inicial

| Métrica           | Valor      |
|-------------------|------------|
| Reglas pasadas    | 190        |
| Reglas fallidas   | 99         |
| Puntuación        | 78.903061  |
| Máximo posible    | 100.000000 |
| Porcentaje        | **78.9%**  |

Severidad de reglas fallidas: 3 high, 2 low, 67 medium, 2 critical.

---

## 12. Remediación con Ansible

### Generar el playbook de remediación desde el datastream

```bash
sudo oscap xccdf generate fix \
  --fix-type ansible \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --output ~/lab-openscap/remediacion/remediacion-cis-l1.yml \
  ~/scap-lab/ssg-rocky9-custom.xml
```

### Instalar sssd y configurar authselect

```bash
sudo dnf install -y sssd
sudo authselect select sssd --force
```

### Simulación del playbook (dry-run)

```bash
cd ~/lab-openscap/remediacion
sudo ansible-playbook -i "localhost," -c local remediacion-cis-l1.yml --check --diff
```

> ✅ Sin errores en la simulación.

### Aplicar el playbook

```bash
sudo ansible-playbook -i "localhost," -c local remediacion-cis-l1.yml
```

---

## 13. Recuperación del sistema

Tras aplicar la remediación, la política de contraseñas se volvió muy estricta, bloqueando el acceso al sistema. Se procedió a recuperar el acceso mediante el arranque de emergencia de GRUB.

### Pasos de recuperación

1. En el menú GRUB, editar la entrada de arranque y agregar `rd.break` al final de la línea `linux`:

```
linux ($root)/vmlinuz-5.14.0-611.55.1.el9_7.x86_64 root=/dev/mapper/rlm-root ro \
  crashkernel=1g-2g:192M,2g-64G-:512M resume=/dev/mapper/rlm-swap \
  rd.lvm.lv=rlm/root rd.lvm.lv=rlm/swap rd.break
```

2. Montar el sistema de archivos y hacer chroot:

```bash
mount -o remount,rw /sysroot
chroot /sysroot
```

3. Cambiar contraseña del usuario (la política requiere mínimo 14 caracteres):

```bash
passwd chester-ferrer
# Contraseña: Lab4Ciber4OpenScap!@#
```

4. Resetear expiración de cuenta y faillock:

```bash
chage -M -1 -E -1 -I -1 chester-ferrer
faillock --user chester-ferrer --reset
```

5. Repetir para root:

```bash
passwd root
# Contraseña: Lab4Ciber4OpenScap!@#
chage -M -1 -E -1 -I -1 root
faillock --user root --reset
```

6. Salir y reiniciar:

```bash
exit
reboot -f
```

---

## 14. Escaneo post-remediación

```bash
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --fetch-remote-resources \
  --results ~/lab-openscap/resultados/post-ansible.xml \
  --report ~/lab-openscap/reportes/post-ansible.html \
  ~/scap-lab/ssg-rocky9-custom.xml
```

### Corregir permisos para transferir el reporte via WinSCP

```bash
sudo chown -R chester-ferrer:chester-ferrer /home/chester-ferrer/lab-openscap
ls -l /home/chester-ferrer/lab-openscap/reportes/
```

---

## 15. Resultados finales

### Comparación antes y después de la remediación

| Métrica           | Baseline (antes) | Post-Ansible (después) |
|-------------------|------------------|------------------------|
| Reglas pasadas    | 190              | 256                    |
| Reglas fallidas   | 99               | 3                      |
| Puntuación        | 78.903061        | 98.039116              |
| Porcentaje        | 78.9%            | **98.04%**             |

### Severidad de reglas fallidas (post-remediación)

| Severidad | Cantidad |
|-----------|----------|
| Low       | 1        |
| Medium    | 1        |
| High      | 1        |

> ✅ Se logró pasar de **78.9%** a **98.04%** de cumplimiento con el perfil CIS Server Level 1, reduciendo las fallas de 99 a solo 3 reglas.

---

## Herramientas utilizadas

| Herramienta       | Propósito                                      |
|-------------------|------------------------------------------------|
| VMware Workstation| Virtualización                                 |
| Rocky Linux 9.7   | Sistema operativo objetivo                     |
| OpenSCAP          | Escaneo y auditoría de seguridad               |
| SCAP Security Guide | Contenido de políticas CIS                   |
| Ansible           | Automatización de remediación                  |
| Fail2Ban          | Protección contra fuerza bruta                 |
| Firewalld         | Gestión del firewall                           |
| PuTTY / WinSCP    | Acceso remoto y transferencia de archivos      |
| auditd            | Auditoría del sistema                          |

---

*Laboratorio realizado el 17 de mayo de 2026 — Ciberseguridad, Facultad de Ciencias y Tecnologías*
