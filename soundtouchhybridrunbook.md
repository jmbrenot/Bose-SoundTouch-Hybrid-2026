# Mettre à jour Bose SoundTouch Hybrid

Manuel de procédure — dépôt GitHub, conteneur Docker sur le NAS, et intégration Home Assistant / Music Assistant. Établi à partir de la mise à jour v4.1 → v4.2 du 7 août 2026.

## Topologie

| Élément | Valeur |
|---|---|
| Fork GitHub | `jmbrenot/Bose-SoundTouch-Hybrid-2026` |
| Dépôt amont | `TJGigs/Bose-SoundTouch-Hybrid` |
| Hôte Docker | PC-LINUX-GRUISSAN (192.168.1.44) |
| Dossier config | `/home/jean-marc/GitHub/Bose-SoundTouch-Hybrid-2026` |
| Hôte Home Assistant | 192.168.1.42 (add-on MASS) |
| Poste de travail | DELL-GRUISSAN (GitHub Desktop) |

## Résumé express

Trois volets indépendants, dans cet ordre :
- **(A)** mettre le fork GitHub à jour sur la nouvelle version amont
- **(B)** mettre à jour le conteneur Docker sur `.44`
- **(C, optionnel)** reconnecter le redémarrage automatique de Music Assistant si HA l'exige

Compter 30–45 min si tout se passe comme la dernière fois ; plus si l'accès GitHub de Claude est toujours cassé (voir §A).

---

## §A — Mettre à jour le dépôt GitHub

Synchroniser le fork `jmbrenot/Bose-SoundTouch-Hybrid-2026` avec la nouvelle release du dépôt amont `TJGigs/Bose-SoundTouch-Hybrid`.

### A1 — Vérifier d'abord si Claude peut pousser directement

La dernière fois, l'app GitHub de Claude était **autorisée** (identité) mais **pas installée** (aucun accès en écriture à un dépôt), ce qui a bloqué tout push direct. Demande à Claude de tenter un push/commit normal en premier — si ça marche, tout le reste de cette section est inutile.

> **Où vérifier côté GitHub** : `github.com/settings/installations` → onglet *Installed GitHub Apps*. Si "Claude" n'y figure pas (même si elle apparaît dans *Authorized GitHub Apps*), l'écriture échouera avec `403 Resource not accessible by integration`.

### A2 — Si ça échoue : méthode de secours (patch + push manuel)

Celle-ci a fonctionné intégralement la dernière fois. Claude prépare le commit et génère un fichier `.patch` ; tu l'appliques et le pousses toi-même depuis un poste où GitHub Desktop/Git est déjà authentifié.

1. Claude prépare la synchronisation et t'envoie un fichier `.patch` (peut faire plusieurs Mo si des fichiers binaires ont changé).
2. Sur ton PC, ouvre le dépôt dans **GitHub Desktop**, puis *Repository → Open in Command Prompt*.
3. Clone une copie fraîche **en dehors de tout dossier synchronisé OneDrive** (voir §D) :

```bash
# change de lecteur ET de dossier en une seule commande
cd /d F:\GitHub
git clone https://github.com/jmbrenot/Bose-SoundTouch-Hybrid-2026 Bose-SoundTouch-Hybrid-2026-vX
cd Bose-SoundTouch-Hybrid-2026-vX
git checkout -b claude/update-vX
```

4. Vérifie le nom exact du fichier patch avant de l'utiliser (il est facile de le taper de travers) :

```bash
dir F:\GitHub\*.patch
```

5. Applique le patch puis pousse :

```bash
git am "F:\GitHub\<nom-exact-du-fichier>.patch"
git push -u origin claude/update-vX
```

6. Merge directement dans `main` (dépôt personnel, pas de review nécessaire) :

```bash
git checkout main
git merge claude/update-vX
git push origin main
```

Si `git am` affiche `Deletion of directory 'xxx' failed`, réponds `n` aux invites — le commit passe quand même ; termine avec `git rebase --continue`. C'est le symptôme du piège OneDrive (§D).

---

## §B — Mettre à jour le conteneur Docker

Sur `PC-LINUX-GRUISSAN` (192.168.1.44), en SSH root, dans `/home/jean-marc/GitHub/Bose-SoundTouch-Hybrid-2026`.

### B1 — Localiser et sauvegarder

```bash
# si le dossier de config n'est pas déjà connu :
docker inspect -f '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' bose-soundtouch-hybrid

cd /home/jean-marc/GitHub/Bose-SoundTouch-Hybrid-2026
cp -r . ../bose-soundtouch-hybrid-backup-$(date +%F)
```

### B2 — Récupérer le nouveau fichier compose

> **Piège — ne pas réutiliser l'ancien `.yml` tel quel.** Le mainteneur republie un `bose-soundtouch-hybrid.yml` à chaque version majeure, parfois avec de nouvelles variables d'environnement (ex : `TRUST_PROXY` apparue en v4.2). Repartir du neuf et reporter uniquement *TZ* et le chemin de volume.

```bash
mv bose-soundtouch-hybrid.yml bose-soundtouch-hybrid.yml.backup-$(date +%F)
curl -O https://raw.githubusercontent.com/TJGigs/Bose-SoundTouch-Hybrid/main/bose-soundtouch-hybrid.yml

# reporter la timezone (le fichier neuf arrive en America/Los_Angeles)
sed -i 's|TZ=America/Los_Angeles.*|TZ=Europe/Paris|' bose-soundtouch-hybrid.yml
grep TZ bose-soundtouch-hybrid.yml
```

Le volume `./:/app/config` (Méthode A dans le fichier) n'a besoin d'aucune modification tant que la commande `docker compose` est lancée depuis ce même dossier.

### B3 — Recréer le conteneur

```bash
docker compose -f bose-soundtouch-hybrid.yml down
docker compose -f bose-soundtouch-hybrid.yml pull
docker compose -f bose-soundtouch-hybrid.yml up -d
docker image prune -f
```

### B4 — Premier boot : migration `.env`

L'app détecte l'ancien schéma, sauvegarde l'existant en `.env.bak`, écrit un nouveau template, puis **s'arrête volontairement** en attendant que tu remplisses le nouveau fichier (`ACTION REQUIRED: Setup Incomplete` dans les logs — c'est normal).

Comme le schéma des champs reste identique d'une version à l'autre, reporter les valeurs par script plutôt qu'à la main — et pour ne pas faire transiter de secret (jeton MASS, etc.) par la conversation avec Claude :

```bash
APP_IP=$(grep '^APP_IP=' .env.bak | cut -d= -f2)
MASS_IP=$(grep '^MASS_IP=' .env.bak | cut -d= -f2)
MASS_TOKEN=$(grep '^MASS_TOKEN=' .env.bak | cut -d= -f2-)

sed -i "s|^APP_IP=.*|APP_IP=$APP_IP|" .env
sed -i "s|^MASS_IP=.*|MASS_IP=$MASS_IP|" .env
sed -i "s|^MASS_TOKEN=.*|MASS_TOKEN=$MASS_TOKEN|" .env

# vérif sans réafficher le secret en clair
echo "APP_IP: $(grep '^APP_IP=' .env)"
echo "MASS_TOKEN rempli: $([ -n "$MASS_TOKEN" ] && echo OUI || echo NON)"
```

```bash
docker compose -f bose-soundtouch-hybrid.yml restart
docker logs -f bose-soundtouch-hybrid
```

Les logs doivent finir par la découverte des enceintes puis `App is up to date (vX.Y)` et l'URL de l'interface web.

| Fichier | Comportement au boot | Action requise |
|---|---|---|
| `.env` | Backup auto → nouveau template vide | Reporter les valeurs (ci-dessus) |
| `settings.json` | Backup auto → régénéré | Aucune — préférences à refaire dans l'UI si besoin |
| `speakers.json` / `library.json` | Conservés si déjà présents | Aucune |

---

## §C — Redémarrage auto de Music Assistant (optionnel)

MASS tourne comme add-on Home Assistant sur 192.168.1.42. Sans cette étape, l'app fonctionne quand même — elle détecte MASS en ligne et continue sans le redémarrer.

### C1 — Trouver le slug de l'add-on

Dans HA : clique *Music Assistant* dans la barre latérale, lis l'URL — tout ce qui suit le port et le slash (ex : `d5369777_music_assistant`).

### C2 — Générer un jeton HA longue durée

Profil HA (compte **administrateur**) → *Sécurité* → *Jetons d'accès de longue durée* → *Créer un jeton*. Copier immédiatement, il ne s'affiche qu'une fois.

### C3 — Renseigner le `.env`

> **Coller le jeton directement dans nano — pas de script sed.** Un script intermédiaire (`sed` injectant le jeton via une variable shell) a provoqué un `401` persistant lors de la mise à jour v4.2, sans qu'on identifie précisément quel caractère posait problème. Éditer `.env` à la main dans `nano` et coller le jeton directement dans le fichier a résolu le souci du premier coup.

```bash
nano /home/jean-marc/GitHub/Bose-SoundTouch-Hybrid-2026/.env
```

Modifie ces deux lignes, colle le jeton directement après le signe égal, sauvegarde (`Ctrl+O`, `Entrée`, `Ctrl+X`) :

```
MASS_CONTAINER_NAME=<slug-de-l-add-on>
HA_TOKEN=<colle-le-jeton-ici>
```

### C4 — Tester avant de redémarrer le conteneur

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" \
  -H "Authorization: Bearer $(grep '^HA_TOKEN=' .env | cut -d= -f2)" \
  http://192.168.1.42:8123/api/
```

`200` → jeton valide, on peut redémarrer le conteneur.

> **Résolu le 7 août 2026.** Le `401` rencontré lors de la mise à jour v4.2 (rejeté par HA 2026.8.0 avec "invalid authentication" dans `homeassistant.components.http.ban`, y compris après redémarrage complet de HA et régénération du jeton) n'était donc pas un bug HA — l'édition directe via `nano` a suffi. Si le `401` revient malgré un collage direct, alors seulement creuser du côté HA (version, droits admin du compte).

---

## §D — Pièges déjà rencontrés

Pour ne pas reperdre le même temps la prochaine fois.

> **OneDrive verrouille des fichiers pendant `git am`.** Un dossier de travail Git synchronisé par OneDrive (ex : `C:\Users\...\OneDrive\Documents\GitHub\...`) fait échouer les suppressions de dossier pendant `git am` (`Deletion of directory 'xxx' failed`). Toujours cloner et travailler dans un dossier **hors OneDrive** (ex : `F:\GitHub\...`).

> **`cd` seul ne change pas de lecteur sous cmd.exe.** `cd F:\GitHub` depuis un prompt ouvert sur `C:\` ne fait rien — l'invite reste sur C:. Utiliser `cd /d F:\GitHub`.

> **Vérifier le remote avant de réutiliser un vieux dossier local.** Un ancien dossier `F:\GitHub\Bose-SoundTouch-Hybrid-2026` pointait en fait vers `TJGigs/Bose-SoundTouch-Hybrid-2026` (pas le fork `jmbrenot`) — le push échouait en `403 Permission denied` pour une tout autre raison que celle attendue. Toujours lancer `git remote -v` avant de retravailler dans un dossier existant ; en cas de doute, cloner un dossier neuf.

> **Ne jamais coller un secret dans le chat.** Un token HA ou MASS collé en clair dans une réponse à Claude finit dans l'historique de conversation. Utiliser un script local (`nano` + `sed`) qui lit/écrit le secret directement sur le serveur, sans jamais l'afficher intégralement.

---

## §E — Checklist finale

- [ ] Tenter un push GitHub direct par Claude (sinon méthode patch, §A2)
- [ ] Fork synchronisé et poussé sur `main`
- [ ] Sauvegarde du dossier de config sur `.44`
- [ ] Nouveau `.yml` téléchargé, TZ et volume vérifiés
- [ ] `down / pull / up -d / prune` exécutés
- [ ] `.env` rempli, conteneur redémarré (logs jusqu'à « App is up to date »)
- [ ] Interface web testée sur `http://192.168.1.44:3000/control.html` (presets physiques + lecture)
- [ ] (Optionnel) Redémarrage auto MASS reconfiguré — sujet à l'état connu ci-dessus

---

*Établi le 7 août 2026 à partir de la mise à jour v4.1 → v4.2. À mettre à jour au fil des prochaines versions.*
