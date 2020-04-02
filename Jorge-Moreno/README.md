# Sistemas distribuidos
## Primer Examen Parcial.

Implemente una arquitectura que contenga:
 
 * Un productor de mensajes (puede ser en la máquina física o en una máquina virtual)
 * Un broker de RabbitMQ (puede ser en la máquina física o en una máquina virtual)
 * Dos consumidores de mensajes (pueden ser en una máquina virtual o contenedores)
    * El primer consumidor recibirá los mensajes de la cola "Grupo 01"
    * El segundo consumidor recibirá los mensajes de la cola "Grupo 02"
    * Ambos consumidores recibirán los mensajes enviados al grupo "General"

### Actividades
1. Documento README.md en formato markdown:  
  * Formato markdown (5%).
  * Nombre y código del estudiante (5%).
  * Ortografía y redacción (5%).
2. Documentación del procedimiento para el aprovisionamiento del productor (10%). Evidencias del funcionamiento (5%).
3. Documentación del procedimiento para el aprovisionamiento de los consumidores (10%). Evidencias del funcionamiento (5%).
4. Documentación del procedimiento para el aprovisionamiento del broker (10%). Evidencias del funcionamiento (5%).
5. Documentación de las tareas de integración (10%). Evidencias de la integración (10%).
6. El informe debe publicarse en un repositorio de github el cual debe ser un fork de https://github.com/ICESI-Training/Rabbitmqtest y para la entrega deberá hacer un Pull Request (PR) al upstream (10%). Tenga en cuenta que el repositorio debe contener todos los archivos necesarios para el aprovisionamiento.
7. Documente algunos de los problemas encontrados y las acciones efectuadas para su solución al aprovisionar la infraestructura y aplicaciones (10%).

___

## Requisitos
* [Vagrant](https://www.vagrantup.com)
* [Virtualbox](https://www.virtualbox.org)
* [Ansible](https://www.ansible.com/)
* [Java](https://www.java.com/es)
* [RabbitMQ](https://www.rabbitmq.com)

## Instalación

### **1. Vagrant.**
Ingresmos al siguiente link:

https://www.vagrantup.com/docs/installation/

En la esquina superior derecha, seleccionamos la opción Download y después seleccionamos el instalador o paquete apropiado para nuestro sistema operativo (en este caso es ubuntu 18.04 LTS).

Para validar la correcta instalación utilizamos el comando dentro de la consola de la maquina que instalamos Vagrant:

~~~
vagrant -v
~~~

### **2. VirtualBox.**

Ingresamos al siguiente link:

https://vitux.com/how-to-install-virtualbox-on-ubuntu/

Agregamos un repositorio fuente y actualizamos el sistema.
~~~
sudo add-apt-repository multiverse && sudo apt-get update
~~~

Ahora instalamos VirtualBox con el comando.
~~~
sudo apt install virtualbox
~~~

Para abrirlo directamente desde la terminal se escribe:
~~~
virtualbox
~~~

### **3. Ansible**

Ingresamos al siguiente link:

https://github.com/Jorge-Andres-Moreno/Ansible_examples

y ejecutamos los siguientes comandos para la instalación en ubuntu:

~~~
sudo apt install software-properties-common -y
sudo apt-add-repository ppa:ansible/ansible -y
sudo apt install ansible -y
~~~

Por ultimo validamos si quedo instalado:

~~~
ansible -v
~~~

**NOTA**: Los demás requisitos no los instalaremos manualmente. Su instación  va dentro del aprovisionamiento que haremos con ansible.

___

## Desarrollo e implementación

En esta fase utilizaremos una arquitectura que tendra cuatro actuadores. Esta se realizo de esta manera para evicenciar el funcionamiento de cada componente del sistema por separado.

Lo actuadores o componentes de nuestra arquitectura son:
* Servidor RabbitMQ que actuare como nuestro sistema de colas para las peticiones que se realicen.
* Un productor de mensajes el cual tiene como funcionamiento la producción de mensajes para enviarlo al servidor RabbitMQ
* Dos consumidores los cuales tendran que consumir los mensajes que le sean entregados por el servidor y que pertenezcan al grupo que pertenezcan cada consumidor.

la interacción entre los componentes o actuadores del sistema es como describe en la imagen a continuación:

![alt text](https://www.rabbitmq.com/img/tutorials/python-two.png)

## Vagrant + Ansible

Para el desarrollo utilizaremos 4 maquinas virtuales, las cuales estan divididas por cada actuador del sistema.

Empezaremos definiendo en un archivo llamado **hosts** las direcciónes IPs que llevaran las maquinas virtuales que crearemos.

En primer lugar definiremos la IP y el usuario de la maquina donde se encuentra encontrara servidor **RabbitMQ**. El nombre de la maquina utilizara es **RabbitMQ** y tendra la **IP 192.168.56.12**.

En segundo lugar definiremos la IP y el usuario de la maquina donde se encuentra productor de mensajes. El nombre de la maquina que utilizara es **Sender** y tendra la **IP 192.168.56.13**.

En Tercer lugar definiremos la IP y el usuario de la maquina donde se encuentra productor de mensajes. El nombre de la maquina que utilizara es **Receiver1** y tendra la **IP 192.168.56.14**.

En Cuarto lugar definiremos la IP y el usuario de la maquina donde se encuentra productor de mensajes. El nombre de la maquina que utilizara es **Receiver2** y tendra la **IP 192.168.56.15**.

A continuación el contenido del archivo **hosts** .

~~~
server_provision ansible_ssh_user=vagrant

[broker]
RabbitMQ ansible_host=192.168.56.12

[tx]
Sender ansible_host=192.168.56.13

[rx1]
Receiver1 ansible_host=192.168.56.14

[rx2]
Receiver2 ansible_host=192.168.56.15

~~~

El generará un archivo llamado Vagrantfile el cual pondremos la siguiente configuración.

Luego con el comando se creara un archivo llamado Vagrantfile que editaremos y el entorno que necesita un proyecto en vagrant:
~~~
vagrant init
~~~

Ahora en el archivo **Vagrantfile** proceder a definir las cualidades de la maquina virtual, con los nombres que les hemos asignado, tales como: CPU, Memoria RAM, puerto ssh y dirección IP.
~~~
machines = {
  "RabbitMQ" => { :ip => "192.168.56.12", :ssh_port => 2200 },
  "Receiver1" => { :ip => "192.168.56.14", :ssh_port => 2220 },
  "Receiver2" => { :ip => "192.168.56.15", :ssh_port => 2220 },
  "Sender" => { :ip => "192.168.56.13", :ssh_port => 2222 },
}
~~~

Seguimos con la configuración **ssh** de las maquinas virtuales. Decidimos utilizar la **box** de vagrant **ubuntu/bionic64** ya que es la distro de linux **Ubuntu 18.04**, la cual es compatible con los playbooks desarrollados.

~~~
Vagrant.configure("2") do |config|
    
    machines.each_with_index do |(hostname, info), index|
      config.vm.define hostname do |cfg|
        cfg.vm.provider :virtualbox do |vb, override|
          config.vm.box = "ubuntu/bionic64"
          override.vm.network "private_network", ip: "#{info[:ip]}"
          override.vm.network "forwarded_port", guest: 22, host: "#{info[:ssh_port]}", id: "ssh", auto_correct: true
          override.vm.hostname = hostname
          vb.name = hostname
        end
      end
    end
  
~~~

Por ultimo aprovisionamos a las maquinas virtuales con el archivo **server.yml** en el cual se encuetra los aprovisionamientos correspondientes.

**NOTA**: Es importante saber la jerarquia de las carpetas y como se ha estructurado el archivo server.yml ya que en el se encuentran los dos aprovisionamientos que funcionan independientes y tienen el nombre de **roles** en **Ansible**.

~~~
 #Ansible
    config.vm.provision "ansible" do |ansible|
          ansible.inventory_path = 'hosts'
          ansible.verbose = 'vvv'
          ansible.playbook = 'playbooks/servers.yml'
    end
end
~~~

En el archivo de configuración **ansible.cfg** pondremos por defecto que los host que utilizara Vagrant seran los que estan en el archivo **hosts**.

La configuración se hace de la siguiente manera:

~~~
[defaults]
hostfile = hosts
host_key_checking = False
allow_world_readable_tmpfiles=true
~~~

### Java

Para el desarrollo del **Productor de mensajes** he decidido utilizar el lenguaje de programación por experiencia y comodida.

Ahora procederemos con el productor que tendra que conectarse a la maquina del servidor de **RabbitMQ** con la dirección **192.168.52.12** y enviara 3 tipos de mensaje los cuales seran **General, Message1 y Message2** cada segundo enviare estos 3 mensajes al servidor RabbitMQ.


A continuación apreciamos el codigo fuente del productos que la clase la nombramos **Sender.java**:

```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;

public class Sender {

	private static final String EXCHANGE_NAME = "direct_logs";

	public static void main(String[] argv) throws Exception {
		
		System.out.println("Productor going to start");

		ConnectionFactory factory = new ConnectionFactory();
		factory.setHost("192.168.56.12");
		factory.setPort(5672);
		factory.setUsername("Sender");
		factory.setPassword("Sender");

		try (Connection connection = factory.newConnection(); Channel channel = connection.createChannel()) {
			channel.exchangeDeclare(EXCHANGE_NAME, "direct");

			String message = argv.length < 1 ? "info: Hello World!" : String.join(" ", argv);
		
			while (true) {
				
				channel.basicPublish(EXCHANGE_NAME, "General", null, (message + " General").getBytes("UTF-8"));
				System.out.println(" [x] Sent '" + message + "'"+" General");
				
				channel.basicPublish(EXCHANGE_NAME, "Message1", null, (message + " Message1").getBytes("UTF-8"));
				System.out.println(" [x] Sent '" + message + "'"+" Message1");
				
				channel.basicPublish(EXCHANGE_NAME, "Message2", null, (message + " Message2").getBytes("UTF-8"));
				System.out.println(" [x] Sent '" + message + "'"+" Message2");
				Thread.sleep(1000);
			}
		} catch (Exception e) {
			// TODO: handle exception
			e.printStackTrace();
		}
	}

}

```

Ahora procederemos con primer **consumidor** que tendra que conectarse a la maquina del servidor de y resolvera 2 tipos de mensaje o peticion los cuales son: **General y Message1**.

A continuación apreciamos el codigo fuente del productos que la clase la nombramos **Consumer1.java**:

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class Consumer1 {

	private static final String EXCHANGE_NAME = "direct_logs";

	public static void main(String[] argv) throws Exception {
		System.out.println("Consumer1 going to start");

		ConnectionFactory factory = new ConnectionFactory();
		factory.setHost("192.168.56.12");
		factory.setPort(5672);
		factory.setUsername("Receiver1");
		factory.setPassword("Receiver1");

		Connection connection = factory.newConnection();
		Channel channel = connection.createChannel();

		channel.exchangeDeclare(EXCHANGE_NAME, "direct");
		String queueName = channel.queueDeclare().getQueue();
		channel.queueBind(queueName, EXCHANGE_NAME, "General");
		channel.queueBind(queueName, EXCHANGE_NAME, "Message1");

		System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

		DeliverCallback deliverCallback = (consumerTag, delivery) -> {
			String message = new String(delivery.getBody(), "UTF-8");
			System.out.println(" [x] Received '" + message + "'");
		};
		channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {
		});
	}

}

```
Por ultimo procederemos con segundo **consumidor** que tendra que conectarse a la maquina del servidor de y resolvera 2 tipos de mensaje o peticion los cuales son: **General y Message2**.

A continuación apreciamos el codigo fuente del productos que la clase la nombramos **Consumer2.java**:

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class Consumer2 {

	private static final String EXCHANGE_NAME = "direct_logs";

	public static void main(String[] argv) throws Exception {
		
		System.out.println("Consumer2 going to start");

		ConnectionFactory factory = new ConnectionFactory();
		factory.setHost("192.168.56.12");
		factory.setPort(5672);
		factory.setUsername("Receiver1");
		factory.setPassword("Receiver1");

		Connection connection = factory.newConnection();
		Channel channel = connection.createChannel();

		channel.exchangeDeclare(EXCHANGE_NAME, "direct");
		String queueName = channel.queueDeclare().getQueue();
		channel.queueBind(queueName, EXCHANGE_NAME, "General");
		channel.queueBind(queueName, EXCHANGE_NAME, "Message2");

		System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

		DeliverCallback deliverCallback = (consumerTag, delivery) -> {
			String message = new String(delivery.getBody(), "UTF-8");
			System.out.println(" [x] Received '" + message + "'");
		};
		
		channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {
		});
	}

}

```

Cabe añadir que los ejecutables correspondientes a cada modulo desarrollado se encuentran en la carpeta **executables** Los cuales se compilaron se nombraron de la siguiente manera:
* Sender.java => sender.jar
* Consumer1.java => consumer1.jar
* Consumer2.java => consumer2.jar

**Nota**: Para el desarrollo de cada modulo compilamos en un archivo **.jar** para que pudieramos ejecutarlo posteriormente por consola y no cuando se hiciera el aprovisionamiento compilarlo cada vez. Además, para el desarrollo utilizamos una libreria que es necesario para conectarse a los servicios de  **RabbitMQ**.

### Ansible

Ahora comenzaremos con el desarrollo del aprovisionamiento de las maquinas virtuales.
El archivo de condiguración se encuentra en **playbooks/servers.yml**

Conmenzaremos con el servidor RabbitMQ  que lo nombramos **RabbitMQ** en el archivo de configuración **hosts**.a continuación el archivo correspondiente a su aprovisionamiento:

**Nota**: Para el desarrollo de esta guía lo que se hizo fue realizar un paso a paso de una tutorial en este [link](https://github.com/geerlingguy/ansible-role-rabbitmq) y la documentación oficial de la pagina de Ansible [aqui](https://docs.ansible.com/ansible/latest/modules/rabbitmq_queue_module.html).

```yaml
 server installation
- hosts: broker
  become: yes
  become_user: root
  become_method: sudo
  tasks:
  
    - name: Install nessesary package
      apt: 
        name: apt-transport-https
        state: present
        update_cache: yes

    # Install Erlang
    - name: Download erlang
      get_url:
        url: https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
        dest: /home/vagrant/erlang-solutions_1.0_all.deb
        mode: 0755

    - name: Install erlang repository
      apt:
        deb: /home/vagrant/erlang-solutions_1.0_all.deb

    - name: Add an Apt signing key
      apt_key:
        url: https://packages.erlang-solutions.com/debian/erlang_solutions.asc
        state: present

    - name: Fix unmet dependencies
      shell: apt-get -f install

    # Install RabbitMQ
    - name: Add the rabbitmq repository's key
      apt_key: url=https://keyserver.ubuntu.com/pks/lookup?op=get&fingerprint=on&search=0x6B73A36E6026DFCA

    - name: Add the rabbitmq repository
      shell: echo "deb https://dl.bintray.com/rabbitmq/debian stretch main" | tee /etc/apt/sources.list.d/bintray.rabbitmq.list

    - name: Install rabbitmq-server
      apt:
        name: rabbitmq-server
        state: present
        update_cache: yes

    - name: Enable rabbitmq-server to survive reboot
      service: name=rabbitmq-server enabled=yes

    - name: Create configuration file
      shell: |
        echo "listeners.tcp.1 = 0.0.0.0:5672" >> /etc/rabbitmq/rabbitmq.conf
        service rabbitmq-server restart

```
Hasta aqui lo que hemos hecho es instalar y arrancar el servicio de RabbitMQ en la maquina virtual. Además, crear el archivo de configuración. Y posteriormente reiniciar el servio.

Ahora procederemos a confirurar los actuadores del sistema configuraremos un usuario y contraseña de la siguiente manera:

**Productor**
* **Usuario:** Sender
* **Constraseña:** Sender

**Consumidor 1**
* **Usuario:** Receiver1
* **Constraseña:** Receiver1

**Consumidor 2**
* **Usuario:** Receiver2
* **Constraseña:** Receiver2

A continuación el archivo de configuración:

```yaml

    - name: add rabbitmq user Sender
      rabbitmq_user:
        user: Sender
        password: Sender
        permissions:
          - vhost: /
            configure_priv: .*
            read_priv: .*
            write_priv: .*
        state: present

    - name: add rabbitmq user Receiver1
      rabbitmq_user:
        user: Receiver1
        password: Receiver1
        permissions:
          - vhost: /
            configure_priv: .*
            read_priv: .*
            write_priv: .*
        state: present

    - name: add rabbitmq user Consumidor2
      rabbitmq_user:
        user: Consumidor2
        password: Consumidor2
        permissions:
          - vhost: /
            configure_priv: .*
            read_priv: .*
            write_priv: .*
        state: present

```

Ahora procederemos a configurar la maquina virtual correspondiente al primer consumidor que lo llamamos en el archivo **hosts** **Receiver1**.

Para ello necesitamos tener claro que lo unico que necesitamos es instalar la maquina virtual y aprovisionar con el archivo **consumer1.jar**. Además, de que imprimir en un archivo llamado **log.txt** todas las peticiones atendidas como se muestra en la siguiente configuración:

**NOTA**: Para que la tarea se ejecutara en segundo plano fue necesario hacer una investigación sobre ansible para poder realizar dicha hazaña. Por lo cual se implementa un comando llamado **nohub** para ello como se muestra en la siguiente configuración.

```yaml
- hosts: Receiver1
  become: yes
  become_method: sudo
  tasks:

    - name: apt-get update
      apt:
        update-cache: yes
      changed_when: 0

    - name: Install latest version of default-jre 'java'
      apt:
        name: default-jre
        state: latest

    - name: Execute the program
      shell: nohup java -jar /vagrant/executables/consumer1.jar > log.txt &

```
Ahora procederemos a configurar la maquina virtual correspondiente al segundo consumidor que lo llamamos en el archivo **hosts** **Receiver2**.

Para ello necesitamos tener claro que lo unico que necesitamos es instalar la maquina virtual y aprovisionar con el archivo **consumer2.jar**. Además, de que imprimir en un archivo llamado **log.txt** todas las peticiones atendidas como se muestra en la siguiente configuración:


```yaml

- hosts: Receiver2
  become: yes
  become_method: sudo
  tasks:

    - name: apt-get update
      apt:
        update-cache: yes
      changed_when: 0

    - name: Install latest version of default-jre 'java'
      apt:
        name: default-jre
        state: latest

    - name: Execute the program
      shell: nohup java -jar /vagrant/executables/consumer2.jar > log.txt &

```
pro ultimo, configuramos la maquina virtual correspondiente al productor de mensajes que lo llamamos en el archivo **hosts** **Sender**.

Para ello necesitamos tener claro que lo unico que necesitamos es instalar la maquina virtual y aprovisionar con el archivo **sender.jar**. Además, de que imprimir en un archivo llamado **log.txt** todas las peticiones que se envien como se muestra en la siguiente configuración:


```yaml
- hosts: Sender
  become: yes
  become_method: sudo
  tasks:

    - name: apt-get update
      apt:
        update-cache: yes
      changed_when: 0

    - name: Install latest version of default-jre 'java'
      apt:
        name: default-jre
        state: latest

    - name: Execute the program
      shell: nohup java -jar /vagrant/executables/sender.jar > log.txt &
```
___

## Conclusiones

Es importante resaltar el uso de un servicio de colas como **RabbitMQ** el cual sirve para darle solución a muchos softwares que necesitan atender una carga pesada de trabajo ya que atienden muchas solicitudes. Usualmente esto se podria llevar a un sevidor HTTP que solucione estas peticiones. Debido a que es el protocolo mayormente usudo para servicios API REST.