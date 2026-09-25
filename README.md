
Ce projet a pour objectif de simplifier et fiabiliser l'installation d'une application sur plusieurs serveurs Rocky Linux.

Comment le lancer :

 eval "$(ssh-agent -s)"
 ssh-add ~/.ssh/id_ed25519

 ansible -i inventory.ini serveurs -m ping

 ansible-playbook -i inventory.ini install.yml



L'automatisation prend notamment en charge :

    🔐 Connexion aux serveurs via SSH

    🐧 Vérification du système Rocky Linux

    💾 Vérification de l'espace disque

    📁 Vérification des répertoires nécessaires

    📦 Transfert de l'archive ZIP

    🗜️ Décompression de l'application

    🛑 Arrêt du service avant installation

    ⚙️ Exécution de install.sh

    📝 Capture des logs d'installation

    🔎 Recherche automatique des erreurs

    ▶️ Redémarrage du service

    📊 Affichage du résultat final

L'objectif est de bloquer automatiquement une installation lorsqu'un prérequis critique n'est pas respecté.
🏗️ Architecture

                         ┌─────────────────────┐
                         │    Connexion SSH     │
                         │  server1 / server2   │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Vérification Rocky  │
                         │       Linux         │
                         └──────────┬──────────┘
                                    │
                                    ▼
                      ┌──────────────────────────┐
                      │ Vérification des chemins │
                      │                          │
                      │       /reception         │
                      │          /var            │
                      └────────────┬─────────────┘
                                   │
                     ┌─────────────┴─────────────┐
                     │                           │
                  > 95 %                      ≤ 95 %
                     │                           │
                     ▼                           ▼
              ┌─────────────┐           Vérification > 80 %
              │     STOP    │                    │
              │   Message   │          ┌─────────┴─────────┐
              └─────────────┘          │                   │
                                    > 80 %              ≤ 80 %
                                       │                   │
                                       ▼                   ▼
                                ┌─────────────┐      Transfert ZIP
                                │    STOP     │           │
                                └─────────────┘           ▼
                                                   Décompression
                                                         │
                                                         ▼
                                                   Arrêt service
                                                         │
                                                         ▼
                                                    install.sh
                                                         │
                                                         ▼
                                                   Capture logs
                                                         │
                                                         ▼
                                             Recherche des erreurs
                                             ERR / ERROR / ERREUR
                                                         │
                                                         ▼
                                               Redémarrage service
                                                         │
                                                         ▼
                                            Résultat installation

📁 Arborescence

ansible-install/
│
├── inventory.ini
│
├── install.yml
│
└── files/
    └── file.zip

inventory.ini

Fichier d'inventaire Ansible contenant les serveurs cibles.

Exemple :

[serveurs]
server1 ansible_host=192.168.1.101
server2 ansible_host=192.168.1.102

Le groupe utilisé par le playbook est :

serveurs

install.yml

Playbook Ansible principal.

Le playbook utilise notamment les paramètres suivants :

- name: Installation de file.zip sur les serveurs Rocky Linux
  hosts: serveurs
  serial: 1
  become: true

Signification
Paramètre	Description
name	Nom du playbook affiché pendant l'exécution
hosts	Groupe de serveurs ciblés dans inventory.ini
serial: 1	Installation d'un seul serveur à la fois
become: true	Exécution des tâches avec élévation de privilèges

L'utilisation de serial: 1 permet notamment d'éviter de lancer simultanément l'installation sur tous les serveurs.

server1
   │
   ▼
Installation
   │
   ▼
Terminé
   │
   ▼
server2
   │
   ▼
Installation
   │
   ▼
Terminé

files/file.zip

Archive contenant les fichiers nécessaires à l'installation de l'application.

files/
└── file.zip

    💡 Pour des fichiers volumineux ou des applications distribuées à grande échelle, 
    il peut être préférable d'utiliser un dépôt d'artefacts plutôt que de versionner directement l'archive dans Git.

⚙️ Fonctionnement
1. Connexion SSH

Ansible se connecte aux serveurs définis dans inventory.ini.

Ansible
   │
   ├── SSH → server1
   │
   └── SSH → server2

La clé SSH doit être disponible pour permettre à Ansible de se connecter aux serveurs.
2. Vérification du système

Le playbook vérifie que le serveur cible utilise bien Rocky Linux.

Si le système ne correspond pas au système attendu, l'installation est interrompue.

Rocky Linux ?
     │
 ┌───┴────┐
Oui       Non
 │         │
 ▼         ▼
Continue  STOP

3. Vérification des répertoires

Les répertoires nécessaires au processus sont contrôlés :

/reception
/var

Le playbook vérifie également leur disponibilité avant de commencer l'installation.
💾 Contrôles d'espace disque

Une partie importante du playbook consiste à empêcher une installation lorsque l'espace disque disponible est insuffisant.
Seuil critique : 95 %

Lorsque l'utilisation du disque dépasse 95 %, l'installation est immédiatement arrêtée.

Utilisation disque

       > 95 %
          │
          ▼
    🛑 STOP INSTALLATION

Un message explicite est retourné afin d'indiquer la raison de l'arrêt.
Seuil d'avertissement : 80 %

Lorsque l'utilisation est comprise entre 80 % et 95 %, une vérification supplémentaire est effectuée avant de continuer.

          ≤ 95 %
             │
             ▼
       Vérification
             │
      ┌──────┴──────┐
      │             │
   > 80 %         ≤ 80 %
      │             │
      ▼             ▼
    STOP          Continue

L'objectif est d'éviter de démarrer une installation alors que l'espace disque restant risque d'être insuffisant.
📦 Transfert de l'application

Lorsque toutes les vérifications sont validées, l'archive est transférée vers le serveur cible.

files/file.zip
      │
      │ Ansible
      ▼
/reception/file.zip

L'archive peut ensuite être décompressée sur le serveur.
🗜️ Décompression

Une fois le transfert terminé :

file.zip
   │
   ▼
Décompression
   │
   ▼
Fichiers de l'application

Les fichiers nécessaires à l'installation sont alors disponibles sur le serveur.
🛑 Arrêt du service

Avant de lancer l'installation, le service concerné est arrêté afin d'éviter qu'il utilise des fichiers pendant leur remplacement ou leur mise à jour.

Service actif
     │
     ▼
STOP SERVICE
     │
     ▼
Installation

⚙️ Exécution de install.sh

Le script d'installation est ensuite exécuté :

./install.sh

Le playbook récupère la sortie du script afin de pouvoir analyser son résultat.
📝 Capture des logs

Les logs générés pendant l'installation sont capturés par Ansible.

Ils permettent notamment de :

    suivre le déroulement de l'installation ;

    identifier les étapes exécutées ;

    détecter les messages d'erreur ;

    faciliter le diagnostic en cas d'échec.

🔎 Détection automatique des erreurs

Après l'exécution du script, les logs sont analysés à la recherche de termes indiquant une erreur.

Les chaînes recherchées sont notamment :

ERR
ERROR
ERREUR

Exemple :

Installation
     │
     ▼
Capture des logs
     │
     ▼
Recherche :
  ├── ERR
  ├── ERROR
  └── ERREUR
     │
     ▼
Résultat

    ⚠️ La présence d'un mot comme ERROR dans un log ne signifie pas nécessairement que l'installation a échoué. Le code retour du script et le contexte du message doivent idéalement être pris en compte.

▶️ Redémarrage du service

Une fois l'installation terminée et les contrôles effectués, le service est redémarré.

Installation
     │
     ▼
Analyse logs
     │
     ▼
Redémarrage service
     │
     ▼
Installation terminée

🚀 Utilisation
1. Se placer dans le projet

cd ansible-install

2. Charger la clé SSH dans ssh-agent

Avant d'utiliser Ansible, charger la clé SSH utilisée pour se connecter aux serveurs :

eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

Vérifier que la clé est bien chargée :

ssh-add -l

La clé SSH est alors disponible pour les connexions SSH effectuées par Ansible.
3. Tester la connexion aux serveurs

Avant de lancer l'installation, vérifier que tous les serveurs du groupe serveurs sont accessibles :

ansible -i inventory.ini serveurs -m ping

Résultat attendu :

server1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

server2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

Si les serveurs répondent avec :

"ping": "pong"

la connexion Ansible est fonctionnelle.
4. Lancer le playbook

Une fois la connexion validée, lancer l'installation :

ansible-playbook -i inventory.ini install.yml

Le playbook va alors exécuter automatiquement les différentes étapes :

Connexion SSH
      │
      ▼
Vérification Rocky Linux
      │
      ▼
Vérification des répertoires
      │
      ▼
Vérification espace disque
      │
      ▼
Transfert file.zip
      │
      ▼
Décompression
      │
      ▼
Arrêt du service
      │
      ▼
Exécution install.sh
      │
      ▼
Capture des logs
      │
      ▼
Analyse des erreurs
      │
      ▼
Redémarrage du service
      │
      ▼
Résultat

5. Afficher davantage de détails

Pour obtenir davantage d'informations pendant l'exécution, utiliser l'option -vv :

ansible-playbook -i inventory.ini install.yml -vv

Le mode -vv est particulièrement utile pour le diagnostic.

Il permet notamment de visualiser davantage d'informations concernant :

    les tâches exécutées ;

    les commandes lancées ;

    les résultats retournés ;

    les variables utilisées ;

    les éventuelles erreurs.

Pour un niveau de détail encore supérieur :

ansible-playbook -i inventory.ini install.yml -vvv

🧪 Mode Dry Run

Avant une installation réelle, il est possible d'utiliser le mode check d'Ansible :

ansible-playbook \
  -i inventory.ini \
  install.yml \
  --check

Cela permet de détecter certains problèmes sans appliquer les modifications.

    ⚠️ Le mode --check ne reproduit pas parfaitement toutes les opérations, notamment certaines commandes shell ou interactions avec des services.

📊 Résultat de l'installation

À la fin du playbook, un résultat est affiché pour chaque serveur.

Exemple :

========================================
 Installation
========================================

Serveur : server1
OS      : Rocky Linux
Disque  : OK
Archive : OK
Install : OK
Logs    : Aucun ERROR détecté
Service : RUNNING

Résultat : SUCCESS
========================================

En cas de problème :

========================================
 Installation
========================================

Serveur : server2
OS      : Rocky Linux
Disque  : 96%

Installation interrompue.

Raison :
Espace disque supérieur au seuil critique.

Résultat : FAILED
========================================

🧰 Pré-requis
Machine de contrôle

La machine depuis laquelle Ansible est exécuté doit disposer de :

    Ansible ;

    Python ;

    SSH ;

    accès réseau aux serveurs cibles ;

    une clé SSH permettant l'accès aux serveurs.

Vérifier l'installation d'Ansible :

ansible --version

Serveurs cibles

Les serveurs doivent :

    utiliser Rocky Linux ;

    être accessibles en SSH ;

    disposer d'un utilisateur autorisé à exécuter les opérations nécessaires ;

    disposer de sudo ;

    avoir les commandes nécessaires à la décompression et à l'installation.

🔐 Configuration SSH

Tester manuellement la connexion SSH si nécessaire :

ssh server1

Puis vérifier que la clé est bien chargée :

ssh-add -l

🐛 Débogage

Pour obtenir davantage d'informations :

ansible-playbook -i inventory.ini install.yml -vv

Ou en mode très détaillé :

ansible-playbook -i inventory.ini install.yml -vvv

Les niveaux de verbosité Ansible sont particulièrement utiles pour identifier :

    une erreur SSH ;

    un problème de permissions ;

    une tâche qui échoue ;

    une commande distante incorrecte ;

    un problème de chemin ;

    un problème avec sudo.

🔒 Bonnes pratiques
Ne pas stocker les secrets dans Git

Éviter de mettre directement dans le repository :

password
private_key
token
API_KEY
SECRET

Pour les variables sensibles, privilégier Ansible Vault.

Exemple :

ansible-vault create secrets.yml

Puis :

ansible-playbook \
  -i inventory.ini \
  install.yml \
  --ask-vault-pass

🔑 Utiliser des clés SSH

L'utilisation d'une clé SSH est recommandée pour les automatisations.

La clé peut être chargée dans ssh-agent avec :

eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

📦 Ne pas versionner les gros fichiers

Si file.zip est volumineux, il est préférable d'utiliser :

    un repository d'artefacts ;

    un stockage objet ;

    un serveur HTTP interne ;

    un système de releases.

Le repository Git doit idéalement rester léger.
📈 Évolutions possibles

Quelques pistes d'amélioration :

    Ajouter une vérification de la version de Rocky Linux

    Vérifier automatiquement les dépendances

    Ajouter un rollback en cas d'échec

    Sauvegarder la version précédente

    Ajouter une vérification du statut du service

    Ajouter une vérification HTTP après installation

    Ajouter une gestion des versions de l'application

    Ajouter des tags Ansible

    Ajouter Ansible Vault

    Ajouter des handlers Ansible

    Ajouter une gestion centralisée des logs

    Intégrer le playbook dans une CI/CD

    Remplacer l'archive locale par un repository d'artefacts

    Ajouter une stratégie de rollback automatique

🔄 Flux global

┌──────────────────────┐
│      INVENTORY       │
│ server1 / server2    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│    Connexion SSH     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Vérification Rocky   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Vérification disque  │
└──────────┬───────────┘
           │
      ┌────┴────┐
      │         │
    >95%      ≤95%
      │         │
     STOP       ▼
          Vérification >80%
               │
          ┌────┴────┐
          │         │
        >80%       ≤80%
          │         │
         STOP       ▼
              Transfert ZIP
                   │
                   ▼
              Décompression
                   │
                   ▼
              Stop service
                   │
                   ▼
                install.sh
                   │
                   ▼
               Capture logs
                   │
                   ▼
             Analyse erreurs
                   │
                   ▼
             Start service
                   │
                   ▼
              🎉 Résultat

👨‍💻 Auteur

Projet d'automatisation de déploiement basé sur Ansible et destiné aux environnements Rocky Linux.
