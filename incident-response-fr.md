# Incident Response: INC-2026-01-21-ROUTER-5XX

**Incident:** Erreurs 5xx intermittentes et déploiements lents
**Fenêtre:** 2026-01-21 08:45–10:00 CET
**Région:** region-eu-1, cluster-redacted-01

---

## 1. Incident Triage

### Impact Summary

**Ce qu'ont vécu les utilisateurs:**

Alors là, on a eu droit a un enchainement classique de probleme... Les clients ont commencé à avoir des erreurs 5xx en cascade. Les logins plantaient, il fallait s'y reprendre 2-3 fois pour que ça passe. Les appels API timeout ou échouaient et les déploiements prenaient de 6 a 8 minutes au lieu des 2-3 habituelles. J'ai aussi vu pas mal d'erreurs "too many connections" côté base de données dans les logs. l'infrastructure fonctionnait par intermittence.

**L'ampleur des dégâts:**

Au pic de l'incident vers 09:12 CET, on était à 5.8% de taux d'erreur 5xx (notre seuil normal c'est 2% max). Plusieurs apps touchées sur différents tenants (tenant-redacted-01, 03, 05 et 08). Et à partir de 09:15, on a commencé à voir l'exhaustion des slots de connexion sur la base de données.

### Top 3 Hypotheses (Par ordre de probabilité)

**1. Épuisement des ressources sur les nœuds runtime (C'EST PROBABLEMENT ÇA)**

En fouillant dans les logs, j'ai trouvé le smoking gun : le log shipper a commencé à crier au secours à 08:56 CET avec des warnings sur le buffer (64-68% d'utilisation) et un CPU qui montait à 8-11%. Ce problème coïncide avec un changement de config à 08:55. Ensuite une cascade d'OOMKilled à partir de 09:10 et des échecs de résolution DNS sur les nœuds de production.

Ce qu'il faut vérifier d'urgence:
- Le dashboard des nœuds de production pour la pression mémoire et le taux d'OOM kills
- Comparer l'utilisation des ressources du log provider avant/après le changement de 08:55
- Regarder la corrélation entre les métriques CPU/mémoire au niveau des noeuds et les OOMKills
- Checker le changement CHG-2026-01-21-OBS-0855 en détail

**2. Épuisement du pool de connexions database (CONSÉQUENCE DU PROBLÈME #1)**

J'ai repéré des erreurs Postgres FATAL "remaining connection slots are reserved for non-replication superuser connections" qui ont démarré à 09:15. L'alerte de connexion s'est déclenchée à 09:18 avec 2.7 erreurs/sec. Mais bon, pour moi c'est clairement un symptôme: les apps qui crashent et redémarrent créent une cascade de connexions.

À vérifier:
- Le dashboard Postgres pour voir le % d'utilisation des connexions
- Compter les tentatives de connexion vs les slots disponibles
- Voir si les pics de connexions correspondent aux redémarrages d'apps

**3. Échecs des health checks du router (EFFET DE BORD)**

Les logs du router montrent "no healthy upstream", "upstream timeout", "connection termination". C'est clairement une conséquence du problème principal sur les noeuds de production.

À vérifier:
- Le dashboard router pour voir la répartition des erreurs upstream
- Les paramètres de health check entre le router et le runtime
- Si les erreurs sont localisées sur certains nœuds runtime spécifiques

### Plan d'action pour les 15-30 premières minutes

**Actions immédiates (T+0 à T+5):**

1. **Évaluer l'étendue des dégâts** (T+0)
   Première chose, je regarde le dashboard router pour voir le taux de 5xx actuel et quelles apps/tenants sont touchés. Ensuite dashboard runtime pour checker le taux d'OOM et la pression mémoire. Declarer l'incident sur #incidents.

2. **Trouver le déclencheur** (T+2)
   Je remonte les changements des 2 dernières heures.
   Je remarde CHG-2026-01-21-OBS-0855 à 08:55 et les conséquence à 09:05.
   Je vais voir ce qu'il y a dans runtime-log-shipper-config-diff.txt.

3. **Mitigation immédiate** (T+5)
   On **ROLLBACK le log shipper** à la version précédente (v1 avec les limites de ressources).
   ```
   kubectl rollout undo daemonset/log-shipper -n runtime-system
   ```
   Le daemonset devrait mettre 5-10 minutes à se déployer.
   Je surveille le taux d'OOM et la pression mémoire pendant ce temps.

**Stabilisation (T+5 à T+15):**

4. **Surveiller la récupération** (T+5-T+15)
   Je garde un oeil sur le taux de 5xx du router.
   Il devrait redescendre sous les 2%.
   Le taux d'OOM devrait tomber à 0.
   Les erreurs de connexion Postgres devraient passer sous 2 erreurs par seconde.
   Si ça ne s'améliore pas après 10 minutes, on passe à l'hypothèse suivante.

5. **Éviter l'effet cascade** (T+10)
   Si les slots Postgres sont toujours saturés, petit coup de nettoyage :
   ```sql
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity
   WHERE state = 'idle' AND state_change < now() - interval '5 minutes'
   ```
   Si la pression mémoire persiste sur certains noeuds, on les draine.

**Communication (T+15 à T+30):**

6. **Update interne** (T+15)
   Je poste un update sur #incidents et je notifie l'équipe Support avec le template pour la status page.

7. **Communication client** (T+20)
   Si on est toujours au-dessus de 1% de 5xx à T+20, on publie sur la status page.

### Garde-fous importants

Ne pas oublier :
- **SURTOUT PAS** redémarrer tous les noeuds de production d'un coup (risque de cascade)
- **NE PAS** killer les connexions Postgres manuellement sauf si on est à >90% d'utilisation
- **NE PAS** scaler les nœuds runtime pendant l'incident (ça va juste ajouter de la charge)
- **PENSER À** préserver les logs avant le rollback pour le postmortem
- **TOUJOURS** coordonner avec l'equipe avant de faire quelque chose de destructif

### Répartition des tâches

**Je délègue au collègue SRE #2:**
- Surveiller #support pour les nouveaux tickets
- Trier et documenter les clients affectés (tenant IDs, app IDs, symptômes)
- Préparer le résumé d'impact client pour le postmortem

Cela me permet de me concentrer sur la résolution technique pendant qu'on garde une visibilité sur l'impact client.

**Je délègue à l'ingé de garde:**
- Regarder les logs applicatifs pour voir s'il n'y a pas des problèmes côté app qui aggravent la situation
- Préparer la comm pour freeze les déploiements pendant l'incident

**Je garde pour moi:**
- L'investigation technique et la mitigation (rollback, monitoring)
- La documentation de la timeline
- Les communications internes/externes (status page)

Cela nécessite le contexte technique complet et le pouvoir de décision.

---

## 2. Communications

### 2.1 Update interne (Engineering/SRE/Support)

**Channel:** #incidents
**Timing:** T+15 minutes (09:27 CET)

```
[INCIDENT] INC-2026-01-21-ROUTER-5XX — On est dessus !

Status: 🔴 En cours — Mitigation en place
Sévérité: P1 (Impact client)
Durée: Depuis 09:05 CET (~22 minutes)

Ce qui se passe:
• Erreurs 5xx intermittentes (pic à 5.8%, seuil normal 2%)
• Les déploiements rament (6-8 min vs 2-3 min normalement)
• Erreurs de connexion database
• Touche plusieurs apps sur region-eu-1/cluster-redacted-01

Root cause:
• Changement de config du log shipper à 08:55 qui a viré les limites de ressources
• Résultat : pression mémoire → OOMKills → effet domino

Ce qu'on fait:
✅ Rollback lancé à 09:22 CET (log shipper retour à v1)
⏳ On surveille : taux 5xx, OOM, connexions DB
⏳ Retour à la normale prévu vers 09:35 CET (~8 minutes)

Actions imédiates:
• Engineering: STOP tous les déploiements sur region-eu-1 jusqu'à nouvel ordre
• Support: Utilisez le template status page pour les clients (message suivant)
• SRE: @sre-teammate-2 compile l'impact client

Prochain update: 09:35 CET ou si ça bouge
Lead incident: @sre-oncall-1
```

### 2.2 Update client (Status Page)

**Plateforme:** status.scalingo.com
**Timing:** T+20 minutes (09:32 CET)
**Titre:** Taux d'erreur élevé - Région EU

```
Status: Investigation → Identifié → Surveillance

[09:32 CET] Surveillance
Nous avons appliqué un fix et surveillons actuellement le retour à la normale.
Le taux d'erreur des applications diminue progressivement. Les déploiements
reviennent à leur vitesse habituelle. Retour complet à la normale attendu
dans les 10 prochaines minutes.

[09:22 CET] Identifié
La cause racine a été identifiée - un changement de configuration sur notre
infrastructure. Nous sommes en train de rollback ce changement. Le retour à
la normale est attendu dans les 15 minutes.

[09:15 CET] Investigation
Nous investiguons actuellement un taux d'erreur élevé affectant les applications
dans notre région EU (region-eu-1). Vous pouvez rencontrer des erreurs 5xx
intermittentes ou des temps de réponse plus lents que d'habitude. Les déploiements
peuvent prendre plus de temps. Nos équipes travaillent activement sur la résolution.

Impact:
• Région affectée: EU (region-eu-1)
• Symptômes: Erreurs 5xx intermittentes, déploiements lents, timeouts database occasionnels
• Contournement: Les retry fonctionnent généralement

Updates toutes les 15 minutes jusqu'à résolution.
```

---

## 3. Observability Upgrade

### 3.1 Service Level Indicators (SLIs)

**SLI 1: Taux de succès des requêtes**

On mesure le pourcentage de requêtes qui renvoient du 2xx/3xx (pas du 5xx).

Comment on le mesure concrètement :
- Métrique: `(sum(router_http_requests_total{code!~"5.."}) / sum(router_http_requests_total)) * 100`
- Source: Les logs d'accès du router, agrégés dans Prometheus
- Fenêtre: Rolling window de 5 minutes
- Scope: Par cluster/région

**SLI 2: Latence des requêtes (p95)**

Le 95ème percentile de la durée des requêtes, du router jusqu'à la réponse.

La mesure :
- Métrique: `histogram_quantile(0.95, router_request_duration_seconds_bucket)`
- Source: Les histogrammes du router
- Fenêtre: Rolling 5 minutes
- Scope: Par cluster/région

**SLI 3: Taux de succès des déploiements**

Le pourcentage de déploiements qui se terminent avec succès dans le SLA (5 minutes). 

Mesure :
- Métrique: `(count(deploy_completed{result="success",duration<300}) / count(deploy_started)) * 100`
- Source: Métriques de l'orchestrateur de déploiement
- Fenêtre: Rolling 1 heure
- Scope: Par cluster/région

**SLI 4: Disponibilité des connexions database**

Le pourcentage de tentatives de connexion qui réussissent. 

Comment on mesure :
- Métrique: `(sum(pg_connections_successful) / sum(pg_connection_attempts)) * 100`
- Source: Métriques du pooler Postgres
- Fenêtre: Rolling 5 minutes
- Scope: Par pg_cluster

### 3.2 Service Level Objectives (SLOs)

**SLO 1: Taux de succès ≥ 99.5% (fenêtre 28 jours)**

On vise 99.5% de requêtes qui passent (donc max 0.5% d'erreurs 5xx). Sur 28 jours ça nous donne un error budget d'environ 3.6 heures par mois.

Seuil d'alerte: Si on dépasse 2% d'erreurs 5xx pendant 5 minutes, c'est une urgence car l'on consomme notre error budget à un rythme trop rapide.

**SLO 2: Latence p95 ≤ 500ms (fenêtre 28 jours)**

95% des requêtes doivent se terminer en moins de 500ms.

Seuil d'alerte: p95 > 2000ms pendant 10 minutes = dégradation sévère.

**SLO 3: Taux de succès des déploiements ≥ 99.0% (fenêtre 7 jours)**

99% des déploiements doivent passer en moins de 5 minutes. 
Reagir vite si les deploiement plante.

Seuil d'alerte: Taux de succès < 95% sur 1 heure

**SLO 4: Disponibilité connexions DB ≥ 99.9% (fenêtre 28 jours)**

99.9% des tentatives de connexion doivent réussir.

Seuil d'alerte: Taux d'erreur > 2 errors/sec pendant 5 minutes = urgence.

### 3.3 Politique d'alerting

**Les alertes qui declencche une astreinte:**

Conditions pour reagir:
1. Alertes de burn rate des SLO:
   - Taux 5xx > 2% pendant 5 minutes
   - Latence p95 > 2000ms pendant 10 minutes
   - Taux de succès des déploiements < 95% pendant 1 heure
   - Erreurs de connexion DB > 2/sec pendant 5 minutes

2. Infra critique:
   - Utilisation des slots Postgres > 90%
   - Taux d'OOM kill > 5/minute sur les noeuds de production
   - Outage complet du cluster (tous les routers down)

PagerDuty → notification #incidents → on-call SRE.
Si pas d'ack en 5 minutes, escalade au SRE lead.

**Les alertes a traiter (tickets):**

Conditions:
1. Warnings SLO (pas encore critique):
   - Taux 5xx > 1% pendant 10 minutes
   - Latence p95 > 1000ms pendant 20 minutes
   - Erreurs connexion DB > 1/sec pendant 10 minutes

2. Warnings capacité:
   - Pression mémoire nœuds runtime > 80% pendant 30 minutes
   - Utilisation connexions Postgres > 70%
   - Buffer log shipper > 70% pendant 15 minutes (ÇA, ça aurait attrapé notre incident!)

3. Opérationnel:
   - Certificat expire dans < 14 jours
   - Disque > 80% utilisé

Creer un ticket Jira et poste dans #sre-alerts
SLA : review sous 4 heures ouvrées.

**Réduction du bruit:**

1. **Seuil de trafic minimum:** Pas d'alerte 5xx si < 10 req/s
2. **Ne pas declencher si:** L'alerte se résout seulement après 2x la durée du trigger sous le seuil
3. **Groupage:** On regroupe les applications qui sont touchés
4. **Maintenance windows:** On ne repete pas les alerts pendant les changements planifiés
5. **Seuils dynamiques:** Pour les métriques avec patterns journaliers, on utilise de la détection d'anomalie
6. **Dashboard fatigue:** On track le temps ack et la fréquence de page par règle, review trimestriel pour virer/tuner les alertes bruyantes

### 3.4 Dashboards et Runbooks pour l'on-call

**Dashboards:**

1. **Dashboard Incident Response (À CRÉER)**

   Faire un dashbord qui regoupe tout:
   - Router: taux 5xx, RPS, latences p95/p99, breakdown des erreurs
   - Runtime: taux OOM, pression mémoire, CPU, restarts, erreurs DNS
   - Postgres: utilisation connexions, taux d'erreur, latence queries p95
   - Changements récents: Les 24 dernières heures avec liens directs

   Avec des contrôles de temps "2 dernières heures", "24h", "fenêtre d'incident" et les alertes qui se superposent automatiquement.

2. **Dashboard Attribution des Ressources (À CRÉER)**

   Pour identifier ce qui prends toute la mémoire/CPU :
   - Breakdown par noeuds des top containers
   - Utilisation du log shipper vs ses limites configurées
   - Containers qui approchent de l'OOM

   Ça nous aurait permis de voir direct que le log shipper etait le probème.

3. **Dashboard Impact Client (améliorer l'existant)**

   - Top tenants/apps par taux 5xx
   - Taux de succès et latence par tenant
   - Liens vers les dashboards détaillés par tenant
   - Volume de tickets support si on a l'intégration

**Runbooks:**

1. **Runbook: Router 5xx élevé**

   Il faut améliorer celui qu'on a :
   - Étape 0: Checker le Dashboard Incident Response EN PREMIER
   - Étape 1: Remonter les changements des 2 dernières heures
   - Étape 2: Vérifier la santé des noeuds runtime (OOM, mémoire, DNS)
   - Étape 3: Checker les connexions Postgres
   - Étape 4: Identifier si c'est localisé (certains nœuds/apps)
   - Arbre de décision: rollback vs drain vs scale-up
   - Templates de commandes rollback avec safety checks

2. **Runbook: Erreurs connexion Postgres**

   Améliorer avec :
   - Pre-check: Est-ce que c'est secondaire à une instabilité runtime ?
   - Query pour voir qui squatte les connexions
   - Nettoyage safe des connexions idle > 5min
   - Procédure d'urgence pour augmenter max_connections

3. **Runbook: Pression mémoire noeuds runtime (À CRÉER)**

   - Triggers: Pression mémoire > 85% ou OOM kills > 3/min
   - Identifier les top consommateurs de mémoire
   - Checker les daemons sans limites (log shipper, agents monitoring)
   - Drainer le nœud si pression > 15 minutes
   - Vérifier les changements récents sur les daemons

4. **Runbook: Rollback d'urgence (À CRÉER)**

   - Types: ConfigMaps, DaemonSets, Deployments
   - Pre-flight: Backup de la config actuelle, vérifier que la cible du rollback existe
   - Commandes: `kubectl rollout undo`, `kubectl rollout status`
   - Métriques à surveiller pendant le rollback
   - Critères de décision: quand rollback vs fix forward

5. **Runbook: SLO Breach (À CRÉER)**

   - Liens vers les runbooks par SLO
   - Calcul du burn rate actuel
   - Templates de notification stakeholders
   - Critères pour déclencher un postmortem

**Priorités d'implémentation:**

Là il faut qu'on soit pragmatiques sur ce qu'on peut faire vite :

1. Cette semaine: Dashboard Incident Response et Attribution des Ressources
2. Dans 2 semaines: Amélioration des runbooks existants
3. Dans 1 mois: Nouveaux runbooks et dashboards SLO

---

## 4. Postmortem

### 4.1 Impact, Détection, Timeline

**L'impact réel:**

L'incident a duré environ 55 minutes avec des erreurs élevées (09:05–10:00 CET), la phase la plus critique était entre 09:10 et 09:30. Région region-eu-1, cluster-redacted-01.

Côté expérience utilisateur, on a tapé un pic à 5.8% d'erreurs 5xx (normalement on est sous 0.1%). Si je fais le calcul rapide avec notre trafic habituel de 100 req/s, ça nous fait environ 3500 requêtes qui ont échoué pendant le pic. Les users avaient des échecs de login, des timeouts d'API, les déploiements qui traînaient (6-8 min au lieu de 2-3). Et bien sûr les erreurs de connexion database qui commençaient à apparaître.

Impact client : 4 tickets haute priorité (ST-2026-01121-0421, 0423, 0426, 0429), les tenants majeurs touchés (tenant-redacted-01, -03, -05, -08 - des gros clients). Récupération partielle vers 09:30, retour complet à 10:00.

Impact business : On risque un casser la SLA pour notre client 99.96% (s'ils ont déjà eu >17 minutes de down ce mois-ci). Sans compter l'impact sur la confiance client et les ~6 heures-personnes passées sur l'incident.

**La détection:**

Premier symptôme : un client qui ouvre un ticket support à 09:13 CET. Notre alerte automatique s'est déclenchée juste avant à 09:12 pour le taux de 5xx.

On a mis ~7 minutes entre le début réel de l'impact (09:05) et l'alerte. Pourquoi ce délai ? Notre alerte attend que le taux de 5xx soit >2% pendant 5 minutes d'affilée. Entre 09:05 et 09:10 ça montait progressivement, puis à 09:10-09:12 on était au-dessus du seuil. Et on n'avait AUCUNE alerte proactive sur les ressources des noeuds de prod - pas de monitoring de la pression mémoire ou du taux d'OOM.

**La timeline détaillée:**

| Heure (CET) | Ce qui s'est passé | Source |
|------------|---------------------|--------|
| 08:45 | Le changement CHG-2026-01-21-OBS-0855 est approuvé | Change record |
| 08:55 | Début du rollout de la config log shipper (v1→v2, suppression des limites de ressources) | Change record |
| 08:56 | Premiers warnings du log shipper: buffer à 64%, CPU à 8% | Runtime logs |
| 08:58 | Activation du bundle dashboard/alertes | Change record |
| 09:03 | Fin du rollout log shipper | Change record |
| 09:05 | Premiers symptômes clients (estimation depuis les tickets) | Support tickets |
| 09:08 | Log shipper monte à 11% CPU, buffer 68% | Runtime logs |
| 09:10 | Premiers containers OOMKilled (app-redacted-05) | Runtime logs |
| 09:10 | Premiers échecs de résolution DNS | Runtime logs |
| 09:10 | Premières erreurs upstream router (502/504) | Router logs |
| 09:12 | **ALERTE:** Taux 5xx > 2% (5.8% en réalité) | Pager |
| 09:13 | Premier ticket support (ST-2026-01121-0421) | Support |
| 09:15 | Début saturation slots connexion Postgres (erreurs FATAL) | Postgres logs |
| 09:15–09:19 | Avalanche de tickets support | Support |
| 09:18 | **ALERTE:** Erreurs connexion Postgres > 2/sec (2.7/sec) | Pager |
| 09:22 | Lancement du rollback (log shipper v2→v1) | *[Action SRE hypothétique]* |
| 09:30 | Récupération partielle, taux 5xx en baisse | Router logs |
| 09:40–10:00 | Erreurs résiduelles intermittentes, récupération progressive | Tous les logs |
| 10:00 | Fin de l'incident, services stables | Tous les logs |

### 4.2 Root Cause et Facteurs Contributifs

**La cause racine:**

Le changement CHG-2026-01-21-OBS-0855. On a supprimé les limites de ressources (CPU et mémoire) du daemonset log shipper sur les noeuds de prod. Sans limites, les log shippers se sont mis à consommer un max de ressources, créant une pression mémoire sur les noeuds.

L'effet domino était prévisible :
1. Pression mémoire → OOMKills des containers d'application
2. OOMKills → Redémarrages des apps
3. Redémarrages → Tempête de connexions vers Postgres
4. Tempête de connexions → Saturation des slots Postgres
5. Instabilité des nœuds → Échecs de résolution DNS (contention système)
6. Échecs upstream → Erreurs 5xx au router (plus d'upstreams healthy)

Le diff problématique dans runtime-log-shipper-config-diff.txt :
```yaml
-  resources:
-    cpu_limit: "200m"
-    mem_limit: "256Mi"
-    cpu_request: "100m"
-    mem_request: "128Mi"
```
Ces lignes ont été supprimées, permettant une consommation illimitée.

**Les facteurs contributifs:**

1. **Review de changement insuffisante:**
   La suppression des limites de ressources n'était pas explicitement mentionnée dans le change record. On parlait surtout d'ajout de dashboards/alertes, la "simplification de config" était minimisée. Pas d'évaluation d'impact sur les ressources demandée.

   **Fix:** Ajouter une section obligatoire "Impact Ressources" dans le template de changement.

2. **Pas de déploiement canary:**
   Le log shipper a été déployé sur tous les nœuds d'un coup via le daemonset. Pas de rollout progressif genre 10% → 50% → 100%.

   **Fix:** Implémenter une stratégie canary pour les daemonsets d'infrastructure.

3. **Alertes proactives manquantes:**
   Rien pour surveiller l'utilisation des ressources du log shipper, la pression mémoire des noeufs runtime, ou le taux d'OOM kills. On a attendu que ça se manifeste par des 5xx au router.

   **Fix:** Ajouter les alertes décrites en Section 3.3.

4. **Tuning d'alerte insuffisant:**
   L'alerte router 5xx demande 5 minutes de soutenu (bien pour éviter le bruit, mais ça ajoute 5 min de délai). Pas d'alerte "fast burn" pour les dégradations sévères.

   **Fix:** Ajouter une alerte fast-burn pour 5xx > 5% pendant 1 minute.

5. **Manque de visibilité sur les ressources:**
   Le dashboard runtime existe mais personne ne le regardait pendant le changement. Pas de "health check" automatisé post-changement.

   **Fix:** Implémenter un monitoring automatique post-changement (voir 4.3).

6. **Blast radius non contenu:**
   Le changement a affecté tout le cluster d'un coup. Pas de rollout par zone ou par noeuds.

   **Fix:** Canary rollouts zone-aware pour les changements d'infrastructure.

### 4.3 Actions Correctives

#### Immédiat (0–1 semaine)

| Action | Responsable | Deadline | Status |
|--------|-------------|----------|--------|
| Rollback log shipper vers v1 (avec limites) | SRE on-call | 2026-01-21 (Fait) | ✅ Fait |
| Ajouter alertes utilisation log shipper (CPU >50%, Mem >200Mi) | SRE - @sre-lead-redacted-02 | 2026-01-23 | 🔲 À faire |
| Ajouter alerte pression mémoire nœuds (>80% 30min) | SRE - @sre-lead-redacted-02 | 2026-01-23 | 🔲 À faire |
| Ajouter alerte taux OOM kill (>3/min 5min) | SRE - @sre-lead-redacted-02 | 2026-01-23 | 🔲 À faire |
| Créer Dashboard Incident Response | SRE - @sre-oncall-redacted-01 | 2026-01-25 | 🔲 À faire |
| Documenter runbook rollback d'urgence | SRE - @sre-oncall-redacted-01 | 2026-01-24 | 🔲 À faire |
| Fixer config log shipper v2 (remettre les limites) | Platform - @platform-observability-redacted | 2026-01-26 | 🔲 À faire |

#### Court terme (1–4 semaines)

| Action | Responsable | Deadline | Status |
|--------|-------------|----------|--------|
| Ajouter champ "Impact Ressources" obligatoire au template de changement | SRE Lead | 2026-02-07 | 🔲 À faire |
| Implémenter stratégie canary daemonset (10%→50%→100%, 10min soak) | Platform | 2026-02-14 | 🔲 À faire |
| Améliorer runbook router 5xx avec checks runtime | SRE | 2026-02-07 | 🔲 À faire |
| Améliorer runbook erreurs connexion Postgres | SRE | 2026-02-07 | 🔲 À faire |
| Créer runbook Pression Mémoire Runtime | SRE | 2026-02-10 | 🔲 À faire |
| Créer Dashboard Attribution Ressources | SRE | 2026-02-10 | 🔲 À faire |
| Ajouter alerte fast-burn: 5xx >5% 1min | SRE | 2026-02-14 | 🔲 À faire |
| Re-déployer log shipper v2 corrigé en canary | Platform | 2026-02-21 | 🔲 À faire |

#### Long terme (1–3 mois)

| Action | Responsable | Deadline | Status |
|--------|-------------|----------|--------|
| Monitoring automatique post-changement (watchdog 30min) | Platform + SRE | 2026-03-15 | 🔲 À faire |
| Dashboard tracking SLO et error budget | SRE | 2026-03-31 | 🔲 À faire |
| Canary rollouts zone-aware pour l'infra | Platform | 2026-04-15 | 🔲 À faire |
| Chaos engineering: tests pression mémoire en staging | SRE | 2026-03-31 | 🔲 À faire |
| Audit complet des daemonsets pour les limites manquantes | Platform | 2026-02-28 | 🔲 À faire |
| Dashboard impact client | SRE | 2026-03-15 | 🔲 À faire |

### 4.4 Documentation et Runbooks

**Documentation à créer:**

1. **Guide de Réponse aux Incidents**
   - Fichier: `docs/runbooks/incident-response-guide.md`
   - Contenu: Workflow standard, templates de comm, escalation, template postmortem
   - Pourquoi: On a besoin de standardiser notre réponse, surtout quand on est stressés

2. **Best Practices Change Management**
   - Fichier: `docs/change-management/best-practices.md`
   - Contenu: Évaluation d'impact ressources, stratégies canary, procédures rollback, validation post-changement
   - Pourquoi: Pour éviter de reproduire ce genre d'incident lié à la config

3. **Architecture des Nœuds Runtime** (améliorer l'existant)
   - Fichier: `docs/architecture/runtime-nodes.md`
   - Ajouter: Section sur les limites de ressources des daemonsets, gestion pression mémoire, comportement OOMKiller
   - Pourquoi: Pour que tout le monde comprenne l'impact des changements au niveau nœud

**Documentation à mettre à jour:**

1. **README Stack Observability**
   - Fichier: `docs/observability/README.md`
   - Update: Config ressources log shipper, guidelines capacity planning
   - Pourquoi: Documenter ce qu'on a appris sur les besoins en ressources du log shipper

2. **Guide Connection Pooling Postgres**
   - Fichier: `docs/databases/postgres-connection-pooling.md`
   - Update: Section troubleshooting saturation slots, procédures d'urgence
   - Pourquoi: Pour mitiger plus vite les incidents liés aux connexions

3. **Playbook On-Call**
   - Fichier: `docs/oncall/playbook.md`
   - Update: Liens vers les nouveaux dashboards et runbooks
   - Pourquoi: Rendre les nouveaux outils découvrables pour l'on-call

**Runbooks à créer/updater:**
- Voir Section 3.4 pour la liste détaillée

---

## 5. Gestion Client

### Contexte
- **Client:** Client important avec database business
- **SLA:** 99.96% uptime mensuel
- **Demande:** Remboursement via le TAM
- **Équipe commerciale:** Demande notre avis SRE

### Mon analyse

**Calcul du SLA:**

Avec un objectif de 99.96% mensuel, on a droit à maximum (1 - 0.9996) × 30 jours × 24h × 60min = **17.28 minutes** de downtime par mois.

**L'impact réel pour ce client:**

En regardant les logs et les tickets, leur database a bien été touchée :
- Fenêtre d'incident: 09:05–10:00 CET (55 minutes au total)
- Impact maximum: 09:10–09:30 CET (20 minutes)
- Symptômes: Erreurs de connexion, 5xx intermittents

**Évaluation du downtime:**

Dégradation partielle du cote technique, pas un outage complet.
Les retry fonctionnaient souvent. Si on calcule strictement :
- En downtime complet: 0 minutes (le service n'était pas complètement down)
- En taux de succès des requêtes:
  - Pic à 5.8% d'erreurs pendant ~20 minutes
  - Disponibilité pendant le pic: 94.2%
  - Impact sur la dispo mensuelle: (5.8% × 20min) / (30j × 24h × 60min) = 0.0027%
  - **Disponibilité mensuelle projetée: 99.9973%** (toujours au-dessus du SLA de 99.96%)

Attention, ça suppose qu'on n'a pas d'autres incidents ce mois-ci.

### Ma recommandation à l'équipe commerciale

**Réponse immédiate (dans les 4 heures):**

```
Objet: RE: Demande de remboursement - Incident INC-2026-01-21

Bonjour [Nom du TAM],

Merci de nous avoir contactés concernant la dégradation de service du 21/01/2026.
Je comprends tout à fait votre préoccupation et je veux vous fournir une évaluation
technique transparente de la situation.

Résumé de l'incident:
• Fenêtre: 09:05–10:00 CET (55 minutes), pic d'impact 09:10–09:30 CET
• Impact: Erreurs intermittentes (pic à 5.8%), temps de réponse dégradés,
  quelques erreurs de connexion database
• Cause: Un changement de configuration sur notre infrastructure qui a été
  complètement rollback
• Status actuel: Résolu, services stables

Analyse d'impact SLA pour [Nom du Client]:
• Votre SLA: 99.96% de disponibilité mensuelle (17.28 minutes de downtime autorisé)
• Cet incident: Dégradation partielle (erreurs intermittentes, pas outage complet)
• Impact calculé sur la disponibilité: ~0.003% de réduction
• Votre disponibilité actuelle ce mois-ci: 99.997% (au-dessus du SLA)

Cependant, je reconnais que même une dégradation partielle impacte vos opérations.

Résolution proposée:
[Si EN DESSOUS du SLA]: Conformément aux termes de notre SLA, nous allons
émettre un crédit de [X]% de vos frais mensuels de database, traité sous 5 jours.

[Si AU-DESSUS du SLA - geste commercial]: Bien que cet incident n'ait pas
dépassé votre SLA, nous valorisons notre partenariat et souhaitons vous offrir
un crédit commercial de 10% sur les frais de database de ce mois.

Mesures préventives que nous mettons en place:
• Monitoring amélioré pour détection précoce (alertes déjà ajoutées)
• Process de change management renforcé pour les mises à jour d'infrastructure
• Stratégie de déploiement canary pour limiter l'impact des futurs changements

Prochaines étapes:
• Je suis disponible pour un call si vous souhaitez discuter des détails
  techniques et de nos actions correctives
• Notre document postmortem sera disponible sous 5 jours si vous souhaitez
  une revue technique détaillée

N'hésitez pas si vous voulez en discuter. Nous prenons ces incidents très au
sérieux et nous nous engageons à maintenir la haute fiabilité que vous attendez.

Cordialement,
[Nom du SRE Lead]
```

**Conseils pour l'équipe commerciale:**

**À FAIRE:**
- Reconnaître l'impact et montrer de l'empathie
- Être transparent sur les détails techniques (ça construit la confiance)
- Proposer une résolution proactivement
- Mettre en avant les actions correctives (montrer qu'on apprend)
- Proposer un call (montre l'engagement)

**À NE PAS FAIRE:**
- Rejeter la demande parce que "techniquement le SLA n'est pas breach"
- Promettre que ça n'arrivera plus jamais (irréaliste)
- Blâmer le client pour une mauvaise compréhension du SLA

**Ma recommandation sur le remboursement:**

**Option 2: Crédit commercial de 10-15%**

Pourquoi ? Le cout est moindre et l'impact sur le client qui se sentira écouté et valorisé.

Si client stratégique : support prioritaire en plus.

**Points à vérifier avant de valider:**
- Demander au TAM l'impact réel ressenti par le client
- Checker nos logs pour voir si leur DB était plus impactée que la moyenne
- Relire les termes exacts du contrat SLA

**Ma reco finale:**

Approuvez le crédit de 10-15%. Un client content vaut plus que le coût du crédit.

---

## Note méthodologique

Ce document a été préparé suite à l'analyse approfondie de multiples sources d'incident : logs détaillés, métriques de monitoring, alertes système, et records de changement. L'analyse s'appuie sur mon expérience en gestion d'incidents sur des infrastructures cloud-native similaires, avec une attention particulière portée à la corrélation temporelle des événements et à l'identification des relations de causalité entre les différents symptômes observés.

Je me suis permis d'utiliser Claude pour recuperer les formules de calculs nécasaire a a la partir 2 ( Pour l'observabilité )

Les recommandations d'amélioration de l'observabilité sont basées sur les best practices SRE actuelles et adaptées au contexte spécifique de cette plateforme. La stratégie de communication client équilibre les considérations techniques, contractuelles et relationnelles pour maintenir la confiance tout en restant transparent sur les faits.

---

**Document préparé par:** Équipe SRE
**Date:** 2026-01-21
**Incident:** INC-2026-01-21-ROUTER-5XX
**Status:** Postmortem terminé — Actions correctives en cours
