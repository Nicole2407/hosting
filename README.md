# hosting
hosting linux
#!/bin/bash

prefijo="usuario"
sitio_prefijo="aso-site"
i=1

# Buscar siguiente usuario disponible
while id -u "${prefijo}$(printf "%02d" $i)" >/dev/null 2>&1; do
    i=$((i+1))
done

usuario="${prefijo}$(printf "%02d" $i)"
sitio="${sitio_prefijo}$(printf "%02d" $i).com"

password=$(openssl rand -base64 12 | cut -c1-10)
# Crear usuario sin shell
sudo useradd -m -d /home/$usuario -s /usr/sbin/nologin -G www-data $usuario
echo "$usuario:$password" | sudo chpasswd
echo "$usuario" | sudo tee -a /etc/vsftpd.chroot_list > /dev/null

# Cambiar shell a bash para pruebas si se desea luego entrar
sudo usermod -s /bin/bash $usuario


# Crear estructura web
sudo mkdir -p /home/$usuario/public_html

sudo tee /home/$usuario/public_html/index.html > /dev/null <<EOF

<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Bienvenido Usuario</title>
<style>
  body, html {
    margin: 0; padding: 0; height: 100%;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    overflow: hidden;
    background:
      linear-gradient(rgba(0,0,0,0.85), rgba(0,0,0,0.85)),
      url('https://www.wnpower.com/blog/wp-content/uploads/sites/3/2019/11/que-es-hosting-y-dominio.jpg') no-repeat center center/cover;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
  }

  .text-container {
    font-size: 6rem;
    font-weight: 900;
    letter-spacing: 10px;
    text-transform: uppercase;
    user-select: none;
    position: relative;
    color: white;

    /* Texto con brillo azul oscuro */
    background: linear-gradient(
      270deg,
      #ffffff,
      #0a3d62,
      #ffffff
    );
    background-size: 600% 600%;
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    animation: shine 3s ease-in-out infinite,
               dropIn 1.5s ease forwards;
    opacity: 0;
    transform: translateY(-100px);
  }
  @keyframes shine {
    0% {
      background-position: 0% 50%;
    }
    50% {
      background-position: 100% 50%;
    }
    100% {
      background-position: 0% 50%;
    }
  }

  @keyframes dropIn {
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  /* Responsividad */
  @media (max-width: 768px) {
    .text-container {
      font-size: 3.5rem;
      letter-spacing: 5px;
      text-align: center;
      padding: 0 15px;
    }
  }
</style>
</head>
<body>
  <div class="text-container" id="welcomeText">
    Bienvenido Usuario $(printf "%02d" $i)
  </div>
</body>
</html>

EOF

#echo <h1>Bienvenido $usuario al sitio $sitio</h1>" | sudo tee /home/$usuario/public_html/index.html > /dev/null

#echo <h1>Bienvenido a $sitio</h1>" | sudo tee /home/$usuario/public_html/index.html > /dev/null


# Permisos
sudo chown -R $usuario:www-data /home/$usuario
sudo chmod 755 /home/$usuario
sudo chmod -R 755 /home/$usuario/public_html

# Crear sitio Nginx
site_file="/etc/nginx/sites-available/$sitio"
sudo tee $site_file > /dev/null <<EOF

erver {

    listen 80;
    server_name $sitio;
    root /home/$usuario/public_html;
    index index.html index.php;

    location / {
        try_files \$uri \$uri/ =404;
    }

    # Configuración para phpMyAdmin
    location /phpmyadmin {
        root /usr/share/;
        index index.php index.html index.htm;

        location ~ ^/phpmyadmin/(.+\.php)$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;  # ⚠️ Ajusta esta línea según tu versión de PHP
            fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;  # Debería funcionar correctamente
            include fastcgi_params;
        }

        location ~* ^/phpmyadmin/(.+\.(jpg|jpeg|gif|css|png|js|ico|html|xml|txt))$ {
            root /usr/share/;
        }
    }
    # Bloque genérico para manejar archivos PHP
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;  # Asegúrate de que esté configurado correctamente
    }

    location ~ /\.ht {
        deny all;
    }
}
EOF

###################

# Crear base de datos en MySQL/MariaDB
sudo mysql -e "CREATE DATABASE ${usuario}_db;"
sudo mysql -e "CREATE USER '${usuario}'@'localhost' IDENTIFIED BY '${password}';"
sudo mysql -e "GRANT SELECT, INSERT, UPDATE, DELETE ON ${usuario}_db.* TO '${usuario}'@'localhost';"
sudo mysql -e "FLUSH PRIVILEGES;"

####################


# Activar sitio

sudo ln -s $site_file /etc/nginx/sites-enabled/
#phpmyadmin acceso
# Recargar Nginx
sudo ln -s /usr/share/phpmyadmin /home/$usuario/public_html/phpmyadmin
sudo nginx -t && sudo systemctl reload nginx


# Mostrar resultado
echo "✅ Usuario y sitio creados"
echo "👤 Usuario FTP: $usuario"
echo "🔑 Contraseña: $password"
echo "🌍 Agrega esto al /etc/hosts de tu máquina:"
echo "127.0.0.1   $sitio"
echo "Luego accede a: http://$sitio:8021"
echo "Luego accede a: http://$sitio/phpmyadmin"



# Guardar usuario y contraseña

echo "$usuario:$password" | sudo tee -a /root/usuarios_creados.txt > /dev/null
