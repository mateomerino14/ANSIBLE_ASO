# ANSIBLE_ASO

Despliegue remoto de **Apache, PHP, phpMyAdmin, MariaDB, PostgreSQL, Samba y FTP** con roles de Ansible sobre **openSUSE 16.0**.

Repositorio: https://github.com/mateomerino14/ANSIBLE_ASO

## Entorno

| Máquina | Función | IP |
|---|---|---|
| VM 01 | Master (Ansible) | 192.168.1.203 |
| VM 02 | Destino (clon de openSUSE 16.0 con MAC nueva) | 192.168.1.204 |

## Estructura

```
ANSIBLE_ASO/
├── ansible.cfg
├── hosts
├── site.yml
├── reset.yml
├── group_vars/
│   └── all/
│       ├── vars.yml
│       └── vault.yml
└── roles/
    ├── apache/        tasks, handlers
    ├── mariadb/       tasks, handlers, templates/my.cnf.j2
    ├── postgresql/    tasks, handlers, templates/pg_hba.conf.j2
    ├── samba/         tasks, handlers, templates/smb.conf.j2
    └── vsftpd/        tasks, handlers, templates/vsftpd.conf.j2
```

| Archivo | Descripción |
|---|---|
| `ansible.cfg` | Configuración de Ansible: inventario, usuario `root`, clave del vault y opciones de rendimiento. |
| `hosts` | Inventario. Solo contiene la IP del destino, sin usuario ni contraseña. |
| `group_vars/all/vars.yml` | Variables comunes: red permitida, usuarios, directorio compartido, permisos y bases de datos. |
| `group_vars/all/vault.yml` | Contraseñas de los servicios cifradas con `ansible-vault` (AES256). |
| `site.yml` | Playbook principal. Ejecuta todos los roles; cada rol tiene su propio tag. |
| `reset.yml` | Deja el destino limpio para volver a probar desde cero. |

## Roles

| Rol | Qué hace |
|---|---|
| `apache` | Instala Apache, PHP 8 y phpMyAdmin. Activa el módulo PHP, crea la página de prueba `index.php`, abre HTTP en el firewall y permite a Apache conectarse a las bases de datos (SELinux). |
| `mariadb` | Instala MariaDB y aplica la plantilla `my.cnf`. Elimina usuarios anónimos y la base `test`, asigna la contraseña de `root` y crea la base `aso_db` con el usuario `aso_user`. |
| `postgresql` | Instala PostgreSQL y la librería de conexión `php8-pgsql`. Asigna la contraseña de `postgres`, crea el usuario `aso_pg` con la base `aso_pgdb` y habilita la conexión remota solo por SSL. |
| `samba` | Crea el usuario del sistema, el usuario Samba y el directorio `/srv/samba/compartido`. Comparte el recurso con `create mask 0664` y `directory mask 2775`. |
| `vsftpd` | Instala el servidor FTP, crea el usuario `user01` y configura el acceso local con escritura y el modo pasivo (puertos 30000-30100). |

## Instalación paso a paso

### 1. Preparar el destino (VM 02, como root)

```bash
systemctl stop packagekit; systemctl mask packagekit
systemctl enable --now sshd
firewall-cmd --permanent --add-service=ssh; firewall-cmd --reload
echo "PermitRootLogin yes" > /etc/ssh/sshd_config.d/10-root.conf
systemctl restart sshd
```

PackageKit se detiene porque bloquea a `zypper`. La línea de `PermitRootLogin` es necesaria porque openSUSE 16 no permite el acceso de root por SSH con contraseña, y se necesita para copiar la llave del master.

### 2. Preparar el master (VM 01, como root)

```bash
systemctl stop packagekit; systemctl mask packagekit
zypper install -y ansible git
git clone https://github.com/mateomerino14/ANSIBLE_ASO.git
cd ANSIBLE_ASO
```

### 3. Clave del vault

La clave no se sube al repositorio. Se guarda en el archivo `.vault_pass`, que está en `.gitignore`:

```bash
echo 'CLAVE_DEL_VAULT' > .vault_pass
chmod 600 .vault_pass
```

Para ver o editar las contraseñas:

```bash
ansible-vault view group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
```

### 4. Conexión por llave SSH

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
ssh-copy-id root@192.168.1.204
```

Si el destino tiene otra IP, se cambia en el archivo `hosts`. Si la red no es `192.168.1.x`, se cambian `red_local` y `mariadb_red` en `group_vars/all/vars.yml`.

### 5. Probar la conexión

```bash
ansible all -m ping
```

Resultado esperado: `192.168.1.204 | SUCCESS` con `"ping": "pong"`.

### 6. Ejecutar el despliegue

Todos los servicios:

```bash
ansible-playbook site.yml
```

Un solo servicio:

```bash
ansible-playbook site.yml --tags apache
ansible-playbook site.yml --tags mariadb
ansible-playbook site.yml --tags postgresql
ansible-playbook site.yml --tags samba
ansible-playbook site.yml --tags ftp
```

Al terminar se muestra el tiempo de cada tarea. Una segunda ejecución debe terminar con `changed=0`, porque todo ya está configurado.

Si aparece el error `couldn't resolve module`, se instalan las colecciones:

```bash
ansible-galaxy collection install ansible.posix community.general
```

### 7. Verificar

Estado de los servicios:

```bash
ansible all -a "systemctl is-active apache2 mariadb postgresql smb vsftpd"
```

| Servicio | Prueba |
|---|---|
| Apache y PHP | Abrir `http://192.168.1.204/` en el navegador |
| phpMyAdmin | Abrir `http://192.168.1.204/phpMyAdmin/` e ingresar con `aso_user` |
| MariaDB | `mysql -h 192.168.1.204 -u aso_user -p aso_db` |
| PostgreSQL | `psql "host=192.168.1.204 dbname=aso_pgdb user=aso_pg sslmode=require"` |
| Samba | `smbclient //192.168.1.204/compartido -U sambauser` |
| FTP | `ftp -p 192.168.1.204` con el usuario `user01` |

Para comprobar que PostgreSQL usa una conexión cifrada, dentro de `psql`:

```sql
SELECT ssl FROM pg_stat_ssl WHERE pid = pg_backend_pid();
```

Debe devolver `t`. Una conexión sin SSL es rechazada.

### 8. Volver a probar desde cero

```bash
ansible-playbook reset.yml
ansible-playbook site.yml
```

`reset.yml` pide escribir `si` antes de borrar los servicios del destino.

## Características del proyecto

- **Seguridad del inventario:** el archivo `hosts` no contiene usuarios ni contraseñas. La conexión se realiza por llave SSH y las contraseñas de los servicios están cifradas con `ansible-vault`.
- **Rendimiento:** `gather_facts` desactivado, `pipelining`, reutilización de la conexión SSH (`ControlPersist`), instalación de los paquetes de cada rol en una sola operación sin actualizar todo el sistema, y reinicio de servicios solo cuando cambia su configuración.
- **Organización por roles:** un rol por servicio, con sus plantillas de configuración en `templates/`.

## Integrantes

- Mateo Alejandro Merino Vidal
- Carlos Daniel Bascopé Espejo
- Gabriel Orlando Camacho Terceros
- Israel Gonzalo Orosco Zeballos
- Mijael Lujan Gandarillas
