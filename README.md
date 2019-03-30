# ROL ANSIBLE DE AUTENTIFICACION DE USUARIXS UNIX EN N SERVIDORES

Para más información visita el repositorio 

## REQUERIMIENTOS

Esta solución se basa en el software de automatización de despliegue Ansible.
Es una librería en Python, que interpreta instrucciones de automatización que se encuentran en archivos YAML (.yml)
Ansible lo que hace es conectarse por SSH a los hosts y ejecutar estas instrucciones por ti.

Se puede instalar fácilmente por *pip* como paquete python
$ pip install ansible

Las demás dependencias las instalará automáticamente, como parte del procedimiento que levanta el entorno de test.

---

## COMO FUNCIONA

En este esquema de administracion de sesiones de usuarixs Unix, la clave es el archivo *group_vars/all/user_auth.yml*. Este consiste en una lista (_users_).

Para que aplique a todos los hosts, este archivo se encuentra en el directorio _group_vars/all_ (profundizaremos en esto más adelante...)

Luego, basta con ejecutar el procedimiento *auth.yml* para que despliegue automáticamente a los hosts las configuraciones definidas en el archivo users_auth.yml

### Rol Ansible

El grueso de las configraciones las hace la tarea principal del rol user_auth _roles/user_auth/tasks/main.yml_
El procedimiento _auth.yml_ en realidad lo único que hace es llamar al rol user_auth.
Esta estructura corresponde a la de un rol en Ansible, y es así para que pueda ser reusable modularmente.

### Parámetros

Cada item de la lista _users_ se compone básicamente de:
- un nombre de usuaria (_name_)
- un comentario (_comment_)
- una lista de los hosts a los que tiene accceso (_allowed_).

Además puede aceptar dos parámetros extra:
- _sudo_ Para decidir si tendrá privilegios de administrador (booleano).
- _shell_ Este puede ser booleano: tener o no acceso a consola (por defecto, la consola será bash). 
También puede ser un string, para especificar una consola específica (útil, por ej., para dar acceso por Git).

#### Grupos de usuarixs

Con ésto se puede emular, de forma simple, los tres grupos de usuario propuestos en el ejercicio:
- Normal: usuarix sin acceso a consola (útil, por ejemplo, para usuarixs ftp)
- Shell: usuarix con acceso a consola
- Admin: usuarix con privilegio de sudo. en este caso si tendrá consola, la cual por defecto será bash.

Más adelante, se podrían definir esquemas más complejos mediante el uso de grupos de usuarixs Unix.
Es un TODO

---

## LEVANTAR ENTORNO DE DESARROLLO

Para probarlo, escribí *tasks/rise.yml* un procedimiento que prepara el entorno y levanta un ambiente de test, con diez servidoras capaces de SSH en una red local.

$ ansible-playbook tasks/rise.yml

### Primera llave pública de SSH

La primera vez, la persona administradora de sistema solo necesita dejar su llave pública en el directorio _files/ssh/user.pub_
en donde el nombre del archivo _user_.pub debe coincidir con la variable *sysadmin* en el archivo _rise.yml_

Esa llave estará autorizada para *root* en todos los hosts, de forma que podamos configurarlos con Ansible.

### Saltarse la instalación de requerimientos

Parte de la preparación del entorno es que nuestra máquina local tenga la paquetería requerida por el sistema de automatización Ansible, estas tareas están marcadas con el tag _installation_. Solo actuarán la primera vez, por tanto después podemos saltárnoslas así

$ ansible-playbook tasks/rise.yml --skip-tags=installation

### Contenedores Docker de Alpine

Las servidoras en realidad son contenedores de Docker, muy delgados porque se basan en Alpine Linux, a cuya imágen agregamos solamente las librerías mínimas para esta situación (el servidor de SSH openssh, shadow para la gestión de usuarixs, python para la automatización con Ansible, y sudo.)

### Archivos de Hosts de prueba

Finalmente, el procedimiento genera *hosts.test* un archivo de hosts para Ansible, el cual utilizarán los procedimientos subsiguientes.
Esto se configura en el archivo ansible.cfg, donde además se especifica que ansible utilizará el usuario root, y que relaje el checkeo de las huellas SSH, que cambiarán.
Este archivo de hosts es local y es válido solo para este ejercicio, en la realidad manejariamos un archivo de hosts con nombres de dominio en vez de IPs locales.

---

## ADMINISTRAR USUARIXS

La primera vez, la persona administradora de sistemas ejecutará el procedimiento *auth.yml*, entonces se crearán los demás usuarixs de la lista, los cuales a su vez podrán crear a otrxs si es que tienen el privilegio sudo.

$ ansible-playbook auth.yml

Para cada usuarix, es necesario que su llave pública de SSH esté copiada en el directorio files/ssh con su nombre de usuario y la extensión .pub *files/ssh/user.pub*

La idea es que una organización mantenga el archivo _user_auth.yml_ junto con la carpeta _files/ssh_ por medio de un sistema de control de versiones como *Git*.

### Crear un nuevx usuarix

La arquitectura de un rol ansible con un archivo de configuración (_user_auth.yml_), permite tener un solo comando a ejecutar para todo.
Es decir que para crear una nueva persona usuaria, basta con añadirlo a la lista y volver a ejecutar el procedimiento *auth.yml*

  - name: lgtbiq
    comment: "LGTBIQplus"
    allowed:
      - multiuser_1_1
      - multiuser_2_1

$ ansible-playbook auth.yml

Ansible trabaja de manera idempotente, por lo que repasará todas las demás configuraciones, y además agregará a la nueva persona usuaria.

### Cambiar privilegios de un usuarix

Así mismo, para cambiar los privilegios de una persona usuarix, basta con editar el item correspondiente en la lista

  - name: shuk
    comment: "uno"
    sudo: yes
    allowed:
      - multiuser_1_1

En este caso le hemos agregado la propiedad sudo, por lo cual este usuarix será un sudoer en las máquinas listadas en _allowed_

Para que esto sea efectivo, el comando a ejecutar es el mismo procedimiento *auth.yml*

$ ansible-playbook -D auth.yml

(En este caso le hemos agregado la opcion -D para que muestre un diff de los cambios que realiza)

#TODO degradar priviegios no funciona

### Eliminar un usuarix

Para eliminar un usuarix si que se requiere un procedimiento particular. Este se encuentra en _tasks/rm.yml_ y se debe invocar con un parámetro extro *username*

$ ansible-playbook tasks/rm.yml -e "username=monsieur"

Simplemente itera por los hosts eliminando el usuarix.

Luego, necesitaremos borrarlo manualmente del archivo _users\_auth.yml_ para que no se vuelva a crear la próxima vez que se ejecute.

---

## Grupos de Hosts

Una funcionalidad del archivo de hosts (hosts.test) de Ansible es que se pueden definir grupos de Hosts entre corchetes []

    [concepcion]
    numerica.cl
    nube.numerica.cl
    
    [temuco]
    mail.numerica.cl

Para hacer uso de esto, dentro de la carpeta _group\_vars_ se utilizan subdirectorios con el nombre del grupo.
Entonces si ponemos un archivo _users\_auth.yml_ en *group_vars/concepcion/all* crearíamos usuarixs solamente en los hosts de ese grupo.
Este nos permite enfrentar casos más complejos, en que queramos dar privilegios de sudo en algunxs servidores y otros no, por ejemplo.

