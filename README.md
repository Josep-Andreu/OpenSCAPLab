# OpenSCAPLab

## Resumen

El objetivo de este lab es evaluar y comprobar políticas de Openscap usando un perfil estándar de seguridad. Para ello comprobaremos si nuestros servidores son compliance usando el perfil de DISA STIG. Después aplicaremos una customización sencilla partiendo de ese perfil, comprobaremos parámetros de password de los usuarios, instalaremos software y chequearemos que todos los comandos con sudo son auditados. Finalmente generaremos una remediación con Ansible para aquellos checks de compliance que no cumplimos y la aplicaremos en los servidores. 

## Instalación de software

Todo el lab se ejecutará en el mismo servidor, por ello vamos a instalar toda la paquetería en el mismo sistema:

```
[root@client ~]# dnf install -y openscap-scanner scap-security-guide scap-workbench
```

El paquete `openscap-scanner` instala la herramienta `oscap` desde la cuál podremos lanzar nuestros scans, generar reports o incluso generar las remediaciones con Ansible.

El paquete `scap-security-guide` instala los data streams necesarios en la ruta `/usr/share/xml/scap/ssg/content/`. Este data stream contiene la información de todos los perfiles de SCAP con sus reglas asociadas para evaluar el compliance de un scan o también para generar las remediaciones. Esta información está en el fichero `XCCDF` (Extensible Configuration Checklist Description File) `ssg-rhel9-xccdf.xml`.

Por último el paquete `scap-workbench` nos servirá para generar perfiles custom de compliance, como veremos más adelante. Sólo añadir que también permite evaluar el compliance en local o en sistemas remotos a través de `SSH`.

## Evaluar el compliance

Podemos ver todos los perfiles disponibles para evaluar el compliance de nuestro sistema con el comando:

```
[root@client ~]# oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
...
Title: DISA STIG for Red Hat Enterprise Linux 9
				Id: xccdf_org.ssgproject.content_profile_stig
...
```

El profile DISA STIG es el que nosotros usaremos. Para evaluar el compliance usando este estándar y guardar los resultados en un fichero ejecutaremos:

```
[root@client ~]# oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_stig --results /root/results.xml /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
...
Title   Record Events that Modify User/Group Information - /etc/shadow
Rule    xccdf_org.ssgproject.content_rule_audit_rules_usergroup_modification_shadow
Ident   CCE-83725-2
Result  fail

Title   System Audit Directories Must Be Group Owned By Root
Rule    xccdf_org.ssgproject.content_rule_directory_group_ownership_var_log_audit
Ident   CCE-90516-6
Result  pass
...
```

Podemos ver como en el ejemplo para cada una de las políticas del estándar, cuáles han pasado el test y cuáles han fallado.

También podemos generar un report html a partir del anterior fichero de resultados con:

```
[root@client ~]# oscap xccdf generate report /root/results.xml > results.html
```

Si abrimos el report vemos lo siguiente:

![image](https://github.com/user-attachments/assets/d440578a-bf06-42dc-95fc-b731de119987)

## Customizar un perfil de compliance

Para customizar un perfil de compliance hemos de crear un `tailoring file`, para ello ejecutaremos el programa de `scap-workbench`:

```
[root@client ~]# scap-workbench
```

El programa carga los `data streams` que estén instalados en el servidor:

![image](https://github.com/user-attachments/assets/4ba4f310-171d-4471-bf14-bdf204ae759b)

Vamos a `Customize` y seleccionamos un nombre de perfil. A continuación marcamos `Deselect all`, así empezaremos con todas las reglas deshabilitadas:

![image](https://github.com/user-attachments/assets/9ce8155b-bd90-419e-bd21-daa95e5532e3)

Marcaremos las siguientes opciones `Ensure PAM Enforces Password Requirements - Minimum Lenght` y editamos la opción de `minlen` para setearlo a 12:

![image](https://github.com/user-attachments/assets/d98a6e1b-dd4e-405a-96d8-3d8481d7cf61)

![image](https://github.com/user-attachments/assets/07852d99-5b32-4698-8a1a-dbe22c7c07a4)

A continuación marcaremos la opción para instalar `AIDE` y también para hacer que `audit` registre todas las invocaciones de `sudo` definidas en `sudoers` y en `sudoers.d`:

![image](https://github.com/user-attachments/assets/5e20c1f0-1758-411c-a273-f32504f064f8)

Con lo que tendremos un `tailoring file` con estas 4 reglas definidas:

![image](https://github.com/user-attachments/assets/62dca0a2-2aa2-4dff-87c5-b440bcd31e90)

A continuación, vamos a `File -> Save Customization Only` y le damos el nombre de `lab-tailoring.xml` y lo guardamos dentro de `/root`.

Con este `tailoring file`, podemos ejecutar un nuevo scan que sólo compruebe estas 4 reglas que hemos añadido al perfil vacío. Para ello, primero necesitamos obtener el nombre del perfil ya que lo hemos customizado:

```
[root@client ~]# oscap info lab-tailoring.xml 
Document type: XCCDF Tailoring
Imported: 2024-09-12T15:28:28
Benchmark Hint: /tmp/scap-workbench-feqbsL/ssg-rhel9-ds.xml
Profiles:
	Title: ANSSI-BP-028 (enhanced) [CUSTOMIZED]
		Id: **xccdf_org.ssgproject.content_profile_anssi_bp28_enhanced_customized**
```

Vemos que el nombre del perfil nos aparece al final, donde pone Id. Con este nombre de perfil podemos lanzar un scan que compruebe las 4 reglas y guardarlas también en fichero de resultados:

```
[root@client ~]# oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_anssi_bp28_enhanced_customized --tailoring-file lab-tailoring.xml --results /root/lab-results-tailoring.xml /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml 
--- Starting Evaluation ---

Title   Install AIDE
Rule    xccdf_org.ssgproject.content_rule_package_aide_installed
Ident   CCE-90843-4
Result  fail

Title   Ensure PAM Enforces Password Requirements - Minimum Length
Rule    xccdf_org.ssgproject.content_rule_accounts_password_pam_minlen
Ident   CCE-83579-3
Result  fail

Title   Ensure auditd Collects System Administrator Actions - /etc/sudoers
Rule    xccdf_org.ssgproject.content_rule_audit_rules_sudoers
Ident   CCE-90176-9
Result  fail

Title   Ensure auditd Collects System Administrator Actions - /etc/sudoers.d/
Rule    xccdf_org.ssgproject.content_rule_audit_rules_sudoers_d
Ident   CCE-89498-0
Result  fail
```
Vemos que ha fallado el test de las 4 reglas. Para corregirlo, crearemos un `playbook` de Ansible que lo hará automáticamente.

## Generar un playbook para aplicar remediaciones automáticamente

Con el fichero de resultados del anterior apartado `/root/lab-results-tailoring.xml` podemos crear un playbook usando:

```
[root@client ~]# oscap xccdf generate fix --profile xccdf_org.ssgproject.content_profile_anssi_bp28_enhanced_customized --tailoring-file lab-tailoring.xml --fix-type ansible --result-id "" /root/lab-results-tailoring.xml > fix.yml
```

Necesitamos crear un inventario ya que el `playbook` tiene la sección `hosts: all` para limitarlo hemos de añadir lo siguiente `echo client > inventory`, y con esto ya podemos ejecutar el playbook generado:

```
[root@client ~]# ansible-playbook -i inventory fix.yml

PLAY [all] *******************************************************************************

TASK [Gathering Facts] *******************************************************************
ok: [client]

TASK [Ensure aide is installed] **********************************************************
changed: [client]

TASK [Gather the package facts] **********************************************************
ok: [client]

TASK [Ensure PAM Enforces Password Requirements - Minimum Length - Ensure PAM variable minlen is set accordingly] ***
changed: [client]

TASK [Gather the package facts] **********************************************************
ok: [client]

TASK [Check if watch rule for /etc/sudoers already exists in /etc/audit/rules.d/] ********
ok: [client]

TASK [Search /etc/audit/rules.d for other rules with specified key actions] **************
ok: [client]

TASK [Use /etc/audit/rules.d/actions.rules as the recipient for the rule] ****************
ok: [client]

TASK [Use matched file as the recipient for the rule] ************************************
skipping: [client]

TASK [Add watch rule for /etc/sudoers in /etc/audit/rules.d/] ****************************
changed: [client]

TASK [Check if watch rule for /etc/sudoers already exists in /etc/audit/audit.rules] *****
ok: [client]

TASK [Add watch rule for /etc/sudoers in /etc/audit/audit.rules] *************************
changed: [client]

TASK [Gather the package facts] **********************************************************
ok: [client]

TASK [Check if watch rule for /etc/sudoers.d/ already exists in /etc/audit/rules.d/] *****
ok: [client]

TASK [Search /etc/audit/rules.d for other rules with specified key actions] **************
ok: [client]

TASK [Use /etc/audit/rules.d/actions.rules as the recipient for the rule] ****************
skipping: [client]

TASK [Use matched file as the recipient for the rule] ************************************
ok: [client]

TASK [Add watch rule for /etc/sudoers.d/ in /etc/audit/rules.d/] *************************
changed: [client]

TASK [Check if watch rule for /etc/sudoers.d/ already exists in /etc/audit/audit.rules] ***
ok: [client]

TASK [Add watch rule for /etc/sudoers.d/ in /etc/audit/audit.rules] **********************
changed: [client]

PLAY RECAP *******************************************************************************
client                     : ok=18   changed=6    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

Un nuevo scan en el servidor nos debería dar los 4 checks como ok:

```
[root@client ~]# oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_anssi_bp28_enhanced_customized --tailoring-file lab-tailoring.xml --results /root/lab-results-tailoring.xml /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml 
--- Starting Evaluation ---

Title   Install AIDE
Rule    xccdf_org.ssgproject.content_rule_package_aide_installed
Ident   CCE-90843-4
Result  pass

Title   Ensure PAM Enforces Password Requirements - Minimum Length
Rule    xccdf_org.ssgproject.content_rule_accounts_password_pam_minlen
Ident   CCE-83579-3
Result  pass

Title   Ensure auditd Collects System Administrator Actions - /etc/sudoers
Rule    xccdf_org.ssgproject.content_rule_audit_rules_sudoers
Ident   CCE-90176-9
Result  pass

Title   Ensure auditd Collects System Administrator Actions - /etc/sudoers.d/
Rule    xccdf_org.ssgproject.content_rule_audit_rules_sudoers_d
Ident   CCE-89498-0
Result  pass
```


