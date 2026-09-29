# Lab : Packaging et déploiement d'applications (PowerShell Remoting + Intune)

## Objectif

Reproduire un scénario complet de gestion d'applications Windows sur un environnement Microsoft Entra ID sans domaine Active Directory classique :
1. Déploiement à distance via **PowerShell Remoting** (WinRM)
2. Déploiement via **Microsoft Intune** (packaging `.intunewin`)

Ce lab fait suite aux dépôts [packaging-des-applications](https://github.com/adjatoto72-cmyk/packaging-des-applications) et [Packagin-d-application-sur-intune](https://github.com/adjatoto72-cmyk/Packagin-d-application-sur-intune).

## Environnement

| VM | Rôle | Adresse IP | OS | Jonction |
|---|---|---|---|---|
| `packager` | Poste utilisé pour empaqueter et déployer l'application | `172.16.0.4` | `<VERSION_WINDOWS>` | Microsoft Entra ID |
| `client` | Poste cible du déploiement | `172.16.0.5` | `<VERSION_WINDOWS>` | Microsoft Entra ID |

- **Jonction** : Microsoft Entra ID (Entra joined), sans contrôleur de domaine AD
- **Tenant** : Microsoft 365 / Entra ID (« Default Directory »)
- **MDM** : Microsoft Intune

> Remplace `<VERSION_WINDOWS>` par la valeur obtenue avec `winver` sur chaque VM.

## Application utilisée

| Application | Fichier | Lien de téléchargement |
|---|---|---|
| 7-Zip | `7z2603-x64.msi` | https://www.7-zip.org/download.html |

> Télécharger la version **.msi** (pas le `.exe`), nécessaire pour une installation silencieuse via `msiexec`.

<!-- Capture : page de téléchargement 7-Zip / fichier .msi dans l'explorateur -->

---

## Partie 1 — Jonction Entra ID et enrôlement Intune

### 1.1 Jonction des VM

Sur chaque VM : *Paramètres > Comptes > Accès à l'entreprise ou à l'école > Se connecter > **Joindre cet appareil à Microsoft Entra ID*** (et non « Ajouter un compte professionnel ou scolaire », qui ne fait qu'un enregistrement).


<img width="300" height="300" alt="Capture d&#39;écran 2026-09-29 112646" src="https://github.com/user-attachments/assets/03a8d6d5-a9e1-46b6-b1a2-7913c07b737f" />
<img width="300" height="300" alt="Capture d&#39;écran 2026-09-29 112656" src="https://github.com/user-attachments/assets/4a216cf1-d2a7-48fa-b33d-a42047c5c567" />

Vérification :

```powershell
dsregcmd /status | Select-String "AzureAdJoined|DomainJoined|TenantName|MdmUrl"
```

Résultat attendu : `AzureAdJoined : YES`, `DomainJoined : NO`.

<!-- Capture : écran d'accès à l'entreprise ou à l'école -->
<!-- Capture : résultat dsregcmd /status -->
<img width="405" height="356" alt="Capture d&#39;écran 2026-09-23 193016" src="https://github.com/user-attachments/assets/015f3914-6c5f-4c75-a187-4241a381f03c" />

### 1.2 Enrôlement automatique Intune

Deux prérequis pour que `MdmUrl` se renseigne automatiquement :

1. **Licence Intune** assignée à l'utilisateur (*admin.microsoft.com > Utilisateurs actifs > Licences et applications*)
2. **Étendue MDM** couvrant l'utilisateur (*Centre d'administration Entra > Appareils > Inscription des appareils > Mobilité (MDM et MAM) > Microsoft Intune*, étendue utilisateur sur **Tout**)

Vérification finale sur chaque VM :

```powershell
dsregcmd /status | Select-String "MdmUrl"
```

Un `MdmUrl` de type `https://enrollment.manage.microsoft.com/...` confirme l'enrôlement.

**Résultat observé** : les deux VM se sont enrôlées avec succès. Preuve concrète : des applications déjà présentes dans Intune (7-Zip, KeePass, en `.intunewin`) se sont installées automatiquement dès la jonction de `client`.

<img width="944" height="332" alt="Capture d&#39;écran 2026-09-29 114023" src="https://github.com/user-attachments/assets/59115134-ca50-4cf2-a6e0-dd63fb54c223" />


<!-- Capture : écran étendue MDM dans Entra -->
<!-- Capture : dsregcmd /status avec MdmUrl renseigné -->
<!-- Capture : appareils visibles dans Intune > Appareils > Windows -->
<!-- Capture : applications installées automatiquement sur client -->

---

## Partie 2 — Déploiement via PowerShell Remoting

### 2.1 Préparation réseau et WinRM sur chaque VM

```powershell
Enable-PSRemoting -Force -SkipNetworkProfileCheck
Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
Enable-NetFirewallRule -DisplayName "Windows Remote Management (HTTP-In)"
```

> **Point de blocage rencontré** : `Enable-PSRemoting` annonce que le pare-feu est configuré, mais les règles `Windows Remote Management (HTTP-In)` restaient à `Enabled : False`. Il a fallu les activer manuellement avec `Enable-NetFirewallRule`, en plus de passer le profil réseau de `Public` à `Private`.

### 2.2 TrustedHosts sur packager

Comme les VM sont jointes à Entra ID (pas à un domaine AD), l'authentification Kerberos par défaut ne fonctionne pas entre elles : il faut déclarer `client` en confiance explicite.

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "172.16.0.5" -Force
```

### 2.3 Test de connectivité

```powershell
Test-NetConnection 172.16.0.5 -Port 5985
```

`TcpTestSucceeded : True` confirme que WinRM est joignable.

<!-- Capture : Test-NetConnection réussi -->

### 2.4 Connexion à distance

```powershell
Enter-PSSession -ComputerName 172.16.0.5 -Credential client\azureuser
```



<img width="459" height="258" alt="Capture d&#39;écran 2026-09-29 115234" src="https://github.com/user-attachments/assets/ba4d3759-efbe-4c05-81d7-d52f43ee6cac" />

<img width="205" height="151" alt="Capture d&#39;écran 2026-09-29 115911" src="https://github.com/user-attachments/assets/cd74f524-7f9f-4979-92cd-3f448e6dff74" />


> **Point de blocage rencontré** : première tentative en erreur `PSRemotingTransportException` (TrustedHosts manquant), résolu par 2.2. Deuxième erreur liée au pare-feu / profil réseau public, résolue par 2.1.

<!-- Capture : session distante active [172.16.0.5]: PS ... -->


<img width="536" height="371" alt="Capture d&#39;écran 2026-09-29 120644" src="https://github.com/user-attachments/assets/2bcd9590-8fba-4945-882b-295dae2cb596" />


### 2.5 Copie et installation silencieuse de 7-Zip

```powershell
Exit-PSSession

$cred = Get-Credential          # client\azureuser
$s = New-PSSession -ComputerName 172.16.0.5 -Credential $cred

Invoke-Command -Session $s { New-Item -ItemType Directory C:\Temp -Force }

Copy-Item "C:\Lab\7z2603-x64.msi" -Destination C:\Temp\ -ToSession $s

Invoke-Command -Session $s {
  $p = Start-Process msiexec.exe -ArgumentList '/i C:\Temp\7z2603-x64.msi /qn /norestart /l*v C:\Temp\7zip-install.log' -Wait -PassThru
  $p.ExitCode
}
```
<img width="415" height="222" alt="Capture d&#39;écran 2026-09-29 121906" src="https://github.com/user-attachments/assets/a4b7837a-28ac-4b95-8eab-5c3878462546" />


> **Point de blocage rencontré** : premier essai avec un nom de fichier erroné dans la commande (`7z2408-x64.msi` au lieu de `7z2603-x64.msi`), provoquant un code retour **1619** (« package MSI introuvable »), confirmé dans le log par l'erreur `2203 ... -2147287038`. Correction : utiliser le nom de fichier exact présent sur `client`.

Résultat : `ExitCode : 0` → installation réussie.

<img width="461" height="185" alt="Capture d&#39;écran 2026-09-29 121933" src="https://github.com/user-attachments/assets/a51b0325-92b4-4d4b-b7a8-2cd8603d1a8e" />


<!-- Capture : ExitCode 0 -->

### 2.6 Vérification finale

```powershell
Invoke-Command -Session $s { Get-ItemProperty "HKLM:\SOFTWARE\7-Zip" -ErrorAction SilentlyContinue }
```

Résultat confirmé : `Path : C:\Program Files\7-Zip\` sur `172.16.0.5`.

<!-- Capture : clé de registre 7-Zip confirmant l'installation -->

---

## Points de blocage rencontrés — récapitulatif

| Problème | Cause | Résolution |
|---|---|---|
| `MdmUrl` vide après jonction Entra ID | Jonction faite via « Ajouter un compte » (enregistrement) plutôt que « Joindre » ; à vérifier aussi côté licence/étendue MDM | Rejoindre avec l'option « Joindre cet appareil à Microsoft Entra ID » |
| VM `packager` non enrôlée | Jonction jamais effectuée sur cette VM | Jonction manuelle |
| `Enter-PSSession` : `PSRemotingTransportException` (TrustedHosts) | Authentification Kerberos impossible entre VM Entra-joined | `Set-Item WSMan:\localhost\Client\TrustedHosts` |
| `Enter-PSSession` : erreur WinRM/pare-feu | Profil réseau `Public` + règles de pare-feu WinRM désactivées malgré `Enable-PSRemoting` | Passage en profil `Private` + `Enable-NetFirewallRule` manuel |
| `msiexec` : code retour **1619** | Nom de fichier incorrect dans la commande (version différente du fichier réellement présent) | Vérifier le nom exact avec `Get-Item` avant de lancer l'installation |

---

<img width="576" height="279" alt="Capture d&#39;écran 2026-09-29 115656" src="https://github.com/user-attachments/assets/b1684e35-55c9-4ba3-b2bb-133905807f8d" />


---

## Partie 3 — Packaging et déploiement via Microsoft Intune (.intunewin)

### 3.1 Récupérer l'outil de packaging

Sur `packager` :

```powershell
Invoke-WebRequest -Uri "https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool/raw/master/IntuneWinAppUtil.exe" -OutFile "C:\Lab\IntuneWinAppUtil.exe"
```

### 3.2 Organiser les fichiers source

```powershell
New-Item -ItemType Directory C:\Lab\7zip-source -Force
Copy-Item C:\Lab\7z2603-x64.msi C:\Lab\7zip-source\
New-Item -ItemType Directory C:\Lab\7zip-output -Force
```

### 3.3 Créer le package .intunewin

```powershell
cd C:\Lab
.\IntuneWinAppUtil.exe -c C:\Lab\7zip-source -s 7z2603-x64.msi -o C:\Lab\7zip-output
```

Génère `7z2603-x64.intunewin` dans `C:\Lab\7zip-output`.

<!-- Capture : fichier .intunewin généré -->
<img width="515" height="152" alt="Capture d&#39;écran 2026-09-29 121906" src="https://github.com/user-attachments/assets/9213159a-a171-4ab1-b664-f3452f374cf4" />


### 3.4 Créer l'application dans Intune

*Intune > Applications > Windows > Créer > Application Windows (Win32)* :

- **Fichier de package** : le `.intunewin` généré
- **Informations** : nom « 7-Zip 26.03 (x64 edition) », éditeur, description
- **Programme** :
  - Installation : `msiexec /i "7z2603-x64.msi" /qn /norestart`
  - Désinstallation : `msiexec /x "{PRODUCT-CODE}" /qn /norestart`
- **Règles de détection** : « Utiliser un fichier MSI » (code produit détecté automatiquement)
- **Attribution** : Tous les appareils (attribution obligatoire)

<!-- Capture : création de l'application dans Intune -->
<!-- Capture : règles de détection -->
<img width="999" height="868" alt="Capture d&#39;écran 2026-09-29 122448" src="https://github.com/user-attachments/assets/b1414736-890a-4567-96eb-738126564e57" />
<img width="904" height="797" alt="Capture d&#39;écran 2026-09-29 122354" src="https://github.com/user-attachments/assets/31e1aaa7-7016-4692-aa15-05c9b2246846" />


### 3.5 Vérification du déploiement

*Intune > Applications > 7-Zip > État de l'installation de l'appareil* :

| Appareil | Version | État |
|---|---|---|
| Client | 26.03.00.0 | Installed |
| Packager | 26.03.00.0 | Installed |

<img width="598" height="284" alt="Capture d&#39;écran 2026-09-29 123930" src="https://github.com/user-attachments/assets/08b5d637-b698-4e80-bfb0-efd2cc754d2b" />


> **Point d'attention** : le rapport d'état peut afficher un délai d'affichage (« L'affichage des informations les plus récentes peut retarder la mise à jour de ce rapport »). Si rien n'apparaît immédiatement après le déploiement, attendre quelques minutes et actualiser plutôt que de conclure à un échec.

<!-- Capture : état de l'installation Installed sur les deux VM -->

---

---

## Partie 4 — Déploiement self-service via le Portail d'entreprise

Contrairement aux parties précédentes (déploiement forcé, sans action de l'utilisateur), cette partie couvre le mode **self-service** : l'utilisateur installe lui-même une application depuis un catalogue, comme un app store interne.

### 4.1 Choix de l'application : Notepad++

Notepad++ ne propose **aucun installeur MSI officiel** — uniquement un `.exe` (NSIS) et une version portable. C'est volontaire pour ce lab : ça impose une règle de détection **personnalisée** plutôt que la détection MSI automatique utilisée pour 7-Zip.

| Application | Fichier | Lien de téléchargement |
|---|---|---|
| Notepad++ | `npp.8.9.8.1.Installer.x64.exe` | https://notepad-plus-plus.org/downloads/ |

### 4.2 Packaging en .intunewin

```powershell
New-Item -ItemType Directory C:\Lab\npp-source -Force
Copy-Item C:\Lab\npp.8.9.8.1.Installer.x64.exe C:\Lab\npp-source\
New-Item -ItemType Directory C:\Lab\npp-output -Force

cd C:\Lab
.\IntuneWinAppUtil.exe -c C:\Lab\npp-source -s npp.8.9.8.1.Installer.x64.exe -o C:\Lab\npp-output
```

> **Point de blocage rencontré** : premier essai avec `-s npp.8.9.8.1.Installer.x64` sans l'extension `.exe`, provoquant `ERROR The setup file you specified cannot be accessed.` — même type d'erreur que le nom de fichier incorrect en partie 2. Toujours vérifier le nom exact avec `Get-Item` avant de lancer la commande.

<!-- Capture : fichier .intunewin Notepad++ généré -->

### 4.3 Création de l'application dans Intune

*Intune > Applications > Windows > Créer > Application Windows (Win32)* :

- **Commande d'installation** : `npp.8.9.8.1.Installer.x64.exe /S`
- **Commande de désinstallation** : `"C:\Program Files\Notepad++\uninstall.exe" /S`
- **Règles de détection** : **« Configurer manuellement les règles de détection »** (pas « script de détection personnalisé », qui demande un vrai script `.ps1`) → type **Fichier** :
  - Chemin : `C:\Program Files\Notepad++`
  - Fichier ou dossier : `notepad++.exe`
  - Méthode : « Le fichier ou dossier existe »
- **Attribution** : groupe **« All Company »**, dans la section **« Disponible pour les appareils inscrits »** (pas « Obligatoire ») — c'est ce qui bascule l'app en mode self-service plutôt qu'en installation forcée

<!-- Capture : configuration de la règle de détection manuelle -->
<!-- Capture : attribution en "Disponible" -->
<img width="588" height="318" alt="Capture d&#39;écran 2026-09-29 203805" src="https://github.com/user-attachments/assets/f06bae85-e0f5-41e7-b3bd-3244d6ca7c7b" />

<img width="600" height="385" alt="Capture d&#39;écran 2026-09-29 204813" src="https://github.com/user-attachments/assets/74d10ef7-af2a-45d0-8eb0-ba14cca4df5b" />



### 4.4 Installation depuis le Portail d'entreprise

Sur `client` (ou `packager`) : ouvrir l'app **Company Portal**, se connecter avec le compte Entra ID, puis :

1. L'app apparaît dans *Recently published apps* sur la page d'accueil
2. Cliquer sur la vignette, puis sur **Install**
3. Suivre la progression dans l'interface

**Résultat observé** : Notepad++ visible dans le catalogue peu après la création de l'app côté Intune, installation réussie depuis l'interface du Portail d'entreprise.

<!-- Capture : Notepad++ visible dans "Recently published apps" -->
<!-- Capture : installation réussie depuis le Portail d'entreprise -->

---

## Points de blocage rencontrés — récapitulatif

| Problème | Cause | Résolution |
|---|---|---|
| `MdmUrl` vide après jonction Entra ID | Jonction faite via « Ajouter un compte » (enregistrement) plutôt que « Joindre » ; à vérifier aussi côté licence/étendue MDM | Rejoindre avec l'option « Joindre cet appareil à Microsoft Entra ID » |
| VM `packager` non enrôlée | Jonction jamais effectuée sur cette VM | Jonction manuelle |
| `Enter-PSSession` : `PSRemotingTransportException` (TrustedHosts) | Authentification Kerberos impossible entre VM Entra-joined | `Set-Item WSMan:\localhost\Client\TrustedHosts` |
| `Enter-PSSession` : erreur WinRM/pare-feu | Profil réseau `Public` + règles de pare-feu WinRM désactivées malgré `Enable-PSRemoting` | Passage en profil `Private` + `Enable-NetFirewallRule` manuel |
| `msiexec` : code retour **1619** | Nom de fichier incorrect dans la commande (version différente du fichier réellement présent) | Vérifier le nom exact avec `Get-Item` avant de lancer l'installation |
| État d'installation Intune vide juste après déploiement | Délai d'affichage du rapport Intune | Attendre et actualiser plutôt que conclure à un échec |
| `IntuneWinAppUtil.exe` : `ERROR The setup file you specified cannot be accessed` | Extension `.exe` manquante dans le paramètre `-s` | Vérifier le nom exact du fichier avec `Get-Item` avant de lancer le packaging |

---

## Prochaines étapes possibles

- [ ] Script de désinstallation à distance (`msiexec /x ... /qn`) et test, sur les méthodes Remoting et Intune
- [ ] Personnaliser le Portail d'entreprise (logo, nom d'organisation, catégorie et mise en avant de l'app)
- [ ] Restreindre l'attribution Intune à un groupe pilote plutôt qu'à tous les appareils pour les apps obligatoires
- [ ] Comparer avec un packaging via PSAppDeployToolkit (PSADT)
- [ ] Tester une mise à jour de version (ex. 7-Zip 24.x → 26.03) sans désinstallation manuelle préalable
