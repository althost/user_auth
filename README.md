# AUTO-AUTENTIFICACION SSH DE USUARIXS UNIX EN N SERVIDORES

Para más información visita el repositorio https://github.com/althost/user_auth/

## REQUERIMIENTOS

Esta solución se basa en el software de automatización de despliegue [Ansible]().

Es una librería en Python, que interpreta instrucciones de automatización que se encuentran en archivos YAML (.yml)
Ansible lo que hace es conectarse por SSH a los hosts y ejecutar estas instrucciones por ti.

Se puede instalar fácilmente por *pip* como paquete python

```bash
$ pip install ansible
```

Las demás dependencias las instalará automáticamente, como parte del procedimiento que levanta el entorno de test.

---

## COMO FUNCIONA

En este esquema de administracion de sesiones de usuarixs Unix, la clave es la lista de usuarixs (_users_) definida en el archivo [*group_vars/all/user_auth.yml*](group_vars/all/user_auth.yml). 

Luego, basta con ejecutar el procedimiento [*auth.yml*](auth.yml) para que despliegue automáticamente a los hosts las configuraciones allí definidas.

``` bash
$ ansible-playbook auth.yml
```


#### group_vars

Para que aplique a todos los hosts, este archivo se encuentra en el directorio __group_vars/all__

Esta estructura de directorios nos permitirá definir usuarixs para grupos de hosts (profundizaremos en esto más adelante...)

### Rol Ansible

El grueso de las configraciones las hace *la tarea principal del rol user_auth* [_roles/user_auth/tasks/main.yml_](roles/user_auth/tasks/main.yml)

El procedimiento [_auth.yml_](auth.yml) en realidad lo único que hace es *llamar al rol user_auth.*

Esta estructura corresponde a la de un rol en Ansible, y es así para que pueda ser reusable modularmente.

### Parámetros

Cada item de la lista _users_ se compone básicamente de:
- un nombre de usuaria (__name__)
- un comentario (__comment__)
- una lista de los hosts a los que tiene accceso (__allowed__).

Además puede aceptar dos parámetros extra:
- __sudo__ Para decidir si tendrá privilegios de administrador (booleano).
- __shell__ Este puede ser booleano: tener o no acceso a consola (por defecto, la consola será bash). 
            También puede ser un string, para especificar una consola específica (útil, por ej., para dar acceso por Git).

#### Grupos de usuarixs

Con ésto se puede emular, de forma simple, los tres grupos de usuario propuestos en el ejercicio:
- Normal: usuarix sin acceso a consola (útil, por ejemplo, para usuarixs ftp)
- Shell: usuarix con acceso a consola
- Admin: usuarix con privilegio de sudo. en este caso si tendrá consola, la cual por defecto será bash.

(Más adelante, se podrían definir esquemas más complejos mediante el uso de grupos de usuarixs Unix.)

---

## LEVANTAR ENTORNO DE DESARROLLO

Para probarlo, escribí [*tasks/rise.yml*](tasks/rise.yml) un procedimiento que prepara el entorno y levanta un ambiente de test, con diez servidoras capaces de SSH en una red local.

    $ ansible-playbook tasks/rise.yml

### Primera llave pública de SSH

La primera vez, la persona administradora de sistema solo necesita dejar su llave pública en el directorio __files/ssh/user.pub__
en donde el nombre del archivo _user_.pub debe coincidir con la variable **sysadmin** en el archivo [_rise.yml_](tasks/rise.yml)

Esa llave estará autorizada como **root** en todos los _hosts_, de forma que podamos configurarlos con Ansible.

### Saltarse la instalación de requerimientos

Parte de la preparación del entorno es que nuestra máquina local tenga la paquetería requerida por el sistema de automatización Ansible, estas tareas están marcadas con el tag _installation_. 

Solo actuarán la primera vez, por tanto después podemos saltárnoslas así

    $ ansible-playbook tasks/rise.yml --skip-tags=installation

### Contenedores Docker de Alpine

Las servidoras en realidad son _contenedores de Docker_, muy delgados porque se basan en **Alpine Linux**, a cuya imágen agregamos solamente las librerías mínimas para esta situación:
- el servidor de SSH **openssh**
- **shadow** para la gestión de usuarixs
- **python** para la automatización con Ansible
- y **sudo**

### Archivos de Hosts de prueba

Finalmente, el procedimiento genera **hosts.test** un _archivo de hosts para Ansible_, el cual utilizarán los procedimientos subsiguientes.

(Esto se configura en el archivo [ansible.cfg](ansible.cfg), donde además se especifica que ansible utilizará el usuario root, y que relaje el checkeo de las huellas SSH, que cambiarán.)

Este archivo de hosts es local y es válido solo para este ejercicio, en la realidad manejariamos un archivo de hosts con nombres de dominio en vez de IPs locales.

---

## ADMINISTRAR USUARIXS

La primera vez, la persona administradora de sistemas ejecutará el procedimiento [*auth.yml*](auth.yml), entonces se crearán los demás usuarixs de la lista, los cuales a su vez podrán crear a otrxs si es que tienen el privilegio sudo.

    $ ansible-playbook auth.yml

La idea es que una organización mantenga un archivo como _user_auth.yml_ por medio de un sistema de control de versiones como **Git**.

### Crear un nuevo usuarix

Tenemos un msimo comando para todo: [auth.yml](auth.yml) que hace efectivos los cambios al archivo [user_auth.yml](group_vars/all/user_auth.yml)

Es decir que para crear un nuevo usuarix, basta con añadirlo a la lista y volver a ejecutar el procedimiento **auth.yml**

```yaml
  - name: lgtbiq
    comment: "LGTBIQplus"
    allowed:
      - multiuser_1_1
      - multiuser_2_1
```
        $ ansible-playbook auth.yml

Ansible trabaja de manera idempotente, por lo que repasará todas las demás configuraciones, y además agregará a la nueva persona usuaria.

Es buena práctica ejecutar regularmente este tipo de procedimientos una vez al día con un cron.


### Cambiar privilegios de un usuarix

Así mismo, para cambiar los privilegios de una persona usuarix, basta con editar el item correspondiente en la lista

```yaml
  - name: shuk
    comment: "uno"
    sudo: yes
    allowed:
      - multiuser_1_1
```

En este caso le hemos agregado la propiedad **sudo**, por lo cual este usuarix será un *sudoer* en las máquinas listadas en _allowed_.

Para que esto sea efectivo, el comando a ejecutar es el mismo, **auth.yml**

    $ ansible-playbook -D auth.yml

(En este caso le hemos agregado la opcion -D para que muestre un diff de los cambios que realiza)

#TODO degradar privilegios no funciona

### Eliminar un usuarix

Para eliminar un usuarix si que se requiere un procedimiento particular. Este se encuentra en _tasks/rm.yml_ y se debe invocar con un parámetro extra **username**

    $ ansible-playbook tasks/rm.yml -e "username=monsieur"

Simplemente itera por los hosts eliminando el usuarix.

Luego, necesitaremos borrarlo manualmente del archivo __users\_auth.yml__ para que no se vuelva a crear la próxima vez que se ejecute.

---

### Test: deshabilitar checkeo de huellas SSH

Al testear la conexión SSH a los contenedores de prueba, recomendamos **deshabilitar el checkeo de huella de identidad SSH** utilizando el parámetro **StrictHostKeyChecking**, ya que será distinta en cada contenedor, y SSH dará una advertencia de seguridad.

Por ejemplo, si vamos a testear la conexión del usuarix _john_ en la i.p. 172.20.0.11 (con la llave privada _files/ssh/john_):

    ssh -i files/ssh/john -oStrictHostKeyChecking=no john@172.20.0.11

---

## Grupos de Hosts

Una funcionalidad del archivo de hosts (hosts.test) de Ansible es que se pueden definir grupos de Hosts entre corchetes []

    [concepcion]
    numerica.cl
    nube.numerica.cl
    
    [temuco]
    mail.numerica.cl

Para hacer uso de esto, dentro de la carpeta __group\_vars__ se utilizan subdirectorios con el nombre del grupo.
Entonces si ponemos un archivo _users\_auth.yml_ en **group_vars/concepcion/all** crearíamos usuarixs solamente en los hosts de ese grupo.

Este nos permite enfrentar casos más complejos, en que queramos dar privilegios de sudo en algunxs servidores y otros no.

