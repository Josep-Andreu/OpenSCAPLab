# OpenSCAPLab

## Resumen

El objetivo de este lab es evaluar y comprobar políticas de Openscap usando un perfil estándar de seguridad. Para ello comprobaremos si nuestros servidores son compliance usando el perfil de Centro Criptológico Nacional (CCN) para RHEL 9. Después aplicaremos una customización sencilla partiendo de ese perfil, comprobaremos parámetros de password de los usuarios, instalaremos software y chequearemos que un servicio está arrancado. Finalmente, generaremos una remediación con Ansible para aquellos checks de compliance que no cumplimos y la aplicaremos en los servidores. 

## Instalación de software

Todo el lab se ejecutará en el mismo servidor, por ello vamos a instalar toda la paquetería en el mismo sistema (la imagen que hemos instalado ya tiene los paquetes necesarios por lo que no hace falta este paso, lo hemos añadido para tener toda la información completa):

```
[root@client ~]# dnf install -y openscap-scanner scap-security-guide scap-workbench
```

El paquete `openscap-scanner` instala la herramienta `oscap` desde la cual podremos lanzar nuestros scans, generar reports o incluso generar las remediaciones con Ansible.

El paquete `scap-security-guide` instala los `data streams` necesarios en la ruta `/usr/share/xml/scap/ssg/content/`. Este `data stream` contiene la información de todos los perfiles de SCAP con sus reglas asociadas para evaluar el compliance de un scan o también para generar las remediaciones. Esta información está en el fichero `XCCDF` (Extensible Configuration Checklist Description File) `ssg-rhel9-xccdf.xml`.

Por último el paquete `scap-workbench` nos servirá para generar perfiles custom de compliance, como veremos más adelante. Solamente añadir que también permite evaluar el compliance en local y en sistemas remotos a través de `SSH`.

## Evaluar el compliance

Podemos ver todos los perfiles disponibles para evaluar el compliance de nuestro sistema con el comando:

```
[root@client ~]# oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
...
Title: Centro Criptológico Nacional (CCN) - STIC for Red Hat Enterprise Linux 9 - Advanced
				Id: xccdf_org.ssgproject.content_profile_ccn_advanced
...
```

El profile Centro Criptológico Nacional (CCN) para RHEL 9 Advanced es el que nosotros usaremos. Para evaluar el compliance usando este estándar y guardar los resultados en un fichero ejecutaremos:

```
[root@client ~]# oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_ccn_advanced --results /root/results.xml /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
...
Title   System Audit Logs Must Have Mode 0640 or Less Permissive
Rule    xccdf_org.ssgproject.content_rule_file_permissions_var_log_audit
Ident   CCE-83720-3
Result  pass

Title   Record Events that Modify the System's Discretionary Access Controls - chmod
Rule    xccdf_org.ssgproject.content_rule_audit_rules_dac_modification_chmod
Ident   CCE-83830-0
Result  fail
...
```

Podemos ver como en el ejemplo para cada una de las políticas del estándar, cuáles han pasado el test y cuáles han fallado.

También podemos generar un report html a partir del anterior fichero de resultados con:

```
[root@client ~]# oscap xccdf generate report /root/results.xml > results.html
```

Si abrimos el report vemos lo siguiente:

![image](https://github.com/user-attachments/assets/5f10cb7a-12d3-470a-8f4b-bf8a43da6bab)


## Customizar un perfil de compliance

Para customizar un perfil de compliance hemos de crear un `tailoring file`, para ello ejecutaremos el programa de `scap-workbench`:

```
[root@client ~]# scap-workbench
```

El programa carga los `s` que estén instalados en el servidor:

![image](https://github.com/user-attachments/assets/4ba4f310-171d-4471-bf14-bdf204ae759b)

Seleccionamos el profile de Centro Criptológico Nacional (CCN) - STIC for Red Hat Enterprise Linux 9 - Advanced. Vamos a `Customize` y seleccionamos un nombre de perfil, dejaremos el que crea por defecto. A continuación marcamos `Deselect all`, así empezaremos con todas las reglas deshabilitadas:

![image](https://github.com/user-attachments/assets/44aabe65-ad63-44ff-9e6d-aaee2ed42429)

Marcaremos las siguientes opciones `Ensure PAM Enforces Password Requirements - Minimum Lenght` y buscamos un poco más arriba la opción de `minlen` para setearlo a 12:

![image](https://github.com/user-attachments/assets/42305c9f-4ca4-44b5-98fe-19efd6487c0d)

A continuación marcaremos la opción para instalar `AIDE` y también para instalar y habilitar el servicio de `USBGuard`:

![image](https://github.com/user-attachments/assets/154e33a7-2318-489f-b9db-e34ebd0efd4a)

Con lo que tendremos un `tailoring file` con estas 4 reglas definidas:

![image](https://github.com/user-attachments/assets/553c1fbf-6b28-425f-a320-05979d5e19e7)

A continuación, vamos a `File -> Save Customization Only` y le damos el nombre de `lab-tailoring.xml` y lo guardamos dentro de `/root`.

Con este `tailoring file`, podemos ejecutar un nuevo scan que solo compruebe estas 4 reglas que hemos añadido al perfil vacío. Para ello, primero necesitamos obtener el nombre del perfil, ya que lo hemos customizado:

```
[root@client ~]# oscap info lab-tailoring.xml 
Document type: XCCDF Tailoring
Imported: 2024-09-25T15:05:59
Benchmark Hint: /tmp/scap-workbench-aywUnZ/ssg-rhel9-ds.xml
Profiles:
	Title: Centro Criptológico Nacional (CCN) - STIC for Red Hat Enterprise Linux 9 - Advanced [CUSTOMIZED]
		Id: xccdf_org.ssgproject.content_profile_ccn_advanced_customized

```

Vemos que el nombre del perfil nos aparece al final, donde pone Id. Con este nombre de perfil podemos lanzar un scan que compruebe las 4 reglas y guardarlas también en fichero de resultados:

```
[root@client ~]# oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_ccn_advanced_customized --tailoring-file lab-tailoring.xml --results /root/lab-results-tailoring.xml /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml 
--- Starting Evaluation ---

Title   Install AIDE
Rule    xccdf_org.ssgproject.content_rule_package_aide_installed
Ident   CCE-90843-4
Result  fail

Title   Ensure PAM Enforces Password Requirements - Minimum Digit Characters
Rule    xccdf_org.ssgproject.content_rule_accounts_password_pam_dcredit
Ident   CCE-83566-0
Result  fail

Title   Install usbguard Package
Rule    xccdf_org.ssgproject.content_rule_package_usbguard_installed
Ident   CCE-84203-9
Result  fail

Title   Enable the USBGuard Service
Rule    xccdf_org.ssgproject.content_rule_service_usbguard_enabled
Ident   CCE-84205-4
Result  fail
```
Vemos que ha fallado el test de las 4 reglas. Para corregirlo, crearemos un `playbook` de Ansible que lo hará automáticamente.

## Generar un playbook para aplicar remediaciones automáticamente

Con el fichero de resultados del anterior apartado `/root/lab-results-tailoring.xml` podemos crear un playbook usando:

```
[root@client ~]# oscap xccdf generate fix --profile xccdf_org.ssgproject.content_profile_ccn_advanced_customized --tailoring-file lab-tailoring.xml --fix-type ansible --result-id "" /root/lab-results-tailoring.xml  > fix.yml
```

Necesitamos crear un inventario, ya que el `playbook` tiene la sección `hosts: all` para limitarlo hemos de añadir lo siguiente `echo client > inventory`, y con esto ya podemos ejecutar el playbook generado:

```
[root@client ~]# ansible-playbook -i inventory fix.yml

PLAY [all] *************************************************************************************************

TASK [Gathering Facts] *************************************************************************************
ok: [client]

TASK [Gather the package facts] ****************************************************************************
ok: [client]

TASK [Ensure PAM Enforces Password Requirements - Minimum Digit Characters - Ensure PAM variable dcredit is set accordingly] ***
changed: [client]

TASK [Ensure usbguard is installed] ************************************************************************
changed: [client]

TASK [Gather the package facts] ****************************************************************************
ok: [client]

TASK [Enable the USBGuard Service - Enable Service usbguard] ***********************************************
changed: [client]

PLAY RECAP *************************************************************************************************
client                     : ok=6    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

Un nuevo scan en el servidor nos debería dar los 4 checks como ok:

```
[root@client ~]# oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_ccn_advanced_customized --tailoring-file lab-tailoring.xml --results /root/lab-results-tailoring.xml /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml 
--- Starting Evaluation ---

Title   Install AIDE
Rule    xccdf_org.ssgproject.content_rule_package_aide_installed
Ident   CCE-90843-4
Result  pass

Title   Ensure PAM Enforces Password Requirements - Minimum Digit Characters
Rule    xccdf_org.ssgproject.content_rule_accounts_password_pam_dcredit
Ident   CCE-83566-0
Result  pass

Title   Install usbguard Package
Rule    xccdf_org.ssgproject.content_rule_package_usbguard_installed
Ident   CCE-84203-9
Result  pass

Title   Enable the USBGuard Service
Rule    xccdf_org.ssgproject.content_rule_service_usbguard_enabled
Ident   CCE-84205-4
Result  pass
```


