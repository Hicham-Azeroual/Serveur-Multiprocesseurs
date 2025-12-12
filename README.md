# Architecture du Système Multiprocesseurs en Mode Flux/Datagramme

## Vue d'ensemble

Ce système implémente un serveur multiprocesseurs distribué avec les composants suivants:

### 1. Serveur Esclave (serveur_esclave.c)
- **Type**: Démon UDP (datagramme)
- **Port**: Configurable (10001, 10002, 10003 par défaut)
- **Fonctionnalité**:
  - Reçoit des commandes shell via UDP du serveur maître
  - Exécute les commandes avec `system()`
  - Retourne le code de sortie et le message de résultat
  - Boucle infinie en attente de commandes

### 2. Serveur Maître (serveur_maitre.c)
- **Type**: Démon TCP/UDP mixte
- **Port**: 9999 (TCP pour clients)
- **Fonctionnalité**:
  - Charge la liste des serveurs esclaves depuis `slaves.conf`
  - Reçoit les fichiers de commandes des clients (TCP)
  - Distribue les commandes en parallèle aux serveurs esclaves (UDP)
  - Assure la transparence de la distribution du travail

### 3. Client (client.c)
- **Type**: Application cliente
- **Fonctionnalité**:
  - Se connecte au serveur maître via TCP
  - Envoie le nom du fichier de commandes
  - Attend l'exécution des commandes
  - Reçoit les résultats

## Structure des Datagrammes/Messages

### CommandRequest (Esclave ← Maître)
```c
typedef struct {
    char command[1024];      // Commande shell à exécuter
    char client_addr[50];    // Adresse IP du client
    int client_port;         // Port du client
} CommandRequest;
```

### CommandResult (Esclave → Maître)
```c
typedef struct {
    char command[1024];      // Commande exécutée
    int return_code;         // Code de retour de system()
    char result[256];        // Message de résultat
} CommandResult;
```

## Flux d'Exécution

1. **Initialisation**:
   - Le maître charge `slaves.conf` et crée des sockets UDP vers chaque esclave
   - Les esclaves se lancent et se mettent en écoute UDP

2. **Requête Client**:
   - Client se connecte au maître (TCP, port 9999)
   - Client envoie le nom du fichier de commandes
   - Maître confirme avec "OK"

3. **Traitement des Commandes**:
   - Maître lit le fichier ligne par ligne
   - Pour chaque commande, maître sélectionne un esclave disponible
   - Maître envoie CommandRequest via UDP à l'esclave
   - Esclave exécute la commande avec `system()`
   - Esclave retourne CommandResult au maître

4. **Parallélisme**:
   - Avec 3 esclaves, jusqu'à 3 commandes s'exécutent en parallèle
   - Le maître assigne chaque nouvelle commande à un esclave libre

## Configuration

### slaves.conf
Format: `hostname port` (une ligne par esclave)

Exemple:
```
localhost 10001
localhost 10002
localhost 10003
```

### Fichiers de commandes
Un fichier texte avec une commande shell par ligne

Exemples fournis:
- `test_basic.txt` - Commandes basiques (echo, dir, date, etc.)
- `test_parallel.txt` - Commandes parallèles (3-4 commandes pour tester le parallélisme)
- `test_stress.txt` - Test de charge (10 commandes)

## Compilation sur Windows

### Prérequis:
- MinGW GCC (avec support Winsock)
- Windows 7 ou plus récent

### Option 1: Script de compilation (RECOMMANDÉ)
```cmd
cd c:\Users\EliteBook 840 G7\Desktop\tp
compile.bat
```

### Option 2: Compilation manuelle
```cmd
gcc -o serveur_esclave.exe serveur_esclave.c -lws2_32
gcc -o serveur_maitre.exe serveur_maitre.c -lws2_32
gcc -o client.exe client.c -lws2_32
```

## Test du Système

### Étape 1: Démarrer les serveurs
Double-cliquez sur `start_servers.bat` ou exécutez:
```cmd
start_servers.bat
```

Cela va:
1. Compiler automatiquement si nécessaire (au premier lancement)
2. Démarrer 3 serveurs esclaves (ports 10001, 10002, 10003)
3. Démarrer le serveur maître (port 9999)
4. Ouvrir chaque serveur dans sa propre fenêtre

**Note**: Chaque serveur s'affichera dans sa propre fenêtre de console.

### Étape 2: Exécuter le client
Dans une nouvelle fenêtre de commande, naviguez jusqu'au répertoire et lancez:

**Test basique:**
```cmd
client.exe test_basic.txt
```

**Test parallèle:**
```cmd
client.exe test_parallel.txt
```

**Test de charge:**
```cmd
client.exe test_stress.txt
```

### Étape 3: Observer les résultats
- Regardez les fenêtres de console des serveurs esclaves
- Chaque esclave affichera les commandes qu'il reçoit et exécute
- Le maître affichera la distribution des commandes
- Le client affichera le statut de la connexion

### Étape 4: Arrêter les serveurs
Double-cliquez sur `stop_servers.bat` ou:
```cmd
stop_servers.bat
```

## Exemple d'Exécution Complète

```
1. Ouvrir une fenêtre de commande
2. cd c:\Users\EliteBook 840 G7\Desktop\tp
3. compile.bat                    (compilation)
4. start_servers.bat              (démarrage des serveurs)
   - 3 fenêtres s'ouvrent pour les esclaves
   - 1 fenêtre pour le maître
5. Ouvrir une nouvelle fenêtre de commande
6. cd c:\Users\EliteBook 840 G7\Desktop\tp
7. client.exe test_parallel.txt   (exécution du client)
8. Observer les résultats dans les fenêtres des serveurs
9. stop_servers.bat               (arrêt des serveurs)
```

## Fichiers de Test Fournis

### test_basic.txt
Commandes basiques pour vérifier le fonctionnement:
```
echo Test Command 1: Print Hello World
dir
date /T
echo This is a test file > test_output.txt
```

### test_parallel.txt
Teste le parallélisme avec 4 commandes:
- Echo simple
- Directory listing
- IP Config
- Commande finale

### test_stress.txt
Test de charge avec 10 commandes en boucle

## Résultats Attendus

### Console du Master:
```
[Master Server] Maître lancé sur le port 9999 avec 3 esclaves (PID=xxxx)
[Master Server] Nouvelle connexion client: 127.0.0.1:xxxxx
[Master Server] Fichier demandé: test_parallel.txt
[Master Server] Traitement commande: echo Command 1...
[Master Server] Commande envoyée à localhost:10001
[Master Server] Traitement commande: echo Command 2...
[Master Server] Commande envoyée à localhost:10002
...
```

### Console des Esclaves:
```
[Slave Server] Esclave lancé sur le port 10001 (PID=xxxx)
[Slave Server] Reçu commande: echo Command 1...
[Slave Server] Résultat: Commande exécutée avec succès (code=0)
```

### Console du Client:
```
[Client] Connexion au serveur maître 127.0.0.1:9999...
[Client] Connecté au serveur maître
[Client] Fichier 'test_parallel.txt' envoyé au maître
[Client] Maître a accepté les commandes
[Client] Attente de l'exécution des commandes...
[Client] Commandes traitées
```

## Troubleshooting

### Erreur: "Cannot open config file: slaves.conf"
- Vérifiez que `slaves.conf` existe dans le répertoire courant
- Le fichier doit contenir les adresses des serveurs esclaves

### Erreur: "Port already in use"
- Un serveur est peut-être déjà en cours d'exécution
- Exécutez `stop_servers.bat` pour arrêter tous les processus
- Attendez 5-10 secondes avant de relancer

### Erreur: "Cannot open file" (commandes)
- Vérifiez que le fichier de commandes existe
- Les chemins relatifs sont supportés
- Utilisez le chemin complet si nécessaire

### Pas de sortie visuelle
- Vérifiez que les serveurs esclaves affichent les messages
- Vérifiez que le master affiche "Maître lancé..."
- Vérifiez que le client affiche "Connecté au serveur maître"

## Avantages de cette Architecture

1. **Scalabilité**: Facile d'ajouter/enlever des esclaves en modifiant `slaves.conf`
2. **Parallélisme**: Exploit le multiprocesseur naturellement
3. **Transparence**: Le client voit un seul serveur
4. **Robustesse**: Perte d'un esclave = seulement quelques commandes perdues
5. **Flexibilité**: Commandes shell arbitraires supportées

## Limitations et Améliorations Possibles

1. **Pas de persistence**: Perte de la connexion = perte des résultats
2. **Pas de timeout**: Si une commande s'exécute trop longtemps
3. **Ordre d'exécution**: Pas garanti (parallélisme)
4. **Pas d'authentification**: Sécurité minimale
5. **Buffer limité**: Commandes max 1024 caractères

### Améliorations futures:
- Ajouter des timeouts
- Implémenter les statuts d'exécution (en cours, terminé)
- Utiliser TCP au lieu d'UDP pour la fiabilité
- Ajouter de l'authentification
- Implémenter la rééquilibrage de charge
