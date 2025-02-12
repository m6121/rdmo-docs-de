# Apache und mod_wsgi

Im Betrieb sollten Sie einen dedizierten Benutzer-Account für RDMO erstellen. Alle Schritte der Installation, für die Sie keine Root-Rechte benötigen, sollten mit diesem Benutzer ausgeführt werden. Wie überall gehen wir davon aus, dass der Benutzer `rdmo` genannt wurde und sein Home-Verzeichnis `/srv/rdmo` ist. Demnach ist ihre `rdmo-app` unter `/srv/rdmo/rdmo-app` zu finden.

Installieren Sie den Apache Server und `mod_wsgi` unter Debian oder Ubuntu mit:

```bash
sudo apt install apache2 libapache2-mod-wsgi-py3
```

Unter CentOS7 müssen sie zunächst das [IUS repositiry](https://ius.io/) einbinden. Dann könne sie Apache und `mod_wsgi` wie folgt instalieren:

```
sudo yum install httpd python35u-mod_wsgi
```

Als nächstes erstellen Sie eine virtuelle Host-Konfiguration. Leider werden verschiedene Versionen von apache und mod-wsgi bei den verschiedenen Distributionen verwendet und daher ist jeweils ein leicht unterschiedliches Setup notwendig:

Für Debian/Ubuntu:

```
# in /etc/apache2/sites-available/000-default.conf
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        Alias /static /srv/rdmo/rdmo-app/static_root/
        <Directory /srv/rdmo/rdmo-app/static_root/>
            Require all granted
        </Directory>

        WSGIDaemonProcess rdmo user=rdmo group=rdmo \
            home=/srv/rdmo/rdmo-app python-home=/srv/rdmo/rdmo-app/env
        WSGIProcessGroup rdmo
        WSGIScriptAlias / /srv/rdmo/rdmo-app/config/wsgi.py process-group=rdmo
        WSGIPassAuthorization On

        <Directory /srv/rdmo/rdmo-app/config/>
            <Files wsgi.py>
                Require all granted
            </Files>
        </Directory>
</VirtualHost>
```

Für CentOS 7:

```
# in /etc/httpd/conf.d/vhosts.conf                 on RHEL/CentOS
<VirtualHost *:80>
        ServerAdmin webmaster@localhost

        DocumentRoot /var/www/html/

        Alias /static /srv/rdmo/rdmo-app/static_root/
        <Directory /srv/rdmo/rdmo-app/static_root/>
            Require all granted
        </Directory>

        WSGIDaemonProcess rdmo user=rdmo group=rdmo home=/srv/rdmo/rdmo-app \
            python-path=/srv/rdmo/rdmo-app:/srv/rdmo/rdmo-app/env/lib/python2.7/site-packages
        WSGIProcessGroup rdmo
        WSGIScriptAlias / /srv/rdmo/rdmo-app/config/wsgi.py process-group=rdmo
        WSGIPassAuthorization On

        <Directory /srv/rdmo/rdmo-app/config/>
            <Files wsgi.py>
                Require all granted
            </Files>
        </Directory>
</VirtualHost>
```

Starten Sie den Apache-Server neu: `sudo /etc/init.d/apache2 restart` oder `sudo service apache2 restart` . RDMO sollte nun unter `YOURDOMAIN` verfügbar sein. Beachten Sie, dass der Apache-Benutzer Zugang zu `/srv/rdmo/rdmo-app/static_root/` haben muss.

Wie Sie der virtuellen Host-Konfigurationen entnehmen können, werden die statischen Inhalte wie CSS und JavaScript unabhängig vom WSGI-Python-Skript bedient. Um dies zu erreichen, müssen diese in dem `static_root`-Ordner innerhalb der virtuellen Umgebung erfasst werden:

```bash
python manage.py collectstatic
```

Um Veränderungen am RDMO -Code vorzunehmen (z.B: nach dem [Upgrade](../../upgrade/index.html)), muss der Webserver neu geladen werden oder die Datei `config/wsgi.py` muss (scheinbar) verändert worden sein indem der `touch`-Befehl verwendet wird:

```bash
touch config/wsgi.py
```

Außerdem muss das `collectstatic`-Kommando innerhalb der virtuellen Umgebung neu ausgeführt werden. Beide Schritte können mit dem folgenden Kommando kombiniert ausgeführt werden:

```bash
python manage.py deploy
```
