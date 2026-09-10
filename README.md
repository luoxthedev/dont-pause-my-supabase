# 🟢 dont-pause-my-supabase

Petite GitHub Action qui envoie une requête à ton projet Supabase tous les
**5 jours**, pour éviter que les projets du plan **Free** de Supabase soient
automatiquement **mis en pause après 7 jours d'inactivité**.

## Comment ça marche

Le workflow défini dans [`.github/workflows/keep-alive.yml`](.github/workflows/keep-alive.yml) :

1. Se déclenche automatiquement selon une planification cron (`schedule`),
   réglée sur `0 12 */5 * *` → tous les 5 jours à 12h00 UTC.
2. Envoie une requête `curl` vers l'endpoint REST de ton projet Supabase :
   `https://<ton-projet>.supabase.co/rest/v1/`.
3. Utilise les headers `apikey` et `Authorization` avec ta clé publique
   (anon key) pour authentifier la requête.
4. Affiche le code HTTP retourné dans les logs de l'action. Peu importe
   que la réponse soit `200`, `401` ou `404` : le simple fait que Supabase
   ait reçu et traité la requête suffit à considérer le projet comme actif
   et donc à repousser sa mise en pause automatique.
5. Peut aussi être lancé manuellement à tout moment via l'onglet **Actions**
   du repo (bouton "Run workflow"), grâce au déclencheur `workflow_dispatch`.

## ⚠️ Limite importante concernant le cron `*/5`

Le planificateur cron de GitHub Actions ne raisonne pas en "tous les X jours
à partir d'aujourd'hui", mais en jours du mois. `*/5` sur le champ
"jour du mois" déclenche l'action les jours **1, 6, 11, 16, 21, 26, 31**
(division du numéro du jour par 5). Concrètement :

- La plupart des écarts entre deux exécutions font bien 5 jours.
- Mais entre le jour 31 et le jour 1 du mois suivant (ou 26 → 1 sur les
  mois à 28/29/30 jours), l'écart peut être plus court (1 à 5 jours).

Dans tous les cas, l'écart maximum entre deux exécutions **ne dépasse
jamais 5 jours**, ce qui est largement suffisant pour rester sous la
limite des 7 jours d'inactivité de Supabase. Si tu veux un espacement
strictement régulier de 5 jours en 5 jours, il faudrait stocker la date
de dernière exécution quelque part (ex: un fichier commité dans le repo,
ou une variable d'environnement GitHub) et faire un calcul de date dans
le script — possible à ajouter si besoin, mais overkill pour ce cas
d'usage.

À noter aussi que GitHub peut retarder l'exécution des workflows planifiés
de quelques minutes à (rarement) quelques heures en cas de forte charge sur
la plateforme, ce qui n'a aucun impact pratique ici.

## 🔧 Configuration

Il faut renseigner deux secrets dans le repo GitHub :
**Settings → Secrets and variables → Actions → New repository secret**

| Secret               | Valeur                                                                 |
|----------------------|-------------------------------------------------------------------------|
| `SUPABASE_URL`       | URL de ton projet, ex: `https://xxxxxxxxxxxx.supabase.co`              |
| `SUPABASE_ANON_KEY`  | Ta clé publique "anon" (Project Settings → API dans le dashboard Supabase) |

Tu peux utiliser la clé `anon` (publique) : elle suffit pour "réveiller"
le projet via l'API REST, aucune donnée sensible n'est accédée par ce
workflow. Il n'est pas nécessaire (et pas recommandé) d'utiliser la
`service_role` key ici.

## ✅ Vérifier que ça fonctionne

- Onglet **Actions** du repo → sélectionne le workflow **Keep Supabase Alive**
  → clique sur **Run workflow** pour le tester manuellement.
- Regarde les logs de l'étape "Ping the Supabase REST endpoint" : tu dois
  voir le code HTTP retourné par Supabase.
- Dans le dashboard Supabase, la date de dernière activité du projet doit
  se mettre à jour.

## 📁 Structure du repo

```
.
├── .github/
│   └── workflows/
│       └── keep-alive.yml   # Le workflow planifié
└── README.md                 # Ce fichier
```
