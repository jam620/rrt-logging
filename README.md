# Responsable Red Teaming - Terminal Logging

Recientemente termine un curso dictado por ***[HuskyHacks](https://taggartinstitute.org/p/responsible-red-teaming)*** el cual nos menciona todo lo relacionado con OPSEC, responsabilidades y éticas, la legalidad y como parte responsable en un ejercicio de red team debemos contar con la manera de registrar todo nuestras pruebas y actividades de manera responsable. La imagen es un representación que no debemos olvidar que nuestro OPSEC es una amenaza emulada. Responsable, legal y ética. **Responsible Red Teaming Operate With Honor**

![1](img/1.png)

Este articulo es una guía paso a paso de como mantener nuestra responsabilidad y auditar  nuestras actividad durante los engagements,  nos permitirá crear un script para registrar nuestra terminal y consumir estos logs con Elastic, Kibana y Fleet para custodiar, auditar y registrar nuestra actividad como red teamers responsables.

**Nota:** el contenido aquí no es nuestro y es un paso a paso del curso mencionado anteriormente, pero me parece adecuado compartir el conocimiento.

##### Requerimientos

* Máquina virtual con Kali 4GB de Ram mínimo, deseable 8GB de Ram.

* Acceso a los puertos 5601, 8220, 9200 si estamos en un VPS 

* Cambiar las contraseñas en el archivo .env de ELASTIC Y KIBANA

* Instalar Docker y Docker-compose



1. ##### Configurando Timestamps en la terminal de Kali

Debemos modificar la terminal `~/.zshrc` y agregamos la siguiente función:

 `PROMPT=$PROMPT'%F{yellow}%}[%D{%m/%f/%y} %D{%L:%M:%S}] '`

```shell
nano ~/.zshrc
```

![2](img/2.png)

Una vez que guardemos podemos usar el comando `source ~/.zshrc`

![3](img/3.png)

Ahora podemos visualizar la hora, existen otras optimizaciones que de igual manera permiten ver esto como son el [Powerlevel9k](https://github.com/Powerlevel9k/powerlevel9k/wiki/Show-Off-Your-Config)

2. ##### Terminal Logging

Volvemos a editar el archivo `~/.zshrc `y agregamos el siguiente script:

```shell
# smart_script will continuously log the input and output of the terminal into a logfile located in ~/Terminal_typescript/raw/


smart_script(){
    # if there's no SCRIPT_LOG_FILE exported yet
    if [ -z "$SCRIPT_LOG_FILE" ]; then
        # make folder paths
        logdirparent=~/Terminal_typescripts
        logdirraw=raw/$(date +%F)
        logdir=$logdirparent/$logdirraw
        logfile=$logdir/$(date +%F_%T).$$.rawlog
                                txtfile=$logdir/$(date +%F_%T).$$.txt
        # if no folder exist - make one
        if [ ! -d $logdir ]; then
            mkdir -p $logdir
        fi
        export SCRIPT_LOG_FILE=$logfile
        export SCRIPT_LOG_PARENT_FOLDER=$logdirparent
        export TXTFILE=$txtfile


        # quiet output if no args are passed
        if [ ! -z "$1" ]; then
            script -f $logfile
            cat $logfile| perl -pe 's/\\e([^\\[\\]]|\\[.*?[a-zA-Z]|\\].*?\\a)//g' | col -b > $txtfile
        else
            script -f -q $logfile
            cat $logfile | perl -pe 's/\\e([^\\[\\]]|\\[.*?[a-zA-Z]|\\].*?\\a)//g' | col -b > $txtfile
        fi
        exit
    fi
}
# Start logging into new file
alias startnewlog='unset SCRIPT_LOG_FILE && smart_script -v'


# savelog manually saves the current terminal in/out into a logfile: 
# Example: $ savelog logname
savelog(){
    # make folder path
    manualdir=$SCRIPT_LOG_PARENT_FOLDER/manual
    # if no folder exists - make one
    if [ ! -d $manualdir ]; then
        mkdir -p $manualdir
    fi
    # make log name
    logname=${SCRIPT_LOG_FILE##*/}
    logname=${logname%.*}
    # add user logname if passed as argument
    if [ ! -z $1 ]; then
        logname=$logname'_'$1
    fi
    # make filepaths
    txtfile=$manualdir/$logname'.txt'
    rawfile=$manualdir/$logname'.rawlog'
    # make .rawlog readable and save it to .txt file
    cat $SCRIPT_LOG_FILE | perl -pe 's/\\e([^\\[\\]]|\\[.*?[a-zA-Z]|\\].*?\\a)//g' | col -b > $txtfile
    # copy corresponding .rawfile
    cp $SCRIPT_LOG_FILE $rawfile
    printf '[+] Saved logs'
    echo ""
    printf '  \\\\-> '$txtfile''
    echo ""
    printf '  \\\\-> '$rawfile''
}


# Run Smart Script at terminal initialization
smart_script
```

El script crea un directorio en el home donde guarda los logs de los comandos utilizados podemos verlo, para iniciar el login debemos ejecutar el comando **smart_script**

![4](img/4.png)

Accedemos al directorio Terminal_typescritps

![5](img/5.png)

Podemos ver los comandos registrados hasta el momento

![6](img/6.png)

El script nos permite guardar los logs de manera manual con la función **savelog**

![7](img/7.png)

##### 3. Instalación de Elastic y Fleet

Clonamos el repositorio 

```shell
 git clone https://github.com/The-Taggart-Institute/elastic-container.git
```

![8](img/8.png)

Entramos al repositorio y editamos el archivo .env

![9](img/9.png)

Modificamos las variables **ELASTIC_USERNAME**, **ELASTIC_PASSWORD** Y **KIBANA_PASSWORD**

![10](img/10.png)



Le damos permisos de ejecución e iniciamos la instalación

```shell
chmod +x elastic-container.sh
./elastic-container.sh start
```

![11](img/11.png)

Si todo sale bien podemos ir al navegador a nuestro ip https://localhost:5601 y accedemos con el usuario y contraseña

![12](img/12.png)

Debemos ver el dashboard

![13](img/13.png)



Vamos a enrolar un agente, vamos al menu de la izquierda **Management** --> **Fleet**

![14](img/14.png)

Ahora vamos a instalar el agente

![15](img/15.png)

Debemos configurar el host de fleet

![16](img/16.png)



Especificamos el ip y puerto del host y guardamos

![17](img/17.png)

Guardamos y desplegamos

![18](img/18.png)

Aparecerá el host

![19](img/19.png)

Ahora si regresamos a crear agente, en nombre lo podemos dejar como **Agent Policy 1** y chicheamos en **Create Policy**

![20](img/20.png)

En el siguiente paso instalamos el agente en el host

![21](img/21.png)



Debemos copiar el comando y ejecutarlo en el host (recordemos que es un vm de demo por lo que deben tener cuidado con los tokens). **Nota:**  se agrego la opción **-i** al final del comando

```shell
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.5.0-linux-x86_64.tar.gz
tar xzvf elastic-agent-8.5.0-linux-x86_64.tar.gz
cd elastic-agent-8.5.0-linux-x86_64
sudo ./elastic-agent install --url=https://4.188.232.120:8220 --enrollment-token=cDV0YXA0VUJieDZPbU1LV3ZVZUY6VUVGcHhDaWFSR3VvUHFfOENhdHFjQQ== -i
```

![22](img/22.png)

Si sale todo bien debemos ver lo siguiente

![24](img/24.png)

Y en el elastic

![23](img/23.png)

Podemos observar el agente desplegado en el host de Kali

![25](img/25.png)

Vamos ahora agregar una salida, vamos al menu principal de Fleet --> Settings

![26](img/26.png)

Colocamos los datos siguientes:

![27](img/27.png)

En la sección de configuración avanzada se coloco lo siguiente, por temas de laboratorio deshabilitamos la verificación de SSL, en un entorno en producción debemos activarlo.

```
ssl.verification_mode: none
```

Marcamos las opciones que aparecen a continuación, desplegamos y guardamos

![28](img/28.png)



4. ##### Personalizando la recolección de datos

Ahora que tenemos el Kali en el fleet Pipeline, debemos agregar los logs del scripts de terminal.

Procedemos al menu de la izquierda Fleet --> Agent Policies --> clickeamos en **Agent Policy 1**

![29](img/29.png)

Vamos a seleccionar **Add Integration**

![30](img/30.png)

Hacemos una búsqueda con la palabra **custom** y seleccionamos **Custom Logs**

![31](img/31.png)

Agregamos los logs 

![32](img/32.png)

Configuramos de la siguiente manera, se agrega una fila adicional y guardamos y desplegamos

![33](img/33.png)

**Nota:** después de guardar sale una notificación debemos aceptarla

Debemos ver las dos integraciones configuradas

![34](img/34.png)

##### 5. Kibana

Ahora que tenemos los logs indexados e infestados, debemos poder ir a Kibana y en la Sección de Discovery  ala izquierda nos debe aparecer lo siguiente

![35](img/35.png)

En la sección de búsqueda colocamos path y agregamos log.file.path en el campo![36](img/36.png)

Si deseamos ordenar los logs por el campo de Terminal Scripts clickeamos lo siguiente

![37](img/37.png)

Procedemos a entrar al modo full screen

![38](img/38.png)

Una vez en full screen podemos seleccionar los logs que queremos

![39](img/39.png)

Por último podemos interactuar con los logs 

![40](img/40.png)

##### Conclusión

Normalmente en un ejercicio de red team desplegamos nuestra infraestructura en la nube, eso quiere decir que los datos confidenciales de los clientes están en el proveedor de nube que escogemos para desplegar nuestra pruebas, es decir en la nube de otras personas. a pregunta es como red teamers, desplegar una infraestructura de manera rápida, que sea fácil de mantener es un fundamental. Pero esta conveniencia no puede dejar de lado que ponemos en riesgo nuestra reputación y la de nuestro cliente.

En próximos articulo explicaremos como desplegar C2 y responder las interrogantes 

¿Quién es el dueño de la data almacenada en la nube? ¿soy yo? ¿la empresa? o el proveedor de servicios en a nube.

