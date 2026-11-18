# Exemple n8n en local — « Déclarer une embauche » (Séance 3)

Un mini-scénario de 5 nœuds qui reprend le début du scénario Habitéo du chapitre :
**déclencheur → mapping → routeur → sortie**. Il n'utilise ni compte externe, ni clé API, ni LLM :
tout tourne sur le poste, et aucune donnée ne quitte la machine.

| Brique du chapitre | Nœud n8n dans l'exemple |
|---|---|
| Déclencheur « formulaire rempli » | **1. Formulaire** (Form Trigger) |
| Mapping (brancher les données) | **2. Mapping et calcul** (Edit Fields) |
| Routeur + filtre « arrivée < 5 jours » | **3. Urgent ?** (If) |
| Sortie prévue pour chaque cas | **4a. Alerte RH** / **4b. Dossier standard** (Edit Fields) |

La règle de calcul (jours avant l'arrivée) est volontairement une **règle, pas un LLM** :
c'est le critère du chapitre « règle ou LLM ? » (calcul exact, gratuit, explicable).

---

## 1. Lancer n8n en local

Prérequis : **Node.js 22.16 ou plus** (vérifié avec n8n 2.10.4), ou Docker.

**Option A — Node.js**

```bash
npx n8n
```

**Option B — Docker** (d'après la documentation officielle n8n, non testé ici)

```bash
docker run -it --rm --name n8n -p 5678:5678 \
  -e GENERIC_TIMEZONE=Europe/Paris -e TZ=Europe/Paris \
  -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

Ouvrir ensuite <http://localhost:5678> et créer le compte propriétaire local à la première visite.

> Pour la discussion « souveraineté » : n8n envoie par défaut des statistiques d'usage anonymes.
> Elles se coupent avec la variable d'environnement `N8N_DIAGNOSTICS_ENABLED=false`.

## 2. Importer le workflow

1. **Create workflow** (nouveau workflow vide).
2. Menu **⋯** en haut à droite → **Import from file…** → choisir `habiteo-declaration-embauche.json`.
   (Variante : ouvrir le JSON, tout copier, puis `Ctrl+V` sur le canvas.)

Le fuseau du workflow est déjà réglé sur `Europe/Paris` : « aujourd'hui » est donc bien la date
française, ce qui compte pour le calcul des jours.

## 3. Tester par morceaux (« Run once », comme dans Make)

1. Cliquer sur **Execute workflow**. n8n ouvre le **formulaire de test** dans un nouvel onglet
   (sinon : double-clic sur le nœud 1 → onglet *Test URL*).
2. Remplir et envoyer le formulaire, puis revenir à l'éditeur.
3. Cliquer sur chaque nœud : on voit les données **en entrée et en sortie**. C'est le *journal*
   du chapitre : la première chose à ouvrir quand quelque chose a mal tourné.

### Les trois cas de test du chapitre

| Cas | À saisir | Résultat attendu |
|---|---|---|
| **Normal** | Date d'arrivée dans 10 jours | `statut = Nouveau`, sortie **4b** |
| **Urgent** | Date d'arrivée dans 2 jours | `statut = URGENT`, sortie **4a** |
| **Incomplet** | Laisser un champ vide | Le formulaire refuse l'envoi (champs obligatoires) |

Cas limites à faire constater aux étudiants (la règle est « strictement moins de 5 jours ») :

| Arrivée dans… | Résultat |
|---|---|
| 4 jours | URGENT |
| 5 jours | Nouveau |
| 0 jour (aujourd'hui) | URGENT |
| −1 jour (date passée) | URGENT, avec « dans -1 jour(s) » → **à discuter** : une date passée devrait-elle être une erreur ? |

## 4. Passer en « production »

1. Cliquer sur **Publish** (selon la version, l'interrupteur **Active**).
2. Le formulaire est alors servi en permanence sur :
   **<http://localhost:5678/form/habiteo-embauche>**
   (l'URL de test contient `form-test` ; celle de production, `form`).
3. Chaque envoi laisse une trace dans l'onglet **Executions**.

> Au démarrage de n8n, patienter quelques secondes : l'URL de production répond 404 tant que
> le workflow n'est pas réactivé.

---

## Ce qui a été vérifié

Le workflow a été importé et exécuté sur une instance n8n 2.10.4 isolée (via l'URL de production du formulaire).
Sept envois, tous en succès, avec le routage attendu : +10 j → Nouveau, +3 j et +2 j → URGENT,
0 j et −1 j → URGENT, +4 j → URGENT, +5 j → Nouveau ; accents (« Hélène Lefèvre », « Saint-Étienne ») préservés.
L'interface graphique n'a pas été ouverte : les libellés de menus ci-dessus (Import from file, Publish, Execute workflow)
suivent la documentation et peuvent varier légèrement selon la version.

## Pour aller plus loin (non testé)

- **Branche « incomplet »** : rendre la date facultative, remplacer le nœud If par un **Switch** à trois sorties
  (date vide / urgent / standard).
- **Nœud LLM** après le routeur : *Basic LLM Chain* + *Ollama Chat Model* pour rédiger la checklist avec un
  modèle local (pas de donnée envoyée à un fournisseur), en réutilisant le prompt du chapitre 4
  (`{{poste}}`, `{{agence}}`, `{{date_arrivee}}` sont déjà les noms des champs produits par le nœud 2).
- **Registre des intégrations** : écrire chaque dossier dans un fichier ou une *Data Table* n8n.
- **Validation humaine** : ajouter un nœud d'attente / d'approbation avant tout envoi réel.
