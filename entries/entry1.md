# Protege tus aplicaciones en Kubernetes con estos pasos

El movimiento de las cargas de trabajo de empresas en servidores propios y máquinas virtuales hacia **arquitecturas cloud** basadas en contenedores **ofrece grandes beneficios y despierta un [gran interés](https://insights.stackoverflow.com/survey/2021#most-loved-dreaded-and-wanted-tools-tech-want).** Pero, esta transición se ha visto obstaculizada por preocupaciones acerca de la seguridad de este nuevo modelo. 

> El [reporte del estado de seguridad de Kubernetes 2022](https://www.redhat.com/en/resources/state-kubernetes-security-report) de Red Hat identifica que **el 55% de los encuestados han atrasado nuevos desarrollos en Kubernetes debido a preocupaciones de seguridad.**
> 

Por esto, realizaremos una serie de blogs donde se expondrá **el modelo de seguridad recomendado** para implementar una arquitectura basada en contenedores manejados con Kubernetes**.** Este, **es un modelo de seguridad que se basa en capas.** En este blog veremos el nivel a alto nivel cada una junto con algunas recomendaciones para protegerlas, después profundizaremos en cada capa y la implementación de las medidas de seguridad necesarias.

## Modelo de seguridad por capas

La [documentación oficial de Kubernetes](https://kubernetes.io/es/docs/concepts/security/overview/) establece que se puede pensar en la seguridad en un ambiente nativo en la nube como un sistema de capas. Estas son las siguientes:

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8fb35ebb-377b-413b-8c02-acc806df0973/Untitled.png)

La **capa exterior** es donde se encuentra el computo que corre nuestras aplicaciones. Esto puede ser un proveedor de servicios en la nube (cloud) pública como AWS, Google Cloud Platform (GCP), Azure, entre otros servidores en una nube privada;, o servidores que mantengamos en un datacenter propios.

La siguiente **capa es la del clúster** que comprende todos los nodos del plano de control y trabajadores que usemos**.** Esta es como tal la manejada por la plataforma abierta y la especificación de Kubernetes, comprende todas las configuraciones que realizamos sobre nuestros nodos, su comunicación, servicios disponibles y comunicaciones.

Luego, tenemos **la capa de los contenedores**. Estos son todos los contenedores **que corren en nuestros nodos trabajadores** y que contienen los ambientes necesarios para correr el código de nuestras aplicaciones.

Por último, tenemos el **código como tal** de nuestras aplicaciones. **Este es que nos permite construir aplicaciones que brindan productos y servicios a nuestros clientes y/o usuarios.**

Cada una de estas tiene un contexto distinto y por lo tanto requieren de medidas de protección distintas. Ahora veremos en un alto nivel, las principales medidas por cada una de las capas del modelo.

## Cómo puedes proteger la Capa de Nube

En esta capa es importante destacar que si usas un proveedor de servicios en la nube pública (AWS, GCP, Azure, etc.), estos cuentan con un modelo de seguridad compartido. Lo cual significa que, en general, los proveedores se encargan de proteger su infraestructura física y tú te encargas de proteger lo que ocurra dentro de su infraestructura. 

Igual, **es recomendable consultar el apartado de seguridad en la documentación del proveedor que uses para tener mayor claridad sobre tus responsabilidades.**

> Puedes encontrar acá los apartados de seguridad de:
> 
> - [AWS](https://aws.amazon.com/es/security/)
> - [GCP](https://cloud.google.com/security/)
> - [Azure](https://learn.microsoft.com/es-mx/azure/security/fundamentals/overview)

Las principales recomendaciones que puedes aplicar para proteger tu parte de la capa de la nube o infraestructura son

### Restringe el acceso de red al Plano de Control

Se debe restringir todo acceso público desde Internet al plano de control de Kubernetes. Estos es restringir mediante el uso de listas de control de acceso o firewall, el acceso a los nodos del plano de control y en especial a los puertos donde corren servicios sensibles de Kubernetes como lo son:

| Puerto | Protocolo | Servicio |
| --- | --- | --- |
| 6443 | TCP | Kubernetes API Server |
| 2379-2380 | TCP | etcd server client API |
| 10250 | TCP | Kubelet API |
| 10251 | TCP | kube-scheduler |
| 10252 | TCP | kube-controller-manager |
| 10255 | TCP | Read-Only Kubelet API |

### Protege el acceso de red de nodos trabajadores

Similar al punto anterior, se deben restringir las conexiones que puedan recibir los nodos trabajadores mediante el uso de listas de control de acceso o firewall. Lo ideal es que solo puedan recibir conexiones desde los nodos del plano de control y limitar lo máximo posible el acceso público a internet. Los principales puertos donde corren servicios en estos nodos son:

| Puerto | Protocolo | Servicio |
| --- | --- | --- |
| 10250 | TCP | Kubelet API |
| 10255 | TCP | Read-Only Kubelet API |
| 30000-32767 | TCP | NodePort Services |

### Limita el acceso remoto a los nodos

Se debe limitar en lo posible el acceso SSH a los nodos del clúster para reducir el riesgo de un acceso no autorizado al host de los nodos. Si se requiere una conexión directa, se debe usar `kubectl exec`.

### Encripta los datos en reposo

Cuando sea posible, es bueno encriptar los discos duros de las maquinas donde corren los nodos, en especial donde se encuentre el **etcd** (banco de datos de Kubernetes).

## Cómo puedes proteger la Capa de Clúster

Estas son medidas que dependiendo del modelo de despliegue de Kubernetes que manejemos (manejado por proveedores como [Amazon EKS](https://aws.amazon.com/es/eks/), o [GKE](https://cloud.google.com/kubernetes-engine) de GCP, o administrado de forma propia, si lo desplegamos en instancias de cómputo) pueden variar un poco en la forma de implementación o de quién sería el responsable de implementarlas (si nuestra o de los proveedores cloud que usemos).

### Implementación de políticas de Autenticación y Autorización

Se deben establecer medidas para autenticar las peticiones a la API del clúster de Kubernetes que se despliegue desde la cual se pueden ver, crear, modificar o eliminar recursos del clúster; como nodos, pods, servicios, secrets, entre otros. Para esto se pueden usar distintos métodos como Certificados TLS, Bearer Tokens, Basic Auth o Proxy de autenticación. 

De igual forma, se deben establecer mecanismos que evalúen cada petición al API contra una serie de reglas establecidas para establecer si se permiten o se niegan; la documentación oficial de Kubernetes recomienda el uso de webhooks, RBAC (control de acceso basado en roles) y ABAC (control de acceso basado en atributos).

En adición, se deben deshabitar algunas configuraciones predeterminadas de Kubernetes como el acceso anónimo al servidor API, controlar el uso de personificación y establecer controles de admisión a las peticiones que buscan llegar a un volumen.

### Implementa las políticas de seguridad especificas a Kubernetes

Diferentes políticas deben ser establecidas según los siguientes niveles:

1. **Políticas de red**

De manera predeterminada todos los pods de un clúster de Kubernetes se pueden comunicar entre sí, esto debe ser cambiado según el caso de uso especifico y permitir conexiones solo entre pods que lo requieran. Adicionalmente, se deben configurar políticas de Firewall para restringir las peticiones de red externas a las mínimas necesarias para el correcto funcionamiento de las aplicaciones que se corran. Plugins de políticas de red como [Calico](https://www.projectcalico.org/calico-networking-for-kubernetes/) son recomendados.

1. **Políticas de pods**

Se deben definir contextos de seguridad para los contenedores corriendo dentro de cada pod. Cuando sea posible, los contenedores deben correr sin privilegios de root dentro del pod y herramientas que implementen MAC como [AppArmor](https://apparmor.net/) o [SELinux](https://es.wikipedia.org/wiki/SELinux) deben ser usadas.

### Activa las herramientas de logs y monitoreo

Kubernetes incluye herramientas que permiten llevar logs detallados de las aplicaciones, los contenedores dentro de cada pod y los clústeres activos. Estos deben ser activados debido a que no se realiza de manera predeterminada. Para que esto sea efectivo, se debe crear una política de monitoreo de logs de manera regular de tal manera que un comportamiento extraño pueda ser descubierto a tiempo.

### Cifra y restringe el acceso a etcd

Etcd es la base de datos interna usada por cada clúster de Kubernetes, allí se guarda información sensible como nombres de usuario de bases de datos, contraseñas, y consultas. Se recomienda aislar etcd tras un firewall para evitar que se pueda tener acceso a esta por medio de la API. Aunque Kubernetes encripta etcd, su llave de encripción se guarda como un texto plano en el nodo máster, por esto, se recomienda usar herramientas de manejo de secretos como [Vault](https://www.vaultproject.io/).

### Actualiza continuamente los componentes

Dado que nuevos ataques y vulnerabilidades se van descubriendo en las herramientas de Kubernetes, es de vital importancia que se mantengan actualizados todos los componentes del clúster de manera continua y tan pronto como aplicar las actualizaciones sea posible sin disrumpir el funcionamiento de las aplicaciones.

### Limita el uso de CPU y memoria

El uso de CPU y memoria se pueden limitar mediante una herramienta provista por el Kernel de Linux llamada CGroups. Esto se debe realizar para prevenir ataques de denegación de servicios (DOS). En adición, Kubernetes permite configurar el número máximo de instancias para un contenedor y el número máximo de contenedores corriendo en un pod.

### Habilita el SSL/TLS

Se recomienda la configuración de certificados TLS para asegurar que una comunicación encriptada entre los componentes de Kubernetes como el servidor API, etcd, Kubelet y Kubectl.

### Asegura el acceso a metadatos

Kubernetes genera metadatos que incluye información que, aunque en primera instancia no pueda parecer importante, en manos de un atacante experto puede ser usada para buscar fallas en las versiones de las herramientas o SO que se manejen en los distintos componentes del clúster de Kubernetes. Debido a esto, el acceso a esta información debe ser restringido solo a usuarios autorizados. Esto se puede hacer mediante el uso de RBAC asignando RoleBindings de roles que tengan acceso a los endpoints de la API de Kubernetes que tengan esta información solo a ciertos usuarios de confianza.

### Separa las aplicaciones sensitivas

En caso de que sea necesario, aplicaciones que manejen información o procesos considerados como sensibles deben ser desplegados en un set de máquinas o ambientes específicos separados del resto de aplicaciones. De esta forma, en caso de una brecha de seguridad, los atacantes no podrán tener acceso a estas aplicaciones.

## Cómo puedes proteger la Capa de contenedores

Estas medidas en su mayoría corresponderán en nuestra propia responsabilidad.

### Aisla los contenedores con Namespaces

El Kernel de Linux permite aislar distintos procesos para que estos no tengan acceso a recursos de otros procesos. Cada instancia de los contenedores que estén corriendo debe ser configurada para correr en un espacio distinto.

### Limita el uso de CPU y memoria con CGroups

El uso de CPU y memoria se pueden limitar mediante una herramienta provista por el Kernel de Linux llamada [CGroups](https://docs.docker.com/engine/security/#control-groups). Esto se debe realizar para prevenir ataques de denegación de servicios (DOS).

### Escanea las vulnerabilidades

Se recomienda el uso de herramientas para el escaneo de vulnerabilidades en contenedores como [Dockscan](https://github.com/kost/dockscan), [CoreOS Chair](https://github.com/quay/clair) o [Snyk](https://snyk.io). De igual forma, el establecimiento de flujos de CI/CD es recomendado para establecer varios puntos de chequeo en el proceso de despliegue y prevenir errores que puedan haber pasado por alto.

### Firma de Imágenes

Una vez has escaneado las imágenes que usaras y has corregido las posibles vulnerabilidades que se hayan encontrado, es buena práctica aplicar una firma digital sobre esta nueva imagen que luego pueda ser verificada y que certifique que las imágenes que usas están aprobadas y son seguras.

### Implementa políticas MAC

Herramientas que implementen MAC como AppArmor o SELinux deben ser implementadas para controlar el acceso no autorizado a recursos de usuarios o procesos maliciosos.

La configuración de estas políticas es la de mayor importancia para evitar ataques y accesos no autorizados; aunque, este proceso es complejo y una mala configuración puede exponer los contendores a mayores vulnerabilidades.

Para ayudar a crear las políticas MAC, existen herramientas como [Docker-Sec](https://github.com/FotisLouk/docker-sec) o [Lic-Sec](https://arxiv.org/abs/2009.11572) que mediante el uso de análisis estáticos y dinámicos aprenden del comportamiento ‘usual’ de un contenedor; con lo cual se crea una lista blanca de comportamientos permitidos y se bloquea todo lo demás.

## Cómo puedes proteger la Capa de código

Esta es la capa sobre la que tenemos mayor control. En esta debemos aplicar las mejores prácticas de desarrollo seguro. Estas recomendaciones van más allá de aplicaciones que corran en contenedores o en Kubernetes, si no que se pueden aplicar a cualquier proceso de desarrollo en general. 

Una buena guía para aplicar prácticas de seguridad es [OWAS ASVS](https://owasp.org/www-project-application-security-verification-standard/) (Application Security Verification Standard). Este es un estándar creado por la comunidad que se puede usar como base de un checklist de prácticas que se pueden evaluar o implementar para considerar a una aplicación somo segura.

> La [versión en español de este estándar](https://raw.githubusercontent.com/OWASP/ASVS/v4.0.3/4.0/OWASP%20Application%20Security%20Verification%20Standard%204.0.3-es.pdf) está disponible públicamente para que la consultes.
> 

### Aplica Seguridad en dependencias de terceros

Actualmente es común el uso de librerías, frameworks y recursos externos en nuestro código que solucionan problemas y necesidades generales. Esto nos ayuda a enfocarnos en las funcionalidades específicas que brindarán valor a nuestros usuarios, pero pueden introducir nuevos vectores de ataque a nuestras aplicaciones. 

Herramientas como [GitHub dependabot](https://docs.github.com/es/code-security/dependabot) nos ayudan a escanear de forma automatizada y alertarnos cuando usemos una dependencia que contenga vulnerabilidades.

### Realiza un análisis estático de código (SAST)

Esto es una práctica que se enfoca en analizar el código fuente de las aplicaciones. Al analizar nuestro código contra una serie de reglas predefinidas y buenas prácticas conocidas, podremos lograr una detección temprana de problemas de desarrollo. Tenemos un [blog donde usamos SonarCloud](https://platzi.com/blog/sonarcloud-mejora-codigo-sast/) para realizar este tipo de análisis; otra herramienta recomendada es [OWASP Source Code Analysis Tools](https://owasp.org/www-community/Source_Code_Analysis_Tools).

### Efectúa un análisis dinámico de código (DAST)

Este tipo de análisis, a diferencia del anterior se realiza sobre la aplicación ya corriendo. Se intenta validar el comportamiento de la aplicación ante distintos tipos de entradas generadas de forma dinámica; esto puede incluir la inyección de SQL, CSRF y XSS. Una de las herramientas de análisis dinámico más populares es [OWASP ZAP](https://www.zaproxy.org/) (Zed Attack proxy).

## ¿Cómo puedes seguir aprendiendo?

Ahora que viste a alto nivel las principales recomendaciones, podrás ver en mayor detalle cómo implementar las recomendaciones por cada una de las capas que vimos en blogs que te dejaré acá a medida que vayan saliendo.

Si te interesa seguir aprendiendo más conceptos de Kubernetes, te invito a que conozcas nuestro [curso de Kubernetes](https://platzi.com/cursos/k8s/). Allí aprenderás que los clústeres de K8s se configuran mediante archivos declarativos, aprenderás qué son *deployments*, *volúmenes*, *RBAC*, *balanceo de carga*, *replica sets, namespaces*, entre muchas otras cosas.

También, puedes aprender mucho más sobre contenedores con nuestro [Curso de Docker](https://platzi.com/cursos/docker/). En este, aprenderás sobre los conceptos fundamentales de contenedores, el ciclo de vida de un contenedor, el proceso de construcción de imágenes, el manejo de red y múltiples formas de construir contenedores.

Por último, si te interesa la seguridad en general; puedes ver nuestra [ruta de aprendizaje de Seguridad Informática](https://platzi.com/seguridad-informatica/) donde encontraras una serie de cursos que te llevarán desde lo más básico hasta los temas más avanzados sobre seguridad.

De igual forma, puedes revisar la [documentación oficial de K8s sobre seguridad](https://kubernetes.io/es/docs/concepts/security/overview/). Aunque lo más importante es que sigas practicando e investigando hasta donde te lleve tu deseo de **nunca parar de aprender.**
