
# 🔐 Synchroniser automatiquement KeePassXC avec le cloud

## 🌟 Objectif

Ce guide vous explique **pas à pas** comment configurer votre ordinateur Linux pour **synchroniser automatiquement** votre base de données KeePassXC avec le cloud (dans mon cas **OneDrive**) à l’aide d’un script Bash et de deux services `systemd`.

Cela vous permettra d’utiliser KeePassXC sur votre PC **et** sur votre téléphone via l'application mobile comme *KeePass2Android* 📱, en gardant **tout synchronisé automatiquement** ✅

---

## 🧰 Ce dont vous avez besoin

### 🟢 Logiciels à installer

| Logiciel                    | Description                                      | Installation                                                                  |
| --------------------------- | ------------------------------------------------ | ----------------------------------------------------------------------------- |
| 🧠 **KeePassXC**             | Le gestionnaire de mots de passe libre           | via le gestionnaire de paquets ou [keepassxc.org](https://keepassxc.org)      |
| ☁️ **rclone**                | Pour accéder à OneDrive depuis Linux             | via le gestionnaire de paquets ou [rclone.org](https://rclone.org/downloads/) |
| 🔔 `notify-send` (optionnel) | Pour recevoir des notifications sur votre bureau | Paquet : `libnotify-bin` (Debian/Ubuntu)                                      |

```bash
sudo apt update
# Installer KeePassXC (gestionnaire de mots de passe)
sudo apt install -y keepassxc

# Installer notify-send pour les notifications (optionnel mais recommandé)
sudo apt install -y libnotify-bin
```
***Installer rclone (accès à OneDrive)***
Installer rclone depuis le site officiel : [rclone.org/downloads/](https://rclone.org/downloads/) pour avoir la dernière version. Je l'ai d'abord installé depuis les dépôts de ma distribution Linux et j'ai rencontré un problème avec la copie de la base de données gérée par KeePassXC.
![image](assets/error.png)

### 📁 Structure recommandée

```swift
/home/<utilisateur>/.local/bin/sync_kp.sh              ← Script Bash de synchronisation
/etc/systemd/system/sync-kp.service                 ← Service lancé au démarrage
/etc/systemd/system/sync-kp-shutdown.service        ← Service lancé à l’extinction
```
---

## ☁️🔧 Étape 1 – Configurer rclone avec OneDrive

1. Ouvrez un terminal.
2. Tapez :

```bash
   rclone config
   n/s/q> n                   👈 Tapez `n` pour créer un nouveau remote
   name> onedrive             👈 Donnez-lui le nom "onedrive"

   Storage> 30                👈 Choisissez "30" pour Microsoft OneDrive

   client_id> [Entrée]        👈 Laissez vide et appuyez sur Entrée
   client_secret> [Entrée]    👈 Laissez vide et appuyez sur Entrée

   region> 1                  👈 Choisissez "1" pour Microsoft Global

   Edit advanced config? n    👈 Tapez `n` (non)

   Use auto config? y         👈 Tapez `y` (oui) — ouvrivra une fenêtre dans votre navigateur
                              🔓 Connectez-vous à votre compte Microsoft

   config_type> 1             👈 Choisissez "1" : OneDrive Personnel ou Business

   config_driveid> 3          👈 Choisissez "3" : votre stockage OneDrive personnel

   Drive OK? y                👈 Tapez `y` (oui)

   Keep this "onedrive"? y    👈 Tapez `y` (oui)

   e/n/d/r/c/s/q> q           👈 Tapez `q` pour quitter
```

 3. Suivez les étapes pour :

   * Créer une nouvelle **remote**
   * Choisir le type : `OneDrive`
   * Suivre l’authentification
   * Donnez-lui un nom, par exemple : `onedrive`

👉 **Notez bien ce nom**, car vous devrez l’utiliser dans le script.

---

## 📄 Étape 2 – Créer le script de synchronisation

1. Créez le dossier s’il n’existe pas :

   ```bash
   mkdir -p ~/.local/bin
   ```

2. Copiez le script ci-dessous dans le fichier `~/.local/bin/sync_kp.sh` :

   > 💡 **N’oubliez pas d’adapter** les variables `DRIVE_NAME`, `DB_FILE_NAME`, `REMOTE_LOCATION` si besoin.

```bash
   #!/bin/bash

# Name of your remote storage as defined in Rclone
DRIVE_NAME="onedrive"

# Name and locations of the passwords file
DB_FILE_NAME="Mots_de_passe.kdbx"   # prioritize a name without spaces (optional but recommended)
LOCAL_LOCATION="$HOME/.config/.db"
REMOTE_LOCATION="kp"

#####################

# Compose full path to local and remote database files
LOCAL_PATH="$LOCAL_LOCATION/$DB_FILE_NAME"
REMOTE_PATH="$REMOTE_LOCATION/$DB_FILE_NAME"

# Create local directory if it doesn't exist
mkdir -p "$LOCAL_LOCATION"

function format_datetime_from_string () {
    date -d "$1" +"%F %T.%3N"
}

# Parse local passwords file modification time
function get_local_passwords_mtime () {
    if [ -f "$LOCAL_PATH" ]; then
        local string=$(stat -c %y "$LOCAL_PATH" | cut -d ' ' -f 1,2)
        format_datetime_from_string "$string"
    else
        echo "1970-01-01 00:00:00.000"  # Date très ancienne si fichier n'existe pas
        return 1
    fi
}

# Parse remote passwords file modification time
function get_remote_passwords_mtime () {
    output=$(rclone lsl "$DRIVE_NAME:$REMOTE_PATH" 2>/dev/null)
    if [ $? -ne 0 ]; then
        echo "1970-01-01 00:00:00.000"  # Date très ancienne si fichier n'existe pas
        return 1
    else
        local string=$(echo "$output" | tr -s ' ' | cut -d ' ' -f 3,4)
        format_datetime_from_string "$string"
    fi
}

function sync_passwords () {
    # Get modification times
    local local_mtime=$(get_local_passwords_mtime)
    local remote_mtime=$(get_remote_passwords_mtime)
    
    # Printing modification times
    printf "Local passwords file modification time:\t\t%s\n" "$local_mtime"
    printf "Remote passwords file modification time:\t%s\n" "$remote_mtime"

    # Convert to seconds for comparison
    local local_sec=$(date -d "$local_mtime" +%s)
    local remote_sec=$(date -d "$remote_mtime" +%s)

    # Handle cases
    if [ ! -f "$LOCAL_PATH" ] && [ "$remote_sec" -gt 0 ]; then
        echo "No local file found, importing from remote..."
        notify-send "No local file found, importing from remote..."
        rclone copy "$DRIVE_NAME:$REMOTE_PATH" "$LOCAL_LOCATION/"
        echo "Done!"
        notify-send "Done!"
    elif [ "$remote_sec" -eq 0 ]; then
        echo "No remote file found, exporting local file..."
        notify-send "No remote file found, exporting local file..."
        rclone copy "$LOCAL_PATH" "$DRIVE_NAME:$REMOTE_LOCATION/"
        echo "Done!"
        notify-send "Done!"
    elif [ "$local_sec" -gt "$remote_sec" ]; then
        echo "Local file is newer than remote, exporting..."
        notify-send "Local file is newer than remote, exporting..."
        rclone copy "$LOCAL_PATH" "$DRIVE_NAME:$REMOTE_LOCATION/"
        echo "Done!"
        notify-send "Done!"
    elif [ "$local_sec" -lt "$remote_sec" ]; then
        echo "Remote file is newer than local, importing..."
        notify-send "Remote file is newer than local, importing..."
        rclone copy "$DRIVE_NAME:$REMOTE_PATH" "$LOCAL_LOCATION/"
        echo "Done!"
        notify-send "Done!"
    else
        echo "Password files are synchronized."
        notify-send "Password files are synchronized."
    fi
}

sync_passwords

```

3. Rendez le script exécutable :

   ```bash
   chmod +x ~/.local/bin/sync_kp.sh
   ```

---

## 🛠️ Étape 3 – Créer les services systemd utilisateur

1. Créez le dossier :

   ```bash
   mkdir -p /etc/systemd/system/
   ```

2. Créez le **service au démarrage** :

   Fichier : `/etc/systemd/system/sync-kp.service`

   ```ini
   [Unit]
   Description=Synchronisation automatique de la base de données de KeePassXC avec le cloud.
   After=network-online.target

   [Service]
   Type=oneshot
   ExecStart=/home/votre-nom/.local/bin/sync_kp.sh
   TimeoutStartSec=0

   [Install]
   WantedBy=default.target
   ```

3. Créez le **service à l’extinction** :

   Fichier : `/etc/systemd/system/sync-kp-shutdown.service`

   ```ini
   [Unit]
   Description=Synchronisation du fichier KeePass avant extinction
   DefaultDependencies=no
   Before=shutdown.target reboot.target halt.target

   [Service]
   Type=oneshot
   ExecStart=/home/votre-nom/.local/bin/sync_kp.sh
   TimeoutStartSec=0

   [Install]
   WantedBy=halt.target reboot.target shutdown.target
   ```

4. Activez les deux services :

   ```bash
   sudo systemctl enable sync-kp.service
   sudo systemctl enable sync-kp-shutdown.service
   ```

---

## 🚀 Étape 4 – Premier test

1. Lancez le script manuellement pour tester :

   ```bash
   ~/.local/bin/sync_kp.sh
   ```

   👉 Vous verrez des messages s’afficher dans le terminal (et des notifications si `notify-send` est installé).

2. Vérifiez que la base est bien copiée dans le cloud ou depuis le cloud.

---

## 📱 Utiliser sur votre téléphone

1. Installez [KeePass2Android](https://play.google.com/store/apps/details?id=keepass2android.keepass2android).
2. Connectez-vous à votre compte OneDrive.
3. Sélectionnez le fichier `.kdbx` synchronisé.
4. C’est prêt ✅

---

## ❗ Remarques

* Le script est conçu pour :

  * Synchroniser au démarrage (télécharger si besoin)
  * Synchroniser à l’extinction (envoyer si modifié)
* Les **notifications** (`notify-send`) sont optionnelles. Supprimez-les si vous ne les souhaitez pas.

---

## 🧽 Désinstallation (facultatif)

```bash
sudo systemctl disable sync-kp.service
sudo systemctl disable sync-kp-shutdown.service
sudo rm /etc/systemd/system/sync-kp.service
sudo rm /etc/systemd/system/sync-kp-shutdown.service
rm ~/.local/bin/sync_kp.sh
```

---

## 🙏 Crédits

📝 Script original :
[Z. Baratz – KeePassXC avec synchronisation cloud](https://medium.com/@z.baratz/setting-up-keepassxc-on-linux-with-cloud-synchronization-85ccce837365)

✨ Adaptation, documentation et services : \[Betzalel75]

---

## ✅ Résultat

Vous disposez maintenant d’un gestionnaire de mots de passe **sécurisé, libre, multi-plateforme** et **synchronisé automatiquement** entre vos appareils. Félicitations 🎉 !



---
