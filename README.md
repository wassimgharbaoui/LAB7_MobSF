# LAB 7 — Analyse de Sécurité Mobile avec MobSF & DIVA

> **Établissement :** EMSI &nbsp;|&nbsp; **Domaine :** Cybersécurité &nbsp;|&nbsp; **Auteur :** Mhttaj Zakariyae

---

## Vue d'ensemble

Ce laboratoire explore l'analyse de sécurité d'applications Android à travers deux approches complémentaires :

- **Analyse Statique** : décompilation de l'APK et inspection du code sans exécuter l'application
- **Analyse Dynamique** : surveillance du comportement de l'application en cours d'exécution sur un émulateur rooté

L'application cible est **DIVA** (*Damn Insecure and Vulnerable App*), une APK Android intentionnellement truffée de failles de sécurité, utilisée comme terrain d'entraînement.

L'outil principal est **MobSF** (*Mobile Security Framework*), une plateforme open-source qui automatise les deux types d'analyse et génère des rapports détaillés.

---

## Prérequis

Avant de commencer, assurez-vous d'avoir :

- **Android Studio** avec le SDK Android (image API 29 x86)
- **Docker** installé et en cours d'exécution
- **ADB** (Android Debug Bridge) accessible depuis le terminal
- **Git** pour cloner le dépôt MobSF

---

## Étape 1 — Création d'un émulateur Android rooté

L'analyse dynamique avec MobSF exige un accès root à l'émulateur. Les images **Google Play Store** bloquent ce type d'accès par mesure de sécurité. Il faut donc utiliser une image **Google APIs** ou **AOSP**, qui autorisent le root.

### Créer l'AVD dans Android Studio

1. Aller dans `Tools` → `Device Manager`
2. Cliquer sur **Create Device**
3. Choisir un modèle (ex : Pixel 4) → **Next**
4. Sélectionner une image **sans** l'icône Play Store — par exemple `x86 — API 29 — Google APIs`
5. Cliquer sur **Finish**

> **Pourquoi API 29 ?** Les versions plus récentes restreignent davantage les accès root, ce qui peut empêcher MobSF de fonctionner correctement. L'API 29 (Android 10) offre le meilleur compromis stabilité / compatibilité.

Vérifier que l'AVD a bien été créé :

```bash
emulator -list-avds
```

Résultat attendu :

```
Pixel_4_API_29
```

<img width="483" height="345" alt="Screenshot 2026-03-31 183424" src="https://github.com/user-attachments/assets/9a712194-86b4-4a72-ba26-832235bf8625" />


---

## Étape 2 — Cloner MobSF et préparer les scripts AVD

MobSF fournit des scripts officiels pour démarrer l'émulateur dans un état rooté et configuré pour l'interception réseau.

```bash
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
cd Mobile-Security-Framework-MobSF
ls scripts/
```

Les scripts utiles dans ce dossier :

| Script | Rôle |
|--------|------|
| `scripts/android_emulator.sh` | Lance l'émulateur avec root, proxy réseau et Frida Server |
| `scripts/android_tools.sh` | Installe les dépendances Android nécessaires |

<!-- 📸 IMAGE À AJOUTER : capture du résultat de `ls scripts/` dans le dossier MobSF -->

<img width="795" height="169" alt="Screenshot 2026-03-31 183440" src="https://github.com/user-attachments/assets/066eb36a-c1c7-444f-934e-0320a116036d" />

---

## Étape 3 — Démarrer l'émulateur via le script MobSF

Ce script configure automatiquement l'émulateur pour qu'il soit compatible avec l'analyse dynamique : accès root activé, certificat MobSF installé, proxy réseau configuré.

```bash
bash scripts/android_emulator.sh Pixel_4_API_29
```

### Vérifier la connexion ADB

```bash
adb devices
```

Résultat attendu :

```
List of devices attached
emulator-5554   device
```

### Activer le root

```bash
adb root
adb remount
```

Résultat attendu :

```
remount succeeded
```

> Si `adb remount` retourne une erreur, c'est presque toujours parce que l'image choisie contient le Play Store. Recréer l'AVD avec une image **Google APIs uniquement**.

<img width="619" height="790" alt="Screenshot 2026-03-31 183512" src="https://github.com/user-attachments/assets/8516a032-566e-4f56-8cf9-320a62df04ec" />



<img width="934" height="502" alt="Screenshot 2026-03-31 183534" src="https://github.com/user-attachments/assets/b701a905-f5f8-4f82-9d63-d64cb118785d" />


## Étape 4 — Déployer MobSF via Docker

Docker est la façon la plus rapide de lancer MobSF sans installer de dépendances complexes sur la machine hôte.

```bash
docker pull opensecurity/mobile-security-framework-mobsf:latest

docker run -it --rm \
  -p 8000:8000 \
  -p 1337:1337 \
  opensecurity/mobile-security-framework-mobsf:latest
```

Accéder ensuite à l'interface web :

```
http://127.0.0.1:8000
```

**Identifiants par défaut :**

| Champ | Valeur |
|-------|--------|
| Utilisateur | `mobsf` |
| Mot de passe | `mobsf` |

> Le port `1337` est utilisé pour la communication entre MobSF et l'émulateur lors de l'analyse dynamique (via Frida).

---

## Étape 5 — Récupérer et installer l'APK DIVA

DIVA est une application conçue exprès pour être vulnérable. Elle regroupe les failles les plus courantes du développement Android non sécurisé.

```bash
# Télécharger l'APK
wget https://github.com/payatu/diva-android/raw/master/DivaApplication.apk -O diva.apk

# Installer sur l'émulateur
adb install diva.apk
```

Résultat attendu :

```
Performing Streamed Install
Success
```

Ouvrir l'application **DIVA** depuis l'écran d'accueil de l'émulateur pour confirmer l'installation.

---

## Étape 6 — Analyse avec MobSF

### 6.1 Analyse Statique

Dans l'interface MobSF, glisser-déposer le fichier `diva.apk`. MobSF décompile l'APK, inspecte son manifeste, ses permissions et son code Java, puis génère un rapport.

Sections importantes du rapport :

| Section | Ce qu'on y cherche |
|---------|--------------------|
| Security Score | Score global de sécurité de l'application |
| Permissions | Permissions déclarées jugées dangereuses |
| Certificate | Validité et informations du certificat de signature |
| Hardcoded Secrets | Mots de passe, clés API ou tokens écrits en dur dans le code |
| Manifest Issues | Mauvaises configurations dans `AndroidManifest.xml` |


<img width="928" height="529" alt="Screenshot 2026-03-31 183549" src="https://github.com/user-attachments/assets/9f1902f4-624c-424f-8f6e-bf197804fdd8" />

---

### 6.2 Analyse Dynamique

1. Dans MobSF, cliquer sur **Dynamic Analysis**
2. Sélectionner l'émulateur connecté (`emulator-5554`)
3. Cliquer sur **Start Dynamic Analysis**

MobSF va instrumenter l'application avec Frida et surveiller son comportement en temps réel : appels réseau, accès fichiers, appels d'API sensibles, etc.


<img width="335" height="73" alt="Screenshot 2026-03-31 183557" src="https://github.com/user-attachments/assets/d2895f0d-50b3-4775-987e-e9c1b2bf0ccc" />

---

#### Test 1 — Journalisation Non Sécurisée

**Dans DIVA :** ouvrir `Insecure Logging`, saisir des données et valider.

```bash
adb logcat | grep -i diva
```

**Ce qu'on observe :** les identifiants saisis apparaissent en clair dans les journaux Android.

```
D/DIVA: Credentials: username=admin password=1234
```

**Pourquoi c'est dangereux :** sur un appareil physique, tout processus ayant accès aux logs peut lire ces données. Sur Android < 4.1, n'importe quelle application pouvait lire les logs système.

---

#### Test 2 — Identifiants en Dur dans le Code

**Dans DIVA :** ouvrir `Hardcoded Issues`.

**Ce qu'on observe via MobSF :** des secrets sont écrits directement dans le code Java de l'application.

```java
String vendorKey = "VendorKey123";
String secret    = "SUPERSecret@123";
```

**Pourquoi c'est dangereux :** n'importe qui peut décompiler une APK avec `jadx` ou `apktool` en quelques secondes et récupérer ces valeurs. Ces secrets ne doivent jamais figurer dans le code source.

---

#### Test 3 — Stockage Non Sécurisé

**Dans DIVA :** ouvrir `Insecure Data Storage`, saisir et enregistrer des données.

```bash
adb shell
cd /data/data/jakhar.aseem.diva/
cat shared_prefs/jakhar.aseem.diva_preferences.xml
```

**Ce qu'on observe :**

```xml
<string name="password">MonMotDePasse123</string>
```

**Pourquoi c'est dangereux :** les données sont stockées en clair dans les préférences partagées, un fichier accessible sans chiffrement sur tout appareil rooté ou compromis.

---

#### Test 4 — WebView sans Validation d'Entrée

**Dans DIVA :** ouvrir `Input Validation Issues` et entrer une URL arbitraire.

**Ce qu'on observe :** l'application charge l'URL sans aucun contrôle dans son WebView intégré.

**Pourquoi c'est dangereux :** cette faille peut permettre des attaques XSS, du phishing interne à l'application, ou même l'exécution de code JavaScript malveillant si `addJavascriptInterface` est activé.

---

#### Test 5 — Algorithmes Cryptographiques Obsolètes

**Ce que MobSF détecte :** utilisation de `MD5` ou `DES` dans le code de l'application.

**Pourquoi c'est dangereux :** MD5 et DES sont considérés comme cassés. Des outils comme `hashcat` peuvent retrouver un hash MD5 en quelques secondes. Il faut privilégier AES-256 pour le chiffrement et SHA-256/bcrypt pour le hachage.

---

### 6.3 Tableau de Synthèse

| Vulnérabilité | Sévérité | Méthode de détection |
|---------------|----------|----------------------|
| Journalisation non sécurisée | 🔴 Critique | ADB Logcat |
| Identifiants codés en dur | 🔴 Haute | MobSF — Statique |
| Stockage non sécurisé | 🔴 Haute | ADB Shell |
| WebView sans validation | 🟠 Haute | MobSF — Dynamique |
| Cryptographie faible | 🟡 Moyenne | MobSF — Statique |
| Application déboguable | 🟡 Moyenne | MobSF — Statique |

---

## Étape 7 — Exploration Complémentaire (Autonome)

Ces points sont à explorer par vous-même pour aller plus loin :

- [ ] **Access Control Issues** : tester les contrôles d'accès insuffisants dans DIVA
- [ ] **Injection SQL** : tester les champs vulnérables dans `Input Validation Issues`
- [ ] **Interception réseau** : configurer Burp Suite comme proxy et passer par MobSF
- [ ] **Hooking avec Frida** : analyser les appels de fonctions en temps réel via MobSF Dynamic Analysis
- [ ] **Rapport PDF** : générer le rapport complet depuis MobSF et l'annoter
- [ ] **Comparaison** : analyser les différences entre les résultats statiques et dynamiques

> Documentez chaque test avec des captures d'écran pour constituer un rapport complet.

---

## Dépannage

| Problème rencontré | Message d'erreur | Solution |
|--------------------|------------------|----------|
| `adb remount` échoue | `remount failed` | Utiliser un AVD sans Play Store (Google APIs uniquement) |
| MobSF ne voit pas l'émulateur | Timeout de connexion | Lancer `adb root` avant de démarrer l'analyse |
| Docker ne démarre pas | Port 8000 déjà utilisé | Modifier le port : `-p 8001:8000`, accéder via `localhost:8001` |
| APK refusé | Upload échoue | Vérifier que le fichier est bien un `.apk` et fait moins de 50 MB |
| Émulateur très lent | Freeze / ralentissements | Activer la virtualisation matérielle (Intel HAXM ou KVM sous Linux) |
| Root refusé | Permission denied | Revenir à l'API 29 ou une version inférieure |

---

## Références

- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
- [Documentation officielle MobSF](https://mobsf.github.io/Mobile-Security-Framework-MobSF/)
- [DIVA Android — GitHub](https://github.com/payatu/diva-android)
- [Bonnes pratiques de sécurité Android](https://developer.android.com/topic/security/best-practices)
- [Frida — Instrumentation dynamique](https://frida.re/)

---

<div align="center">
  <sub>Laboratoire réalisé dans le cadre du cursus Cybersécurité — EMSI</sub>
</div>
