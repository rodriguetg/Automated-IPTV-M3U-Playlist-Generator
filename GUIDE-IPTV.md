# Guide de prise en main — Ma playlist IPTV auto-actualisée

> Fork : **[rodriguetg/Automated-IPTV-M3U-Playlist-Generator](https://github.com/rodriguetg/Automated-IPTV-M3U-Playlist-Generator)**
> Dernière mise à jour du guide : 09/08/2026

---

## 1. Ce que fait le système

Chaque jour, GitHub Actions (les serveurs gratuits de GitHub) exécute un script Python qui :

1. **Récupère** 13 listes de chaînes TV (sources M3U) sur internet
2. **Fusionne et déduplique** les chaînes (par hash + similarité de nom)
3. **Teste chaque flux un par un** (60 connexions en parallèle) et jette les morts, les 404, les géo-bloqués
4. **Exporte** le résultat en 3 formats : `.m3u` (pour les lecteurs IPTV), `.json`, `.txt`
5. **Committe** les fichiers dans le dépôt → l'URL de la playlist est donc toujours à jour

```
Sources M3U ──► Fusion/Dédup ──► Validation (60 threads) ──► Export ──► Commit GitHub
   (13)          ~14 000 ch.        ~10 000 actives          m3u/json/txt
```

**Aucun serveur à gérer, aucun coût** : les minutes GitHub Actions sont illimitées et gratuites pour les dépôts publics.

---

## 2. Les URLs à connaître

| Quoi | URL |
|---|---|
| **Playlist à mettre dans ton lecteur IPTV** | `https://raw.githubusercontent.com/rodriguetg/Automated-IPTV-M3U-Playlist-Generator/main/LiveTV/Mikhoul/LiveTV.m3u` |
| Version JSON (pour scripts/n8n) | `.../main/LiveTV/Mikhoul/LiveTV.json` |
| Rapport du dernier run (stats, erreurs) | `.../main/LiveTV/Mikhoul/processing_report.txt` |
| Page Actions (suivre les runs) | `https://github.com/rodriguetg/Automated-IPTV-M3U-Playlist-Generator/actions` |

L'URL de la playlist **ne change jamais** : ton lecteur (VLC, Kodi, TiviMate…) recharge automatiquement la dernière version.

---

## 3. Quand ça tourne

- **Automatiquement** : tous les jours à **19h45 UTC** (21h45 heure de Paris en été, 20h45 en hiver)
- **À chaque push** sur la branche `main`
- **Manuellement** : onglet *Actions* → workflow *« TV-Mikhoul Update Files »* → bouton **« Run workflow »** (Exécuter le flux de travail)

Un run dure environ **20 minutes** avec la configuration actuelle.

### ⚠️ Règle de concurrence (le piège classique)

Le workflow n'accepte qu'**un seul run à la fois** : si un nouveau run démarre pendant qu'un autre tourne, l'un des deux est **annulé** (statut rouge « Annulé »). Ce n'est **pas une erreur** — c'est le comportement voulu.

- Un run annulé pour cette raison → ignore, ou relance avec « Run workflow »
- Ne jamais utiliser « Re-run » (Relancer les tâches) sur un **vieux** run après avoir modifié le code : un re-run rejoue **l'ancien commit**, donc l'ancienne version du script. Toujours préférer **« Run workflow »**, qui part du dernier commit.

---

## 4. Où tout se configure

Tout est dans **un seul fichier** : `BugsfreeMain/TV-Mikhoul.py`, tout en bas, dans la fonction `main()` (autour de la ligne 1760).

### 4.1 Ajouter / retirer des sources

Cherche `source_urls = [` (~ligne 1775). C'est une simple liste d'URLs :

```python
source_urls = [
    "https://github.com/Sphinxroot/QC-TV/raw/.../Quebec.m3u",   # Québec
    "https://iptv-org.github.io/iptv/countries/fr.m3u",          # France
    "https://iptv-org.github.io/iptv/countries/be.m3u",          # Belgique
    "https://iptv-org.github.io/iptv/countries/ch.m3u",          # Suisse
    "https://iptv-org.github.io/iptv/languages/fra.m3u",         # Tout le francophone
    "https://iptv-org.github.io/iptv/index.m3u",                 # MONDE ENTIER (~10 000 ch.)
    # ... etc
]
```

- **Retirer une source** : supprime la ligne, ou commente-la avec `#` en début de ligne
- **Ajouter une source** : ajoute l'URL entre guillemets, suivie d'une virgule
- La playlist trop grosse ? Le plus efficace est de **retirer `index.m3u`** (l'index mondial) — tu retombes à ~1 500-2 000 chaînes francophones/canadiennes

Catalogue de sources fiables (iptv-org) :
- Par pays : `https://iptv-org.github.io/iptv/countries/XX.m3u` (fr, be, ch, ca, us, gb…)
- Par langue : `https://iptv-org.github.io/iptv/languages/fra.m3u` (fra, eng, spa…)
- Par catégorie : `https://iptv-org.github.io/iptv/categories/sports.m3u` (sports, movies, news, music, kids…)

### 4.2 Filtrer des catégories ou des pays

Cherche `excluded_groups = [` (~ligne 1764). Actuellement **vide** (on garde tout). Pour exclure, ajoute des mots-clés — la comparaison est **partielle et insensible à la casse** (le mot-clé matche s'il est contenu dans le nom du groupe) :

```python
excluded_groups = [
    "Religious",      # exclut toutes les chaînes du groupe Religious
    "Shop",           # exclut Shop, Teleshopping…
    "Undefined",      # exclut les ~2 500 chaînes sans catégorie
]
```

### 4.3 Régler la performance

Cherche `config = {` (~ligne 1790) :

```python
config = {
    'max_workers': 60,        # threads de validation en parallèle (15 = doux, 60 = rapide)
    'request_timeout': 12,    # secondes avant d'abandonner un flux
    'max_retries': 1,         # nouvelles tentatives sur flux en erreur
    ...
}
```

Règle simple : plus de sources → augmente `max_workers` ou le `timeout-minutes` du workflow (fichier `.github/workflows/TV-Mikhoul.yml`, actuellement 120 min).

### 4.4 Comment éditer un fichier (sans rien installer)

1. Ouvre le fichier sur GitHub (ex. `BugsfreeMain/TV-Mikhoul.py`)
2. Clique sur le **crayon** ✏️ en haut à droite
3. Modifie, puis **« Commit changes »** (bouton vert)
4. Le push déclenche automatiquement un run → ta playlist se régénère

💡 Pour committer **sans** déclencher de run, termine le message de commit par `[skip ci]`.

---

## 5. Lire le rapport d'un run

Ouvre `LiveTV/Mikhoul/processing_report.txt`. Les lignes qui comptent :

```
Total Channels Found: 14368     ← chaînes trouvées dans les sources
Urls Validated: 12905           ← flux testés
Active Urls: 10065              ← flux VIVANTS gardés dans la playlist
Geo Blocked Urls: 817           ← bloqués hors de leur pays (jetés)
Inactive By Error: {...}        ← détail des morts (404, timeout…)
Total Processing Time: 1214 s   ← durée (~20 min)
```

Suivi de la santé : si `Active Urls` chute brutalement d'un jour à l'autre, une grosse source est probablement tombée — vérifie ses URLs.

---

## 6. Faire tourner en local (optionnel)

Utile pour tester une modification sans attendre un run GitHub :

```bash
git clone https://github.com/rodriguetg/Automated-IPTV-M3U-Playlist-Generator.git
cd Automated-IPTV-M3U-Playlist-Generator
pip install requests beautifulsoup4 pytz
python BugsfreeMain/TV-Mikhoul.py
```

Les fichiers sont générés dans `LiveTV/Mikhoul/`. (Python 3.9+ requis ; sous Windows, installe Python depuis python.org en cochant « Add to PATH ».)

Pour renvoyer tes modifs sur GitHub :

```bash
git add -A
git commit -m "Ma modification"
git push
```

---

## 7. Dépannage

| Symptôme | Cause | Solution |
|---|---|---|
| Run « Annulé » avec message *« requête prioritaire en attente »* | Deux runs simultanés (règle de concurrence) | Normal. Relance via « Run workflow » si besoin |
| Avertissement jaune *« Node.js 20 obsolète »* | Actions `checkout`/`setup-python` un peu anciennes | Sans conséquence, ignorer |
| Le cron ne tourne plus | GitHub suspend les crons d'un fork après **60 jours sans activité** | Un clic « Run workflow » ou n'importe quel commit réactive |
| Run échoue (rouge, pas « annulé ») | Regarder les logs : Actions → run → job `update-files` | Souvent une source qui timeout ; retirer la source fautive |
| Run dépasse 120 min | Trop de sources vs workers | Augmenter `max_workers` (80-100) ou `timeout-minutes` |
| Lecteur IPTV qui rame | Playlist trop grosse (10 000 ch.) | Retirer `index.m3u` des sources, ou exclure `Undefined` |
| Playlist pas à jour dans le lecteur | Cache du lecteur | Forcer le rafraîchissement de la playlist dans l'appli |

---

## 8. Maintenance & bonnes pratiques

- **Récupérer les nouveautés du repo d'origine** (mikhoul) : sur la page de ton fork, bouton **« Sync fork »** → « Update branch ». ⚠️ S'il a modifié `TV-Mikhoul.py`, un conflit est possible avec tes sources — dans le doute, note tes sources quelque part avant de synchroniser (elles sont aussi dans l'historique git de toute façon).
- **Ménage ponctuel** : les branches `claude-push` et `claude-bytecheck` (résidus de la mise en place) peuvent être supprimées dans l'onglet *Branches*.
- **Historique** : chaque run = un commit « TV-Mikhoul Update files ». Pour revenir à une ancienne playlist : onglet *Commits* → parcourir le fichier à cette date.
- **Légalité** : l'outil et les sources iptv-org agrègent des flux publiquement diffusés, mais la légitimité de chaque flux dépend de sa source. Pour un usage perso, privilégie les chaînes publiques et FAST channels officiels (Pluto, Samsung TV Plus, Plex, Rakuten — déjà dans tes sources).

---

## 9. Récap des modifications faites le 09/08/2026

Par rapport au fork d'origine de mikhoul (commit `b102907`) :

1. **+5 sources** : France, Belgique, Suisse, langue française mondiale, index mondial iptv-org
2. **Filtres d'exclusion vidés** (40+ pays/catégories étaient exclus)
3. **`max_workers` : 15 → 60** et `max_retries` : 2 → 1 (validation 4× plus rapide)
4. **Timeout du workflow : 30 → 120 min**

Résultat : **810 → 10 065 chaînes actives**, run en ~20 min.

Pour revenir à la version « petite playlist franco-canadienne » d'origine : re-commente les 5 sources ajoutées et remets l'ancienne liste `excluded_groups` (visible dans l'historique git, commits antérieurs à `b102907`).
