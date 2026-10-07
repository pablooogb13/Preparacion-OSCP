## Jeeves

## Índice

- [Enumeración](#enumeración)
- [Acceso inicial](#acceso-inicial)
- [Escalada de privilegios](#escalada-de-privilegios)
- [Conclusión](#conclusión)

## Enumeración

Puertos encontrados:
![nmap](image-2.png)

Analizando el puerto 80, no hemos sacado nada.
![80](image-4.png)

Realizando fuzzing al puerto 50000 encontramos lo siguiente: 

![wfuzz](image-3.png)

![jeenkins](image-5.png)

Si seleccionamos `Manage Jenkins`, hay una opción que permite ejecutar comandos: `Script Console`.
![script console](image-6.png)

Desde aquí vamos a conseguir acceso a la shell. 


## Acceso inicial

Para acceder a la shell, tendremos que ponernos en escucha desde nuestra máquina atacante y, a continuacion, ejecutar el siguiente código que hemos encontrado en github sobre reverse shell en jenkins. Se trata de Groovy script:

![groovy](image-7.png)

Una vez dentro, buscamos la primera flag, `user.txt`. Sabemos que hay un usuario llamado `kohsuke`, por tanto, vamos a buscar su directorio.
![user_flag](image-8.png)


## Escalada de privilegios

Para la escalada de privilegios, encontramos en el directorio `Documents` de `kohsuke` un archivo llamado `CEH.kdbx`.

CEH.kdbx es muy probablemente una base de datos de contraseñas de KeePass.

.kdbx → formato de base de datos de KeePass.
CEH → probablemente el nombre que le pusieron a la base de datos.
Puede contener usuarios, contraseñas, URLs, notas, etc., protegidos mediante cifrado.

Por tanto, lo transferimos a nuestra máquina atacante y lo abrimos con KeePassXC:

```bash
keepassxc CEH.kdbx
```

Nos pide una contraseña, por lo que tenemos que crackear el archivo para obtenerla. Lo hacemos de la siguiente forma:
- Con keepass2john convertimos el archivo en un hash, y con john lo crackeamos

![keepass](image-9.png)

Una vez conseguida la contraseña, accedemos a la base de datos de KeePass y obtenemos un hash interesante en `Backup stuff`.
![stuff](image-10.png)
![stuff2](image-11.png)

Con NetExec podemos proporcionar un hash para comprobar si conseguimos acceso a los recursos compartidos como `Administrator`:

En NetExec, -H significa que vas a proporcionar un hash NTLM en lugar de una contraseña.

```bash
nxc smb <IP> -u Administrator -H <NTLM_HASH> --shares
```

Es decir:

- `-u`: usuario.
- `-p`: contraseña.
- `-H`: hash NTLM.
- `--shares`: enumera los recursos SMB.
Esto se conoce como Pass-the-Hash (PtH): autenticarte usando el hash sin necesitar conocer la contraseña original.

![pass_the_hash](image-12.png)

Como tenemos acceso, podemos probar a obtener una shell de Windows con privilegios usando `impacket-psexec`:

`impacket-psexec` sirve para ejecutar comandos remotamente en una máquina Windows mediante SMB, normalmente aprovechando credenciales válidas o un hash NTLM.

![privileges](image-13.png)

Para encontrar la root_flag, nos movemos hasta el directorio de Administrador y vemos que nos dice que busquemos más a fondo. 

En CMD de Windows:

```cmd
dir /r
```

muestra los archivos y, además, sus Alternate Data Streams (ADS).

Podemos ver que hay un ADS y leerlo usando el comando `more`.

![root_flag](image-1.png)

## Conclusión

Skills: 

Jenkins Exploitation (Groovy Script Console)

PassTheHash (Psexec) 

Breaking KeePass Alternate Data Streams (ADS)
