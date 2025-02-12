# Shibboleth

Um Shibboleth mit RMDO zu verwenden, ist es notwendig in der Betriebsumgebung einen Apache2-Service bereitzustellen. Das Setup ist [hier](../../../../deployment/apache.html) dokumentiert.

Als nächstes installieren Sie das Shibboleth Apache Modul für den Service-Provider (SP) aus ihrem Distributionsrepositorium, z.B. für Debian/Ubuntu:

```bash
sudo apt-get install libapache2-mod-shib2
```

Unter Ubuntu 18.04 LTS ist leider das `libapache2-mod-shib2` Paket defekt. Eine funktionierende Anleitung findet sich unter https://www.switch.ch/aai/guides/sp/installation/?os=ubuntu.

Zusätzlich muss der [Django-shibboleth-remoteuser](https://github.com/Brown-University-Library/django-shibboleth-remoteuser) in der virtuellen RDMO-Umgebung installiert werden:

```bash
pip install -r requirements/shibboleth.txt
```

Konfigurieren Sie ihren Shibboleth Service-Anbieter mithilfe der Dateien in `/etc/shibboleth/`. Dies kann zwischen den Identity-Providern variieren. Für RDMO muss der `REMOTE_SERVER` gesetzt sein, sowie vier weitere Attribute ihres Identity-Providers:

* ein Username (meistens `eppn`)
* eine E-Mail-Addresse (meistens `mail` oder `email`)
* ein Vorname (meistens `givenName`)
* ein Familienname (meistens `sn`)

In unserer Testumgebung kann dies durch das Verändern von `/etc/shibboleth/shibboleth2.xml` erreicht werden:

```xml
<ApplicationDefaults entityID="https://sp.vbox/shibboleth"
    REMOTE_USER="uid eppn persistent-id targeted-id">
```

und `/etc/shibboleth/attribute-map.xml`:

```xml
<Attribute name="urn:oid:0.9.2342.19200300.100.1.1" id="uid"/>
<Attribute name="urn:oid:2.5.4.4" id="sn"/>
<Attribute name="urn:oid:2.5.4.42" id="givenName"/>
<Attribute name="urn:oid:0.9.2342.19200300.100.1.3" id="mail"/>
```

Starten Sie den Shibboleth Service neu:

```bash
service shibd restart
```

In ihrer virtuellen Host-Konfiguration (Apache2) fügen Sie folgendes hinzu:

```
<Location /Shibboleth.sso>
    SetHandler shib
</Location>
<LocationMatch /(account|domain|options|projects|questions|tasks|conditions|views)>
    AuthType shibboleth
    require shibboleth
    ShibRequireSession On
    ShibUseHeaders On
</LocationMatch>
```

In ihrer `config/settings/local.py` fügen Sie Folgendes hinzu oder entfernen die Kommentarsymbole:

```python
from rdmo.core.settings import INSTALLED_APPS, AUTHENTICATION_BACKENDS, MIDDLEWARE_CLASSES

SHIBBOLETH = True
PROFILE_UPDATE = False
PROFILE_DELETE = False

INSTALLED_APPS += ['shibboleth']

AUTHENTICATION_BACKENDS.append('shibboleth.backends.ShibbolethRemoteUserBackend')
MIDDLEWARE_CLASSES.insert(
    MIDDLEWARE_CLASSES.index('django.contrib.auth.middleware.AuthenticationMiddleware') + 1,
    'shibboleth.middleware.ShibbolethRemoteUserMiddleware'
)

SHIBBOLETH_ATTRIBUTE_MAP = {
    'uid': (True, 'username'),
    'givenName': (True, 'first_name'),
    'sn': (True, 'last_name'),
    'mail': (True, 'email'),
}

LOGIN_URL = '/Shibboleth.sso/Login?target=/projects'
LOGOUT_URL = '/Shibboleth.sso/Logout'
```

wobei die Werte von `SHIBBOLETH_ATTRIBUTE_MAP`, `LOGIN_URL`, und `LOGOUT_URL` entsprechend ihrem Setup geändert werden müssen. Die Einstellung `SHIBBOLETH = True` deaktiviert das reguläre Login-Formular von RDMO und deaktiviert das Udpateformular für das Benutzerprofil, so dass der Benutzer seine Zugangsdaten nicht mehr ändern kann. Durch `PROFILE_DELETE = False` wird die Seite deaktiviert, auf der der Nutzer sein Profil löschen kann. Da es RDMO unmöglich ist auf die Shibboleth Nutzerdatenbank schreibend zuzugreifen, sollten die beiden genannten Seiten auf jeden Fall deaktiviert werden. Alles andere ist aus technischen Gründen nicht ratsam. Die `INSTALLED_APPS`, `AUTHENTICATION_BACKENDS`, und `MIDDLEWARE_CLASSES` Einstellungen erlauben es, den Django-Shibboleth-Remoteuser mit RDMO zu verwenden.

Starten Sie den Webserver neu.

```bash
service apache2 restart
```

Certain Attributes from the Shibboleth Identity Provider can be mapped to Django groups, in particular to restrict the access to Catalogs and Views. This can be done by adding the following settings:

```
SHIBBOLETH_GROUP_ATTRIBUTES = ['eduPersonScopedAffiliation']
```

In this case, the attribute `eduPersonScopedAffiliation` contains a comma seperated list of groups which will be created in RDMO (if they don't exist yet) and the user will be added into these groups. Due to a limitation by [django-shibboleth-remoteuser](https://github.com/Brown-University-Library/django-shibboleth-remoteuser) the user will also be **removed from all other groups**.


```eval_rst
.. warning::
    Das folgende Feature is noch nicht Teil der veröffentlichten Version von RDMO. Es wird Teil eines späteren Releases.
```

Sie können auch Attribute des Shibboleth Identity Provider auf Django Gruppen abbilden, insbesondere um den Zugang zu Katalogen und Ansichten zu kontrollieren. Hierfür werden die folgenden Einstellunge benötigt:

```
SHIBBOLETH_GROUP_ATTRIBUTES = ['eduPersonScopedAffiliation']
```

In diesem Fall, enhält das Attribut `eduPersonScopedAffiliation` eine Komma-separierte Liste von Gruppen, die nach RDMO gespiegelt werden. Aufgrund einer Limitierung von [django-shibboleth-remoteuser](https://github.com/Brown-University-Library/django-shibboleth-remoteuser) wird der User **aus allen anderen Gruppen entfernt**.
