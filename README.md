# Talk LDAP

1. Regarder le fichier docker-compose
2. Regarder le fichier config/env
3. Les certificats sont auto-signés pour la démo
4. docker-compose up -d
5. ls (ça a créé data et slap.d)
6. regarder le naming context créé depuis le bash:
  docker-compose exec openldap bash
  ldapsearch -x -LLL -s base -b "" namingContexts
7. se connecter avec Apache Directory Studio
  => Paramètres réseau
  => Authentification
8. dc=sleede,dc=com 
  => clic droit, Nouvelle entrée
  => à partir de zéro
  => organizationalUnit
  => RDN: ou=people
  => Terminer
8. ou=people,dc=sleede,dc=com
  => clic droit, Nouvelle entrée
  => à partir de zéro
  => inetOrgPerson
  => RDN: uid=cyril.farre
  => cn (Common Name): Cyril, sn (Surname): Farré
  => Terminer
9. uid=cyril.farre,ou=people,dc=sleede,dc=com
  => clic droit, Nouvel attribut
  => jpegPhoto
  => cyril-fdebddfb.jpg
  => OK
  => Voir la photo : clic droit/éditer la valeur
10. On veut stocker un attribut custom (ex. l'ID github)
  => Fenêtre > Ouvrir la perspective > Éditeur de schéma
  => Projets > Clic droit, nouveau projet de schema
  => Schéma hors ligne
  => OpenLDAP
  
  (on peut voir les schemas actifs sur le serveur avec :
   ldapsearch -LLLQY EXTERNAL -H ldapi:/// -b cn=schema,cn=config "(objectClass=olcSchemaConfig)" dn
  )
  
  => [+] system [+] inetorgperson [+] core
  => Terminer
  => Clic droit / nouveau schema
  => Clic droit / nouveau type d'attribut
  => OID: 1.3.6.1.4.1.57924.1.1
  => Alias: githubId
  => Syntax: Printable string
11. On crée un class custom qui intègre le nouvel attribut
  => Clic droit / nouvelle classe d'objet
  => OID: 1.3.6.1.4.1.57924.0.1
  => Alias: sleedePerson
  => Supérieurs: inetOrgPerson
  => Types d'attribut obligatoires: email
  => Types d'attribut optionnels: githubId
  => Terminer
12. On exporte le schema
  => clic droit /export au format openLDAP
  => [+] sleede
13. Examiner le fichier dans un éditeur de texte
14. Le convertir au format LDIF
  On pourrait le convertir automatiquement avec la procédure ci-dessous :
---
  # copy inside container
  cd ~/Documents/sleede
  sudo mv sleede.schema ~/workspace/talk-ldap/data/
  cd -
  docker-compose exec openldap bash
  cd /var/lib/ldap
  mv sleede.schema /etc/ldap/schema 
  
  # include server schemas into config file
  mkdir /tmp/ldif
  cd /tmp
  ls /etc/ldap/schema/*.schema > ./schema_conv.conf
  sed -i -e 's/^/include /' schema_conv.conf
  
  # remove bogus schema (unused)
  sed -i -e '/collective/d' schema_conv.conf
  
  # convert
  slaptest -f ./schema_conv.conf -F /tmp/ldif/
  
  # view generated file
  cat ldif/cn\=config/cn\=schema/cn\=\{12\}sleede.ldif
  # finally change header and remove bottom lines
---
  Mais on va plutôt l'écrire à la main
  
  ```ldif
dn: cn=sleede,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: sleede
olcAttributeTypes: ( 1.3.6.1.4.1.57924.1.1 
  NAME 'githubId' 
  DESC '' 
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.44 )
olcObjectClasses: ( 1.3.6.1.4.1.57924.0.1 
  NAME 'sleedePerson' 
  DESC '' 
  SUP inetOrgPerson 
  STRUCTURAL 
  MUST email 
  MAY githubId )
  ```
15. Copier le LDIF sur le serveur
  docker-compose exec openldap bash
  cd /etc/ldap/schema
  cat > sleede.ldif << EOF
  ...
  EOF
16. ajouter à openldap
  ldapadd -Q -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/sleede.ldif
17. regarder que ça a bien été ajouté
  ldapsearch -x -LLL -b cn=Subschema -s base '(objectClass=subschema)' +
18. créer une nouvelle sleedePerson dans Apache Directory Studio
