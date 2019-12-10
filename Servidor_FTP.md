### 1. Tenemos un sitio web y un sitio ftp, queremos que cuando accedamos a www.iesgn.org/documentos, aparecen una lista de documentos (como NAS). Cuando accedamos de forma anónima al FTP, accedemos al mismo directorio /documentos.

###### Instalamos el paquete del servidor proftpd

~~~
vagrant@ServidorApache:~$ sudo apt install proftpd
~~~

###### Descomentamos las lines de anonymous del fichero /etc/proftpd/proftpd.conf y le indicamos el directorio al cual queraamos acceder.

~~~
 <Anonymous /srv/doc>
   User				ftp
   Group			nogroup
   # We want clients to be able to login with "anonymous" as well as "ftp"
   UserAlias			anonymous ftp
   # Cosmetic changes, all files belongs to ftp user
   DirFakeUser	on ftp#
   DirFakeGroup on ftp
 
   RequireValidShell		off
 
   # Limit the maximum number of anonymous logins
   MaxClients			10
 
   # We want 'welcome.msg' displayed at login, and '.message' displayed
   # in each newly chdired directory.
   DisplayLogin			welcome.msg
   DisplayChdir		.message
 
   # Limit WRITE everywhere in the anonymous chroot
   <Directory *>
     <Limit WRITE>
       DenyAll
     </Limit>
   </Directory>
 
   # Uncomment this if you're brave.
   # <Directory incoming>
   #   # Umask 022 is a good standard umask to prevent new files and dirs
   #   # (second parm) from being group and world writable.
   #   Umask				022  022
   #            <Limit READ WRITE>
   #            DenyAll
   #            </Limit>
   #            <Limit STOR>
   #            AllowAll
   #            </Limit>
   # </Directory>
 
 </Anonymous>
~~~

###### reiniciamos el servicio

~~~
vagrant@ServidorApache:~$ sudo systemctl restart proftpd.service 
~~~

###### Instalamos en el cliente el paquete de ftp para realizar la conexión:

~~~
moralg@padano:~$ sudo apt install ftp
~~~

###### Realizamos la conexión:
~~~
moralg@padano:~$ ftp www.iesgn.org
    ftp: connect to address 172.22.4.231: No route to host
    Trying 172.22.8.34...
    Connected to www.iesgn.org.
    220 ProFTPD Server (Debian) [::ffff:172.22.8.34]
    Name (www.iesgn.org:moralg): ftp
    331 Anonymous login ok, send your complete email address as your password
    Password:
    230 Anonymous access granted, restrictions apply
    Remote system type is UNIX.
    Using binary mode to transfer files.
ftp> ls
    200 PORT command successful
    150 Opening ASCII mode data connection for file list
    lrwxrwxrwx   1 ftp#     ftp            27 Dec 10 08:09 pruebaFTP.txt -> /home/vagrant/  pruebaFTP.txt
    226 Transfer complete
~~~

### 2. Crear un virtualhost en Apache y subir por FTP, con un usuario concreto, ficheros a ese virtualhost.

###### Para indicar a ftp los permisos de usuarios concretos tenemos que modificar una linea en el fichero /etc/proftpd/proftpd.conf

~~~
 DefaultRoot                    /srv/www/%u
~~~

~~~
vagrant@ServidorApache:~$ sudo useradd -m -d /srv/www/user1 user1
~~~

~~~
vagrant@ServidorApache:~$ sudo passwd user1
    New password: 
    Retype new password: 
    passwd: password updated successfully
~~~

~~~
vagrant@ServidorApache:~$ sudo nano /etc/apache2/sites-available/user1.conf
~~~

~~~
<VirtualHost *:80>

        ServerName user1.iesgn.org
        ServerAdmin alejandro@localhost
        DocumentRoot /srv/www/user1

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
~~~

~~~
vagrant@ServidorApache:~$ sudo a2ensite user1
    Enabling site user1.
    To activate the new configuration, you need to run:
      systemctl reload apache2
~~~

~~~
vagrant@ServidorApache:~$ sudo chown -R user1:user1 /srv/www/user1/
~~~

~~~
vagrant@ServidorApache:~$ sudo systemctl restart proftpd.service
vagrant@ServidorApache:~$ sudo systemctl restart apache2.service
~~~

~~~
moralg@padano:~$ ftp www.iesgn.org
    ftp: connect to address 172.22.4.231: No route to host
    Trying 172.22.8.34...
    Connected to www.iesgn.org.
    220 ProFTPD Server (Debian) [::ffff:172.22.8.34]
    Name (www.iesgn.org:moralg): user1
    331 Password required for user1
    Password:
    230 User user1 logged in
    Remote system type is UNIX.
    Using binary mode to transfer files.
ftp> ls
    200 PORT command successful
    150 Opening ASCII mode data connection for file list
    -rw-r--r--   1 root     root           14 Dec 10 09:05 pruebaFTP.txt
    226 Transfer complete
ftp> put amorales.gonzalonazareno.org.csr
    local: amorales.gonzalonazareno.org.csr remote: amorales.gonzalonazareno.org.csr
    200 PORT command successful
    150 Opening BINARY mode data connection for amorales.gonzalonazareno.org.csr
    226 Transfer complete
    1809 bytes sent in 0.27 secs (6.5122 kB/s)
~~~
