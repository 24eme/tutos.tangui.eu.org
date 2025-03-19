---
title: "Tester une configuration SSL de OpenLDAP - Debian"
date:  2025-03-19 12:00
layout: post
---

Pour tester si le SSL est activer et consulter le certificat :

     openssl s_client -connect 127.0.0.1:636 -showcerts -state

Si il y a un certificat, voici la commande `ldapsearch` qui permet de tester la connexion :

     ldapsearch -v -H ldaps://127.0.0.1:636

La commande devrait répondre : `SASL/DIGEST-MD5 authentication started` avant de demander un mot de passe.

En cas de mauvaise configuration, les messages suivants pourraient être renvoyés :

 - `ldap_sasl_interactive_bind_s: Can't contact LDAP server` : ldaps:/// n'est pas activé
 - `additional info: The TLS connection was non-properly terminated` : les certificats ne sont pas créés/activés

Enfin, le test pour un login particulier peut se faire avec la commande suivante :

    ldapsearch  -x -D uid=admin,ou=People,dc=domain,dc=com -w <MOT_DE_PASSE_ADMIN> -H ldaps://127.0.0.1 -b  dc=domain,dc=com -x  uid=login

En adaptant :

 - l'identifiant du compte admin (ici `uid=admin,ou=People,dc=domain,dc=com`)
 - le mot de passe de ce compte (ici `<MOT_DE_PASSE_ADMIN>`)
 - l'annuaire ldap interrogé (ici `dc=domain,dc=com`)
 - le login recherché (ici `uid=login`)

## Activer ldaps

Pour activer le SSL, veiller à ce que `ldaps:///` est activé via l'option `SLAPD_SERVICES` de `/etc/default/slapd` :

     $ grep SLAPD_SERVICES /etc/default/slapd
     SLAPD_SERVICES="ldap:/// ldaps:/// ldapi:///"

## Configurer les certificats

Vérifier que openldap est configuré avec un certificat :

    $ sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config -s base | grep olcTLS
    olcTLSCACertificateFile: /etc/ssl/openldap/certs/cacert.pem
    olcTLSCertificateFile: /etc/ssl/openldap/certs/ldapserver-cert.crt
    olcTLSCertificateKeyFile: /etc/ssl/openldap/private/ldapserver-key.key

Si ce n'est pas le cas, la commande ldapmodify permettra de le faire via un fichier ldif.

Voici le contenu du ldif (appelé `ssl.ldif` dans la suite) :


    dn: cn=config
    changetype: modify
    add: olcTLSCACertificateFile
    olcTLSCACertificateFile: /etc/ssl/openldap/certs/cacert.pem
    -
    replace: olcTLSCertificateFile
    olcTLSCertificateFile: /etc/ssl/openldap/certs/ldapserver-cert.crt
    -
    replace: olcTLSCertificateKeyFile
    olcTLSCertificateKeyFile: /etc/ssl/openldap/private/ldapserver-key.key

Il faut adapter les chemins des certificats/clés en fonction de ceux créés.

Une fois le fichier créé, voici comment l'intégrer dans la configuration :

    sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f ssl.ldif


