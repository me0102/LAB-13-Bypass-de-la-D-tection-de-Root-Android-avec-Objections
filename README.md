# LAB-13-Bypass-de-la-D-tection-de-Root-Android-avec-Objections

> **Domaine :** Sécurité des applications mobiles Android  
> **Difficulté :** Intermédiaire  
> **Cible :** OWASP UnCrackable Level 1 (`owasp.mstg.uncrackable1`)

---

## Table des matières

1. [Objectifs pédagogiques](#objectifs-pédagogiques)
2. [Environnement requis](#environnement-requis)
3. [Étape 1 — Mise en place de l'outillage](#étape-1--mise-en-place-de-loutillage)
4. [Étape 2 — Configuration de l'émulateur et lancement de frida-server](#étape-2--configuration-de-lémulateur-et-lancement-de-frida-server)
5. [Étape 3 — Connexion d'Objection à l'application](#étape-3--connexion-dobjection-à-lapplication)
6. [Étape 4 — Neutralisation de la détection via hook Java](#étape-4--neutralisation-de-la-détection-via-hook-java)
7. [Étape 5 — Vérification du résultat](#étape-5--vérification-du-résultat)
8. [Bonus — Interception des vérifications natives avec frida-trace](#bonus--interception-des-vérifications-natives-avec-frida-trace)
9. [Aide-mémoire des commandes](#aide-mémoire-des-commandes)
10. [Grille d'évaluation](#grille-dévaluation)
11. [Problèmes courants et solutions](#problèmes-courants-et-solutions)

---

## Objectifs pédagogiques

À l'issue de ce laboratoire, vous serez capable de :

- Expliquer les mécanismes qu'une application Android utilise pour détecter un terminal rooté.
- Exploiter **Objection** — surcouche d'Frida — pour neutraliser ces contrôles en cours d'exécution.
- Mettre en œuvre un hook au niveau Java afin de court-circuiter les vérifications de sécurité.
- *(Bonus)* Localiser et désactiver les contrôles implémentés au niveau natif grâce à `frida-trace`.

---

## Environnement requis

| Composant | Version testée |
|-----------|---------------|
| Python | 3.11 ou supérieur |
| Frida | 17.9.1 |
| Objection | 1.12.4 |
| ADB | Fourni avec l'Android SDK |
| Émulateur | Android 8.1.0 (API 27) — rooté |
| Application cible | OWASP UnCrackable Level 1 |

<img width="899" height="191" alt="Environnement de laboratoire" src="https://github.com/user-attachments/assets/8d8da6ba-430c-4223-8c40-fbf1448af48d" />

### Récupérer l'application cible

```
https://github.com/OWASP/owasp-mastg/tree/master/Crackmes/Android/Level_01
```

---

## Étape 1 — Mise en place de l'outillage

### 1.1 Installer Objection avec pipx

```powershell
pip install --user pipx
pipx ensurepath
pipx install objection
```

> **Remarque :** Fermez puis rouvrez votre terminal après `pipx ensurepath` pour que les modifications du PATH soient effectivement prises en compte.  
> Si la commande `objection` reste introuvable, ajoutez le chemin manuellement :
> ```powershell
> $env:PATH += ";C:\Users\<user>\.local\bin"
> ```

### 1.2 Contrôler l'installation

```powershell
objection --version
frida --version
adb devices
```

**Résultat attendu :**

```
objection: 1.12.4
17.9.1
List of devices attached
emulator-5554   device
```

<img width="880" height="107" alt="Vérification des versions" src="https://github.com/user-attachments/assets/8f66ad41-c255-41a4-a622-6888be54479f" />

### 1.3 Déployer l'APK sur l'émulateur

```powershell
adb install UnCrackable-Level1.apk
```

---

## Étape 2 — Configuration de l'émulateur et lancement de frida-server

### 2.1 Transférer frida-server sur l'émulateur

Téléchargez la version adaptée à l'architecture de votre émulateur (x86 pour un AVD standard) depuis :
```
https://github.com/frida/frida/releases
```

```powershell
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
```

### 2.2 Démarrer le serveur Frida

```powershell
adb shell "/data/local/tmp/frida-server &"
```

### 2.3 Confirmer que le processus est actif

```powershell
adb shell "ps | grep frida"
```

**Résultat attendu :**

```
root   9887   1  1053372  172648  poll_schedule_timeout  S  frida-server
```

### 2.4 S'assurer que l'application est visible par Frida

```powershell
frida-ps -Uai | findstr uncrackable
```

**Résultat attendu :**

```
12282  Uncrackable1   owasp.mstg.uncrackable1
```

---

## Étape 3 — Connexion d'Objection à l'application

### 3.1 Identifier le nom exact de l'activité principale

```powershell
adb shell dumpsys package owasp.mstg.uncrackable1 | findstr Activity
```

**Résultat :**

```
cf37351 owasp.mstg.uncrackable1/sg.vantagepoint.uncrackable1.MainActivity filter d878595
```

### 3.2 Lancer l'application

```powershell
adb shell am start -n owasp.mstg.uncrackable1/sg.vantagepoint.uncrackable1.MainActivity
```

### 3.3 Rattacher Objection au processus en cours

```powershell
objection -g owasp.mstg.uncrackable1 explore
```

**Invite de commande attendue :**

```
owasp.mstg.uncrackable1 (run) on (Android: 8.1.0) [usb] #
```

> **Mode spawn (recommandé en cas de fermeture prématurée) :**  
> Si l'application se ferme avant qu'Objection ait eu le temps de s'y accrocher, utilisez plutôt le mode spawn avec le bypass activé dès le démarrage :
> ```powershell
> objection -g owasp.mstg.uncrackable1 explore --startup-command "android root disable"
> ```

---

## Étape 4 — Neutralisation de la détection via hook Java

### 4.1 Désactiver la détection de root

Depuis la console Objection :

```
android root disable
```

**Sortie attendue :**

```
(agent) Registering job 522368. Name: root-detection-disable
```

### 4.2 Consulter les tâches en cours d'exécution

```
jobs list
```

### 4.3 Mécanisme sous-jacent

La commande `android root disable` pose automatiquement des hooks sur les méthodes Java classiquement mobilisées pour détecter un terminal rooté, parmi lesquelles :

- `RootBeer.isRooted()`
- Les lectures de fichiers tels que `/system/app/Superuser.apk`
- La vérification de la présence du binaire `su` dans les répertoires du PATH

---

## Étape 5 — Vérification du résultat

### 5.1 Observer le comportement de l'application

Une fois le bypass appliqué, comparez les deux situations :

- **Sans bypass :** l'application affiche une boîte de dialogue *« Root detected ! »* puis se ferme automatiquement.
- **Avec bypass :** l'application reste ouverte et expose le champ de saisie permettant d'entrer le code secret.

### 5.2 Contournement avancé si l'application continue de se fermer — Script Frida

Créez un fichier nommé `bypass.js` avec le contenu suivant :

```javascript
Java.perform(function() {

    // Intercepter System.exit() pour bloquer la fermeture forcée
    var System = Java.use("java.lang.System");
    System.exit.implementation = function(code) {
        console.log("[*] System.exit(" + code + ") intercepté et bloqué !");
    };

    // Réécrire les méthodes responsables de la détection de root
    var RootDetection = Java.use("sg.vantagepoint.a.c");
    RootDetection.a.implementation = function() { return false; };
    RootDetection.b.implementation = function() { return false; };
    RootDetection.c.implementation = function() { return false; };

    console.log("[*] Bypass de la détection de root appliqué avec succès !");
});
```

Injectez ce script directement via Frida :

```powershell
frida -U -f owasp.mstg.uncrackable1 -l bypass.js
```

**Sortie attendue dans le terminal :**

```
Spawned `owasp.mstg.uncrackable1`. Resuming main thread!
[Android Emulator 5554::owasp.mstg.uncrackable1] -> [*] Bypass de la détection de root appliqué avec succès !
```

---

## Bonus — Interception des vérifications natives avec frida-trace

### Repérer les appels natifs impliqués

```powershell
frida-trace -U -f owasp.mstg.uncrackable1 -i "Java_*"
```

Ou en ciblant spécifiquement les fonctions liées à la détection :

```powershell
frida-trace -U -f owasp.mstg.uncrackable1 -i "*root*" -i "*check*"
```

### Écrire un handler pour un appel natif identifié

Une fois la fonction native repérée (ex. `Java_sg_vantagepoint_a_b_a`), créez un handler dans le répertoire `__handlers__` généré automatiquement par frida-trace :

```javascript
// __handlers__/libfoo.so/Java_sg_vantagepoint_a_b_a.js
{
  onEnter: function(args) {
    console.log("[*] Interception du contrôle natif de root");
  },
  onLeave: function(retval) {
    retval.replace(0);  // Imposer un retour à 0 (false = aucun root détecté)
    console.log("[*] Valeur de retour remplacée par 0 — bypass natif effectif");
  }
}
```

Relancez frida-trace pour activer le hook :

```powershell
frida-trace -U -f owasp.mstg.uncrackable1 -i "Java_sg_vantagepoint*"
```

---

## Aide-mémoire des commandes

```powershell
# Vérification préalable de l'environnement
objection --version
frida --version
adb devices

# Mise en route de frida-server sur l'émulateur
adb shell "/data/local/tmp/frida-server &"
adb shell "ps | grep frida"

# Gestion du cycle de vie de l'application
adb shell am force-stop owasp.mstg.uncrackable1
adb shell am start -n owasp.mstg.uncrackable1/sg.vantagepoint.uncrackable1.MainActivity

# Rattachement via Objection (mode attach)
objection -g owasp.mstg.uncrackable1 explore

# Injection au démarrage avec bypass immédiat (mode spawn)
objection -g owasp.mstg.uncrackable1 explore --startup-command "android root disable"

# Injection du script de bypass avancé via Frida
frida -U -f owasp.mstg.uncrackable1 -l bypass.js

# Bonus — Traçage des fonctions natives
frida-trace -U -f owasp.mstg.uncrackable1 -i "*root*"
```

<img width="557" height="241" alt="Résumé des commandes" src="https://github.com/user-attachments/assets/85cfbc19-8ab1-4b1b-8f13-feb7155f214f" />

---

## Grille d'évaluation

| N° | Exercice | Points | Élément à remettre |
|----|----------|--------|--------------------|
| 1 | Preuve d'installation et de connectivité | 20 pts | Capture d'écran : `objection --version`, `frida --version`, `adb devices` |
| 2 | Démarrage et visibilité de l'application | 20 pts | Capture d'écran : invite `owasp.mstg.uncrackable1 (run) on (…) [usb] #` |
| 3 | Bypass Java via Objection | 40 pts | Captures avant/après le bypass + journaux Objection affichant `root-detection-disable` |
| 4 | Bypass natif (bonus) | 20 pts | Sortie de `frida-trace` + handler JS neutralisant la détection native |

**Total : 100 points**

---

## Problèmes courants et solutions

| Symptôme | Cause probable | Remède |
|----------|---------------|--------|
| `objection` non reconnu par le shell | Répertoire absent du PATH | Exécuter : `$env:PATH += ";C:\Users\<user>\.local\bin"` |
| `Unable to find target application` | Application non démarrée | Lancer l'app avec `adb shell am start` avant d'attacher Objection |
| `process-terminated` immédiatement après l'attachement | Détection native trop rapide pour le mode attach | Injecter `bypass.js` directement via Frida en mode spawn |
| `frida-server` introuvable ou plantant | Architecture binaire incompatible | Vérifier l'ABI avec `adb shell getprop ro.product.cpu.abi` puis télécharger le bon binaire |
| Conflits entre deux installations Python | `python` et `pip` pointant vers des versions différentes | Utiliser explicitement `pip` associé à Python 3.11 pour installer les outils Frida et Objection |

---

> **Références et ressources complémentaires :**  
> - [OWASP Mobile Application Security Testing Guide (MASTG)](https://mas.owasp.org/MASTG/)  
> - [Documentation officielle de Frida](https://frida.re/docs/home/)  
> - [Dépôt GitHub d'Objection](https://github.com/sensepost/objection)
