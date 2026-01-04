# YGG Hybrid Indexer - Guide Complet

> Bypass de la nouvelle protection YGG (signature par torrent) pour Prowlarr/Sonarr/Radarr

## 📋 Table des matières

- [Le problème](#-le-problème)
- [La solution](#-la-solution)
- [Architecture recommandée](#-architecture-recommandée)
- [Installation](#-installation)
- [Génération des cookies](#-génération-des-cookies)
- [Configuration Prowlarr](#-configuration-prowlarr)
- [Erreurs courantes](#-erreurs-courantes)
- [Dépannage](#-dépannage)
- [Maintenance](#-maintenance)
- [FAQ](#-faq)

---

## 🔴 Le problème

Depuis **fin décembre 2024**, YGG a mis en place une **signature individuelle** sur chaque torrent téléchargé.

### Avant (ça marchait)
```
announce_sig=abc123def456789abc123def456789abc123def4  ✅
```

### Maintenant (via API/Prowlarr)
```
announce_sig=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa  ❌
```

### Conséquences
- ❌ Les torrents récupérés via YggAPI ont une signature **invalide**
- ❌ Le tracker **refuse la connexion** → le téléchargement ne démarre pas
- ✅ Les torrents téléchargés **manuellement** depuis le site fonctionnent

---

## ✅ La solution

Un **indexeur hybride** qui combine le meilleur des deux mondes :

| Fonction | Source | Avantage |
|----------|--------|----------|
| **Recherche** | YggAPI (`yggapi.eu`) | Résultats propres, rapide, pas de CloudFlare |
| **Download** | YGG direct (`yggtorrent.org`) | Signature valide avec tes cookies |

### Comment ça marche

```
┌─────────────────────────────────────────────────────────────────┐
│  1. RECHERCHE                                                    │
│     Prowlarr → YggAPI (yggapi.eu/torrents?q=xxx)                │
│             → Résultats JSON (id, titre, seeders, etc.)         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  2. DOWNLOAD .torrent                                            │
│     Prowlarr → YGG direct (yggtorrent.org/engine/download_torrent)
│             → Avec cookies (ygg_ + cf_clearance)                │
│             → Fichier .torrent avec signature VALIDE            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. TÉLÉCHARGEMENT                                               │
│     qBittorrent (via VPN) → Tracker accepte la connexion        │
│                           → Téléchargement démarre ✅            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Architecture recommandée

### Ce qui a besoin du VPN
Seuls les **clients torrent** font du P2P et nécessitent un VPN :
- ✅ qBittorrent
- ✅ Deluge
- ✅ Transmission

### Ce qui n'a PAS besoin du VPN
Les *arr apps font juste de la gestion/recherche :
- ❌ Prowlarr (recherche + récup .torrent)
- ❌ Sonarr (gestion séries)
- ❌ Radarr (gestion films)
- ❌ Lidarr (gestion musique)

### Schéma

```
┌─────────────────────────────────────────────────────────────────┐
│                      SANS VPN (IP publique)                      │
│                                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│  │ Prowlarr│    │ Sonarr  │    │ Radarr  │    │ Lidarr  │      │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘      │
│       │                                                          │
│       ↓ (envoie .torrent)                                       │
└───────┼─────────────────────────────────────────────────────────┘
        │
        ↓
┌─────────────────────────────────────────────────────────────────┐
│                      AVEC VPN (Gluetun)                          │
│                                                                  │
│  ┌─────────────┐    ┌─────────┐                                 │
│  │ qBittorrent │    │ Deluge  │                                 │
│  └─────────────┘    └─────────┘                                 │
│         │                                                        │
│         ↓ (P2P via VPN)                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Pourquoi cette architecture ?

1. **Prowlarr sans VPN** = même IP que ton PC = cookies fonctionnent
2. **Pas besoin de matcher l'IP VPN** = plus simple à maintenir
3. **Clients torrent avec VPN** = protection pour le P2P

---

## 📦 Installation

### Prérequis

- Docker avec Gluetun (pour les clients torrent)
- Prowlarr installé
- Un compte YGG

### Étape 1 : Sortir Prowlarr du VPN

Si Prowlarr passe actuellement par Gluetun, modifie sa config.

**docker-compose.yml - AVANT :**
```yaml
prowlarr:
  image: linuxserver/prowlarr
  container_name: prowlarr
  network_mode: "service:gluetun"  # ❌ À retirer
  # ...
```

**docker-compose.yml - APRÈS :**
```yaml
prowlarr:
  image: linuxserver/prowlarr
  container_name: prowlarr
  network_mode: bridge  # ✅ Réseau normal
  ports:
    - 9696:9696  # ✅ Exposer le port
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=Europe/Paris
  volumes:
    - /chemin/vers/prowlarr/config:/config
  restart: unless-stopped
```

Fais pareil pour Sonarr (8989) et Radarr (7878).

**Redémarre les containers :**
```bash
docker-compose down
docker-compose up -d
```

### Étape 2 : Vérifier les IPs

```bash
# IP de Prowlarr (doit être ton IP publique)
docker exec prowlarr curl -s ifconfig.me

# IP de qBittorrent (doit être l'IP VPN)
docker exec qbittorrent curl -s ifconfig.me
```

### Étape 3 : Installer l'indexeur

```bash
# Copie le fichier ygghybrid.yml
docker cp ygghybrid.yml prowlarr:/config/Definitions/Custom/ygghybrid.yml

# Redémarre Prowlarr
docker restart prowlarr
```

---

## 🍪 Génération des cookies

### ⚠️ IMPORTANT : IPv4 obligatoire !

Le cookie `cf_clearance` est lié à ton **adresse IP**. 

**Problème courant :** Ton PC utilise IPv6, ton NAS utilise IPv4 → les cookies ne marchent pas !

### Étape 1 : Désactiver IPv6 temporairement

**Windows :**
1. `Win + R` → tape `ncpa.cpl` → Entrée
2. Clic droit sur ta connexion (Ethernet/Wi-Fi) → **Propriétés**
3. **Décoche** "Protocole Internet version 6 (TCP/IPv6)"
4. OK

**Linux :**
```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
```

**macOS :**
```bash
sudo networksetup -setv6off Wi-Fi
```

### Étape 2 : Vérifier ton IP

Va sur https://ifconfig.me et vérifie :
- Tu dois voir une IPv4 (ex: `92.168.1.100`)
- **PAS** une IPv6 (ex: `2a01:xxxx:...`)

Compare avec l'IP de Prowlarr :
```bash
docker exec prowlarr curl -s ifconfig.me
```

**Les deux IPs doivent être identiques !**

### Étape 3 : Récupérer les cookies

1. Ouvre ton navigateur (Opera, Chrome, Firefox...)
2. Va sur https://www.yggtorrent.org
3. Connecte-toi à ton compte
4. Ouvre les DevTools : `F12`
5. Va dans l'onglet **Application** (Chrome/Opera) ou **Stockage** (Firefox)
6. Dans **Cookies** → `www.yggtorrent.org`
7. Copie les valeurs de :
   - `ygg_`
   - `cf_clearance`

### Étape 4 : Réactiver IPv6

N'oublie pas de réactiver IPv6 après !

**Windows :** Recoche "Protocole Internet version 6 (TCP/IPv6)"

**Linux :**
```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
```

**macOS :**
```bash
sudo networksetup -setv6automatic Wi-Fi
```

---

## ⚙️ Configuration Prowlarr

### Ajouter l'indexeur

1. Prowlarr → **Settings** → **Indexers**
2. **Add Indexer** → cherche "YGG Hybrid"
3. Configure :

**Cookie ygg_ :**
```
TON_COOKIE_YGG
```
(ex: `xXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxX`)

**Cookie cf_clearance :**
```
TON_COOKIE_CF
```
(ex: `aBcDeFgHiJkLmNoPqRsTuVwXyZ-1234567890-1.2.1.1-xxxxx...`)

**User-Agent :**
```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/141.0.0.0 Safari/537.36 OPR/125.0.0.0
```

> 💡 Utilise le même User-Agent que ton navigateur (F12 → Network → n'importe quelle requête → Headers)

4. **Test** l'indexeur
5. **Save**

### Syncer avec Sonarr/Radarr

1. Prowlarr → **Settings** → **Apps**
2. Ajoute Sonarr et Radarr
3. **Sync App Indexers**

---

## ❌ Erreurs courantes

### 1. "Just a moment..." / CloudFlare bloque

**Symptôme :**
```
<!DOCTYPE html><html lang="en-US"><head><title>Just a moment...
```

**Causes possibles :**

| Cause | Solution |
|-------|----------|
| IP différente (IPv6 vs IPv4) | Désactive IPv6 et régénère les cookies |
| Prowlarr passe par le VPN | Sors Prowlarr de Gluetun |
| Cookies expirés | Régénère les cookies |
| User-Agent différent | Utilise le même User-Agent que ton navigateur |

### 2. Signature invalide (aaaa...)

**Symptôme :** Le torrent se télécharge mais qBittorrent affiche "Tracker non joignable"

**Vérification :**
```bash
# Regarde le début du .torrent
head -c 300 /chemin/vers/fichier.torrent
```

Si tu vois `announce_sig=aaaaaaa...` → le cookie n'a pas été utilisé.

**Solution :** Vérifie que l'indexeur YGG Hybrid est bien configuré avec les cookies.

### 3. Prowlarr et PC ont des IPs différentes

**Vérification :**
```bash
# IP Prowlarr
docker exec prowlarr curl -s ifconfig.me

# IP PC (dans un terminal)
curl ifconfig.me
```

**Solutions :**

| Situation | Solution |
|-----------|----------|
| PC en IPv6, NAS en IPv4 | Désactive IPv6 sur le PC |
| Prowlarr derrière VPN | Sors Prowlarr du VPN |
| Réseaux différents | Génère les cookies depuis le même réseau |

### 4. "No results" dans les recherches

**Causes possibles :**
- YggAPI est down → vérifie https://yggapi.eu
- Mauvaise catégorie sélectionnée
- Terme de recherche trop spécifique

---

## 🔧 Dépannage

### Tester les cookies manuellement

```bash
docker exec prowlarr curl 'https://www.yggtorrent.org/engine/download_torrent?id=1282618' \
  -H 'user-agent: TON_USER_AGENT' \
  -b 'ygg_=TON_YGG_COOKIE; cf_clearance=TON_CF_COOKIE' \
  -o /tmp/test.torrent && docker exec prowlarr head -c 200 /tmp/test.torrent
```

**Résultat attendu :**
```
d8:announce152:http://tracker.p2p-world.net:8080/xxxxx/announce?announce_sig=VRAIE_SIGNATURE...
```

**Si tu vois ça → ❌ cookies invalides :**
```
<!DOCTYPE html><html lang="en-US"><head><title>Just a moment...
```

### Vérifier les IPs

```bash
# Toutes les IPs de tes containers
docker exec prowlarr curl -s ifconfig.me && echo " (prowlarr)"
docker exec sonarr curl -s ifconfig.me && echo " (sonarr)"
docker exec radarr curl -s ifconfig.me && echo " (radarr)"
docker exec qbittorrent curl -s ifconfig.me && echo " (qbittorrent)"
```

**Attendu :**
- Prowlarr/Sonarr/Radarr : ton IP publique (ex: `92.168.1.100`)
- qBittorrent : IP du VPN (ex: `185.210.50.20`)

### Vérifier qu'un torrent a une signature valide

```bash
# Télécharge un torrent via Prowlarr et vérifie
head -c 300 /chemin/vers/downloads/fichier.torrent | grep -o 'announce_sig=[^&]*'
```

- `announce_sig=abc123def456...` → ✅ Valide
- `announce_sig=aaaaaaaaaaaaaa...` → ❌ Invalide

### Logs Prowlarr

```bash
docker logs prowlarr --tail 100 | grep -i "ygg\|error\|fail"
```

---

## 🔄 Maintenance

### Quand régénérer les cookies ?

Les cookies expirent après un certain temps (variable). Régénère-les si :
- Les téléchargements ne démarrent plus
- Tu vois "Just a moment..." dans les tests
- Le tracker refuse les connexions

### Procédure de refresh

1. **Désactive IPv6** sur ton PC
2. **Vérifie ton IP** sur https://ifconfig.me (doit matcher Prowlarr)
3. **Va sur YGG** et connecte-toi
4. **Récupère les cookies** (F12 → Application → Cookies)
5. **Mets à jour** l'indexeur dans Prowlarr
6. **Teste** avec la commande curl
7. **Réactive IPv6**

### Automatisation (avancé)

Pour automatiser le refresh des cookies, tu pourrais :
- Utiliser un script avec Playwright/Selenium
- Créer une extension navigateur qui envoie les cookies
- Utiliser un browser headless sur le NAS

> ⚠️ Ces solutions sont complexes et peuvent casser. Le refresh manuel reste le plus fiable.

---

## ❓ FAQ

### Q: Pourquoi pas juste utiliser FlareSolverr ?

FlareSolverr ne fonctionne plus bien avec YGG depuis leurs dernières protections. De plus, même si tu bypass CloudFlare, la signature du torrent reste invalide sans passer par le site directement.

### Q: Les cookies durent combien de temps ?

Variable. De quelques heures à plusieurs jours selon l'activité de CloudFlare. En moyenne, compte 1 à 7 jours.

### Q: Je peux partager mes cookies ?

**Non !** Les cookies sont liés à :
- Ton compte YGG
- Ton adresse IP
- Ton User-Agent

Si quelqu'un utilise tes cookies, ça peut faire ban ton compte.

### Q: Ça marche avec Jackett ?

Ce guide est pour Prowlarr, mais le principe est le même. Tu peux adapter le fichier YAML pour Jackett.

### Q: Je dois désactiver IPv6 à chaque fois ?

Seulement quand tu régénères les cookies. Une fois les cookies en place, tu peux réactiver IPv6.

### Q: Pourquoi sortir Prowlarr du VPN ?

Pour que Prowlarr et ton PC aient la **même IP publique**. Sinon, les cookies générés sur ton PC ne fonctionneront pas sur Prowlarr.

### Q: Et si je veux garder Prowlarr derrière le VPN ?

C'est possible mais plus compliqué :
1. Ton PC doit se connecter au **même serveur VPN** que Gluetun
2. Les IPs doivent matcher exactement
3. Tu devras peut-être reconnecter plusieurs fois pour tomber sur la bonne IP

---

## 📄 Fichiers

- `ygghybrid.yml` - L'indexeur hybride pour Prowlarr
- `README.md` - Ce guide

---

## 🙏 Crédits

- [YggAPI](https://yggapi.eu/) - API non-officielle pour la recherche
- **Topiary___** - Reverse-engineering de la protection YGG et création de l'indexeur hybride

---
