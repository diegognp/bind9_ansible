# bind9_ansible

## **Instalación y configuración automatizada de BIND9 usando Ansible**
#

### **Instalacion de Ansible**
 Se instala Ansible con los siguientes comandos en Debian
 ~~~
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt update
$ sudo apt install ansible
~~~
Se verifica la instalacion revisando la version con el comando:
~~~
$ ansible --version
~~~
### **Preparacion de conexion por SSH a los servidores DNS**
Ansible se conectara con los servidores remotos mediante SSH, por lo tanto se configura una key de ssh para su acceso sin contraseña.

Primero se genera una llave SSH con:
~~~
$ sudo ssh-keygen
~~~
Y se envia la llave SSH usando el usuario y la direccion IP del host remoto, mediante el comando:
~~~
$ sudo ssh-copy-id <remote_user>@<remote_host>
~~~


 ### **Crear un inventario** 
 El archivo inventario contiene los grupos de servidores que puede acceder Ansible, especificando las IP de cada host.
 
 El inventario debe contener los dos servidores de DNS que serán configurados. Cada servidor tendrá una direccion IP, un usuario y su contraseña.
~~~

[servidordns]
dns1  ansible_host=<ip_addres> ansible_become_pass=<user> ansible_user=<password>

[servidordns2]
dns2  ansible_host=<ip_addres> ansible_become_pass=<user> ansible_user=<password>
~~~

 ### **Generar key's para las diferentes vistas del servidor BIND9**
 Se generan la llaves para que el servidor secundario reciba archivos del servidor primario, con el siguiente comando:
 ~~~
$ sudo tsig-keygen -a HMAC-MD5 <view>
 ~~~
 Insertar en `<view>` el nombre de la vista.
 - internet
 - pit
 - servidores
 - red-estatal

Las key's generadas deben añadirse al archivo "named.conf.options" en los archivos de ambos servidores, uno para cada vista.

Tendra el siguiente formato:
~~~batch
key "internet-key" {
        algorithm hmac-md5;
        secret "AAAABBBBCCCC4poEtQNNg==";
};
~~~

### **Crear el playbook**
Ansible ejecutara las tareas descritas en el playbook que sera un archivo tipo .YML 

El playbook para instalar y configurar el primer servidor DNS sera de la siguiente forma:
~~~yml
---
- name: configure servidordns
  hosts: servidordns
  become: yes
  become_method: sudo
  
  tasks:
 
    - name: install pakage bind9
      apt:
        name: bind9
        state: present

    - name: copy all files
      copy:
        src: /home/debianpas/Descargas/bind/
        dest: /etc/bind/
      notify: restart BIND9

    - name: start Bind9 service
      service:
        name: bind9
        state: started
      notify: restart BIND9

  handlers:
    - name: restart BIND9
      service:
       name: bind9
       state: restarted
~~~

Se debe especificar el directorio de los archivos preconfigurados para el servidor DNS que serán enviados.

El playbook para instalar el segundo servidor DNS tiene el siguiente formato:
~~~yml
---
- name: configure servidordns2
  hosts: servidordns2
  become: yes
  become_method: sudo
  
  tasks:

    - name: Creates directory
      file:
       path: /var/lib/bind/
       state: directory
       mode: 0775
       owner: bind
       group: bind
       recurse: yes
 
    - name: Install pakage bind9
      apt:
        name: bind9
        state: present

    - name: Copy all files
      copy:
        src: /home/debianpas/Descargas/bind2/bind/
        dest: /etc/bind/
      notify: restart BIND9

    - name: start Bind9 service
      service:
        name: bind9
        state: started
      notify: restart BIND9

  handlers:
    - name: restart BIND9
      service:
       name: bind9
       state: restarted
~~~

### **Ejecutar el playbook**
Una vez preparado los archivos inventory y el playbook.yml
se ejecuta con el comando:
~~~
$ sudo ansible-playbook -i <hosts> <playbook.yml>
~~~
En `<hosts>` se especifica el archivo que contiene el inventario y en `<playbook.yml>` se inserta el archivo  playbook en formato YML.

### **Comprobar estado de los dos servidores DNS**
Se comprueba el estado de BIND9 usando el comando:
~~~
$ sudo systemctl status bind9
~~~
Debe estar en estado activo.

### **Solucion de problemas**

Algunos comandos para solucionar problemas:

~~~
named-checkconf
~~~
Este comando identifica en que linea del archivo "named.conf" existe un error.

~~~
sudo tail -f /var/log/syslog
~~~
Se visualiza las ultimas 10 lineas de los logs generados.
